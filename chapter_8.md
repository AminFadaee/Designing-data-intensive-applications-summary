# Designing Data Intensive Applications
## Chapter 8: The Trouble with Distributed Systems
Even though we have talked a lot about faults, the last few chapters have still been too optimistic. The reality is even
darker. We will now turn our pessimism to the maximum and assume that anything that can go wrong will go wrong. In this
chapter, we will get a taste of the problems that arise in practice, and an understanding of the things we can and
cannot rely on.

### Faults and Partial Failure
An individual computer with good software is usually either fully functional or entirely broken, but not something in
between. When you are writing software that runs on several computers, connected by a network, the situation is
fundamentally different. In distributed systems, we are no longer operating in an idealized system model—we have no
choice but to confront the messy reality of the physical world. And in the physical world, a remarkably wide range of
things can go wrong.

In a distributed system, there may well be some parts of the system that are broken in some unpredictable way, even
though other parts of the system are working fine. This is known as a partial failure. The difficulty is that partial
failures are nondeterministic: if you try to do anything involving multiple nodes and the network, it may sometimes work
and sometimes unpredictably fail. As we shall see, you may not even know whether something succeeded or not, as the time
it takes for a message to travel across a network is also nondeterministic!

This nondeterminism and possibility of partial failures is what makes distributed systems hard to work with.

Even in smaller systems consisting of only a few nodes, it’s important to think about partial failure. In a small
system, it’s quite likely that most of the components are working correctly most of the time. However, sooner or later,
some part of the system will become faulty, and the software will have to somehow handle it. The fault handling must be
part of the software design, and you (as operator of the software) need to know what behavior to expect from the
software in the case of a fault.

It would be unwise to assume that faults are rare and simply hope for the best. It is important to consider a wide range
of possible faults—even fairly unlikely ones—and to artificially create such situations in your testing environment to
see what happens. In distributed systems, suspicion, pessimism, and paranoia pay off.

### Unreliable Networks
If you send a request and expect a response, many things could go wrong:

1. Your request may have been lost (perhaps someone unplugged a network cable).
2. Your request may be waiting in a queue and will be delivered later (perhaps the network or the recipient is
   overloaded).
3. The remote node may have failed (perhaps it crashed or it was powered down).
4. The remote node may have temporarily stopped responding (perhaps it is experiencing a long garbage collection
   pause), but it will start responding again later.
5. The remote node may have processed your request, but the response has been lost on the network (perhaps a network
   switch has been misconfigured).
6. The remote node may have processed your request, but the response has been delayed and will be delivered later
   (perhaps the network or your own machine is overloaded).

If you send a request to another node and don’t receive a response, it is impossible to tell why. The usual way of
handling this issue is a timeout: after some time you give up waiting and assume that the response is not going to
arrive. However, when a timeout occurs, you still don’t know whether the remote node got your request or not (and if the
request is still queued somewhere, it may still be delivered to the recipient, even if the sender has given up on it).

#### Network Faults in Practice
If the error handling of network faults is not defined and tested, arbitrarily bad things could happen: for example, the
cluster could become deadlocked and permanently unable to serve requests, even when the network recovers, or it could
even delete all of your data. If software is put in an unanticipated situation, it may do arbitrary unexpected things.

Handling network faults doesn’t necessarily mean tolerating them: if your network is normally fairly reliable, a valid
approach may be to simply show an error message to users while your network is experiencing problems. However, you do
need to know how your software reacts to network problems and ensure that the system can recover from them. It may make
sense to deliberately trigger network problems and test the system’s response.

#### Detecting Faults
Unfortunately, the uncertainty about the network makes it difficult to tell whether a node is working or not. In some
specific circumstances you might get some feedback to explicitly tell you that something is not working.

Rapid feedback about a remote node being down is useful, but you can’t count on it. Even if TCP acknowledges that a
packet was delivered, the application may have crashed before handling it. If you want to be sure that a request was
successful, you need a positive response from the application itself.

