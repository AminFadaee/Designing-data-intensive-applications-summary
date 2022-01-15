# Designing Data Intensive Applications
## Partitioning
For very large datasets, or very high query throughput, that is not sufficient: we need to break the data up into
partitions, also known as sharding.

Normally, partitions are defined in such a way that each piece of data (each record, row, or document) belongs to
exactly one partition. In effect, each partition is a small database of its own, although the database may support
operations that touch multiple partitions at the same time. The main reason for wanting to partition data is
scalability. For queries that operate on a single partition, each node can independently execute the queries for its own
partition, so query throughput can be scaled by adding more nodes. 

In this chapter we will first look at different approaches for partitioning large datasets and observe how the indexing
of data interacts with partitioning. We’ll then talk about rebalancing. Finally, we’ll get an overview of how databases
route requests to the right partitions and execute queries.

### Partitioning and Replication
Partitioning is usually combined with replication so that copies of each partition are stored on multiple nodes. This
means that, even though each record belongs to exactly one partition, it may still be stored on several different nodes
for fault tolerance. Each node may be the leader for some partitions and a follower for other partitions. For the sake
of simplicity, we ignore replication in this chapter.

### Partitioning of Key-Value Data
Say you have a large amount of data, and you want to partition it. How do you decide which records to store on which
nodes?

Our goal with partitioning is to spread the data and the query load evenly across nodes. If the partitioning is unfair,
so that some partitions have more data or queries than others, we call it *skewed*. The presence of skew makes
partitioning much less effective. A partition with disproportionately high load is called a *hot spot*.

Simplest approach to deal with hot spots is to assign each record randomly to a partition but this way you have no way
of knowing which node it is on.

Obviously, we can do better.

#### Partitioning by Key Range
One way of partitioning is to assign a continuous range of keys (from some minimum to some maximum) to each partition.
The ranges of keys are not necessarily evenly spaced, because your data may not be evenly distributed. In order to
distribute the data evenly, the partition bound‐ aries need to adapt to the data.

However, the downside of key range partitioning is that certain access patterns can lead to hot spots. If the key is a
timestamp, then the partitions correspond to ranges of time—e.g., one partition per day. Unfortunately, because we write
data from the sensors to the database as the measurements happen, all the writes end up going to the same partition, so
that partition can be overloaded with writes while others sit idle.

#### Partitioning by Hash of Key
Once you have a suitable hash function for keys, you can assign each partition a range of hashes (rather than a range of
keys), and every key whose hash falls within a partition’s range will be stored in that partition. A good hash function
takes skewed data and makes it uniformly distributed. For partitioning purposes, the hash function need not be
cryptographically strong (like MD5). This technique is good at distributing keys fairly among the partitions. The
partition boundaries can be evenly spaced, or they can be chosen pseudorandomly.

Unfortunately however, by using the hash of the key for partitioning we lose a nice property of key-range partitioning:
the ability to do efficient range queries. Keys that were once adjacent are now scattered across all the partitions, so
their sort order is lost.

#### Skewed Workloads and Relieving Hot Spots
As discussed, hashing a key to determine its partition can help reduce hot spots. However, it can’t avoid them entirely:
in the extreme case where all reads and writes are for the same key, you still end up with all requests being routed to
the same parti‐ tion. This kind of workload is perhaps unusual, but not unheard of (celebrity problem).

Today, most data systems are not able to automatically compensate for such a highly skewed workload, so it’s the
responsibility of the application to reduce the skew. For example, if one key is known to be very hot, a simple
technique is to add a random number to the beginning or end of the key. For example, we can distribute writes between
100 different keys with the cost of having to read the data from all 100 keys and combine it.

### Partitioning and Secondary Indexes
If records are only ever accessed via their primary key, we can determine the partition from that key and use it to
route read and write requests to the partition responsible for that key. The situation becomes more complicated if
secondary indexes are involved. A secondary index usually doesn’t identify a record uniquely but rather is a way of
searching for occurrences of a particular value.

The problem with secondary indexes is that they don’t map neatly to partitions. There are two main approaches to
partitioning a database with secondary indexes: document-based partitioning and term-based partitioning.

