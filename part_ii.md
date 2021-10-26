# Designing Data Intensive Applications
## Part II: Distributed Data

In Part I of this book, we discussed aspects of data systems that apply when data is stored on a single machine. Now, in
Part II, we move up a level and ask: what happens if multiple machines are involved in storage and retrieval of data?

There are various reasons why you might want to distribute a database across multiple machines:

Scalability

If your data volume, read load, or write load grows bigger than a single machine can handle, you can potentially spread
the load across multiple machines.

Fault tolerance/high availability

If your application needs to continue working even if one machine (or several machines, or the network, or an entire
datacenter) goes down, you can use multiple machines to give you redundancy. When one fails, another one can take
over.

Latency

If you have users around the world, you might want to have servers at various locations worldwide so that each user can
be served from a datacenter that is geographically close to them. That avoids the users having to wait for network
pack‐ ets to travel halfway around the world.

### Scaling to Higher Load
If all you need is to scale to higher load, the simplest approach is to buy a more powerful machine (sometimes called
vertical scaling or scaling up). Many CPUs, many RAM chips, and many disks can be joined together under one operating
system, and a fast interconnect allows any CPU to access any part of the memory or disk. In this kind of shared-memory
architecture, all the components can be treated as a single machine.

#### Shared-Nothing Architectures
By contrast, shared-nothing architectures (sometimes called horizontal scaling or scaling out) have gained a lot of
popularity. In this approach, each machine or virtual machine running the database software is called a node. Each node
uses its CPUs, RAM, and disks independently. Any coordination between nodes is done at the software level, using a
conventional network.

No special hardware is required by a shared-nothing system, so you can use whatever machines have the best
price/performance ratio. You can potentially distribute data across multiple geographic regions, and thus reduce latency
for users and potentially be able to survive the loss of an entire datacenter. With cloud deployments of virtual
machines, you don’t need to be operating at Google scale: even for small companies, a multi-region distributed
architecture is now feasible.

In this part of the book, we focus on shared-nothing architectures—not because they are necessarily the best choice for
every use case, but rather because they require the most caution from you, the application developer. If your data is
distributed across multiple nodes, you need to be aware of the constraints and trade-offs that occur in such a
distributed system—the database cannot magically hide these from you.

#### Replication Versus Partitioning
There are two common ways data is distributed across multiple nodes:

Replication

Keeping a copy of the same data on several different nodes, potentially in different locations. Replication provides
redundancy: if some nodes are unavailable, the data can still be served from the remaining nodes. Replication can also
help improve performance. We discuss replication in Chapter 5.

Partitioning

Splitting a big database into smaller subsets called partitions so that different partitions can be assigned to
different nodes (also known as sharding). We discuss partitioning in Chapter 6.

With an understanding of those concepts, we can discuss the difficult trade-offs that you need to make in a distributed
system. We’ll discuss transactions in Chapter 7, as that will help you understand all the many things that can go wrong
in a data system, and what you can do about them. We’ll conclude this part of the book by discussing the fundamental
limitations of distributed systems in Chapters 8 and 9.

Later, in Part III of this book, we will discuss how you can take several (potentially distributed) datastores and
integrate them into a larger system, satisfying the needs of a complex application. But first, let’s talk about
distributed data.