Conversely, if something has gone wrong, you may get an error response at some level of the stack, but in general you
have to assume that you will get no response at all. You can retry a few times (TCP retries transparently, but you may
also retry at the application level), wait for a timeout to elapse, and eventually declare the node dead if you don’t
hear back within the timeout.

#### Timeouts and Unbounded Delays
When a node is declared dead, its responsibilities need to be transferred to other nodes, which places additional load
on other nodes and the network. If the system is already struggling with high load, declaring nodes dead prematurely can
make the problem worse. In particular, it could happen that the node actually wasn’t dead but only slow to respond due
to overload; transferring its load to other nodes can cause a cascading failure (in the extreme case, all nodes declare
each other dead, and everything stops working).

Asynchronous networks have unbounded delays (that is, they try to deliver packets as quickly as possible, but there is
no upper limit on the time it may take for a packet to arrive), and most server implementations cannot guarantee that
they can handle requests within some maximum time (see “Response time guarantees” on page 298). For failure detection,
it’s not sufficient for the system to be fast most of the time: if your timeout is low, it only takes a transient spike
in round-trip times to throw the system off-balance.

##### Network congestion and queueing
When driving a car, travel times on road networks often vary most due to traffic congestion. Similarly, the
variability of packet delays on computer networks is most often due to queueing.

Moreover, TCP considers a packet to be lost if it is not acknowledged within some timeout (which is calculated from
observed round-trip times), and lost packets are automatically retransmitted. Although the application does not see the
packet loss and retransmission, it does see the resulting delay (waiting for the timeout to expire, and then waiting for
the retransmitted packet to be acknowledged).

In public clouds and multi-tenant datacenters, resources are shared among many customers: the network links and
switches, and even each machine’s network interface and CPUs (when running on virtual machines), are shared. Batch
workloads such as MapReduce (see Chapter 10) can easily saturate network links. As you have no control over or insight
into other customers’ usage of the shared resources, network delays can be highly variable if someone near you (a
noisy neighbor) is using a lot of resources.

In such environments, you can only choose timeouts experimentally: measure the distribution of network round-trip times
over an extended period, and over many machines, to determine the expected variability of delays. Then, taking into
account your application’s characteristics, you can determine an appropriate trade-off between failure detection delay
and risk of premature timeouts.

Even better, rather than using configured constant timeouts, systems can continually measure response times and their
variability (jitter), and automatically adjust timeouts according to the observed response time distribution.
#### Synchronous Versus Asynchronous Networks
Distributed systems would be a lot simpler if we could rely on the network to deliver packets with some fixed maximum
delay, and not to drop packets. Why can’t we solve this at the hardware level and make the network reliable so that the
software doesn’t need to worry about it?

When you make a call over the telephone network, it establishes a circuit: a fixed, guaranteed amount of bandwidth is
allocated for the call, along the entire route between the two callers. This circuit remains in place until the call
ends. For example, an ISDN network runs at a fixed rate of 4,000 frames per second. When a call is established, it
is allocated 16 bits of space within each frame (in each direction). Thus, for the duration of the call, each side is
guaranteed to be able to send exactly 16 bits of audio data every 250 microseconds.

This kind of network is synchronous: even as data passes through several routers, it does not suffer from queueing,
because the 16 bits of space for the call have already been reserved in the next hop of the network. And because there
is no queueing, the maximum end-to-end latency of the network is fixed. We call this a bounded delay.

##### Can we not simply make network delays predictable?
Note that a circuit in a telephone network is very different from a TCP connection: a circuit is a fixed amount of
reserved bandwidth which nobody else can use while the circuit is established, whereas the packets of a TCP connection
opportunistically use whatever network bandwidth is available. You can give TCP a variable-sized block of data (e.g., an
email or a web page), and it will try to transfer it in the shortest time possible. While a TCP connection is idle, it
doesn’t use any bandwidth.

