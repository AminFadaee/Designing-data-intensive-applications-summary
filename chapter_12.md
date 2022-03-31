# Designing Data Intensive Applications
## Chapter 12: The Future of Data Systems

> Disclaimer[From auther of this Repo, not the author of the book]: As I consider this chapter a general conclusion and
> summary to the whole book, I just used the book summary here.

In this chapter we discussed new approaches to designing data systems, and I included my personal opinions and
speculations about the future. We started with the observation that there is no one single tool that can efficiently
serve all possible use cases, and so applications necessarily need to compose several different pieces of software to
accomplish their goals. We discussed how to solve this data integration problem by using batch processing and event
streams to let data changes flow between different systems.

In this approach, certain systems are designated as systems of record, and other data is derived from them through
transformations. In this way we can maintain indexes, materialized views, machine learning models, statistical
summaries, and more. By making these derivations and transformations asynchronous and loosely coupled, a problem in one
area is prevented from spreading to unrelated parts of the system, increasing the robustness and fault-tolerance of the
system as a whole.

Expressing dataflows as transformations from one dataset to another also helps evolve applications: if you want to
change one of the processing steps, for example to change the structure of an index or cache, you can just rerun the new
transformation code on the whole input dataset in order to rederive the output. Similarly, if something goes wrong, you
can fix the code and reprocess the data in order to recover.

These processes are quite similar to what databases already do internally, so we recast the idea of dataflow
applications as unbundling the components of a database, and building an application by composing these loosely coupled
components.

Derived state can be updated by observing changes in the underlying data. Moreover, the derived state itself can further
be observed by downstream consumers. We can even take this dataflow all the way through to the end-user device that is
displaying the data, and thus build user interfaces that dynamically update to reflect data changes and continue to work
offline.

Next, we discussed how to ensure that all of this processing remains correct in the presence of faults. We saw that
strong integrity guarantees can be implemented scalably with asynchronous event processing, by using end-to-end
operation identifiers to make operations idempotent and by checking constraints asynchronously. Clients can either wait
until the check has passed, or go ahead without waiting but risk having to apologize about a constraint violation. This
approach is much more scalable and robust than the traditional approach of using distributed transactions, and fits with
how many business processes work in practice.

By structuring applications around dataflow and checking constraints asynchronously, we can avoid most coordination and
create systems that maintain integrity but still perform well, even in geographically distributed scenarios and in the
presence of faults. We then talked a little about using audits to verify the integrity of data and detect corruption.

Finally, we took a step back and examined some ethical aspects of building data- intensive applications. We saw that
although data can be used to do good, it can also do significant harm: making justifying decisions that seriously affect
people’s lives and are difficult to appeal against, leading to discrimination and exploitation, normalizing
surveillance, and exposing intimate information. We also run the risk of data breaches, and we may find that a
well-intentioned use of data has unintended consequences.

As software and data are having such a large impact on the world, we engineers must remember that we carry a
responsibility to work toward the kind of world that we want to live in: a world that treats people with humanity and
respect. I hope that we can work together toward that goal.

