## Table of contents:
   - [index.md](Features): this document
   - [api.md](API): more detailed description of the API
   - [implementation.md](Implementation): implementation details

## Features

This post describes our implementation of a library used to provide
high-performance logging for our Ada applications. It provides a large
number of features that are detailed below.

We compare with existing frameworks when possible. In particular, this
library is a rewrite and extension of `GNATCOLL.Traces`, provided by AdaCore.

### Semantic messages

Although they ultimately will likely be printed as strings in a file, the
messages are internally made of one or more components (a string, an integer,
an instance of a user-defined type,...). Keeping those typed components till
the last minute allows us to filter out messages, to possibly output the
message to a database rather than a string, or to format them as JSON instead
of strings (though this is not implemented currently).

They also provide improved efficiency, since the loggers are careful to avoid
formatting the string (which might be a slow operation) if not needed.

`GNATCOLL.Traces` doesn't have this flexibility, since it only strings for
messages (thus the user typically has to call `Integer'Image` to output an
integer).

### Multi-loggers

Users can define multiple loggers. Each of these logger is in charge of
deciding whether the message will actually be sent or printed anywhere, and
then to forward it to one or more streams.

This setup makes it possible to combine external libraries, each with their own
loggers, and decide which part of your application or these libraries should
be logged.

Loggers are identified by a name, which can be referenced from configuration
files to alter the logger's configuration.

Each message is sent to one logger. The message is emitted with a user-defined
level, like `debug`, `info`, `warning` or `error` (though others can be added
as needed in your application). This level is compared with a logger-specific
threshold, and if the level is greater than the threshold it is forwarded to
the stream associated with the logger.

Compared to `GNATCOLL.Traces`, our loggers are more flexible since the initial
decision on whether a message should be visible or not uses multiple levels,
whereas GNATCOLL uses a simple boolean. Those threshold are fairly common in
other libraries like Python's `logging` module.

### Multi streams

Messages can be sent to one or more streams. Each of these is responsible for
printing the message somewhere, be it in a file, on the console, on a socket,
in syslog for Unix-based systems, in a zip file,...

The predefined streams include files and zip files, each of which can be
further configured. For instance, the name of the file can be computed
automatically to include the current date and time, a unique id, or the value
of any reference variable. It is possible to control the amount of buffering
that occurs, as a balance between efficiency and reliability.

More importantly, it is possible to automatically rotate files. For instance,
at the end of the day, the current day's file is closed and a new file is
created. This helps keep the size of log files smaller for long running
applications.

The library can automatically create missing directories if needed.

Compared to `GNATCOLL.Traces`, our loggers are more flexible, since a message
can end up on multiple streams. We also added support for zip files, and
rotating files.

### Filtering of messages

When it receives a message, a stream can optionally run a filter to decide
whether this message should be printed.

Those filters provide greater control than the levels and thresholds described
above. For instance, it is possible that a stream only wants to show `error`
messages or above, or only messages coming from a given third party library,
or even only if the messages includes a components of a given type.

`GNATCOLL.Traces` has no such notion of filtering. The filtering must be done
in user code, and thus cannot be controlled via a configuration file for
instance.

### Synchronous and asynchronous logging

Depending on your requirements, you might want the message to be written
synchronously to the stream (so that even if your application crashes, the
message is guaranteed to have been printed). This has however a significant
cost in terms of performance, because writing to a file is always a small
operation compared to writing to memory.

Synchronous streams are also required if your application doesn't allow
multiple tasks, perhaps via a configuration pragma.

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

`GNATCOLL.Traces` does not support asynchronous logging.

### Task safe

All logging is task safe. It is possible to emit messages to the same logger
from multiple tasks, without any interference. It is also possible to write
to the same stream via multiple tasks concurrently.

Finally, changing the configuration of loggers (threshold for instance) is
task safe so can be performed at any time while the program is running.

`GNATCOLL.Traces` provides basic task safety, using a lock before it writes
to the file. This lock however limits the performance if multiple tasks write
to the same stream.

### High-performance

The library uses various approaches to ensure the best performance possible.
Our goal was that we could leave as many log calls in our code as we wanted,
and not pay a price unless the messages are actually displayed.

The critical improvement compared to `GNATCOLL.Traces` for instance is that
when a message is at a low-level (`debug`) compare to its logger's threshold,
almost nothing is done. In GNATCOLL, one needs to test whether the logger is
`Active`, then call `Trace` after formatting the string. In our loggers,
one just calls `Log` and passes the various components (as string, integer,
user-defined types,...). Unless you use a type for which you need to explicitly
call an image function, there is no need to explicitly test whether the
message will be output.

For instance, we are able to log 1.3 billion messages per second when the
message is ultimately discarded (7.7 nanoseconds per message). On the same
machine, `GNATCOLL.Traces` manages 8 million (in both cases the message
includes a string and an integer in our test).

In performance-sensitive contexts, you should use asynchronous logging to
defer as much work as possible to the background task (formatting the message,
locking streams, writing to files,...).

When logging a simple string, it takes 570 nanosecond per message on this
machine (17.5 million messages per second in the user task, though it takes
of course longer for the background task to eventually complete writing to the
file).

When the message is slightly more complex (a string and an integer), it takes
590 nanosecond per message, or 16.9 million messages per second).

This library is up to 3 times faster than `GNATCOLL.Traces` when actually
writing messages, and up to 150 times faster for inactive messages.

### Message decorators

There are a lot of information that we would like to be part of every message
we output. These include current timestamp, task id, source location, program
name and more.

The loggers library lets you configure, for each stream, what extra decorators
should be added to the user-provided message. The value of those decorators
is captured when the message is emitted (since with asynchronous logging, for
instance, the timestamp might be much later).

`GNATCOLL.Traces` also has similar decorators, though users only has coarse
control on them. They can only enable or disable them, but not specify in
which order they appear in the output message, for instance. All streams share
the same decorators.

### Configurable

At any time (though it is generally down on application startup), it is
possible to alter the configuration of the loggers (which ones are active,
where the messages are printed,...).

Although everything can be done via direct subprogram calls in Ada, the
simplest is in general to parse configuration files. A default parser is
provided that supports JSON, with a few features added from its YAML support.
As such, it is possible to omit quoting simple strings. It is also possible
to add comments.

The configuration file in GNATCOLL.Traces uses a custom format. There is no
separate configuration for streams, so when multiple loggers are sent to the
same file, the stream's configuration must be duplicated.

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

### Extensible

This library can be extended from user programs.

For instance, it is possible to add your own custom streams. You might want to
display the messages (or a subset of them) in a GUI. Or perhaps send them to
a central server, or store them in some database table...

You can also create your own decorators to add extra information to all
messages. Those might include program-specific information (a current job id
for instance), or system information (e.g. memory used by the program).

Programs can create new filters, that can then be referenced from configuration
files. Those filters have access to all components of a message, and can
therefore lookup additional information to decide whether to display a message.
For instance, if you display a price, you could check whether that price is
is positive or negative.

The most usual extension is to provide operators to add your own types to
messages. This means users will no longer have to call an `Image` function
to convert the data to a string. Filters will also be able to look at the
actual data, not their string representation. And this is more efficient since
the conversion to string only occurs when the message is printed.

