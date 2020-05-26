## Table of contents:
   - [Features](index.md): this document
   - [Implementation](implementation.md): implementation details

## Features

This post describes our implementation of a library used to provide
high-performance logging for our Ada applications. It provides a large
number of features that are detailed below.

We compare this framework with `GNATCOLL.Traces`, provided by AdaCore. The
new framework borrows a number of ideas from GNATCOLL, improving on them in
a number of cases.

### Configurable

At any time (though it is generally done on application startup), it is
possible to alter the configuration of the loggers (which ones are active,
where the messages are printed,...).

Although everything can be done via direct subprogram calls in Ada, the
simplest is in general to parse configuration files. A default parser is
provided that supports JSON, via `GNATCOLL.JSON`. One of the drawbacks of
JSON is the lack of comments, so we added some extensions to it to support
both comments and the possibility to omit quoting simple strings. Both of
these are borrowed from YAML.

In practice, it is possible to implement your own custom format for the
configuration files, since everything is doable via an Ada API. As such,
parsing a configuration file is basically a matter of calling the proper Ada
subprograms.

*The configuration file in GNATCOLL.Traces uses a custom format. There is no
separate configuration for streams, so when multiple loggers are sent to the
same file, the stream's configuration must be duplicated.*

### Semantic messages

Although they ultimately will likely be printed as strings in a file, the
messages are internally made of one or more components (a string, an integer,
an instance of a user-defined type,...). Keeping those typed components till
the last minute allows us to filter out messages, to possibly output the
message to a database rather than a string, or to format them as JSON instead
of strings (though this is not implemented currently).

They also provide improved efficiency, since the loggers are careful to avoid
formatting the string (which might be a slow operation) if not needed.

The Ada code would look like:

```ada
with Loggers;  use Loggers;
package Pkg is
   Me : constant Logger_Type := Create;  --  using a default logger name

   procedure P (A : Integer) is
   begin
      Log (Me.Debug & "Some text" & A);
   end P;
end Pkg;
```

The library provides built-in formatting for the standard types like Integer,
Float, String,... but as will be discussed later you can add support for your
own types and your own formatting functions as needed. For floats, we use the
same algorithm as postgreSQL, resulting in the more readable "12.34" rather
than "1.234E+01" for instance.

*`GNATCOLL.Traces` doesn't have this flexibility, since it only strings for
messages (thus the user typically has to call `Integer'Image` to output an
integer).*

### Multi-loggers

Users can define multiple loggers. Each of these logger is in charge of
deciding whether the message will actually be sent or printed anywhere, and
then to forward it to **one or more streams**.

This setup makes it possible to combine external libraries, each with their own
loggers, and decide which part of your application or these libraries should
be logged.

Loggers are identified by a name, which can be referenced from configuration
files to alter the logger's configuration.

Each message is sent to one logger. The message is emitted with a user-defined
level, like `debug`, `info`, `warning` or `error` (though others can be added
as needed in your application). This level is compared with a logger-specific
**threshold**, and if the level is greater than the threshold it is forwarded to
the streams associated with the logger.

The loggers can be configured via the configuration file (or, again, directly
in Ada though this is less usual). In the following example, we setup the root
logger (which provides the default for all other loggers) to only print
messages at level INFO or above. But messages emitted by the MY_LIB logger
should all be printed.

```yaml
{
   loggers: {
      '': {
         threshold: INFO,
      },
      'MY_LIB': {
         threshold: DEBUG,
      }
   }
}
```

*Compared to `GNATCOLL.Traces`, our loggers are more flexible since the initial
decision on whether a message should be visible or not uses multiple levels,
whereas GNATCOLL uses a simple boolean. Those threshold are fairly common in
other libraries like Python's `logging` module.*

### Multi streams

Messages can be sent to one or more streams. Each of these is responsible for
printing the message somewhere, be it in a file, on the console, on a socket,
in syslog for Unix-based systems, in a zip file,...

The predefined streams include files and zip files, each of which can be
further configured. For instance, the name of the file can be computed
automatically to include the **current date and time**, a **unique id**, or the
value of any environment variable. It is possible to control the amount of
buffering that occurs, as a balance between efficiency and reliability.

More importantly, it is possible to automatically **rotate files**. For instance,
at the end of the day, the current day's file is closed and a new file is
created. This helps keep the size of log files smaller for long running
applications.

The library can automatically create missing directories if needed.

Here is an example of a config file that declares multiple streams, and
associates several loggers with them (including changing for the root logger,
so that all messages by default are sent to that file).

