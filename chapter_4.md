# Designing Data Intensive Applications
## Encoding and Evolution
When a data format or schema changes, a corresponding change to application code often needs to happen. However, in a
large application, code changes often cannot happen instantaneously:

* With server-side applications you may want to perform a rolling upgrade (also known as a staged rollout), deploying the
new version to a few nodes at a time, checking whether the new version is running smoothly, and gradually working your
way through all the nodes. This allows new versions to be deployed without service downtime, and thus encourages more
frequent releases and better evolvability.

* With client-side applications you’re at the mercy of the user, who may not install the update for some time.

In order for the system to continue running smoothly, we need to maintain compatibility in two directions:

* **Backward compatibility**:Newer code can read data that was written by older code.

* **Forward compatibility**: Older code can read data that was written by newer code.

Backward compatibility is normally not hard to achieve: as author of the newer code, you know the format of data written
by older code, and so you can explicitly handle it. Forward compatibility can be trickier, because it requires older
code to ignore additions made by a newer version of the code.

### Formats for Encoding Data
Programs usually work with data in (at least) two different representations:

* In memory, data is kept in objects, structs, lists, arrays, hash tables, trees, and so on. These data structures are
  optimized for efficient access and manipulation by the CPU.

* When you want to write data to a file or send it over the network, you have to encode it as some kind of
  self-contained sequence of bytes (e.g. JSON). This sequence-of-bytes representation looks quite different from the
  data structures that are normally used in memory.

Thus, we need some kind of translation between the two representations. The translation from the in-memory
representation to a byte sequence is called encoding (also known as serialization or marshalling), and the reverse is
called decoding (parsing, deserialization, unmarshalling).

#### Language-Specific Formats
Many programming languages come with built-in support for encoding in-memory objects into byte sequences. For example,
Java has java.io.Serializable, Ruby has Marshal, Python has pickle, and so on. It’s generally a bad idea to use your
language’s built-in encoding for anything other than very transient purposes:

* The encoding is often tied to a particular programming language, and reading the data in another language is very
  difficult.

* In order to restore data in the same object types, the decoding process needs to be able to instantiate arbitrary
  classes. This is frequently a source of security problems.
* Versioning data is often an afterthought in these libraries: as they are intended for quick and easy encoding of data,
  they often neglect the inconvenient problems of forward and backward compatibility.

* Efficiency (CPU time taken to encode or decode, and the size of the encoded structure) is also often an afterthought.

#### JSON, XML, and Binary Variants
JSON and XML are widely supported standardized encodings. XML is often criticized for being too verbose and
unnecessarily complicated. JSON's popularity is mainly due to its built-in support in web browsers and simplicity
relative to XML. CSV is another popular language-independent format, albeit less powerful. All 3 of these formats are
textual and human readable but have some problems:
* There is a lot of ambiguity around encoding of numbers.
* JSON and XML don't support binary strings.
* There is optional schema support for both JSON and XML but applications that don't use schemas need to potentially
  hardcode the appropriate encoding/decoding logic instead.
* CSV does not have any schema and is also a quite vague format.

Despite these flaws, JSON, XML, and CSV are good enough for many purposes. It’s likely that they will remain popular,
especially as data interchange formats (i.e., for sending data from one organization to another). In these situations,
as long as people agree on what the format is, it often doesn’t matter how pretty or efficient the format is. The
difficulty of getting different organizations to agree on anything outweighs most other concerns.

#### Thrift and Protocol Buffers
Apache Thrift and Protocol Buffers (protobuf) are binary encoding libraries that are based on the same principle.
Protocol Buffers was originally developed at Google, Thrift was originally developed at Facebook, and both were made
open source. Both Thrift and Protocol Buffers require a schema for any data that is encoded.

Thrift Example:
```
struct Person {
	1: required string userName,
	2: optional i64 favoriteNumber,
	3: optional list<string> interests
}
```

Protocol Buffers Example:
```
message Person{
	required string user_name = 1;
	optional int64 favorite_number = 2;
	repeated string interests = 3;
}
```

