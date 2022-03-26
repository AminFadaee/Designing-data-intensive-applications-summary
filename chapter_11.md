# Designing Data Intensive Applications
## Chapter 11: Stream Processing
In reality, a lot of data is unbounded because it arrives gradually over timeUnless you go out of business, this process
never ends, and so the dataset is never “complete” in any meaningful way. Thus, batch processors must artificially
divide the data into chunks of fixed duration.

The problem with daily batch processes is that changes in the input are only reflected in the output a day later, which
is too slow for many impatient users. To reduce the delay, we can run the processing more frequently—say, processing a
second’s worth of data at the end of every second—or even continuously, abandoning the fixed time slices entirely and
simply processing every event as it happens. That is the idea behind stream processing.

In general, a “stream” refers to data that is incrementally made available over time. The concept appears in many
places: in the stdin and stdout of Unix, programming languages (lazy lists), filesystem APIs (such as Java’s
FileInputStream), TCP connections, delivering audio and video over the internet, and so on.

In this chapter we will look at event streams as a data management mechanism: the unbounded, incrementally processed
counterpart to the batch data we saw in the last chapter. We will first discuss how streams are represented, stored, and
transmitted over a network. After that, we will investigate the relationship between streams and databases. And finally,
we will explore approaches and tools for processing those streams continually, and ways that they can be used to build
applications.

### Transmitting Event Streams
In a stream processing context, a record is more commonly known as an event. In batch processing, a file is written once
and then potentially read by multiple jobs. Analogously, in streaming terminology, an event is generated once by a
producer (also known as a publisher or sender), and then potentially processed by multiple consumers (subscribers or
recipients). In a filesystem, a filename identifies a set of related records; in a streaming system, related events are
usually grouped together into a topic or stream.

In principle, a file or database is sufficient to connect producers and consumers: a producer writes every event that it
generates to the datastore, and each consumer periodically polls the datastore to check for events that have appeared
since it last ran. This is essentially what a batch process does when it processes a day’s worth of data at the end of
every day.

However, when moving toward continual processing with low delays, polling becomes expensive if the datastore is not
designed for this kind of usage. The more often you poll, the lower the percentage of requests that return new events,
and thus the higher the overheads become. Instead, it is better for consumers to be notified when new events appear.

Databases have traditionally not supported this kind of notification mechanism very well: relational databases commonly
have triggers, which can react to a change, but they are very limited in what they can do and have been somewhat of an
afterthought in database design. Instead, specialized tools have been developed for the purpose of delivering event
notifications.
#### Messaging Systems
We previously talked about message passing dataflow. Within this *publish/subscribe* model, different systems take a
wide range of approaches, and there is no one right answer for all purposes. To differentiate the systems, it is
particularly helpful to ask the following two questions:

1. What happens if the producers send messages faster than the consumers can process them? Broadly speaking, there are
   three options: the system can drop messages, buffer messages in a queue, or apply backpressure (also known as flow
   control; i.e., blocking the producer from sending more messages).
2. What happens if nodes crash or temporarily go offline—are any messages lost? As with databases, durability may
   require some combination of writing to disk and/or replication, which has a cost. If you can afford to sometimes lose
   messages, you can probably get higher throughput and lower latency on the same hardware.

A nice property of the batch processing systems we explored in Chapter 10 is that they provide a strong reliability
guarantee: failed tasks are automatically retried, and partial output from failed tasks is automatically discarded. This
means the output is the same as if no failures had occurred, which helps simplify the programming model. Later in this
chapter we will examine how we can provide similar guarantees in a streaming context.

##### Direct messaging from producers to consumers
A number of messaging systems use direct network communication between producers and consumers without going via
intermediary nodes. Although these direct messaging systems work well in the situations for which they are designed,
they generally require the application code to be aware of the possibility of message loss. The faults they can tolerate
are quite limited: even if the protocols detect and retransmit packets that are lost in the network, they generally
assume that producers and consumers are constantly online.

If a consumer is offline, it may miss messages that were sent while it is unreachable. Some protocols allow the producer
to retry failed message deliveries, but this approach may break down if the producer crashes, losing the buffer of
messages that it was supposed to retry.
##### Message brokers
A widely used alternative is to send messages via a message broker (also known as a message queue), which is essentially
a kind of database that is optimized for handling message streams. It runs as a server, with producers and consumers
connecting to it as clients. Producers write messages to the broker, and consumers receive them by reading them from the
broker.