```yaml
{
    streams: {
       "file1": {
          type: file,               # will write messages to a file
          filename: "log_$D$N.log", # filename includes data and unique number
          rotate_size="1000000",    # create new file every 1MB
          compress_on_close: false  # we could compress when the file is closed
       },
       "file2": {
          type: zip,                 # a compressed file
          filename: "zip_logs/log_$$.log.gz" # include process id in filename
       }
    },

    loggers: {
       '': {
           ...as before...
           stream: file1
       },
       'MY_LIB': {
           ...as before...
           stream: file2
       }
    }
}
```

If you want to send a message to multiple streams, you first need to combine
those streams by declaring another one. For instance:

```yaml
{
    streams: {
       'combined': {   # the name can be anything
          type: dispatcher,   # this stream dispatches msgs to multiple streams
          dispatch: [
              'file1',     # the one we defined earlier
              'file2',     # also defined earlier
              {            # anonymous stream defined here
                  type: file,
                  filename: 'thirdlog.txt'
              }
          ]
       }
    },

    loggers: {
       OTHER_LIB: { stream: combined }  # write to multiple files
    }
}
```

*Compared to `GNATCOLL.Traces`, our loggers are more flexible, since a message
can end up on multiple streams. We also added support for zip files, and
rotating files. The files are only created on the disk when some message is
actually sent to them, whereas GNATCOLL creates them as it parses the config
file.*

### Filtering of messages

When it receives a message, a stream can optionally run a filter to decide
whether this message should be printed.

Those filters provide greater control than the levels and thresholds described
above. For instance, it is possible that a stream only wants to show `error`
messages or above, or only messages coming from a given third party library,
or even only if the messages includes a components of a given type.

The following configuration adds some filtering to one of the streams we
defined earlier:

```yaml
{
    streams: {
       file2: {
           ...as before...
           filter: "level >= ERROR and logger /= OTHER_LIB"
       }
    }
}
```

*`GNATCOLL.Traces` has no such notion of filtering. The filtering must be done
in user code, and thus cannot be controlled via a configuration file for
instance.*

### Synchronous and asynchronous logging

Depending on your requirements, you might want the message to be written
synchronously to the stream (i.e. as soon as you call `Log`, so that even if
your application crashes, the message is guaranteed to have been printed). This
has however a significant cost in terms of performance, because writing to a
file is always a slow operation compared to writing to memory.

Synchronous streams are also required if your application doesn't allow
multiple tasks.

Our loggers therefore provide asynchronous logging as an option. In this case,
the message is copied into a queue, and will be processed later by a
background task in charge of doing the actual write to files. The user task
(or the main application task) only has to copy the message components and
queue them. The formatting of the message into a string (potentially slow) is
done in the background task as well.

Of course, the queuing is task safe. More importantly, it is also not blocking,
which means you can safely log from a protected object for instance (it is an
error in Ada to do a potentially blocking operation from a protected object,
even though in practice a compiler has no way to systematically detect this).

Any of the stream types described earlier (file, zip, socket,...) can be
wrapped into an asynchronous stream. The library does proper locking as needed
to ensure the correctness of operations.

The configuration to declare asynchronous streams is very similar to the one
that combined multiple streams above. Only the `type` changes.

```yaml
{
    streams: {
       'my_async': {  # any name would work
          type: async,
          dispatch: [file1, file2]
       }
    }
}
```

*`GNATCOLL.Traces` does not support asynchronous logging.*

### Message decorators

There are a lot of information that we would like to be part of every message
we output. These include current timestamp, task id, source location, program
name and more.

The loggers library lets you configure, for each stream, what extra decorators
should be added to the user-provided message. The value of those decorators
is captured when the message is emitted (since with asynchronous logging, for
instance, the timestamp might be much later when we actually write to the disk).

We can further extend our example configuration file to override the formats
of the output:

```yaml
{
   streams: {
       file1: {
          ...as before...
          format: "{scope_indent}[{logger}] {msg}{scope_elapsed}"
       },
       file2: {
          ...as before...
          format: "{date_time}{scope_indent} [{logger}@{severity}/{task_id}] {msg}"
       }
   }
}
```

*`GNATCOLL.Traces` also has similar decorators, though users only has coarse
control on them. They can only enable or disable them, but not specify in
which order they appear in the output message, for instance. All streams share
the same decorators.*

### Task safe

All logging is task safe. It is possible to emit messages to the same logger
from multiple tasks, without any interference. It is also possible to write
to the same stream via multiple tasks concurrently.

