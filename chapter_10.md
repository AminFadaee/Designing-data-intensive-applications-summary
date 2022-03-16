# Designing Data Intensive Applications
## Chapter 10: Batch Processing
The web, and increasing numbers of HTTP/REST-based APIs, has made the request/response style of interaction so common
that it’s easy to take it for granted. But we should remember that it’s not the only way of building systems, and that
other approaches have their merits too. Let’s distinguish three different types of systems:

*Services (online systems)*

A service waits for a request or instruction from a client to arrive. When one is received, the service tries to handle
it as quickly as possible and sends a response back. Response time is usually the primary measure of performance of a
service, and availability is often very important (if the client can’t reach the service, the user will probably get an
error message).

*Batch processing systems (offline systems)*

A batch processing system takes a large amount of input data, runs a job to process it, and produces some output data.
Jobs often take a while (from a few minutes to several days), so there normally isn’t a user waiting for the job to
finish. Instead, batch jobs are often scheduled to run periodically (for example, once a day). The primary performance
measure of a batch job is usually throughput (the time it takes to crunch through an input dataset of a certain size).
We discuss batch processing in this chapter.

*Stream processing systems (near-real-time systems)*

Stream processing is somewhere between online and offline/batch processing (so it is sometimes called near-real-time or
nearline processing). Like a batch processing system, a stream processor consumes inputs and produces outputs (rather
than responding to requests). However, a stream job operates on events shortly after they happen, whereas a batch job
operates on a fixed set of input data. This difference allows stream processing systems to have lower latency than the
equivalent batch systems. As stream processing builds upon batch processing, we discuss it in Chapter 11.

In this chapter, we will look at MapReduce and several other batch processing algorithms and frameworks, and explore how
they are used in modern data systems.

### MapReduce and Distributed Filesystems
MapReduce is a bit like Unix tools (`awk`, `uniq`, `sort`, ...), but distributed across potentially thousands of
machines. Like Unix tools, it is a fairly blunt, brute-force, but surprisingly effective tool. A single MapReduce job is
comparable to a single Unix process: it takes one or more inputs and produces one or more outputs.

As with most Unix tools, running a MapReduce job normally does not modify the input and does not have any side effects
other than producing the output. While Unix tools use stdin and stdout as input and output, MapReduce jobs read and
write files on a distributed filesystem. In Hadoop’s implementation of MapReduce, HDFS (Hadoop Distributed File System)
is used.

HDFS is based on the shared-nothing principle. The shared-nothing approach requires no special hardware, only computers
connected by a conventional datacenter network. A central server called the `NameNode` keeps track of which file blocks
are stored on which machine. Thus, HDFS conceptually creates one big filesystem that can use the space on the disks of
all machines running the daemon. In order to tolerate machine and disk failures, file blocks are replicated on multiple
machines as well.

#### MapReduce Job Execution
To create a MapReduce job, you need to implement two callback functions, the mapper and reducer, which behave as
follows:

*Mapper*

The mapper is called once for every input record, and its job is to extract the key and value from the input record. For
each input, it may generate any number of key-value pairs (including none). It does not keep any state from one input
record to the next, so each record is handled independently.

*Reducer*
The MapReduce framework takes the key-value pairs produced by the mappers, collects all the values belonging to the same
key, and calls the reducer with an iterator over that collection of values. The reducer can produce output records (such
as the number of occurrences of the same URL).

##### Distributed execution of MapReduce
MapReduce can parallelize a computation across many machines, without you having to write code to explicitly handle the
parallelism. The mapper and reducer only operate on one record at a time; they don’t need to know where their input is
coming from or their output is going to, so the framework can handle the complexities of moving data between machines.
Usually the mappers and reducers are implemented as functions in a conventional programming language (Java in Hadoop,
JavaScript in MongoDB and CouchDB).