Thrift and Protocol Buffers each come with a code generation tool that takes a schema definition like the ones shown
here, and produces classes that implement the schema in various programming languages. Your application code can call
this generated code to encode or decode records of the schema.

##### Field tags and schema evolution
As you can see from the examples, an encoded record is just the concatenation of its encoded fields. Each field is
identified by its tag number (the numbers 1, 2, 3 in the sample schemas) and annotated with a datatype (e.g., string or
integer). If a field value is not set, it is simply omitted from the encoded record. From this you can see that field
tags are critical to the meaning of the encoded data. You can change the name of a field in the schema, since the
encoded data never refers to field names, but you cannot change a field’s tag, since that would make all existing
encoded data invalid.

You can add new fields to the schema, provided that you give each field a new tag number. If old code (which doesn’t
know about the new tag numbers you added) tries to read data written by new code, including a new field with a tag
number it doesn’t recognize, it can simply ignore that field. The datatype annotation allows the parser to determine how
many bytes it needs to skip. This maintains forward compatibility: old code can read records that were written by new
code.

What about backward compatibility? As long as each field has a unique tag number, new code can always read old data,
because the tag numbers still have the same meaning. The only detail is that if you add a new field, you cannot make it
required. If you were to add a field and make it required, that check would fail if new code read data written by old
code, because the old code will not have written the new field that you added. Therefore, to maintain backward
compatibility, every field you add after the initial deployment of the schema must be optional or have a default value.

Removing a field is just like adding a field, with backward and forward compatibility concerns reversed. That means you
can only remove a field that is optional (a required field can never be removed), and you can never use the same tag
number again (because you may still have data written somewhere that includes the old tag number, and that field must be
ignored by new code).

##### Datatypes and schema evolution
What about changing the datatype of a field? That may be possible but there is a
risk that values will lose precision or get truncated. For example, say you change a 32-bit integer into a 64-bit
integer. New code can easily read data written by old code, because the parser can fill in any missing bits with zeros.
However, if old code reads data written by new code, the old code is still using a 32-bit variable to hold the value. If
the decoded 64-bit value won’t fit in 32 bits, it will be truncated.

Protocol Buffers does not have a list or array datatype, but instead has a `repeated` marker for fields. The encoding of
a repeated field is just what it says on the tin: the same field tag simply appears multiple times in the record.  This
has the nice effect that it’s okay to change an optional (single-valued) field into a repeated (multi-valued) field. New
code reading old data sees a list with zero or one elements (depending on whether the field was present); old code
reading new data sees only the last element of the list.

Thrift has a dedicated list datatype, which is parameterized with the datatype of the list elements. This does not allow
the same evolution from single-valued to multi-valued as Protocol Buffers does, but it has the advantage of supporting
nested lists.

#### Avro
Apache Avro is another binary encoding format and started in 2009 as a subproject of Hadoop. Avro also uses a schema to
specify the structure of the data being encoded.

```json
{
    "type": "record",
    "name": "Person",
    "fields": [
        {"name": "userName",       "type": "string"},
        {"name": "favoriteNumber", "type": ["null", "long"], "default": null},
        {"name": "interests",      "type": {"type": "array", "items": "string"}}
    ]
}
```

Avro does not use any tag numbers in the schema. To parse the binary data, Avro goes through the fields in the order
they appear in the schema and use the schema to tell you the datatype of each field. This means that the binary data can
only be decoded correctly if the code reading the data is using the exact schema as the code that wrote the data. Any
mismatch in the schema between the reader and the writer would mean incorrectly decoded data.

##### The writer's schema and the reader's schema
The key idea with Avro is that the writer's schema and the reader's schema don't have to be the same-they only need to
be compatible. When data is decoded (read), the Avro library resolves the differences by looking at the writer's schema
and the reader's schema side by side and translating data from the writer's schema into the reader's schema.