If datacenter networks and the internet were circuit-switched networks, it would be possible to establish a guaranteed
maximum round-trip time when a circuit was set up. However, they are not: Ethernet and IP are packet-switched protocols,
which suffer from queueing and thus unbounded delays in the network. These protocols do not have the concept of a
circuit.

Why do datacenter networks and the internet use packet switching? The answer is that they are optimized for bursty
traffic. A circuit is good for an audio or video call, which needs to transfer a fairly constant number of bits per
second for the duration of the call. On the other hand, requesting a web page, sending an email, or transferring a file
doesn’t have any particular bandwidth requirement—we just want it to complete as quickly as possible.

If you wanted to transfer a file over a circuit, you would have to guess a bandwidth allocation. If you guess too low,
the transfer is unnecessarily slow, leaving network capacity unused. If you guess too high, the circuit cannot be set up
(because the network cannot allow a circuit to be created if its bandwidth allocation cannot be guaranteed). Thus, using
circuits for bursty data transfers wastes network capacity and makes transfers unnecessarily slow. By contrast, TCP
dynamically adapts the rate of data transfer to the available network capacity.

Currently deployed technology does not allow us to make any guarantees about delays or reliability of the network: we
have to assume that network congestion, queueing, and unbounded delays will happen. Consequently, there’s no “correct”
value for timeouts—they need to be determined experimentally.

### Unreliable Clocks
In a distributed system, time is a tricky business, because communication is not instantaneous: it takes time for a
message to travel across the network from one machine to another. The time when a message is received is always later
than the time when it is sent, but due to variable delays in the network, we don’t know how much later. This fact
sometimes makes it difficult to determine the order in which things happened when multiple machines are involved.

Moreover, each machine on the network has its own clock, which is an actual hardware device: usually a quartz crystal
oscillator. These devices are not perfectly accurate, so each machine has its own notion of time, which may be slightly
faster or slower than on other machines. It is possible to synchronize clocks to some degree: the most commonly used
mechanism is the Network Time Protocol (NTP), which allows the computer clock to be adjusted according to the time
reported by a group of servers. The servers in turn get their time from a more accurate time source, such as a GPS
receiver.

#### Monotonic Versus Time-of-Day Clocks Modern computers have at least two different kinds of clocks: a time-of-day
clock and a monotonic clock. Although they both measure time, it is important to distinguish the two, since they serve
different purposes.

##### Time-of-day clocks
A time-of-day clock does what you intuitively expect of a clock: it returns the current
date and time according to some calendar (also known as wall-clock time).

Time-of-day clocks also have various oddities, as described in the next section. In particular, if the local clock is
too far ahead of the NTP server, it may be forcibly reset and appear to jump back to a previous point in time. These
jumps, as well as the fact that they often ignore leap seconds, make time-of-day clocks unsuitable for measuring elapsed
time. Time-of-day clocks have also historically had quite a coarse-grained resolution, e.g., moving forward in steps of
10 ms on older Windows systems. On recent systems, this is less of a problem.

##### Monotonic clocks
A monotonic clock is suitable for measuring a duration (time interval), such as a timeout or a service’s response time.
The name comes from the fact that they are guaranteed to always move forward (whereas a time-of-day clock may jump back
in time).

You can check the value of the monotonic clock at one point in time, do something, and then check the clock again at a
later time. The difference between the two values tells you how much time elapsed between the two checks. However, the
absolute value of the clock is meaningless: it might be the number of nanoseconds since the computer was started, or
something similarly arbitrary. In particular, it makes no sense to compare monotonic clock values from two different
computers, because they don’t mean the same thing.

In a distributed system, using a monotonic clock for measuring elapsed time (e.g., timeouts) is usually fine, because it
doesn’t assume any synchronization between different nodes’ clocks and is not sensitive to slight inaccuracies of
measurement. 

#### Relying on Synchronized Clocks
If you use software that requires synchronized clocks, it is essential that you
also carefully monitor the clock offsets between all the machines. Any node whose
clock drifts too far from the others should be declared dead and removed from the
cluster. Such monitoring ensures that you notice the broken clocks before they can
cause too much damage.