The MapReduce scheduler tries to run each mapper on one of the machines that stores a replica of the input file,
provided that machine has enough spare RAM and CPU resources to run the map task. This principle is known as putting the
computation near the data: it saves copying the input file over the network, reducing network load and increasing
locality.

After copying the code to appropriate machines, MapReduce framework starts the map task and begins reading the input
file, passing one record at a time to the mapper callback. The output of the mapper consists of key-value pairs. The
reduce side of the computation is also partitioned. While the number of map tasks is determined by the number of input
file blocks, the number of reduce tasks is configured by the job author (it can be different from the number of map
tasks).

To ensure that all key-value pairs with the same key end up at the same reducer, the framework uses a hash of the key to
determine which reduce task should receive a particular key-value pair. The key-value pairs must be sorted, but the
dataset is likely too large to be sorted with a conventional sorting algorithm on a single machine. Instead, the sorting
is performed in stages. First, each map task partitions its output by reducer, based on the hash of the key. Each of
these partitions is written to a sorted file on the mapper’s local disk. Whenever a mapper finishes reading its input
file and writing its sorted output files, the MapReduce scheduler notifies the reducers that they can start fetching the
output files from that mapper. The reducers connect to each of the mappers and download the files of sorted key-value
pairs for their partition. The process of partitioning by reducer, sorting, and copying data partitions from mappers to
reducers is known as the shuffle.

The reduce task takes the files from the mappers and merges them together, preserving the sort order. Thus, if different
mappers produced records with the same key, they will be adjacent in the merged reducer input. The reducer is called
with a key and an iterator that incrementally scans over all records with the same key (which may in some cases not all
fit in memory). The reducer can use arbitrary logic to process these records, and can generate any number of output
records. These output records are written to a file on the distributed filesystem (usually, one copy on the local disk
of the machine running the reducer, with replicas on other machines).

##### MapReduce workflows
It is very common for MapReduce jobs to be chained together into workflows, such that the output of one job becomes the
input to the next job. The Hadoop MapReduce framework does not have any particular support for workflows, so this
chaining is done implicitly by directory name. Thus, each MapReduce is considered an independent job. The flow is like a
sequence of commands where each command’s output is written to a temporary file, and the next command reads from the
temporary file. This design has advantages and disadvantages.

To handle dependencies between job executions, various workflow schedulers for Hadoop have been developed. These
schedulers also have management features that are useful when maintaining a large collection of batch jobs. Workflows
consisting of 50 to 100 MapReduce jobs are common when building recommendation systems, and in a large organization,
many different teams may be running different jobs that read each other’s output. Tool support is important for managing
such complex dataflows.

#### Reduce-Side Joins and Grouping
We discussed joins in Chapter 2 in the context of data models and query languages, but we have not delved into how joins
are actually implemented.

In a database, if you execute a query that involves only a small number of records, the database will typically use an
index to quickly locate the records of interest. If the query involves joins, it may require multiple index lookups.
However, MapReduce has no concept of indexes—at least not in the usual sense.

When a MapReduce job is given a set of files as input, it reads the entire content of all of those files; a database
would call this operation a full table scan. If you only want to read a small number of records, a full table scan is
outrageously expensive compared to an index lookup. However, in analytic queries it is common to want to calculate
aggregates over a large number of records. In this case, scanning the entire input might be quite a reasonable thing to
do, especially if you can parallelize the processing across multiple machines.

When we talk about joins in the context of batch processing, we mean resolving all occurrences of some association
within a dataset. In order to achieve good throughput in a batch process, the computation must be (as much as possible)
local to one machine. Making random-access requests over the network for every record you want to process is too slow.
Moreover, querying a remote database would mean that the batch job becomes nondeterministic, because the data in the
remote database might change.

Thus, a better approach would be to take a copy of the second database and to put it in the same distributed filesystem
as the first data logs. Now you could use MapReduce to bring together all of the relevant records in the same place and
process them efficiently.