By centralizing the data in the broker, these systems can more easily tolerate clients that come and go (connect,
disconnect, and crash), and the question of durability is moved to the broker instead. Some message brokers only keep
messages in memory, while others (depending on configuration) write them to disk so that they are not lost in case of a
broker crash. Faced with slow consumers, they generally allow unbounded queueing (as opposed to dropping messages or
backpressure), although this choice may also depend on the configuration.

A consequence of queueing is also that consumers are generally asynchronous: when a producer sends a message, it
normally only waits for the broker to confirm that it has buffered the message and does not wait for the message to be
processed by consumers. The delivery to consumers will happen at some undetermined future point in time—often within a
fraction of a second, but sometimes significantly later if there is a queue backlog.

##### Message brokers compared to databases
Some message brokers can even participate in two-phase commit protocols. This feature makes them quite similar in nature
to databases, although there are still important practical differences between message brokers and databases:

* Databases usually keep data until it is explicitly deleted, whereas most message brokers automatically delete a
  message when it has been successfully delivered to its consumers. Such message brokers are not suitable for long-term
  data storage.
* Since they quickly delete messages, most message brokers assume that their working set is fairly small—i.e., the
  queues are short. If the broker needs to buffer a lot of messages because the consumers are slow (perhaps spilling
  messages to disk if they no longer fit in memory), each individual message takes longer to process, and the overall
  throughput may degrade.
* Databases often support secondary indexes and various ways of searching for data, while message brokers often support
  some way of subscribing to a subset of topics matching some pattern. The mechanisms are different, but both are
  essentially ways for a client to select the portion of the data that it wants to know about.
* When querying a database, the result is typically based on a point-in-time snapshot of the data; if another client
  subsequently writes something to the database that changes the query result, the first client does not find out that
  its prior result is now outdated (unless it repeats the query, or polls for changes). By contrast, message brokers do
  not support arbitrary queries, but they do notify clients when data changes (i.e., when new messages become
  available).

##### Multiple consumers
When multiple consumers read messages in the same topic, two main patterns of messaging are used:

*Load balancing*

Each message is delivered to one of the consumers, so the consumers can share the work of processing the messages in the
topic. The broker may assign messages to consumers arbitrarily. This pattern is useful when the messages are expensive
to process, and so you want to be able to add consumers to parallelize the processing. 

*Fan-out*

Each message is delivered to all of the consumers. Fan-out allows several independent consumers to each “tune in” to the
same broadcast of messages, without affecting each other—the streaming equivalent of having several different batch jobs
that read the same input file. 

The two patterns can be combined: for example, two separate groups of consumers
may each subscribe to a topic, such that each group collectively receives all messages,
but within each group only one of the nodes receives each message.

##### Acknowledgements and redelivery
Consumers may crash at any time, so it could happen that a broker delivers a message to a consumer but the consumer
never processes it, or only partially processes it before crashing. In order to ensure that the message is not lost,
message brokers use acknowledgments: a client must explicitly tell the broker when it has finished processing a message
so that the broker can remove it from the queue.

If the connection to a client is closed or times out without the broker receiving an acknowledgment, it assumes that the
message was not processed, and therefore it delivers the message again to another consumer.

#### Partitioned Logs
Sending a packet over a network or making a request to a network service is normally a transient operation that leaves
no permanent trace. Although it is possible to record it permanently (using packet capture and logging), we normally
don’t think of it that way. Even message brokers that durably write messages to disk quickly delete them again after
they have been delivered to consumers, because they are built around a transient messaging mindset.

Databases and filesystems take the opposite approach: everything that is written to a database or file is normally
expected to be permanently recorded, at least until someone explicitly chooses to delete it again.

If you add a new consumer to a messaging system, it typically only starts receiving messages sent after the time it was
registered; any prior messages are already gone and cannot be recovered. Contrast this with files and databases, where
you can add a new client at any time, and it can read data written arbitrarily far in the past. Why can we not have a
hybrid, combining the durable storage approach of databases with the low-latency notification facilities of messaging?
This is the idea behind log-based message brokers.

##### Using logs for message storage
A log is simply an append-only sequence of records on disk. The same structure can be used to implement a message
broker: a producer sends a message by appending it to the end of the log, and a consumer receives messages by reading
the log sequentially.

