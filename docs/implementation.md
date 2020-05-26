## Table of contents:
   - [Features](index.md): List the features of the loggers library
   - [Implementation](implementation.md): This document

## Implementation details

You do not need to read or understand this document to use the loggers.
Its aim is to highlight some of the implementation choices that were made
in the design of this library, and how they impact the performance.

### Asynchronous logging

Arguably the most important feature of this library is its support for
asynchronous logging, where user tasks delegate as much work as possible to
one or more background tasks in charge of doing the actual formatting of
messages and writing them to their final destination.

The workflow is as follows:

 1. user builds a message via one or more calls to the `"&"` operators;
 1. user calls `Log` to emit the message
 1. simple filtering based on severity threshold is applied, and message is
   possibly discarded immediately;
 1. if doing synchronous logging, skip the following two steps
    1. all needed information is saved and queued for the background
      task;
    1. background task polls the queue, decodes the message and forwards it
      to one or more streams, depending on the logger the message was sent to;
 1. each stream applies its filter to decide whether to discard the message;
 1. each stream formats the message, applying the decorators specified by the
   user in the configuration, converting the message components to string,...
 1. each stream writes the resulting string to its final destination.

#### building a message (capturing components)

A `Msg_Type` is essentially an array of message components. This array has
a limited size (after various experimentations, we ended up with a maximum
of 15 components). This limitation allows us to avoid any dynamic memory
allocation when logging messages, which speeds things up and eases the
management of memory (no question as to when we should free a message).

The message itself is built via one or more calls to `"&"`, as in:

```ada

   function "&" (M : Msg_Type; Component : Integer) return Msg_Type;
   function "&" (M : Msg_Type; Component : String) return Msg_Type;
   --  one overload per possible type of component

   Log (Me.Debug & "string" & Comp2 & Comp3 & Comp4);
```

where the first component must always be a string (as a way to limit the number
of overloads of `"&"`).

If a user adds more than 15 components in a single call of `Log`, the extra
components are replaced with a single "..." string. Because of the way we
dynamically build a message, the compiler cannot statically detect that we
provided too many messages.

At this stage, components are **captured**, i.e. we simply keep an unsafe
pointer to the object. That object will exist at least as long as `Log` is
executing.

Because the actual parameters to `"&"` might be passed by copy or by reference,
the loggers provide two generic procedures. Their implementation is made
simple by using GNAT's `Deref` attribute, though it is possible to use
a standard `Address` attribute and carefully copying bits.

For instance:

```ada
   generic
      type T (<>) is limited private;
   package Captures_By_Ref is
      procedure Capture (Data : T; Into : out Captured_Parameter_Type)
         with Inline;
      function Get (Comp : Captured_Parameter_Type) return access constant T
         is (T_Access'Deref (Comp'Address));
   end Captures_By_Ref;

   package body Captures_By_Ref is
      By_Reference : constant Boolean := Capture'Mechanism_Code (1) = 2;
      pragma Assert (By_Reference, "must be by reference");

      procedure Capture (Data : T; Into : out Captured_Parameter_Type) is
      begin
         T_Access'Deref (Into'Address) := Data'Unrestricted_Access;
      end Capture;
   end Captures_By_Ref;
```