##### Sort-merge joins
Recall that the purpose of the mapper is to extract a key and value from each input record. This key would be the join
key ID: one set of mappers would go over the logs, while another set of mappers would go over the secondary database.
(extracting the join key ID as the key and any needed values from the secondary databas as the value).

When the MapReduce framework partitions the mapper output by key and then sorts the key-value pairs, the effect is that
all the events and the records from the secondary database with the same  ID become adjacent to each other in the
reducer input. The MapReduce job can even arrange the records to be sorted such that the reducer always sees the record
from the secondary database first, followed by the events in any order—this technique is known as a secondary sort. The
reducer can then perform the actual join logic easily: the reducer function is called once for every join ID.

Since the reducer processes all of the records for a particular ID in one go, it only needs to keep one record in memory
at any one time, and it never needs to make any requests over the network. This algorithm is known as a sort-merge join,
since mapper output is sorted by key, and the reducers then merge together the sorted lists of records from both sides
of the join.

##### Bringing related data together in the same place
In a sort-merge join, the mappers and the sorting process make sure that all the necessary data to perform the join
operation for a particular user ID is brought together in the same place: a single call to the reducer. Having lined up
all the required data in advance, the reducer can be a fairly simple, single-threaded piece of code that can churn
through records with high throughput and low memory overhead.

One way of looking at this architecture is that mappers “send messages” to the reducers. When a mapper emits a key-value
pair, the key acts like the destination address to which the value should be delivered. Even though the key is just an
arbitrary string (not an actual network address like an IP address and port number), it behaves like an address: all
key-value pairs with the same key will be delivered to the same destination (a call to the reducer).

Using the MapReduce programming model has separated the physical network communication aspects of the computation
(getting the data to the right machine) from the application logic (processing the data once you have it). This
separation contrasts with the typical use of databases, where a request to fetch data from a database often occurs
somewhere deep inside a piece of application code. Since MapReduce handles all network communication, it also
shields the application code from having to worry about partial failures, such as the crash of another node: MapReduce
transparently retries failed tasks without affecting the application logic.

##### GROUP BY
Besides joins, another common use of the “bringing related data to the same place” pattern is grouping records by some
key (as in the GROUP BY clause in SQL). All records with the same key form a group, and the next step is often to
perform some kind of aggregation within each group.

The simplest way of implementing such a grouping operation with MapReduce is to set up the mappers so that the key-value
pairs they produce use the desired grouping key. The partitioning and sorting process then brings together all the
records with the same key in the same reducer. Thus, grouping and joining look quite similar when implemented on top of
MapReduce.

#### Map-Side Joins
The join algorithms described in the last section perform the actual join logic in the reducers, and are hence known as
reduce-side joins. The mappers take the role of preparing the input data: extracting the key and value from each input
record, assigning the key-value pairs to a reducer partition, and sorting by key.

The reduce-side approach has the advantage that you do not need to make any assumptions about the input data: whatever
its properties and structure, the mappers can prepare the data to be ready for joining. However, the downside is that
all that sorting, copying to reducers, and merging of reducer inputs can be quite expensive. Depending on the available
memory buffers, data may be written to disk several times as it passes through the stages of MapReduce.

On the other hand, if you can make certain assumptions about your input data, it is possible to make joins faster by
using a so-called map-side join. This approach uses a cut-down MapReduce job in which there are no reducers and no
sorting. Instead, each mapper simply reads one input file block from the distributed filesystem and writes one output
file to the filesystem—that is all.

##### Broadcast hash joins
The simplest way of performing a map-side join applies in the case where a large dataset is joined with a small dataset.
In particular, the small dataset needs to be small enough that it can be loaded entirely into memory in each of the
mappers. There can still be several map tasks: one for each file block of the large input to the join. Each of these
mappers loads the small input entirely into memory.

This simple but effective algorithm is called a *broadcast hash join*: the word broadcast reflects the fact that each
mapper for a partition of the large input reads the entirety of the small input, and the word hash reflects its use of a
hash table.