In order to scale to higher throughput than a single disk can offer, the log can be partitioned. Different partitions
can then be hosted on different machines, making each partition a separate log that can be read and written
independently from other partitions. A topic can then be defined as a group of partitions that all carry messages of the
same type.

Within each partition, the broker assigns a monotonically increasing sequence number, or offset, to every message. Such
a sequence number makes sense because a partition is append-only, so the messages within a partition are totally
ordered. There is no ordering guarantee across different partitions.

##### Logs compared to traditional messaging
The log-based approach trivially supports fan-out messaging, because several consumers can independently read the log
without affecting each other—reading a message does not delete it from the log. To achieve load balancing across a group
of consumers, instead of assigning individual messages to consumer clients, the broker can assign entire partitions to
nodes in the consumer group.

Each client then consumes all the messages in the partitions it has been assigned. Typically, when a consumer has been
assigned a log partition, it reads the messages in the partition sequentially, in a straightforward single-threaded
manner. This coarse-grained load balancing approach has some downsides:

* The number of nodes sharing the work of consuming a topic can be at most the number of log partitions in that topic,
  because messages within the same partition are delivered to the same node.
* If a single message is slow to process, it holds up the processing of subsequent messages in that partition.

Thus, in situations with high message throughput, where each message is fast to process and where message ordering is
important, the log-based approach works very well.

##### Consumer offsets
Consuming a partition sequentially makes it easy to tell which messages have been processed: all messages with an offset
less than a consumer’s current offset have already been processed, and all messages with a greater offset have not yet
been seen. Thus, the broker does not need to track acknowledgments for every single message— it only needs to
periodically record the consumer offsets. The reduced bookkeeping overhead and the opportunities for batching and
pipelining in this approach help increase the throughput of log-based systems.

If a consumer node fails, another node in the consumer group is assigned the failed consumer’s partitions, and it starts
consuming messages at the last recorded offset. If the consumer had processed subsequent messages but not yet recorded
their offset, those messages will be processed a second time upon restart.

##### Disk space usage
If you only ever append to the log, you will eventually run out of disk space. To reclaim disk space, the log is
actually divided into segments, and from time to time old segments are deleted or moved to archive storage. 

This means that if a slow consumer cannot keep up with the rate of messages, and it falls so far behind that its
consumer offset points to a deleted segment, it will miss some of the messages. Effectively, the log implements a
bounded-size buffer that discards old messages when it gets full, also known as a circular buffer or ring buffer.
However, since that buffer is on disk, it can be quite large.

### Databases and Streams
We can also go in reverse: take ideas from messaging and streams, and apply them to databases. In this section we will
first look at a problem that arises in heterogeneous data systems, and then explore how we can solve it by bringing
ideas from event streams to databases.

#### Keeping Systems in Sync
As we have seen throughout this book, there is no single system that can satisfy all data storage, querying, and
processing needs. In practice, most nontrivial applications need to combine several different technologies in order to
satisfy their requirements: for example, using an OLTP database to serve user requests, a cache to speed up common
requests, a full-text index to handle search queries, and a data warehouse for analytics. Each of these has its own copy
of the data, stored in its own representation that is optimized for its own purposes.

As the same or related data appears in several different places, they need to be kept in sync with one another: if an
item is updated in the database, it also needs to be updated in the cache, search indexes, and data warehouse. With data
warehouses this synchronization is usually performed by ETL processes, often by taking a full copy of a database,
transforming it, and bulk-loading it into the data warehouse—in other words, a batch process. Similarly, we saw before
how search indexes, recommendation systems, and other derived data systems might be created using batch processes.

If periodic full database dumps are too slow, an alternative that is sometimes used is dual writes, in which the
application code explicitly writes to each of the systems when data changes: for example, first writing to the database,
then updating the search index, then invalidating the cache entries (or even performing those writes concurrently).

However, dual writes have some serious problems, one of which is a race condition. Another problem with dual writes is
that one of the writes may fail while the other succeeds. This is a fault-tolerance problem rather than a concurrency
problem, but it also has the effect of the two systems becoming inconsistent with each other. Ensuring that they either
both succeed or both fail is a case of the atomic commit problem, which is expensive to solve.

If you only have one replicated database with a single leader, then that leader determines the order of writes, so the
state machine replication approach works among replicas of the database. However, the database may have a leader and the
search index may have a leader, but neither follows the other, and so conflicts can occur.

