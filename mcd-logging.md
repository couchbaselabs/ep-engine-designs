
##Memcached Logging

Couchbase 2.2 currently uses the memcached file logging module to log server activity. This module is rather basic and allows us to log string messages to a file. One problem with this approach is that we leave the message format completely up to the engineer who wrote it and as a result these messages are not easily searchable. Furthermore, everything is logged as text even though certain pieces of the message such as time stamp, VBucket id, and so on could be encoded as binary. The purpose of this design document is to improve the logging format to allow better compression of messages, searchability of the log, and an application that allow engineers and support to debug the logs more easily.

####Logging format

**Event Id (16 bits)**: The event ID will be used to signify important events that take place in the memcached process of Couchbase Server. Examples of event IDs include:

* Memcached Start-up
* Connection Established
* Tap Connect
* File Compaction

**Severity (8 bits)**: The log level.

**Time stamp (64 bits)**: The time when the event occurred.

**VBucket (16 bits)**: The VBucket that the event took place in. Set to -1 if this event did not take place in any VBucket.

**Module (8 bits)**: The module will contain the sub-component of the memcached process that the event took place in. Examples of sub-componenets are:

* Connection Management
* Non-IO Dispatcher
* Checkpointing
* Tap
* Warmup
* KVStore

**Group ID (64 bits)**: The group id can be used to follow a sequence of actions. An example of how a group id could be useful is when we are looking at tap connections. Throughout the life cycle of a tap connection there would be one group id assigned to that connection. If we wanted to see all of the log messages for this connection then we would just search for all of the messages containing this group id.

**Bucket Name**: The name of the bucket that the event took place in.

**Message**: The message that we want to log.

####Compression

The message format above would be logged in binary format in order to save as much space as possible. Moving forward we could also consider compressing the actual messages as well. This scheme would help us increase the amount of messages we could log with the same space overhead.

####Examining log files

In order to examine log files we would need an application that allowed us to both dump the contents of a log file (since the log file will be compressed or in binary) and also run queries against the log file in order to find information. This application would be in the form of a command line tool and would be written in python. Examples of how this tool would work are below.

lquery -h

-f, --file  The log file to examine<br>
-d, --dump  Dumps the specified log file to stdout<br>
-q, --query Speifies a query to be run on the log file

The initial query language is shown below:

	This grammer isn't 100% finished, but it shows what a query would look like.

    START -> FILTER EXTENDER START
           | eps

    FILTER -> vbucket number
           |  period timval timeval
           |  event string
           |  bucket string
           |  group number
           |  module string
           |  level string

    ENTENDER -> AND
             |  OR

####Incremental implementation

Although there are many changes that would need to be made to fully implement this feature it can be done incrementally. In particular having the module name and vbucket in each message would likely go a long way in making the logs more readable and easier to use in debugging.