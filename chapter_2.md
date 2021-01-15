# Designing Data Intensive Applications
## Chapter 2: Data Models and Query Languages
In this chapter we will look at a range of general-purpose data models for data storage and querying.

## Relational Model Versus Document Model
The best-known data model today is probably that of SQL, based on the relational model proposed by Edgar Codd in 1970:
data is organized into relations (tables in SQL), where each relation is an unordered collection of tuples (rows in SQL).

### The Birth of NoSQL:
There are several driving forces behind the adoption of NoSQL databases, including:

* A need for greater scalability than relational databases can easily achieve, including very large datasets or very
high write throughput
* A widespread preference for free and open source software over commercial database products
* Specialized query operations that are not well supported by the relational model
* Frustration with the restrictiveness of relational schemas, and a desire for a more dynamic and expressive data model

It seems likely that in the foreseeable future, relational databases will continue to be used alongside a broad variety
of nonrelational datastores—an idea that is sometimes called *polyglot persistence*.

### The Object-Relational Mismatch
Most application development today is done in object-oriented programming languages, which leads to a common criticism 
of the SQL data model: if data is stored in relational tables, an awkward translation layer is required between the 
objects in the application code and the database model of tables, rows, and columns. The disconnect between the models
is sometimes called an *impedance mismatch*.

Object-relational mapping (ORM) frameworks like ActiveRecord and Hibernate reduce the amount of boilerplate code 
required for this translation layer, but they can’t completely hide the differences between the two models.

Some developers feel that the JSON model reduces the impedance mismatch between the application code and the storage
layer. However, there are also problems with JSON as a data encoding format. 
The lack of a schema is often cited as an advantage.

### Many-to-One and Many-to-Many Relationships
The advantage of using an ID is that because it has no meaning to humans, it never needs to change: the ID can remain
the same, even if the information it identifies changes. Anything that is meaningful to humans may need to change
sometime in the future—and if that information is duplicated, all the redundant copies need to be updated. That incurs
write overheads, and risks inconsistencies (where some copies of the information are updated but others aren’t). 
Removing such duplication is the key idea behind *normalization* in databases.

Unfortunately, normalizing this data requires many-to-one relationships, which don’t fit nicely into the document model.
In relational databases, it’s normal to refer to rows in other tables by ID, because joins are easy. In document 
databases, joins are not needed for one-to-many tree structures, and support for joins is often weak.

If the database itself does not support joins, you have to emulate a join in application code by making multiple 
queries to the database. Moreover, even if the initial version of an application fits well in a join-free document 
model, data has a tendency of becoming more interconnected as features are added to applications.

### Are Document Databases Repeating History?
The design of IBM’s Information Management System (IMS) used a fairly simple data model called the hierarchical model,
which has some remarkable similarities to the JSON model used by document databases. It represented all data as a tree
of records nested within records, much like the JSON structure.

Like document databases, IMS worked well for one-to-many relationships, but it made many-to-many relationships 
difficult, and it didn’t support joins.

Various solutions were proposed to solve the limitations of the hierarchical model. The two most prominent were the
relational model (which became SQL, and took over the world) and the network model (which initially had a large 
following but eventually faded into obscurity).

#### The Network Model
The network model was standardized by a committee called the Conference on Data Systems Languages (CODASYL) and
implemented by several different database vendors; it is also known as the CODASYL model [16].

The CODASYL model was a generalization of the hierarchical model. In the tree structure of the hierarchical model, 
every record has exactly one parent; in the network model, a record could have multiple parents.

The links between records in the network model were not foreign keys, but more like pointers. The only way of accessing
a record was to follow a path from a root record along these chains of links. This was called an *access path*.

A query in CODASYL was performed by moving a cursor through the database by iterating over lists of records and
following access paths. If a record had multiple parents (i.e., multiple incoming pointers from other records), the
application code had to keep track of all the various relationships. Even CODASYL committee members admitted that this
was like navigating around an n-dimensional data space. The problem was that they made the code for querying and
updating the database complicated and inflexible. With both the hierarchical and the network model, if you didn’t have a
path to the data you wanted, you were in a difficult situation.

#### The Relational Model
What the relational model did, by contrast, was to lay out all the data in the open: a relation (table) is simply a
collection of tuples (rows), and that’s it. In a relational database, the query optimizer automatically decides which
parts of the query to execute in which order, and which indexes to use. Those choices are effectively the “access path,”
but the big difference is that they are made automatically by the query optimizer, not by the application developer, so
we rarely need to think about them.