##### Schema evolution rules
With Avro, forward compatibility means that you can have a new version of the schema as writer and an old version of the
schema as reader. Conversely, backward compatibility means that you can have a new version of the schema as reader and
an old version as writer.

To maintain compatibility, you may only add or remove a field that has a default value. If you want to allow a field to
be null, you have to use a *union type*. Consequently, Avro doesn’t have optional and required markers in the same way
as Protocol Buffers and Thrift do.

Changing the datatype of a field is possible, provided that Avro can convert the type. Changing the name of a field is
possible but a little tricky: the reader’s schema can contain aliases for field names, so it can match an old writer’s
schema field names against the aliases. This means that changing a field name is backward compatible but not forward
compatible. Similarly, adding a branch to a union type is backward compatible but not forward compatible.

##### But what is the writer’s schema?
There is an important question that we’ve glossed over so far: how does the reader know the writer’s schema with which a
particular piece of data was encoded? We can’t just include the entire schema with every record, because the schema
would likely be much bigger than the encoded data, making all the space savings from the binary encoding futile.

The answer depends on the context in which Avro is being used. To give a few examples:

* Writer of file can just include the writer’s schema once at the beginning of the file for large files.

* The simplest solution is to include a version number at the beginning of every encoded record, and to keep a list of
  schema versions in your database. 

* Sending records over a network connection When two processes are communicating over a bidirectional network connection.

A database of schema versions is a useful thing to have in any case, since it acts as documentation and gives you a
chance to check schema compatibility. As the version number, you could use a simple incrementing integer, or you could
use a hash of the schema.

##### Dynamically generated schemas
One advantage of Avro's approach, compared to Protocol Buffers and Thrift, is that the schema doesn’t contain any tag
numbers which makes it friendlier to dynamically generated schemas.

#### The Merits of Schemas
So, we can see that although textual data formats such as JSON, XML, and CSV are widespread, binary encodings based on
schemas are also a viable option. They have a number of nice properties:

* They can be much more compact than the various “binary JSON” variants, since they can omit field names from the
  encoded data.

* The schema is a valuable form of documentation, and because the schema is required for decoding, you can be sure that
  it is up to date (whereas manually maintained documentation may easily diverge from reality).

* Keeping a database of schemas allows you to check forward and backward compatibility of schema changes, before
  anything is deployed.

* For users of statically typed programming languages, the ability to generate code from the schema is useful, since it
  enables type checking at compile time.

In summary, schema evolution allows the same kind of flexibility as schemaless/schema-on-read JSON databases provide,
while also providing better guarantees about your data and better tooling.

### Modes of Dataflow
In the rest of this chapter we will explore some of the most common ways how data flows between processes:
* Via databases
* Via service calls
* Via asynchronous message passing


#### Dataflow Through Databases
In a database, the process that writes to the database encodes the data, and the process that reads from the database
decodes it. In an environment where the application is changing, it is likely that some processes accessing the database
will be running newer code and some will be running older code. Backward compatibility is necessary; otherwise a process
won’t be able to decode what you previously wrote.  Also means that a value in the database may be written by a newer
version of the code, and subsequently read by an older version of the code that is still running. Thus, forward
compatibility is also often required for databases.

However, there is an additional snag. Say you add a field to a record schema, and the newer code writes a value for that
new field to the database. Subsequently, an older version of the code (which doesn’t yet know about the new field) reads
the record, updates it, and writes it back. In this situation, the desirable behavior is usually for the old code to
keep the new field intact, even though it couldn’t be interpreted. Solving this is not a hard problem; you just need to
be aware of it.

##### Different values written at different times
A database generally allows any value to be updated at any time. This means that within a single database you may have
some values that were written five milliseconds ago, and some values that were written five years ago.  After you deploy
a new version of your application the five-year-old data will still be there, in the original encoding, unless you have
explicitly rewritten it since then. This observation is sometimes summed up as data outlives code.

Rewriting (migrating) data into a new schema is certainly possible, but it’s an expensive thing to do on a large
dataset, so most databases avoid it if possible. Most relational databases allow simple schema changes, such as adding a
new column with a null default value, without rewriting existing data.