#### Partitioning Secondary Indexes by Document
We can keep a secondary index for each partition, covering only the documents in that partition. It doesn’t care what
data is stored in other partitions. Whenever you need to write to the database you only need to deal with the partition
that contains the document ID that you are writing. For that reason, a document-partitioned index is also known as a
*local index*.

However, in this setting, unless you have done something special with the document IDs, you need to send the query to
*all* partitions, and combine all the results you get back. This approach to querying a partitioned database is
sometimes known as *scatter/gather*, and it can make read queries on secondary indexes quite expensive. Even if you
query the partitions in parallel, scatter/gather is prone to tail latency amplification. Nevertheless, it is widely
used.

#### Partitioning Secondary Indexes by Term
Rather than each partition having its own secondary index (a local index), we can
construct a global index that covers data in all partitions. However, we can’t just store
that index on one node, since it would likely become a bottleneck and defeat the purpose of partitioning. A global index must also be partitioned, but it can be partitioned
differently from the primary key index.

We call this kind of index term-partitioned, because the term we’re looking for determines the partition of the index.
For example, a term would be color:red, would be in the partition for letter *a* to *r*. The name term comes from
full-text indexes (a particular kind of secondary index), where the terms are all the words that occur in a document.

The advantage of a global (term-partitioned) index over a document-partitioned index is that it can make reads more
efficient. However, the downside of a global index is that writes are slower and more complicated, because a write to a
single document may now affect multiple partitions of the index.

In an ideal world, the index would always be up to date, and every document written to the database would immediately be
reflected in the index. However, in a term- partitioned index, that would require a distributed transaction across all
partitions affected by a write, which is not supported in all databases.

### Rebalancing Partitions
All of these changes call for data and requests to be moved from one node to another. The process of moving load from
one node in the cluster to another is called reba‐ lancing.

No matter which partitioning scheme is used, rebalancing is usually expected to meet some minimum requirements:
* After rebalancing, the load (data storage, read and write requests) should be shared fairly between the nodes in the
  cluster.
* While rebalancing is happening, the database should continue accepting reads and writes.
* No more data than necessary should be moved between nodes, to make rebalanc‐ ing fast and to minimize the network and
  disk I/O load.

#### Strategies for rebalancing
##### How not to do it: hash mod N
When partitioning by the hash of a key, it’s best to divide the possible hashes into ranges and assign each range to a
partition. Perhaps you wondered why we don’t just use mod. The problem with the mod N approach is that if the number of
nodes N changes, most of the keys will need to be moved from one node to another. Such frequent moves make rebalancing
excessively expensive. We need an approach that doesn’t move data around more than necessary.

##### Fixed number of partitions
Fairly simple solution: create many more partitions than there are nodes, and assign several partitions to each node.
Now, if a node is added to the cluster, the new node can steal a few partitions from every existing node until
partitions are fairly distributed once again.

Only entire partitions are moved between nodes. The number of partitions does not change, nor does the assignment of
keys to partitions. The only thing that changes is the assignment of partitions to nodes. This change of assignment is
not immediate it takes some time to transfer a large amount of data over the network—so the old assignment of partitions
is used for any reads and writes that happen while the transfer is in progress. In principle, you can even account for
mismatched hardware in your cluster: by assigning more partitions to nodes that are more powerful, you can force those
nodes to take a greater share of the load.

In this configuration, the number of partitions is usually fixed when the database is first set up and not changed
afterward. Although in principle it’s possible to split and merge partitions, however, fixed-partition databases choose
not to implement partition splitting. Thus, the number of partitions configured at the outset is the maximum number of
nodes you can have, so you need to choose it high enough to accommodate future growth. However, each partition also has
management overhead, so it’s counterproductive to choose too high a number.

In this setting, The best performance is achieved when the size of partitions is “just right,” neither too big nor too
small, which can be hard to achieve if the number of partitions is fixed but the dataset size varies.

##### Dynamic partitioning
For databases that use key range partitioning, a fixed number of partitions with fixed boundaries would be very
inconvenient: if you got the boundaries wrong, you could end up with all of the data in one partition and all of the
other partitions empty. Reconfiguring the partition boundaries manually would be very tedious.