As a simple optimization, if the severity of the message
(debug, info, error,...) is lower than the logger's threshold, we do not even
do the capturing (though the compiler still has to call each of the `"&"`
operators.

The actual call to Log and the first level of filtering are simple and require
no special detailing.

####  Queuing messages

When doing asynchronous logging, we need to save everything that the background
task will need later on to write the message to a stream. We can no longer
rely on the unsafe pointers created during the capture phase described
above, because the data might be freed, popped from the stack,...

We now **encode** the message into a binary format. For each component, we
capture the bytes. For instance, an integer requires 4 bytes, a string requires
8 bytes for its bounds and then one byte per character,... All the bytes for
these components are written into a newly allocate memory record, one after
the other. Apart from the memory allocation itself, this is a fast operation,
only involving some copying of bytes in the memory.

A pointer is more complex, of course. Currently we simply encode the 8 bytes
for its address, but it will never be dereferenced afterward (we log the
pointer address itself, not the pointed object).

That binary object is added to a very efficient lock-free queue. We have one
such queue per asynchronous stream.

#### Dequeuing messages

The background queue (a different task) will later retrieve that binary
object, decode it back to its components (4 bytes back to the integer,...)
and pass the decoded result to each stream. This decoding in general requires no
copy because we simply use the bytes where they are in memory.

The decoded message is not exactly the equivalent of the original `Msg_Type`
(Ada does type checking after all !), so the streams have two different
primitive operations to handle both kinds of messages.

### Task safety

This library is fully task safe. Because we care about efficiency, however,
we try to avoid locking when possible.

A call to `Log` (in the user task) requires no locking at all when using
asynchronous logging, because the queue is a lock-free data structure. In
addition to some speed improvement, this also means it is safe to call `Log`
from a protected object (the Ada standard declares that calling a potentially
blocking operation from a protected object is a bounded error).

Each asynchronous stream (i.e. where the `type` is defined as `async` in the
config file) has its own queue and background task. In general, applications
will have a single such asynchronous stream.

Each stream can be called from multiple tasks, either directly by the user
task when using synchronous logging, or by one or more of the background tasks
associated with the synchronous logging. Locking is used here, trying to keep
the locked section to the smallest possibly size to improve throughput.

The code is also split into multiple packages, so that if you decide to
build your application without tasking support, using the loggers doesn't
initialize GNAT's tasking runtime at all. For this, we use low-level primitives
for locks (as opposed to protected types) in a number of places, and the code
that actually requires tasks (e.g. asynchronous logging) is carefully isolated
in optional packages that your application can chose not to import.

### Inlining

A lot of subprograms are marked as inline, and kept small so that the compiler
indeed obeys that request. If we look at the code for `Log`, for instance,
we have:

```ada
   procedure Log_Out_Of_Line
      (Message  : Msg_Type;
       Location : String;
       Entity   : String) with No_Inline;

   procedure Log_Out_Of_Line
      (Message  : Msg_Type;
       Location : String;
       Entity   : String) is
   begin
      ... do the actual logging
   end Log_Out_Of_Line;

   procedure Log
      (Message  : Msg_Type;
       Location : String := GNAT.Source_Info.Source_Location;
       Entity   : String := GNAT.Source_Info.Enclosing_Entity)
   is
   begin
      if Optimization.Likely (Message.Logger = null) then
         return;
      end if;
      Log_Out_Of_Line (Message, Location, Entity);
   end Log;
```

The call to `Optimization.Likely` uses gcc builtins to tell the compiler that
the most likely execution path is that the message will in fact be discarded.
That let's the branch prediction of the processor take full advantage for
further optimization. The `Log_Out_Of_Line` subprogram should not be inlined,
though, as this actually results in slower code in this case.

### Task termination on exception

We want to log a message when a user task terminates because of an
unhandled exception. If we do not, it is sometimes hard to understand why
and when a part of the application has suddenly stopped working.

We could of course require that all user tasks have proper exception handlers
in place, and do their own logging. But it is easy to forget to add them...

So we make use of GNAT's `Ada.Task_Termination` hooks for that purpose.
The code outline looks like:

```ada

with Ada.Exceptions;
with Ada.Task_Termination; use Ada.Task_Termination;
package body Loggers is

   protected Callbacks is
      procedure On_Termination (
        Cause : Ada.Task_Termination.Cause_Of_Termination;
        T     : Ada.Task_Identification.Task_Id;
        X     : Ada.Exceptions.Exception_Occurrence
      );
      procedure Set_Handler;
   private
      Prev : Ada.Task_Termination.Termination_Handler;
   end Callbacks;

   Termination_Handler : constant Ada.Task_Termination.Termination_Handler
      := Callbacks.On_Termination'Access;

   protected body Callbacks is

      procedure On_Termination (
        Cause : Ada.Task_Termination.Cause_Of_Termination;
        T     : Ada.Task_Identification.Task_Id;
        X     : Ada.Exceptions.Exception_Occurrence) is
      begin
         case Cause is
            when Normal => null;
            when Abnormal | Unhandled_Exception =>
               --  do some logging
         end case;
      end On_Termination;

      procedure Set_Handler is
      begin
         Prev := Current_Task_Fallback_Handler;
         Set_Dependents_Fallback_Handler (Termination_Handler);
      end Set_Handler;
   end Callbacks;

begin
   Callbacks.Set_Handler;
end Loggers;
```


### Application finalization

Handling application finalization in the context of asynchronous logging was
one of the most complicated part of the design.

In Ada, the main subprogram will not terminate while any of the task it has
created is still running. None of the global controlled objects will be
`Finalize`d either, because the application is in effect still running.

So potentially that means we need to stop the background tasks associated with
the asynchronous loggers. The usual way to do this is to check whether the
environment task is callable, as in:

```ada
task body Background_Task is
begin
   while Ada.Task_Identification.Is_Callable
      (Ada.Task_Identification.Environment_Task)
   loop
      --  write messages
   end loop;
end Background_Task;
```

The problem is that the user's own tasks might decide to still do some work
at this stage (possibly for a very long time !), and in particular log
messages that we do not want to discard. But the background tasks are not
running any more, so the queue of messages will just keep growing forever...

We tried various approaches, including having `Log` do locking and actual
writing to file when the background task has terminated, but this change of
semantics (and performance profile) is very often not acceptable.

In the end, we settled for using a low-level subprogram provided by the
GNAT runtime:

```ada
task body Background_Task is
   Ignore : constant Boolean := System.Tasking.Utilities.Make_Independent;
begin
   loop
      select
         accept Terminate_After_Dequeue;
         exit;
      or
         delay 0.1;
         --  write messages
      end select;
   end loop;
end Background_Task;
```

The background task is owned by one of the asynchronous streams, via a
controlled type.

When the application is terminated, our background task is no longer blocking
the finalization. So the application will wait for all user tasks to properly
terminate (at which point they are no longer doing any logging). Then as
usual it will `Finalize` all remaining controlled objects, including the one
associated with the stream. That finalization will then rendez-vous with
the `Terminate_After_Dequeue` entry, which will end the background task,
and finally let the application exit cleanly.

### Generics vs tagged types

This library uses a lot of generics to share code. Those are preferred
over tagged types because the compiler can statically optimize the code,
do more inlining,... Those generics are used for the performance critical
parts of the code, in particular the implementation of the `"&"` operators
that are used to build log messages in the user tasks.

However, the streams are implemented as tagged types, so that we keep the
flexibility of configuring them via config files. They are called often, but
from the background task, so they have no impact on the performance of the
user tasks. In fact, streams are controlled types because on finalization
they need to properly close the associated file, socket,...