#### Dataflow Through Services: REST and RPC
Web browsers are not the only type of web clients. For example, a native app running on a mobile device or a desktop
computer can also make network requests to a server. Moreover a server can itself be a client to another service. This
approach is often used to decompose a large application into smaller services by area of functionality, such that one
service makes a request to another when it requires some functionality or data from that other service.  This way of
building applications has traditionally been called a *service-oriented architecture* (SOA), more recently refined and
rebranded as *microservices architecture*.

A key design goal of a service-oriented/microservices architecture is to make the application easier to change and
maintain by making services independently deployable and evolvable. For example, each service should be owned by one
team, and that team should be able to release new versions of the service frequently, without having to coordinate with
other teams. In other words, we should expect old and new versions of servers and clients to be running at the same
time, and so the data encoding used by servers and clients must be compatible across versions of the service
API—precisely what we’ve been talking about in this chapter.

##### Web services
When HTTP is used as the underlying protocol for talking to the service, it is called a *web service*.

There are two popular approaches to web services: *REST* and *SOAP*.

REST is not a protocol, but rather a design philosophy that builds upon the principles of HTTP. It emphasizes simple
data formats, using URLs for identifying resources and using HTTP features for cache control, authentication, and
content type negotiation. REST has been gaining popularity compared to SOAP, at least in the context of
cross-organizational service integration, and is often associated with microservices. An API designed according to the
principles of REST is called *RESTful*.

By contrast, SOAP is an XML-based protocol for making network API requests. Although it is most commonly used over
HTTP, it aims to be independent from HTTP and avoids using most HTTP features. Instead, it comes with a sprawling and
complex multitude of related standards that add various features.

Even though SOAP and its various extensions are ostensibly standardized, interoperability between different vendors’
implementations often causes problems. For all of these reasons, although SOAP is still used in many large enterprises,
it has fallen out of favor in most smaller companies.

RESTful APIs tend to favor simpler approaches, typically involving less code generation and automated tooling. A
definition format such as OpenAPI, also known as Swagger, can be used to describe RESTful APIs and produce
documentation.

##### The problems with remote procedure calls (RPCs)
Web services are merely the latest incarnation of a long line of technologies for making API requests over a network,
many of which received a lot of hype but have serious problems. All of these are based on the idea of a *remote
procedure call* (RPC). The RPC model tries to make a request to a remote network service look the same as calling a
function or method in your programming language, within the same process (this abstraction is called *location
transparency*). Although RPC seems convenient at first, the approach is fundamentally flawed. A network request is very
different from a local function call:

* A local function call is predictable and either succeeds or fails.

* A local function call either returns a result, or throws an exception, or never returns. A network request has another
  possible outcome: it may return without a result, due to a timeout.

* If you retry a failed network request, it could happen that the requests are actually getting through, and only the
  responses are getting lost. In that case, retrying may cause the action to be performed multiple times.

* Every time you call a local function, it normally takes about the same time to execute. A network request is wildly
  variable.

* When you call a local function, you can efficiently pass it references (pointers) to objects in local memory. When you
  make a network request, all those parameters need to be encoded. 

* The client and the service may be implemented in different programming languages, so the RPC framework must translate
  datatypes from one language into another.

All of these factors mean that there’s no point trying to make a remote service look too much like a local object in
your programming language, because it’s a fundamentally different thing. Part of the appeal of REST is that it doesn’t
try to hide the fact that it’s a network protocol (although this doesn’t seem to stop people from building RPC libraries
on top of REST).

##### Current directions for RPC
Despite all these problems, RPC isn’t going away. Various RPC frameworks have been built on top of all the encodings
mentioned in this chapter: for example, Thrift and Avro come with RPC support included, gRPC is an RPC implementation
using Protocol Buffers, Finagle also uses Thrift, and Rest.li uses JSON over HTTP.