For that reason, we can create partitions dynamically. When a partition grows to exceed a configured size, it is split
into two partitions so that approximately half of the data ends up on each side of the split. Conversely, if lots of
data is deleted and a partition shrinks below some threshold, it can be merged with an adjacent partition.

Each partition is assigned to one node, and each node can handle multiple partitions, like in the case of a fixed number
of partitions. After a large partition has been split, one of its two halves can be transferred to another node in order
to balance the load.

An advantage of dynamic partitioning is that the number of partitions adapts to the total data volume. If there is only
a small amount of data, a small number of partitions is sufficient, so overheads are small; if there is a huge amount of
data, the size of each individual partition is limited to a configurable maximum.

##### Partitioning proportionally to nodes
With dynamic partitioning, the number of partitions is proportional to the size of the dataset, since the splitting and
merging processes keep the size of each partition between some fixed minimum and maximum. On the other hand, with a
fixed number of partitions, the size of each partition is proportional to the size of the dataset. In both of these
cases, the number of partitions is independent of the number of nodes.

A third option, is to make the number of partitions proportional to the number of nodes—in other words, to have a fixed
number of partitions per node. In this case, the size of each partition grows proportionally to the dataset size while
the number of nodes remains unchanged, but when you increase the number of nodes, the partitions become smaller again.
Since a larger data volume generally requires a larger number of nodes to store, this approach also keeps the size of
each partition fairly stable.

#### Operations: Automatic or Manual Rebalancing
Fully automated rebalancing can be convenient, because there is less operational work to do for normal maintenance.
However, it can be unpredictable. Rebalancing is an expensive operation, because it requires rerouting requests and
moving a large amount of data from one node to another. Also, such automation can be dangerous in combination with
automatic failure detection. For that reason, it can be a good thing to have a human in the loop for rebalancing. It’s
slower than a fully automatic process, but it can help prevent operational surprises.

### Request Routing
When a client wants to make a request, how does it know which node to connect to? This is an instance of a more general
problem called service discovery, which isn’t limited to just databases. Any piece of software that is accessible over a
network has this problem, especially if it is aiming for high availability (running in a redundant configuration on
multiple machines). Many companies have written their own in-house service discovery tools, and many of these have been
released as open source.

On a high level, there are a few different approaches to this problem:
1. Allow clients to contact any node. If that node coincidentally owns the partition to which the request applies, it
   can handle the request directly; otherwise, it forwards the request to the appropriate node, receives the reply, and
   passes the reply along to the client.
2. Send all requests from clients to a routing tier first, which determines the node that should handle each request and
   forwards it accordingly. This routing tier does not itself handle any requests; it only acts as a partition-aware
   load balancer.
3. Require that clients be aware of the partitioning and the assignment of partitions to nodes. In this case, a client
   can connect directly to the appropriate node, without any intermediary.

In all cases, the key problem is: how does the component making the routing decision (which may be one of the nodes, or
the routing tier, or the client) learn about changes in the assignment of partitions to nodes?

Many distributed data systems rely on a separate coordination service such as Zoo‐Keeper to keep track of this cluster
metadata. Each node registers itself in ZooKeeper, and ZooKeeper maintains the authoritative mapping of partitions to
nodes. Other actors, such as the routing tier or the partitioning-aware client, can subscribe to this information in
ZooKeeper. Whenever a partition changes ownership, or a node is added or removed, ZooKeeper notifies the routing tier so
that it can keep its routing information up to date.

#### Parallel Query Execution
So far we have focused on very simple queries that read or write a single key. This is about the level of access
supported by most NoSQL distributed datastores.

However, massively parallel processing (MPP) relational database products, often used for analytics, are much more
sophisticated in the types of queries they support. A typical data warehouse query contains several join, filtering,
grouping, and aggregation operations. The MPP query optimizer breaks this complex query into a number of execution
stages and partitions, many of which can be executed in parallel on different nodes of the database cluster. Queries
that involve scanning over large parts of the dataset particularly benefit from such parallel execution.

Fast parallel execution of data warehouse queries is a specialized topic, and given the business importance of
analytics, it receives a lot of commercial interest. We will discuss some techniques for parallel query execution in
Chapter 10. For a more detailed overview of techniques used in parallel databases, please see the references.