Finally, changing the configuration of loggers (threshold for instance) is
task safe so can be performed at any time while the program is running.

*`GNATCOLL.Traces` provides basic task safety, using a lock before it writes
to the file. This lock however limits the performance if multiple tasks write
to the same stream.*

### High-performance

The library uses various approaches to ensure the best performance possible.
Our goal was that we could keep as many log calls in our code as we wanted,
and not pay a price unless the messages are actually displayed.

The critical improvement compared to `GNATCOLL.Traces` for instance is that
when a message is at a low-level (`debug`) compare to its logger's threshold,
almost nothing is done. In GNATCOLL, one needs to test whether the logger is
`Active`, then call `Trace` after formatting the string. In our loggers,
one just calls `Log` and passes the various components (as string, integer,
user-defined types,...). The `"&"` operators will do nothing if the message
is to be discarded eventually.

Unless you use a type for which you need to explicitly call an image function
(which the compiler would always call), there is no need to explicitly test
whether the message will be output.

For instance, we are able to log 1.3 billion messages per second when the
message is ultimately discarded (7.7 nanoseconds per message). On the same
machine, `GNATCOLL.Traces` manages 8 million (in both cases the message
includes a string and an integer in our test).

In performance-sensitive contexts, you should use asynchronous logging to
defer as much work as possible to the background task (formatting the message,
locking streams, writing to files,...).

When logging a simple string, it takes 570 nanosecond per message on this
machine (17.5 million messages per second in the user task, though it takes
longer for the background task to eventually complete writing to the file).

When the message is slightly more complex (a string and an integer), it takes
590 nanosecond per message, or 16.9 million messages per second).

*This library is up to 3 times faster than `GNATCOLL.Traces` when actually
writing messages, and up to 150 times faster for inactive messages.*

### Support for long exception messages

One of the limitations in the GNAT's implementation of exceptions is that
exception messages are silently truncated after 200 characters. One of the
workarounds is to log a message with all the details, then raise an exception.

There are however several drawbacks to this approach: the log message might be
much earlier than the exception in the log file; the exception might end up
being caught by another subprogram that deals with the issue and goes on. In
this case, the log message might be just noise in the log file.

Instead, the loggers library provides a subprogram to raise an exception with
a log message. The message will be displayed in full when the exception itself
is logged. If it is never logged (because it was handled), then nothing is
added to the log file.

The library also provides a subprogram to reraise an exception while adding
extra information to its message.

```ada

procedure P1 (A : Integer; Long_String : String) is
begin
   Loggers.Raise_With_Message
      (Constraint_Error'Identity, +"some message" & A & Long_String);
end P1;

procedure P2 is
begin
   P1 (2, "some possibly long string (longer than 200 characters)");
exception
   when E : Constraint_Error =>
      --  Will log the full message, not truncated to 200 characters
      Log (Me.Error & "Caught exception" & E);
   when E : others =>
      --  Also works with exceptions raised via the usual raise statement. In
      --  this case though the exception message has already been truncated by
      --  GNAT.
      Log (Me.Error & "Unexpected" & E);
end P2;
```

### Dynamic configuration

How many times have you run a long-running program to discover there is a bug,
but you have not activated all the log messages you need to fully understand
it ?

The loggers can optionally open a socket, to which users connect via `telnet`
for instance. This socket supports a simple text protocol, which let you
change the threshold of any logger, for instance to decide that you want to
see all messages are priority DEBUG or above, at least temporarily.

### Extensible

This library can be extended from user programs.

For instance, it is possible to add your own **custom streams**. You might want to
display the messages (or a subset of them) in a GUI. Or perhaps send them to
a central server, or store them in some database table...

You can also create **custom decorators** to add extra information to all
messages. Those might include program-specific information (a current job id
for instance), or system information (e.g. memory used by the program).

Programs can create **custom filters**, that can then be referenced from
configuration files. Those filters have access to all components of a message,
and can therefore lookup additional information to decide whether to display a
message.  For instance, if you display a price, you could check whether that
price is positive or negative.

The most usual extension is to provide operators to add your **own types** to
messages. This means users will no longer have to call an `Image` function
to convert the data to a string. Filters will also be able to look at the
actual data, not their string representation. And this is more efficient since
the conversion to string only occurs when the message is printed.

As mentioned earlier in this document, you can create your **own configuration
file** (or perhaps integrate the logger setup in some existing config file you
already have, or read from a database)
