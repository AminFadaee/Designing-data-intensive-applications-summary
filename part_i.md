# Designing Data Intensive Applications
## Part I: Foundations of Data Systems

The first four chapters go through the fundamental ideas that apply to all data systems, whether running on a single
machine or distributed across a cluster of machines:

1. [Chapter 1](chapter_1.md) introduces the terminology and approach that we’re going to use throughout this book. It examines what we
   actually mean by words like reliability, scalability, and maintainability, and how we can try to achieve these goals.
2. [Chapter 2](chapter_2.md) compares several different data models and query languages—the most visible distinguishing factor between
   databases from a developer’s point of view. We will see how different models are appropriate to different situations.
3. [Chapter 3](chapter_3.md) turns to the internals of storage engines and looks at how databases lay out data on disk. Different
   storage engines are optimized for different workloads, and choosing the right one can have a huge effect on
   performance.
4. [Chapter 4](chapter_4.md) compares various formats for data encoding (serialization) and especially examines how they fare in an
   environment where application requirements change and schemas need to adapt over time.

Later, Part II will turn to the particular issues of distributed data systems.