The situation would be better if there really was only one leader—for example, the database—and if we could make the
search index a follower of the database. But is this possible in practice?

#### Change Data Capture
More recently, there has been growing interest in change data capture (CDC), which is the process of observing all data
changes written to a database and extracting them in a form in which they can be replicated to other systems. CDC is
especially interesting if changes are made available as a stream, immediately as they are written.

For example, you can capture the changes in a database and continually apply the same changes to a search index. If the
log of changes is applied in the same order, you can expect the data in the search index to match the data in the
database. The search index and any other derived data systems are just consumers of the change stream.

##### Implementing change data capture
Change data capture is a mechanism for ensuring that all changes made to the system of record are also reflected in the
derived data systems so that the derived systems have an accurate copy of the data.

Essentially, change data capture makes one database the leader, and turns the others into followers. A log-based message
broker is well suited for transporting the change events from the source database, since it preserves the ordering of
messages.

Database triggers can be used to implement change data capture by registering triggers that observe all changes to data
tables and add corresponding entries to a changelog table. However, they tend to be fragile and have significant
performance overheads. Parsing the replication log can be a more robust approach, although it also comes with
challenges, such as handling schema changes.

Like message brokers, change data capture is usually asynchronous: the system of record database does not wait for the
change to be applied to consumers before committing it. This design has the operational advantage that adding a slow
consumer does not affect the system of record too much, but it has the downside that all the issues of replication lag
apply.

#### Event Sourcing
There are some parallels between the ideas we’ve discussed here and event sourcing, a technique that was developed in
the domain-driven design (DDD) community.

Similarly to change data capture, event sourcing involves storing all changes to the application state as a log of
change events. The biggest difference is that event sourcing applies the idea at a different level of abstraction:

* In change data capture, the application uses the database in a mutable way, updating and deleting records at will. The
  log of changes is extracted from the database at a low level (e.g., by parsing the replication log), which ensures
  that the order of writes extracted from the database matches the order in which they were actually written, avoiding
  race conditions. The application writing to the database does not need to be aware that CDC is occurring.
* In event sourcing, the application logic is explicitly built on the basis of immutable events that are written to an
  event log. In this case, the event store is append-only, and updates or deletes are discouraged or prohibited. Events
  are designed to reflect things that happened at the application level, rather than low-level state changes.

Event sourcing is a powerful technique for data modeling: from an application point of view it is more meaningful to
record the user’s actions as immutable events, rather than recording the effect of those actions on a mutable database.
Event sourcing makes it easier to evolve applications over time, helps with debugging by making it easier to understand
after the fact why something happened, and guards against application bugs.

Event sourcing is similar to the chronicle data model, and there are also similarities between an event log and the
fact table that you find in a star schema.

Specialized databases have been developed to support applications using event sourcing, but in general the approach is
independent of any particular tool. A conventional database or a log-based message broker can also be used to build
applications in this style.

##### Deriving current state from the event log
An event log by itself is not very useful, because users generally expect to see the current state of a system, not the
history of modifications. For example, on a shopping website, users expect to be able to see the current contents of
their cart, not an append-only list of all the changes they have ever made to their cart.

Thus, applications that use event sourcing need to take the log of events (representing the data written to the system)
and transform it into application state that is suitable for showing to a user (the way in which data is read from the
system). This transformation can use arbitrary logic, but it should be deterministic so that you can run it again and
derive the same application state from the event log.

#### State, Streams, and Immutability
Like batch processing, the principle of immutability is also what makes event sourcing and change data capture so
powerful.

We normally think of databases as storing the current state of the application. Whenever you have state that changes,
that state is the result of the events that mutated it over time. The key idea is that mutable state and an append-only
log of immutable events do not contradict each other: they are two sides of the same coin. The log of all changes, the
changelog, represents the evolution of state over time. If you store the changelog durably, that simply has the effect
of making the state reproducible.

As Pat Helland puts it:

> Transaction logs record all the changes made to the database. High-speed appends are the only way to change the log.
> From this perspective, the contents of the database hold a caching of the latest record values in the logs. The truth
> is the log. The database is a cache of a subset of the log. That cached subset happens to be the latest value of each
> record and index value from the log.

##### Deriving several views from the same event log
Besides keeping the mistakes in the history and capturing more information than the current state, another benefit of
separating mutable state from the immutable event log is that you can derive several different read-oriented
representations from the same log of events.

