# Desinging Data Intensive Applications
## Chapter 1: Reliable, Scalable and Maintainable Applications

### Readability
The system should continue to work correctly (performing the correct function at the desired level of performance)
even in the face of adversity (hardware or software faults, and even human error). This roughly translates to 
*continuing to work correctly, even when things go wrong*. Some examples of this include:

* The application performs the function that the user expected.
* It can tolerate the user making mistakes or using the software in unexpected ways.
* Its performance is good enough for the required use cases, under the expected load and data volume.
* The system prevents any unauthorised access and abuse.

Fault is usually defined as one component of the system deviating from its spec, whereas a failure is when the system
as a whole stops providing the required service to the user. It is impossible to reduce the probability of a fault to
zero; therfore it is usually best to design fault-tolerance mechanisms that prevent faults from causing failures.

Systems that anticipate faults and can cope with them are called fault-tolerant or resilient. Many critical bugs are
actually due to poor error handling; by deliberately inducing faults, you ensure that the fault-tolerance machinery
is continually exercised and tested, which can increase your confidence that faults will be handled correctly when
they occur naturally (example: Netflix Chaos Monkey).

#### Hardware Faults
Our first response is usually to add redundancy to the individual hardware components in order to reduce the failure
rate of the system, but there is a move toward systems that can tolerate the loss of entire machines, by using software
fault-tolerance techniques in preference or in addition to hardware redundancy. Such systems also have operational
advantages: a single-server system requires planned downtime if you need to reboot the machine (to apply operating 
system security patches, for example), whereas a system that can tolerate machine failure can be patched one node at a
time, without downtime of the entire system (a rolling upgrade).

#### Software Errors
Systematic errors are harder to anticipate than hardware faults, and because they are correlated across nodes, they
tend to cause many more system failures. The bugs that cause these kinds of software faults often lie dormant for a
long time until they are triggered by an unusual set of circumstances. In those circumstances, it is revealed that the
software is making some kind of assumption about its environment—and while that assumption is usually true, it
eventually stops being true for some reason. Some examples of systematic error are as follows:

* A software bug that causes every instance of an application server to crash when given a particular bad input.
For example, consider the leap second on June 30, 2012, that caused many applications to hang simultaneously due to a
bug in the Linux kernel.
* A runaway process that uses up some shared resource—CPU time, memory, disk space, or network bandwidth.
* A service that the system depends on that slows down, becomes unresponsive, or starts returning corrupted responses.
* Cascading failures, where a small fault in one component triggers a fault in another component, which in turn 
triggers further faults.

#### Human Errors
How do we make our systems reliable, in spite of unreliable humans?

* Design systems in a way that minimizes opportunities for error. Well-designed abstractions, APIs, and admin 
interfaces make it easy to do “the right thing” and discourage “the wrong thing.”
* Decouple the places where people make the most mistakes from the places where they can cause failures. In particular,
provide fully featured non-production sandbox environments where people can explore and experiment safely.
* Test thoroughly at all levels, from unit tests to whole-system integration tests and manual tests.
* Allow quick and easy recovery from human errors, to minimize the impact in the case of a failure.
* Set up detailed and clear monitoring.
* Implement good management practices and training.

### Scalability
As the system grows (in data volumes, traffic volumes or complexity), there should be reasonable ways of dealing with
that growth. Discussing scalability means considering questions like “If the system grows in a particular way, what are
our options for coping with the growth?” and “How can we add computing resources to handle the additional load?”.

#### Describing Load
Load can be described with a few numbers which we call *load parameters*. The best choice of parameters depends on the
architecture of your system: it may be requests per second to a web server, the ratio of reads to writes in a database,
the number of simultaneously active users in a chat room, the hit rate on a cache, or something else.
Perhaps the average case is what matters for you, or perhaps your bottleneck is dominated by a small number of extreme
cases.

##### Twitter Example
Two of Twitter’s main operations are:

* **Post tweet**: A user can publish a new message to their followers (4.6k requests/sec on average, over 12k requests/sec at peak).
* **Home timeline**: A user can view tweets posted by the people they follow (300k requests/sec).

Each user follows many people, and each user is followed by many people. There are broadly two ways of implementing 
these two operations:

1. Posting a tweet simply inserts the new tweet into a global collection of tweets. When a user requests their home 
timeline, look up all the people they follow, find all the tweets for each of those users, and merge them.
1. Maintain a cache for each user’s home timeline—like a mailbox of tweets for each recipient user. When a user posts
a tweet, look up all the people who follow that user, and insert the new tweet into each of their home timeline caches.
The request to read the home timeline is then cheap, because its result has been computed ahead of time.

The first approach struggles to keep up with the load of home timeline queries, so the company switched to approach 2.
It’s preferable to do more work at write time and less at read time.

Currently Twitter is moving to a hybrid of both approaches. Most users’ tweets continue to be fanned out to home 
timelines at the time when they are posted, but a small number of users with a very large number of followers 
(i.e., celebrities) are excepted from this fan-out. Tweets from any celebrities that a user may follow are fetched 
separately and merged with that user’s home timeline when it is read, like in approach 1.

#### Describing Performance
Investigating the consequences of increasing load:
* When you increase a load parameter and keep the system resources unchanged, how is the performance of your system affected?
* When you increase a load parameter, how much do you need to increase the resources if you want to keep performance unchanged?

##### Approaches for Coping with Load
People often talk of a dichotomy between scaling up (vertical scaling, moving to a more powerful machine) and scaling 
out (horizontal scaling, distributing the load across multiple smaller machines). Distributing load across multiple 
machines is also known as a shared-nothing architecture. A system that can run on a single machine is often simpler, 
but high-end machines can become very expensive, so very intensive workloads often can’t avoid scaling out. In reality,
good architectures usually involve a pragmatic mixture of approaches: for example, using several fairly powerful
machines can still be simpler and cheaper than a large number of small virtual machines.

Some systems are elastic, meaning that they can automatically add computing resources when they detect a load increase,
whereas other systems are scaled manually (a human analyzes the capacity and decides to add more machines to the 
system). An elastic system can be useful if load is highly unpredictable, but manually scaled systems are simpler and 
may have fewer operational surprises.

### Maintainability
Over time, many different people will work on the system (engineering and operations, both maintaining current
behavior and adapting the system to new use cases), and they should all be able to work on it productively.

Three design principles for software systems:

1. Operability: Make it easy for operations teams to keep the system running smoothly.
1. Simplicity: Make it easy for new engineers to understand the system.
1. Evolvability: Make it easy for engineers to make changes to the system in the future. 