##### Timestamps for ordering events
There are fundamental problems with Last write wins (LWW) strategy:

* Database writes can mysteriously disappear: a node with a lagging clock is unable to overwrite values previously
  written by a node with a fast clock until the clock skew between the nodes has elapsed. This scenario can cause
  arbitrary amounts of data to be silently dropped without any error being reported to the application.
* LWW cannot distinguish between writes that occurred sequentially in quick succession and writes that were truly
  concurrent (neither writer was aware of the other). Additional causality tracking mechanisms, such as version vectors,
  are needed in order to prevent violations of causality.
* It is possible for two nodes to independently generate writes with the same timestamp, especially when the clock only
  has millisecond resolution. An additional tiebreaker value (which can simply be a large random number) is required to
  resolve such conflicts, but this approach can also lead to violations of causality.

Thus, even though it is tempting to resolve conflicts by keeping the most “recent” value and discarding others, it’s
important to be aware that the definition of “recent” depends on a local time-of-day clock, which may well be incorrect.

Logical clocks, which are based on incrementing counters rather than an oscillating quartz crystal, are a safer
alternative for ordering events. Logical clocks do not measure the time of day or the number of seconds elapsed, only
the relative ordering of events (whether one event happened before or after another). In contrast, time-of-day and
monotonic clocks, which measure actual elapsed time, are also known as physical clocks.

##### Clock readings have a confidence interval
You may be able to read a machine’s time-of-day clock with microsecond or even nanosecond resolution. But even if you
can get such a fine-grained measurement, that doesn’t mean the value is actually accurate to such precision. In fact, it
most likely is not—as mentioned previously, the drift in an imprecise quartz clock can easily be several milliseconds,
even if you synchronize with an NTP server on the local network every minute. With an NTP server on the public internet,
the best possible accuracy is probably to the tens of milliseconds, and the error may easily spike to over 100 ms when
there is network congestion.

Thus, it doesn’t make sense to think of a clock reading as a point in time—it is more like a range of times, within a
confidence interval: for example, a system may be 95% confident that the time now is between 10.3 and 10.5 seconds past
the minute, but it doesn’t know any more precisely than that. If we only know the time +/– 100 ms, the microsecond
digits in the timestamp are essentially meaningless.

An interesting exception is Google’s TrueTime API in Spanner, which explicitly reports the confidence interval on the
local clock. When you ask it for the current time, you get back two values: [earliest, latest], which are the earliest
possible and the latest possible timestamp. Based on its uncertainty calculations, the clock knows that the actual
current time is somewhere within that interval. The width of the interval depends, among other things, on how long it
has been since the local quartz clock was last synchronized with a more accurate clock source.

##### Synchronized clocks for global snapshots
The most common implementation of snapshot isolation requires a monotonically increasing transaction ID. If a write
happened later than the snapshot (i.e., the write has a greater transaction ID than the snapshot), that write is
invisible to the snapshot transaction. On a single-node database, a simple counter is sufficient for generating
transaction IDs.

However, when a database is distributed across many machines, potentially in multiple datacenters, a global,
monotonically increasing transaction ID (across all partitions) is difficult to generate, because it requires
coordination. The transaction ID must reflect causality: if transaction B reads a value that was written by transaction
A, then B must have a higher transaction ID than A—otherwise, the snapshot would not be consistent. With lots of small,
rapid transactions, creating transaction IDs in a distributed system becomes an untenable bottleneck.

Can we use the timestamps from synchronized time-of-day clocks as transaction IDs? If we could get the synchronization
good enough, they would have the right properties: later transactions have a higher timestamp. The problem, of course,
is the uncertainty about clock accuracy.