Instead of loading the small join input into an in-memory hash table, an alternative is to store the small join input in
a read-only index on the local disk. The frequently used parts of this index will remain in the operating system’s page
cache, so this approach can provide random-access lookups almost as fast as an in-memory hash table, but without
actually requiring the dataset to fit in memory.


##### Partitioned hash joins
If the inputs to the map-side join are partitioned in the same way, then the hash join approach can be applied to each
partition independently. If the partitioning is done correctly, you can be sure that all the records you might want to
join are located in the same numbered partition, and so it is sufficient for each mapper to only read one partition from
each of the input datasets. This has the advantage that each mapper can load a smaller amount of data into its hash
table.

This approach only works if both of the join’s inputs have the same number of partitions, with records assigned to
partitions based on the same key and the same hash function. If the inputs are generated by prior MapReduce jobs that
already perform this grouping, then this can be a reasonable assumption to make.

##### Map-side merge joins
Another variant of a map-side join applies if the input datasets are not only partitioned in the same way, but also
sorted based on the same key. In this case, it does not matter whether the inputs are small enough to fit in memory,
because a mapper can perform the same merging operation that would normally be done by a reducer: reading both input
files incrementally, in order of ascending key, and matching records with the same key.

If a map-side merge join is possible, it probably means that prior MapReduce jobs brought the input datasets into this
partitioned and sorted form in the first place. In principle, this join could have been performed in the reduce stage of
the prior job. However, it may still be appropriate to perform the merge join in a separate map- only job, for example
if the partitioned and sorted datasets are also needed for other purposes besides this particular join.
##### MapReduce workflows with map-side joins
When the output of a MapReduce join is consumed by downstream jobs, the choice of map-side or reduce-side join affects
the structure of the output. The output of a reduce-side join is partitioned and sorted by the join key, whereas the
output of a map-side join is partitioned and sorted in the same way as the large input (since one map task is started
for each file block of the join’s large input, regardless of whether a partitioned or broadcast join is used).

As discussed, map-side joins also make more assumptions about the size, sorting, and partitioning of their input
datasets. Knowing about the physical layout of datasets in the distributed filesystem becomes important when optimizing
join strategies: it is not sufficient to just know the encoding format and the name of the directory in which the data
is stored; you must also know the number of partitions and the keys by which the data is partitioned and sorted.

#### The Output of Batch Workflows
We have talked a lot about the various algorithms for implementing workflows of MapReduce jobs, but we neglected an
important question: what is the result of all of that processing, once it is done? We previously talked about OLTP and
OLAP queries. Where does batch processing fit in? It is not transaction processing, nor is it analytics. It is closer to
analytics, in that a batch process typically scans over large portions of an input dataset. However, a workflow of
MapReduce jobs is not the same as a SQL query used for analytic purposes. The output of a batch process is often not a
report, but some other kind of structure. Here are two main use cases for MapReduce:

* Building search indexes: Google’s original use of MapReduce was to build indexes for its search engine
* Key-value stores as batch process output: machine learning systems such as classifiers and recommendation systems

##### Philosophy of batch process outputs
The handling of output from MapReduce jobs follows the same philosophy as the Unix philosophy. By treating
inputs as immutable and avoiding side effects (such as writing to external databases),
batch jobs not only achieve good performance but also become much easier to maintain:

* If you introduce a bug into the code and the output is wrong or corrupted, you can simply roll back to a previous
  version of the code and rerun the job, and the output will be correct again. Or, even simpler, you can keep the old
  output in a different directory and simply switch back to it. Databases with read-write transactions do not have this
  property: if you deploy buggy code that writes bad data to the database, then rolling back the code will do nothing to
  fix the data in the database. 
* As a consequence of this ease of rolling back, feature development can proceed more quickly than in an environment
  where mistakes could mean irreversible damage. This principle of minimizing irreversibility is beneficial for Agile
  software development.