#### Comparison to document databases
Document databases reverted back to the hierarchical model in one aspect: storing nested records within their parent
record rather than in a separate table. However, when it comes to representing many-to-one and many-to-many
relationships, relational and document databases are not fundamentally different: in both cases, the related item is
referenced by a unique identifier, which is called a foreign key in the relational model and a document reference in the
document model. That identifier is resolved at read time by using a join or follow-up queries. To date, document
databases have not followed the path of CODASYL.

### Relational Versus Document Databases Today
The main arguments in favor of the document data model are schema flexibility, better performance due to locality, and
that for some applications it is closer to the data structures used by the application. The relational model counters by
providing better support for joins, and many-to-one and many-to-many relationships.

#### Which data model leads to simpler application code?
If the data in your application has a document-like structure, then it’s probably a good idea to use a document model.
The relational technique of shredding—splitting a document-like structure into multiple tables can lead to cumbersome
schemas and unnecessarily complicated application code.

The document model has limitations: for example, you cannot refer directly to a nested item within a document, but
instead you need to say something like “the second item in the list of positions for user 251” (much like an access path
in the hierarchical model). However, as long as documents are not too deeply nested, that is not usually a problem.

The poor support for joins in document databases may or may not be a problem, depending on the application. For example,
many-to-many relationships may never be needed in an analytics application that uses a document database to record which
events occurred at which time.

However, if your application does use many-to-many relationships, the document model becomes less appealing. It’s
possible to reduce the need for joins by denormalizing, but then the application code needs to do additional work to
keep the denormalized data consistent. Joins can be emulated in application code by making multiple requests to the
database, but that also moves complexity into the application and is usually slower than a join performed by specialized
code inside the database. In such cases, using a document model can lead to significantly more complex application code
and worse performance.

#### Schema flexibility in the document model
Document databases are sometimes called schemaless, but that’s misleading, as the code that reads the data usually
assumes some kind of structure—i.e., there is an implicit schema, but it is not enforced by the database. A more
accurate term is schema-on-read (the structure of the data is implicit, and only interpreted when the data is read), in
contrast with schema-on-write (the traditional approach of relational databases, where the schema is explicit and the
database ensures all written data conforms to it).

Schema-on-read is similar to dynamic (runtime) type checking in programming languages, whereas schema-on-write is
similar to static (compile-time) type checking.

Schema changes have a bad reputation of being slow and requiring downtime. This reputation is not entirely deserved:
most relational database systems execute the ALTER TABLE statement in a few milliseconds.

Running the UPDATE statement on a large table is likely to be slow on any database, since every row needs to be
rewritten. If that is not acceptable, the application can leave a field to its default of NULL and fill it in at
read time, like it would with a document database.

The schema-on-read approach is advantageous if the items in the collection don’t all have the same structure for some
reason (i.e., the data is heterogeneous)—for example, because:

* There are many different types of objects, and it is not practical to put each type of object in its own table.
* The structure of the data is determined by external systems over which you have no control and which may change at any
  time.

In situations like these, a schema may hurt more than it helps, and schemaless documents can be a much more natural data
model. But in cases where all records are expected to have the same structure, schemas are a useful mechanism for
documenting and enforcing that structure.

#### Data locality for queries
The locality advantage only applies if you need large parts of the document at the same time. The database typically
needs to load the entire document, even if you access only a small portion of it, which can be wasteful on large
documents. On updates to a document, the entire document usually needs to be rewritten—only modifications that don’t
change the encoded size of a document can easily be performed in place. For these reasons, it is generally
recommended that you keep documents fairly small and avoid writes that increase the size of a document. These
performance limitations significantly reduce the set of situations in which document databases are useful.

#### Convergence of document and relational databases
PostgreSQL since version 9.3, MySQL since version 5.7, and IBM DB2 since version 10.5 have level of support for JSON
documents. Given the popularity of JSON for web APIs, it is likely that other relational databases will follow in their
footsteps and add JSON support.

On the document database side, RethinkDB supports relational-like joins in its query language, and some MongoDB drivers
automatically resolve database references (effectively performing a client-side join, although this is likely to be
slower than a join performed in the database since it requires additional network round-trips and is less optimized).

It seems that relational and document databases are becoming more similar over time, and that is a good thing: the data
models complement each other. If a database is able to handle document-like data and also perform relational queries on
it, applications can use the combination of features that best fits their needs.

A hybrid of the relational and  document models is a good route for databases to take in the future.

## Query Languages for Data
When the relational model was introduced, it included a new way of querying data: SQL is a *declarative* query language
(imperative language tells the computer to perform certain operations in a certain order), whereas IMS and CODASYL
queried the database using *imperative* code (like many commonly used programming languages).