Spanner implements snapshot isolation across datacenters in this way. It uses the clock’s confidence interval as
reported by the TrueTime API, and is based on the following observation: if you have two confidence intervals, each
consisting of an earliest and latest possible timestamp, and those two intervals do not overlap, then B definitely
happened after A—there can be no doubt. Only if the intervals overlap are we unsure in which order A and B happened.

In order to ensure that transaction timestamps reflect causality, Spanner deliberately waits for the length of the
confidence interval before committing a read-write transaction. By doing so, it ensures that any transaction that may
read the data is at a sufficiently later time, so their confidence intervals do not overlap. In order to keep the wait
time as short as possible, Spanner needs to keep the clock uncertainty as small as possible; for this purpose, Google
deploys a GPS receiver or atomic clock in each datacenter, allowing clocks to be synchronized to within about 7ms.

Using clock synchronization for distributed transaction semantics is an area of active research. These ideas are
interesting, but they have not yet been implemented in mainstream databases outside of Google.

#### Process Pauses
Let’s consider another example of dangerous clock use in a distributed system. Say you have a database with a single
leader per partition. Only the leader is allowed to accept writes. How does a node know that it is still leader (that it
hasn’t been declared dead by the others), and that it may safely accept writes?

One option is for the leader to obtain a lease from the other nodes, which is similar to a lock with a timeout. Only one
node can hold the lease at any one time—thus, when a node obtains a lease, it knows that it is the leader for some
amount of time, until the lease expires. In order to remain leader, the node must periodically renew the lease before it
expires. If the node fails, it stops renewing the lease, so another node can take over when it expires.

You can imagine the request-handling loop looking something like this:

```java
while (true) {
	request = getIncomingRequest();
	// Ensure that the lease always has at least 10 seconds remaining
	if (lease.expiryTimeMillis - System.currentTimeMillis() < 10000) {
		lease = lease.renew();
	}
	if (lease.isValid()) {
		process(request);
	}
}
```

What’s wrong with this code? Firstly, it’s relying on synchronized clocks: the expiry time on the lease is set by a
different machine (where the expiry may be calculated as the current time plus 30 seconds, for example), and it’s being
compared to the local system clock. If the clocks are out of sync by more than a few seconds, this code will start doing
strange things.

Secondly, even if we change the protocol to only use the local monotonic clock, there is another problem: the code
assumes that very little time passes between the point that it checks the time (System.currentTimeMillis()) and the time
when the request is processed (process(request)). Normally this code runs very quickly, so the 10 second buffer is more
than enough to ensure that the lease doesn’t expire in the middle of processing a request.

However, what if there is an unexpected pause in the execution of the program? For example, imagine the thread stops for
15 seconds around the line lease.isValid() before finally continuing. In that case, it’s likely that the lease will have
expired by the time the request is processed, and another node has already taken over as leader. However, there is
nothing to tell this thread that it was paused for so long, so this code won’t notice that the lease has expired until
the next iteration of the loop—by which time it may have already done something unsafe by processing the request.

Is it crazy to assume that a thread might be paused for so long? Unfortunately not. There are various reasons why this
could happen.

A node in a distributed system must assume that its execution can be paused for a significant length of time at any
point, even in the middle of a function. During the pause, the rest of the world keeps moving and may even declare the
paused node dead because it’s not responding. Eventually, the paused node may continue running, without even noticing
that it was asleep until it checks its clock sometime later.

### Knowledge, Truth and Lies
Although it is possible to make software well behaved in an unreliable system model, it is not straightforward to do so.
In the rest of this chapter we will further explore the notions of knowledge and truth in distributed systems, which
will help us think about the kinds of assumptions we can make and the guarantees we may want to provide. In Chapter 9 we
will proceed to look at some examples of distributed systems, algorithms that provide particular guarantees under
particular assumptions.

#### The Truth Is Defined by the Majority
A node cannot necessarily trust its own judgment of a situation. A distributed system cannot exclusively rely on a
single node, because a node may fail at any time, potentially leaving the system stuck and unable to recover. Instead,
many distributed algorithms rely on a quorum, that is, voting among the nodes: decisions require some minimum number of
votes from several nodes in order to reduce the dependence on any one particular node.