This new generation of RPC frameworks is more explicit about the fact that a remote request is different from a local
function call. For example, Finagle and Rest.li use futures (promises) to encapsulate asynchronous actions that may
fail. Futures also simplify situations where you need to make requests to multiple services in parallel, and combine
their results. gRPC supports streams, where a call consists of not just one request and one response, but a series
of requests and responses over time.

Custom RPC protocols with a binary encoding format can achieve better performance than something generic like JSON over
REST. However, a RESTful API has other significant advantages: it is good for experimentation and debugging, it is
supported by all mainstream programming languages and platforms, and there is a vast ecosystem of tools available.

For these reasons, REST seems to be the predominant style for public APIs. The main focus of RPC frameworks is on
requests between services owned by the same organization, typically within the same datacenter.

##### Data encoding and evolution for RPC
For evolvability, it is important that RPC clients and servers can be changed and deployed independently. Compared to
data flowing through databases, we can make a simplifying assumption in the case of dataflow through services: it is
reasonable to assume that all the servers will be updated first, and all the clients second. Thus, you only need
backward compatibility on requests, and forward compatibility on responses.

The backward and forward compatibility properties of an RPC scheme are inherited from whatever encoding it uses:

* Thrift, gRPC (Protocol Buffers), and Avro RPC can be evolved according to the compatibility rules of the respective
  encoding format.

* In SOAP, requests and responses are specified with XML schemas. These can be evolved, but there are some subtle
  pitfalls.

* RESTful APIs most commonly use JSON (without a formally specified schema) for responses, and JSON or
  URI-encoded/form-encoded request parameters for requests. Adding optional request parameters and adding new fields to
  response objects are usually considered changes that maintain compatibility.

Service compatibility is made harder by the fact that RPC is often used for communication across organizational
boundaries, so the provider of a service often has no control over its clients and cannot force them to upgrade. Thus,
compatibility needs to be maintained for a long time, perhaps indefinitely. If a compatibility-breaking change is
required, the service provider often ends up maintaining multiple versions of the service API side by side.

There is no agreement on how API versioning should work. For RESTful APIs, common approaches are to use a version number
in the URL or in the HTTP Accept header. For services that use API keys to identify a particular client, another option
is to store a client’s requested API version on the server and to allow this version selection to be updated through a
separate administrative interface.

#### Message-Passing Dataflow
In this section, we will briefly look at *asynchronous message-passing systems*, which are somewhere between RPC and
databases. They are similar to RPC in that a client’s request is delivered to another process with low latency. They are
similar to databases in that the message is not sent via a direct network connection, but goes via an intermediary
called a *message broker*, which stores the message temporarily.

Using a message broker has several advantages compared to direct RPC:

* It can act as a buffer if the recipient is unavailable or overloaded, and thus improve system reliability.

* It can automatically redeliver messages to a process that has crashed, and thus prevent messages from being lost.

* It avoids the sender needing to know the IP address and port number of the recipient.

* It allows one message to be sent to several recipients.

* It logically decouples the sender from the recipient (the sender just publishes messages and doesn’t care who consumes
them).

However, a difference compared to RPC is that message-passing communication is usually one-way: a sender normally
doesn’t expect to receive a reply to its messages. It is possible for a process to send a response, but this would
usually be done on a separate channel. This communication pattern is asynchronous: the sender doesn’t wait for the
message to be delivered, but simply sends it and then forgets about it.

##### Message brokers
The detailed delivery semantics in message brokers vary by implementation and configuration, but in general, message
brokers are used as follows: one process sends a message to a named queue or topic, and the broker ensures that the
message is delivered to one or more consumers of or subscribers to that *queue* or *topic*. There can be many producers
and many consumers on the same topic.

A topic provides only one-way dataflow. However, a consumer may itself publish messages to another topic, or to a reply
queue that is consumed by the sender of the original message.

Message brokers typically don’t enforce any particular data model—a message is just a sequence of bytes with some
metadata, so you can use any encoding format. If the encoding is backward and forward compatible, you have the greatest
flexibility to change publishers and consumers independently and deploy them in any order.