A declarative query language is attractive because it is typically more concise and easier to work with than an
imperative API. But more importantly, it also hides implementation details of the database engine, which makes it
possible for the database system to introduce performance improvements without requiring any changes to queries.
Finally, declarative languages often lend themselves to parallel execution.

### Declarative Queries on the Web
In a web browser, using declarative CSS styling is much better than manipulating styles imperatively in JavaScript.
Similarly, in databases, declarative query languages like SQL turned out to be much better than imperative query APIs.

#### MapReduce Querying
MapReduce is a programming model for processing large amounts of data in bulk across many machines, popularized by
Google. A limited form of MapReduce is supported by some NoSQL datastores, including MongoDB and CouchDB, as a mechanism
for performing read-only queries across many documents.

MapReduce is neither a declarative query language nor a fully imperative query API, but somewhere in between: the logic
of the query is expressed with snippets of code, which are called repeatedly by the processing framework. It is based on
the *map* (also known as collect) and *reduce* (also known as fold or inject) functions that exist in many functional
programming languages.

The map and reduce functions are somewhat restricted in what they are allowed to do. They must be pure functions, which
means they only use the data that is passed to them as input, they cannot perform additional database queries, and they
must not have any side effects. These restrictions allow the database to run the functions anywhere, in any order, and
rerun them on failure. However, they are nevertheless powerful: they can parse strings, call library functions, perform
calculations, and more.

## Graph-Like Data Models
There are cases where many-to-many relationships are very common in your data? The relational model can handle simple
cases of many-to-many relationships, but as the connections within your data become more complex, it becomes more
natural to start modeling your data as a graph.

Graphs do not need to be limited to homogeneous data: an equally powerful use of graphs is to provide a consistent way
of storing completely different types of objects in a single datastore. For example, Facebook maintains a single graph
with many different types of vertices and edges: vertices represent people, locations, events, checkins, and comments
made by users; edges indicate which people are friends with each other, which checkin happened in which location, who
commented on which post, who attended which event, and so on.

There are several different, but related, ways of structuring and querying data in graphs.

### Property Graphs
In the property graph model, each vertex consists of:

* A unique identifier
* A set of outgoing edges
* A set of incoming edges
* A collection of properties (key-value pairs)

Each edge consists of:

* A unique identifier
* The vertex at which the edge starts (the tail vertex)
* The vertex at which the edge ends (the head vertex)
* A label to describe the kind of relationship between the two vertices
* A collection of properties (key-value pairs)

Some important aspects of this model are:

1. Any vertex can have an edge connecting it with any other vertex. There is no schema that restricts which kinds of
   things can or cannot be associated.
1. Given any vertex, you can efficiently find both its incoming and its outgoing edges, and thus traverse the
   graph

1. By using different labels for different kinds of relationships, you can store several different kinds of information
   in a single graph, while still maintaining a clean data model.

Those features give graphs a great deal of flexibility for data modeling.

### The Cypher Query Language
Cypher is a declarative query language for property graphs, created for the Neo4j graph database. (It is named after a
character in the movie The Matrix and is not related to ciphers in cryptography.)

### Graph Queries in SQL
Graph data can be represented in a relational database. But if we put graph data in a relational structure, querying it
would have some difficulty. In a relational database, you usually know in advance which joins you need in your query. In
a graph query, you may need to traverse a variable number of edges before you find the vertex you’re looking for—that
is, the number of joins is not fixed in advance.

Since SQL:1999, this idea of variable-length traversal paths in a query can be expressed using something called
recursive common table expressions (the WITH RECURSIVE syntax). However, the syntax is very clumsy in comparison to
Cypher.

### Triple-Stores
The triple-store model is mostly equivalent to the property graph model, using different words to describe the same
ideas.

In a triple-store, all information is stored in the form of very simple three-part statements: (subject, predicate,
object). For example, in the triple (Jim, likes, bananas), Jim is the subject, likes is the predicate (verb), and
bananas is the object.

The subject of a triple is equivalent to a vertex in a graph. The object is one of two things:
1. A value in a primitive datatype, such as a string or a number. In that case, the predicate and object of the triple
   are equivalent to the key and value of a property on the subject vertex. For example, (lucy, age, 33) is like a
   vertex lucy with properties {"age":33}.

1. Another vertex in the graph. In that case, the predicate is an edge in the graph, the subject is the tail vertex, and
   the object is the head vertex. For example, in (lucy, marriedTo, alain) the subject and object lucy and alain are
   both vertices, and the predicate marriedTo is the label of the edge that connects them.

### Datalog
Datalog’s data model is similar to the triple-store model, generalized a bit. Instead of writing a triple as (subject,
predicate, object), we write it as predicate(subject, object).