That includes decisions about declaring nodes dead. If a quorum of nodes declares another node dead, then it must be
considered dead, even if that node still very much feels alive. The individual node must abide by the quorum decision
and step down. Most commonly, the quorum is an absolute majority of more than half the nodes (although other kinds of
quorums are possible). A majority quorum allows the system to continue working if individual nodes have failed (with
three nodes, one failure can be tolerated; with five nodes, two failures can be tolerated). However, it is still safe,
because there can only be only one majority in the system—there cannot be two majorities with conflicting decisions at
the same time. We will discuss the use of quorums in more detail when we get to consensus algorithms in Chapter 9.

##### The leader and the lock
Frequently, a system requires there to be only one of some thing. For example:

* Only one node is allowed to be the leader for a database partition, to avoid split brain.
* Only one transaction or client is allowed to hold the lock for a particular resource or object, to prevent
  concurrently writing to it and corrupting it.
* Only one user is allowed to register a particular username, because a username must uniquely identify a user.

Implementing this in a distributed system requires care: even if a node believes that it is “the chosen one”, that
doesn’t necessarily mean a quorum of nodes agrees! A node may have formerly been the leader, but if the other nodes
declared it dead in the meantime, it may have been demoted and another leader may have already been elected.

If a node continues acting as the chosen one, even though the majority of nodes have declared it dead, it could cause
problems in a system that is not carefully designed. Such a node could send messages to other nodes in its
self-appointed capacity, and if other nodes believe it, the system as a whole may do something incorrect.

As discussed in the previous problem, if the client holding the lease is paused for too long, its lease expires. Another
client can obtain a lease for the same file, and start writing to the file. When the paused client comes back, it
believes (incorrectly) that it still has a valid lease and proceeds to also write to the file. As a result, the clients’
writes clash and corrupt the file.

##### Fencing tokens
When using a lock or lease to protect access to some resource, we need to ensure that a node that is under a false
belief of being “the chosen one” cannot disrupt the rest of the system. A fairly simple technique that achieves this
goal is called fencing.

Let’s assume that every time the lock server grants a lock or lease, it also returns a fencing token, which is a number
that increases every time a lock is granted (e.g., incremented by the lock service). We can then require that every time
a client sends a write request to the storage service, it must include its current fencing token.

Note that this mechanism requires the resource itself to take an active role in checking tokens by rejecting any writes
with an older token than one that has already been processed—it is not sufficient to rely on clients checking their lock
status themselves. For resources that do not explicitly support fencing tokens, you might still be able work around the
limitation (for example, in the case of a file storage service you could include the fencing token in the filename).
However, some kind of check is necessary to avoid processing requests outside of the lock’s protection.

Checking a token on the server side may seem like a downside, but it is arguably a good thing: it is unwise for a
service to assume that its clients will always be well behaved, because the clients are often run by people whose
priorities are very different from the priorities of the people running the service. Thus, it is a good idea for any
service to protect itself from accidentally abusive clients.

#### Byzantine Faults
Fencing tokens can detect and block a node that is inadvertently acting in error (e.g., because it hasn’t yet found out
that its lease has expired). However, if the node deliberately wanted to subvert the system’s guarantees, it could
easily do so by sending messages with a fake fencing token.

In this book we assume that nodes are unreliable but honest: they may be slow or never respond (due to a fault), and
their state may be outdated, but we assume that if a node does respond, it is telling the “truth”: to the best of its
knowledge, it is playing by the rules of the protocol.

Distributed systems problems become much harder if there is a risk that nodes may “lie”—for example, if a node may claim
to have received a particular message when in fact it didn’t. Such behavior is known as a Byzantine fault, and the
problem of reaching consensus in this untrusting environment is known as the Byzantine Generals Problem.