* If a map or reduce task fails, the MapReduce framework automatically reschedules it and runs it again on the same
  input. If the failure is due to a bug in the code, it will keep crashing and eventually cause the job to fail after a
  few attempts; but if the failure is due to a transient issue, the fault is tolerated. This automatic retry is only
  safe because inputs are immutable and outputs from failed tasks are discarded by the MapReduce framework.
* The same set of files can be used as input for various different jobs, including monitoring jobs that calculate
  metrics and evaluate whether a job’s output has the expected characteristics.
* Like Unix tools, MapReduce jobs separate logic from wiring (configuring the input and output directories), which
  provides a separation of concerns and enables potential reuse of code: one team can focus on implementing a job that
  does one thing well, while other teams can decide where and when to run that job.

#### Comparing Hadoop to Distributed Databases
When the MapReduce paper was published, it was—in some sense—not at all new. All of the processing and parallel join
algorithms that we discussed in the last few sections had already been implemented in so-called massively parallel
processing (MPP) databases more than a decade previously.

The biggest difference is that MPP databases focus on parallel execution of analytic SQL queries on a cluster of
machines, while the combination of MapReduce and a distributed filesystem provides something much more like a
general-purpose operating system that can run arbitrary programs.

##### Diversity of storage
Databases require you to structure data according to a particular model (e.g., relational or documents), whereas files
in a distributed filesystem are just byte sequences, which can be written using any data model and encoding. To put it
bluntly, Hadoop opened up the possibility of indiscriminately dumping data into HDFS, and only later figuring out how to
process it further. By contrast, MPP databases typically require careful up-front modeling of the data and query
patterns before importing the data into the database’s proprietary storage format.

In practice, it appears that simply making data available quickly— even if it is in a quirky, difficult-to-use, raw
format—is often more valuable than trying to decide on the ideal data model up front. Indiscriminate data dumping shifts
the burden of interpreting the data: instead of forcing the producer of a dataset to bring it into a standardized
format, the interpretation of the data becomes the consumer’s problem.

Thus, Hadoop has often been used for implementing ETL processes: data from transaction processing systems is dumped into
the distributed filesystem in some raw form, and then MapReduce jobs are written to clean up that data, transform it
into a relational form, and import it into an MPP data warehouse for analytic purposes. Data modeling still happens, but
it is in a separate step, decoupled from the data collection. This decoupling is possible because a distributed
filesystem supports data encoded in any format.

##### Diversity of processing models
MapReduce gave engineers the ability to easily run their own code over large datasets. If you have HDFS and MapReduce,
you can build a SQL query execution engine on top of it. However, you can also write many other forms of batch processes
that do not lend themselves to being expressed as a SQL query.

Subsequently, people found that MapReduce was too limiting and performed too badly for some types of processing, so
various other processing models were developed on top of Hadoop.

##### Designing for frequent faults
When comparing MapReduce to MPP databases, two more differences in design approach stand out: the handling of faults and
the use of memory and disk. Batch processes are less sensitive to faults than online systems, because they do not
immediately affect users if they fail and they can always be run again.

MapReduce can tolerate the failure of a map or reduce task without it affecting the job as a whole by retrying work at
the granularity of an individual task. It is also very eager to write data to disk, partly for fault tolerance, and
partly on the assumption that the dataset will be too big to fit in memory anyway.

At Google, a MapReduce task that runs for an hour has an approximately 5% risk of being terminated to make space for a
higher-priority process. This rate is more than an order of magnitude higher than the rate of failures due to hardware
issues, machine reboot, or other reasons. At this rate of preemptions, if a job has 100 tasks that each run for 10
minutes, there is a risk greater than 50% that at least one task will be terminated before it is finished.

And this is why MapReduce is designed to tolerate frequent unexpected task termination: it’s not because the hardware is
particularly unreliable, it’s because the freedom to arbitrarily terminate processes enables better resource utilization
in a computing cluster.