Having an explicit translation step from an event log to a database makes it easier to evolve your application over
time: if you want to introduce a new feature that presents your existing data in some new way, you can use the event log
to build a separate read-optimized view for the new feature, and run it alongside the existing systems without having to
modify them. Running old and new systems side by side is often easier than performing a complicated schema migration in
an existing system. Once the old system is no longer needed, you can simply shut it down and reclaim its resources.

Storing data is normally quite straightforward if you don’t have to worry about how it is going to be queried and
accessed; many of the complexities of schema design, indexing, and storage engines are the result of wanting to support
certain query and access patterns. For this reason, you gain a lot of flexibility by separating the form in which data
is written from the form it is read, and by allowing several different read views. This idea is sometimes known as
*command query responsibility segregation* (CQRS).

The traditional approach to database and schema design is based on the fallacy that data must be written in the same
form as it will be queried. Debates about normalization and denormalization become largely irrelevant if you can
translate data from a write-optimized event log to read-optimized application state: it is entirely reasonable to
denormalize data in the read-optimized views, as the translation process gives you a mechanism for keeping it consistent
with the event log.

##### Concurrency control
The biggest downside of event sourcing and change data capture is that the consumers of the event log are usually
asynchronous, so there is a possibility that a user may make a write to the log, then read from a log-derived view and
find that their write has not yet been reflected in the read view. We discussed this problem and potential solutions
previously in “Reading Your Own Writes”.

One solution would be to perform the updates of the read view synchronously with appending the event to the log. This
requires a transaction to combine the writes into an atomic unit, so either you need to keep the event log and the read
view in the same storage system, or you need a distributed transaction across the different systems. Alternatively, you
could use the approach discussed in “Implementing linearizable storage using total order broadcast”.

On the other hand, deriving the current state from an event log also simplifies some aspects of concurrency control.
Much of the need for multi-object transactions stems from a single user action requiring data to be changed in several
different places. With event sourcing, you can design an event such that it is a self-contained description of a user
action. The user action then requires only a single write in one place—namely appending the events to the log—which is
easy to make atomic.

##### Limitations of immutability
Many systems that don’t use an event-sourced model nevertheless rely on immutability: various databases internally use
immutable data structures or multi-version data to support point-in-time snapshots. Many version control systems rely on
immutable data to preserve version history of files.

To what extent is it feasible to keep an immutable history of all changes forever? The answer depends on the amount of
churn in the dataset. Some workloads mostly add data and rarely update or delete; they are easy to make immutable. Other
workloads have a high rate of updates and deletes on a comparatively small dataset; in these cases, the immutable
history may grow prohibitively large, fragmentation may become an issue, and the performance of compaction and garbage
collection becomes crucial for operational robustness.

Besides the performance reasons, there may also be circumstances in which you need data to be deleted for administrative
reasons, in spite of all immutability. For example, privacy regulations may require deleting a user’s personal
information after they close their account, data protection legislation may require erroneous information to be removed,
or an accidental leak of sensitive information may need to be contained.

Truly deleting data is surprisingly hard, since copies can live in many places: for example, storage engines,
filesystems, and SSDs often write to a new location rather than overwriting in place, and backups are often deliberately
immutable to prevent accidental deletion or corruption. Deletion is more a matter of “making it harder to retrieve the
data” than actually “making it impossible to retrieve the data.” Nevertheless, you sometimes have to try, as we shall
see in “Legislation and self-regulation”.

### Processing Streams
You can use the events in a stream to write it to a database, cache, search index or similar storage system; push the
events to users in some way (push notifications, emails or live dashbords); or, you can process one or more input
streams to produce one or more output streams (what we are going to discuss next).

The piece of code that processes streams like this is known as an *operator* or *job*: a stream processor consumes input
streams in a read-only fashion and writes its output to a different location in an append-only fashion.

A one crucial difference between this model and batch processing is that a stream never ends. This difference has many
implications: as discussed at the start of this chapter, sorting does not make sense with an unbounded dataset, and so
sort-merge joins cannot be used. Fault-tolerance mechanisms must also change: with a batch job that has been running for
a few minutes, a failed task can simply be restarted from the beginning, but with a stream job that has been running for
several years, restarting from the beginning after a crash may not be a viable option.