A system is Byzantine fault-tolerant if it continues to operate correctly even if some of the nodes are malfunctioning
and not obeying the protocol, or if malicious attackers are interfering with the network. This concern is relevant in
certain specific circumstances.

However, in the kinds of systems we discuss in this book, we can usually safely assume that there are no Byzantine
faults. In your datacenter, all the nodes are controlled by your organization (so they can hopefully be trusted) and
radiation levels are low enough that memory corruption is not a major problem. Protocols for making systems Byzantine
fault-tolerant are quite complicated, and fault-tolerant embedded systems rely on support from the hardware level. In
most server-side data systems, the cost of deploying Byzantine fault-tolerant solutions makes them impractical.

Web applications do need to expect arbitrary and malicious behavior of clients that are under end-user control, such as
web browsers. This is why input validation, sanitization, and output escaping are so important: to prevent SQL injection
and cross- site scripting, for example. However, we typically don’t use Byzantine fault-tolerant protocols here, but
simply make the server the authority on deciding what client behavior is and isn’t allowed. In peer-to-peer networks,
where there is no such central authority, Byzantine fault tolerance is more relevant.

A bug in the software could be regarded as a Byzantine fault, but if you deploy the same software to all nodes, then a
Byzantine fault-tolerant algorithm cannot save you. Most Byzantine fault-tolerant algorithms require a supermajority of
more than two- thirds of the nodes to be functioning correctly (i.e., if you have four nodes, at most one may
malfunction). To use this approach against bugs, you would have to have four independent implementations of the same
software and hope that a bug only appears in one of the four implementations.

Similarly, it would be appealing if a protocol could protect us from vulnerabilities, security compromises, and
malicious attacks. Unfortunately, this is not realistic either: in most systems, if an attacker can compromise one node,
they can probably compromise all of them, because they are probably running the same software. Thus, traditional
mechanisms (authentication, access control, encryption, firewalls, and so on) continue to be the main protection against
attackers.

#### System Model and Reality
Many algorithms have been designed to solve distributed systems problems—for example, we will examine solutions for the
consensus problem in Chapter 9. In order to be useful, these algorithms need to tolerate the various faults of
distributed systems that we discussed in this chapter.

Algorithms need to be written in a way that does not depend too heavily on the details of the hardware and software
configuration on which they are run. This in turn requires that we somehow formalize the kinds of faults that we expect
to happen in a system. We do this by defining a system model, which is an abstraction that describes what things an
algorithm may assume.

With regard to timing assumptions, three system models are in common use:

*Synchronous model*

The synchronous model assumes bounded network delay, bounded process pauses, and bounded clock error. This does not
imply exactly synchronized clocks or zero network delay; it just means you know that network delay, pauses, and clock
drift will never exceed some fixed upper bound. The synchronous model is not a realistic model of most practical
systems, because (as discussed in this chapter) unbounded delays and pauses do occur.

*Partially synchronous model*

Partial synchrony means that a system behaves like a synchronous system most of the time, but it sometimes exceeds the
bounds for network delay, process pauses, and clock drift. This is a realistic model of many systems: most of the time,
networks and processes are quite well behaved—otherwise we would never be able to get anything done—but we have to
reckon with the fact that any timing assumptions may be shattered occasionally. When this happens, network delay,
pauses, and clock error may become arbitrarily large.

*Asynchronous model*

In this model, an algorithm is not allowed to make any timing assumptions—in fact, it does not even have a clock (so it
cannot use timeouts). Some algorithms can be designed for the asynchronous model, but it is very restrictive.

Moreover, besides timing issues, we have to consider node failures. The three most common system models for nodes are:

*Crash-stop faults*

In the crash-stop model, an algorithm may assume that a node can fail in only one way, namely by crashing. This means
that the node may suddenly stop responding at any moment, and thereafter that node is gone forever—it never comes back.

*Crash-recovery faults*

We assume that nodes may crash at any moment, and perhaps start responding again after some unknown time. In the
crash-recovery model, nodes are assumed to have stable storage (i.e., nonvolatile disk storage) that is preserved across
crashes, while the in-memory state is assumed to be lost.

