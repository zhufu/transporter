# Design

This document contains both a high level overview of the system and then introduces
proposed changes to enable the system to make certain guarantees about message delivery.
It will then detail changes to enable the system to allow adaptors to "resume" processing
after either a graceful or interrupted restart of the system.

## Adaptors

We want to support various types of systems pertaining to data storage/transport. Most systems
will support the ability for both reading/writing, however, the system allows for one or the 
other implementation. Adaptors are somewhat analogous to a producer/consumer model and can be 
depicted below (R = reader, W = writer, M = message):

```
     +-----------+
     |           | -> W^1^
R -> |     M     | -> W^2^
     |           | -> W^3^
     +-----------+
```

## Messages

Because data structure differs from system to system, transporter attempts to be as agnostic as possible
to any particulars... with the exception of MongoDB due to its [extended JSON](https://docs.mongodb.com/manual/reference/mongodb-extended-json/). Every message flowing through the system takes the following form:

```go
type Message struct {
  Op:        int // Insert/Delete/Update/...
  Timestamp: int64 // internal timestamp
  Namespace: string // generic representation of a schema name (i.e. collection/table/queue)
  Data:      map[string]interface{}
}
```

Messages flow through the system ***one at a time*** which means most writer implementations 
will favor a bulk mechanism for performance.

## Namespace Filtering

Since data is typically segregated into different "buckets" within the underlying system, transporter
uses a `namespace` concept to enable filtering of the data at the "bucket" level. A `namespace` is 
configured by providing a regular expression as defined by the [regexp pkg](https://golang.org/pkg/regexp/).
By default, if no `namespace` is provided, the system will use a "catch all" filter in the form of `.*`.
As messages flow through the system, its originating namespace is matched against the configured 
`namespace` to determine whether the message will continue to the next hop in the system.

The goal of namespace filtering is to be able to define a single reader and filter which segments
are sent to 1 or more writers.

## Data Transformation

Transporter differs from other systems (some, not all) by being able to insert transformation functions
into the data flow pipeline. By doing so, it allows users to manipulate the data structure through code
to ensure the system receiving the data can be handled properly. Below is an example diagram depicting
the use of a transformation function (F = transformation function:

```
     +-----------+
     |           | -> F^1^ -> F^2^ -> W^1^
R -> |     M     | -> W^2^
     |           | -> F^3^ -> W^3^
     +-----------+
```

These function not only allow for manipulating the data structure being sent to do downstream writer 
but can also make a decision of whether or not to "drop" messages such that they never reach the writer.

## Continuous Change Data Capture

Many of today's systems support the ability to not only scan/copy data at a point in time but also
allow for continuously reading changes from the system as they occur in near real-time. It is the goal
of transporter to support continuous change data capture wherever possible based on the underlying 
systems capabilities. 

## Message Guarantees

The system will support an ***at least once*** delivery guarantee wherein the possibility of the same
message being sent more than once exists. 

In order to facilitate this, each writer will have an append only commit log and error log. When a 
message is received by the writer, it will process the message and if no error occurs, it will be 
appended to the commit log. In addition to appending the message, the writer will also be able to 
perform offset tracking which will correlate with the last message written to the underlying system. 
_If_ the writer uses a bulk mechanism, any errors returned from the bulk operation should cause the 
system to stop. 

### Normal Operations

Example of a bulk writer (X = message, O = offset):

commit log
```
+--+--+--+--+--+--+--+--+--+--+
|X |X |X |X |XO|X |X |X |  |  |
+--+--+--+--+--+--+--+--+--+--+
   0  1  2  3  4  5  6  7  8  9 
```

In the above scenario, the commit log is at position 7 but the offset is only at position 4. This would
indicate messages 5-7 have either not been committed to the underlying system or are in the process
of being committed and awaiting a response.

Should the system be forcefully terminated and restarted, messages 5-7 will be redelivered to the writer
and the system will wait until the offset (O) is 7 to ensure message delivery. 

If the system performs a "clean" shutdown, it provides a 30 second window to allow all writers to commit
any messages to the underlying system.

In both scenarios, the state of the system would then be:

commit log
```
+--+--+--+--+--+--+--+--+--+--+
|X |X |X |X |X |X |X |XO|  |  |
+--+--+--+--+--+--+--+--+--+--+
   0  1  2  3  4  5  6  7  8  9 
```

### Message Failures

Example of a bulk writer that receives an error response during a commit:


commit log
```
+--+--+--+--+--+--+--+--+--+--+
|X |X |X |X |X |XO|X |X |  |  |
+--+--+--+--+--+--+--+--+--+--+
   0  1  2  3  4  5  6  7  8  9 
```

Writer attempts to commit messages 6-7 but receives an error response.

commit log
```
+--+--+--+--+--+--+--+--+--+--+
|X |X |X |X |X |X |X |XO|  |  |
+--+--+--+--+--+--+--+--+--+--+
   0  1  2  3  4  5  6  7  8  9 
```

error log
```
+--+--+--+--+--+--+--+--+--+--+
|X |X |  |  |  |  |  |  |  |  |
+--+--+--+--+--+--+--+--+--+--+
   0  1  2  3  4  5  6  7  8  9 
```

In the above scenario, the offset will be equal to the last message in the commit log and the messages
that were part of the errored commit appended to the error log. At this point, this system will have 
shutdown and the operator must review the messages in the error log to decide whether they can be discarded
or attempt to be committed again. It will be up to each implemented writer to determine whether the
messages it appends to the error log are all messages that were part of the commit or only the ones that
resulted in an error.

Once the operator resolves messages 0-1 in the error log, it will be truncated and the system can resume
processing under normal operations.

## Reader State

When a reader sends a messages down the pipeline, it can define `State` associated with the message.
The following attributes are available to readers for defining its state:

```Go
type State struct {
	Identifier interface{}
	Timestamp  uint64
	Namespace  string
	Mode       Mode
}

type Mode int

const (
	Copy Mode = iota
	Sync
)
```

As `State` flows through the system, it will be persisted such that on restart, the last known `State` 
can be retrieved. This persistence will work in conjuction with offset tracking of the commit log. When 
the offset is updated, the `State` associated with that message will be persisted and replace any previous
`State`. 

The `Namespace` field will be used as the key for keeping track of `State` changes over time in order to
support multiple `Namespaces` in different `Modes`. If the system should encounter an error and be 
restarted, the reader will be provided with a `[]State` that it can use to resume its work wherever it left
off.