#### Uses of Stream Processing
Stream processing has long been used for monitoring purposes, where an organization wants to be alerted if certain
things happen. However, other uses of stream processing have also emerged over time. In this section we will briefly
compare and contrast some of these applications.

##### Complex event processing
Complex event processing (CEP) is an approach developed in the 1990s for analyzing event streams, especially geared
toward the kind of application that requires searching for certain event patterns. Similarly to the way that a
regular expression allows you to search for certain patterns of characters in a string, CEP allows you to specify rules
to search for certain patterns of events in a stream.

In these systems, the relationship between queries and data is reversed compared to normal databases. Usually, a
database stores data persistently and treats queries as transient: when a query comes in, the database searches for data
matching the query, and then forgets about the query when it has finished. CEP engines reverse these roles: queries are
stored long-term, and events from the input streams continuously flow past them in search of a query that matches an
event pattern.

##### Stream analytics
Another area in which stream processing is used is for analytics on streams. The boundary between CEP and stream
analytics is blurry, but as a general rule, analytics tends to be less interested in finding specific event sequences
and is more oriented toward aggregations and statistical metrics over a large number of events.

Stream analytics systems sometimes use probabilistic algorithms, such as Bloom filters for set membership, HyperLogLog
for cardinality estimation, and various percentile estimation algorithms. Probabilistic algorithms produce approximate
results, but have the advantage of requiring significantly less memory in the stream processor than exact algorithms.

##### Maintaining materialized views
in event sourcing, application state is maintained by applying a log of events; here the application state is also a
kind of materialized view. Unlike stream analytics scenarios, it is usually not sufficient to consider only events
within some time window: building the materialized view potentially requires all events over an arbitrary time period,
apart from any obsolete events that may be discarded by log compaction. In effect, you need a window that stretches all
the way back to the beginning of time.

##### Search on streams
Besides CEP, which allows searching for patterns consisting of multiple events, there is also sometimes a need to search
for individual events based on complex criteria, such as full-text search queries.

#### Reasoning about Time
Stream processors often need to deal with time, especially when used for analytics purposes, which frequently use time
windows such as “the average over the last five minutes.” It might seem that the meaning of “the last five minutes”
should be unambiguous and clear, but unfortunately the notion is surprisingly tricky.

Many stream processing frameworks use the local system clock on the processing machine (the processing time) to
determine windowing. This approach has the advantage of being simple, and it is reasonable if the delay between event
creation and event processing is negligibly short. However, it breaks down if there is any significant processing lag.

##### Knowing when you’re ready
A tricky problem when defining windows in terms of event time is that you can never be sure when you have received all
of the events for a particular window, or whether there are some events still to come. You need to be able to handle
such straggler events that arrive after the window has already been declared complete. Broadly, you have two options:

* Ignore the straggler events, as they are probably a small percentage of events in normal circumstances. You can track
  the number of dropped events as a metric, and alert if you start dropping a significant amount of data.
* Publish a correction, an updated value for the window with stragglers included. You may also need to retract the
  previous output.

##### Whose clock are you using, anyway?
Assigning timestamps to events is even more difficult when events can be buffered at several points in the system. To
adjust for incorrect device clocks, one approach is to log three timestamps:

* The time at which the event occurred, according to the device clock
* The time at which the event was sent to the server, according to the device clock
* The time at which the event was received by the server, according to the server clock

By subtracting the second timestamp from the third, you can estimate the offset between the device clock and the server
clock (assuming the network delay is negligible compared to the required timestamp accuracy). You can then apply that
offset to the event timestamp, and thus estimate the true time at which the event actually occurred (assuming the device
clock offset did not change between the time the event occurred and the time it was sent to the server).

This problem is not unique to stream processing—batch processing suffers from exactly the same issues of reasoning about
time. It is just more noticeable in a streaming context, where we are more aware of the passage of time.

##### Types of windows
Once you know how the timestamp of an event should be determined, the next step is to decide how windows over time
periods should be defined. The window can then be used for aggregations, for example to count events, or to calculate
the average of values within the window. Several types of windows are in common use:

*Tumbling window*

A tumbling window has a fixed length, and every event belongs to exactly one window. For example, if you have a 1-minute
tumbling window, all the events with timestamps between 10:03:00 and 10:03:59 are grouped into one window.

*Hopping Window*