*Byzantine (arbitrary) faults*

Nodes may do absolutely anything, including trying to trick and deceive other nodes, as described in the last section.

For modeling real systems, the partially synchronous model with crash-recovery faults is generally the most useful
model. But how do distributed algorithms cope with that model?

##### Correctness of an algorithm
To define what it means for an algorithm to be correct, we can describe its properties. For example, the output of a
sorting algorithm has the property that for any two distinct elements of the output list, the element further to the
left is smaller than the element further to the right. That is simply a formal way of defining what it means for a list
to be sorted.

##### Safety and liveness
Safety is often informally defined as nothing bad happens, and liveness as something good eventually happens (e.g.
eventual consistency). However, it’s best to not read too much into those informal definitions, because the meaning of
good and bad is subjective. The actual definitions of safety and liveness are precise and mathematical:

* If a safety property is violated, we can point at a particular point in time at which it was broken (for example, if
  the uniqueness property was violated, we can identify the particular operation in which a duplicate fencing token was
  returned). After a safety property has been violated, the violation cannot be undone—the damage is already done.

* A liveness property works the other way round: it may not hold at some point in time (for example, a node may have
  sent a request but not yet received a response), but there is always hope that it may be satisfied in the future
  (namely by receiving a response).

An advantage of distinguishing between safety and liveness properties is that it helps us deal with difficult system
models. For distributed algorithms, it is common to require that safety properties always hold, in all possible
situations of a system model. That is, even if all nodes crash, or the entire network fails, the algorithm must
nevertheless ensure that it does not return a wrong result (i.e., that the safety properties remain satisfied).

However, with liveness properties we are allowed to make caveats: for example, we could say that a request needs to
receive a response only if a majority of nodes have not crashed, and only if the network eventually recovers from an
outage. The definition of the partially synchronous model requires that eventually the system returns to a synchronous
state—that is, any period of network interruption lasts only for a finite duration and is then repaired.

##### Mapping system models to the real world
Safety and liveness properties and system models are very useful for reasoning about the correctness of a distributed
algorithm. However, when implementing an algorithm in practice, the messy facts of reality come back to bite you
again, and it becomes clear that the system model is a simplified abstraction of reality.

For example, algorithms in the crash-recovery model generally assume that data in stable storage survives crashes.
However, what happens if the data on disk is corrupted, or the data is wiped out due to hardware error or
misconfiguration? What happens if a server has a firmware bug and fails to recognize its hard drives on reboot, even
though the drives are correctly attached to the server?

Quorum algorithms rely on a node remembering the data that it claims to have stored. If a node may suffer from amnesia
and forget previously stored data, that breaks the quorum condition, and thus breaks the correctness of the algorithm.
Perhaps a new system model is needed, in which we assume that stable storage mostly survives crashes, but may sometimes
be lost. But that model then becomes harder to reason about.

The theoretical description of an algorithm can declare that certain things are simply assumed not to happen—and in
non-Byzantine systems, we do have to make some assumptions about faults that can and cannot happen. However, a real
implementation may still have to include code to handle the case where something happens that was assumed to be
impossible, even if that handling boils down to `printf("Sucks to be you")` and `exit(666)`—i.e., letting a human
operator clean up the mess.

That is not to say that theoretical, abstract system models are worthless—quite the opposite. They are incredibly
helpful for distilling down the complexity of real systems to a manageable set of faults that we can reason about, so
that we can understand the problem and try to solve it systematically. We can prove algorithms correct by showing that
their properties always hold in some system model.

Proving an algorithm correct does not mean its implementation on a real system will necessarily always behave correctly.
But it’s a very good first step, because the theo‐ retical analysis can uncover problems in an algorithm that might
remain hidden for a long time in a real system, and that only come to bite you when your assumptions (e.g., about
timing) are defeated due to unusual circumstances. Theoretical analysis and empirical testing are equally important.