A hopping window also has a fixed length, but allows windows to overlap in order to provide some smoothing. For example,
a 5-minute window with a hop size of 1 minute would contain the events between 10:03:00 and 10:07:59, then the next
window would cover events between 10:04:00 and 10:08:59, and so on.

*Sliding Window*

A sliding window contains all the events that occur within some interval of each other. For example, a 5-minute sliding
window would cover events at 10:03:39 and 10:08:12, because they are less than 5 minutes apart (note that tumbling and
hopping 5-minute windows would not have put these two events in the same window, as they use fixed boundaries). A
sliding window can be implemented by keeping a buffer of events sorted by time and removing old events when they expire
from the window.

*Session Window*

Unlike the other window types, a session window has no fixed duration. Instead, it is defined by grouping together all
events for the same user that occur closely together in time, and the window ends when the user has been inactive for
some time.

#### Stream Joins
Since stream processing generalizes data pipelines to incremental processing of unbounded datasets, there is exactly the
same need for joins on streams. However, the fact that new events can appear anytime on a stream makes joins on streams
more challenging than in batch jobs. We can distinguish three types of joins that may appear in stream processes:

* Stream-stream joins: Both input streams consist of activity events, and the join operator searches for related events
  that occur within some window of time. For example, it may match two actions taken by the same user within 30 minutes
  of each other. The two join inputs may in fact be the same stream (a self-join) if you want to find related events
  within that one stream.
* Stream-table joins: One input stream consists of activity events, while the other is a database changelog. The
  changelog keeps a local copy of the database up to date. For each activity event, the join operator queries the
  database and outputs an enriched activity event.
* Table-table joins: Both input streams are database changelogs. In this case, every change on one side is joined with
  the latest state of the other side. The result is a stream of changes to the materialized view of the join between the
  two tables.

#### Fault Tolerance
In the final section of this chapter, let’s consider how stream processors can tolerate faults. The batch approach to
fault tolerance ensures that the output of the batch job is the same as if nothing had gone wrong, even if in fact some
tasks did fail. It appears as though every input record was processed exactly once—no records are skipped, and none are
processed twice. Although restarting tasks means that records may in fact be processed multiple times, the visible
effect in the output is as if they had only been processed once. This principle is known as *exactly-once* semantics,
although *effectively-once* would be a more descriptive term.

##### Microbatching and checkpointing
One solution is to break the stream into small blocks, and treat each block like a miniature batch process. This
approach is called *microbatching*. The batch size is typically around one second, which is the result of a performance
compromise: smaller batches incur greater scheduling and coordination overhead, while larger batches mean a longer delay
before results of the stream processor become visible.

A variant approach, is to periodically generate rolling checkpoints of state and write them to durable storage. If a
stream operator crashes, it can restart from its most recent checkpoint and discard any output generated between the
last checkpoint and the crash. The checkpoints are triggered by barriers in the message stream, similar to the
boundaries between microbatches, but without forcing a particular window size.

Within the confines of the stream processing framework, the microbatching and checkpointing approaches provide the same
exactly-once semantics as batch processing. However, as soon as output leaves the stream processor (for example, by
writing to a database, sending messages to an external message broker, or sending emails), the framework is no longer
able to discard the output of a failed batch. In this case, restarting a failed task causes the external side effect to
happen twice, and microbatching or checkpointing alone is not sufficient to prevent this problem.

##### Atomic commit revisited
In order to give the appearance of exactly-once processing in the presence of faults, we need to ensure that all outputs
and side effects of processing an event take effect if and only if the processing is successful. Those effects include
any messages sent to downstream operators or external messaging systems (including email or push notifications), any
database writes, any changes to operator state, and any acknowledgment of input messages (including moving the consumer
offset forward in a log-based message broker). Those things either all need to happen atomically, or none of them must
happen, but they should not go out of sync with each other.

##### Idempotence
Our goal is to discard the partial output of any failed tasks so that they can be safely retried without taking effect
twice. Distributed transactions are one way of achieving that goal, but another way is to rely on idempotence. An
idempotent operation is one that you can perform multiple times, and it has the same effect as if you performed it only
once.

##### Rebuilding state after a failure
Any stream process that requires state—for example, any windowed aggregations (such as counters, averages, and
histograms) and any tables and indexes used for joins—must ensure that this state can be recovered after a failure. One
option is to keep the state in a remote datastore and replicate it. In some cases, it may not even be necessary to
replicate the state, because it can be rebuilt from the input streams.
