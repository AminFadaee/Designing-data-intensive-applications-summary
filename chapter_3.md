# Designing Data Intensive Applications
## Chapter 3. Storage and Retrieval
On the most fundamental level, a database needs to do two things: when you give it some data, it should store the data,
and when you ask it again later, it should give the data back to you.

Why should you, as an application developer, care how the database handles storage and retrieval internally? You’re
probably not going to implement your own storage engine from scratch, but you do need to select a storage engine that is
appropriate for your application, from the many that are available. In order to tune a storage engine to perform well on
your kind of workload, you need to have a rough idea of what the storage engine is doing under the hood.

### Data Structures That Power Your Database
In order to efficiently find the value for a particular key in the database, we need a data structure: an index. In this
chapter we will look at a range of indexing structures and see how they compare; the general idea behind them is to keep
some additional metadata on the side, which acts as a signpost and helps you to locate the data you want.

An index is an additional structure that is derived from the primary data. Many databases allow you to add and remove
indexes, and this doesn’t affect the contents of the database; it only affects the performance of queries. Maintaining
additional structures incurs overhead, especially on writes. For writes, it’s hard to beat the performance of simply
appending to a file, because that’s the simplest possible write operation. Any kind of index usually slows down writes,
because the index also needs to be updated every time data is written.

This is an important trade-off in storage systems: well-chosen indexes speed up read queries, but every index slows down
writes. For this reason, databases don’t usually index everything by default, but require you—the application developer
or database administrator—to choose indexes manually, using your knowledge of the application’s typical query patterns.
You can then choose the indexes that give your application the greatest benefit, without introducing more overhead than
necessary.

#### Hash Indexes
Key-value stores are quite similar to the dictionary type that you can find in most programming languages, and which is
usually implemented as a hash map (hash table).

Let’s say our data storage consists only of appending to a file. Then the simplest possible indexing strategy is this:
keep an in-memory hash map where every key is mapped to a byte offset in the data file—the location at which the value
can be found. Whenever you append a new key-value pair to the file, you also update the hash map to reflect the offset
of the data you just wrote (this works both for inserting new keys and for updating existing keys). When you want to
look up a value, use the hash map to find the offset in the data file, seek to that location, and read the value.

How do we avoid eventually running out of disk space? A good solution is to break the log into segments of a certain
size by closing a segment file when it reaches a certain size, and making subsequent writes to a new segment file. We
can then perform compaction on these segments. Compaction means throwing away duplicate keys in the log, and keeping
only the most recent update for each key.

Each segment now has its own in-memory hash table, mapping keys to file offsets. In order to find the value for a key,
we first check the most recent segment’s hash map; if the key is not present we check the second-most-recent segment,
and so on. The merging process keeps the number of segments small, so lookups don’t need to check many hash maps.

##### Considerations
* File format: CSV is not the best format for a log. It’s faster and simpler to use a binary format that first encodes
  the length of a string in bytes, followed by the raw string (without need for escaping).

* Deleting records: If you want to delete a key and its associated value, you have to append a special deletion record
  to the data file (sometimes called a tombstone). When log segments are merged, the tombstone tells the merging process
  to discard any previous values for the deleted key.

* Crash recovery: If the database is restarted, the in-memory hash maps are lost. In principle, you can restore each
  segment’s hash map by reading the entire segment file from beginning to end and noting the offset of the most recent
  value for every key as you go along. However, that might take a long time if the segment files are large, which would
  make server restarts painful.

* Partially written records: The database may crash at any time, including halfway through appending a record to the
  log. We can use checksums, allowing such corrupted parts of the log to be detected and ignored.

* Concurrency control: As writes are appended to the log in a strictly sequential order, a common implementation choice
  is to have only one writer thread. Data file segments are append-only and otherwise immutable, so they can be read
  concurrently by multiple threads. 

However, the hash table index also has limitations:
* The hash table must fit in memory
* Range queries are not efficient

#### SSTables and LSM-Trees
In *Sorted String Table*, or *SSTable* for short, we require that the sequence of key-value pairs be sorted by key.
For merging segments we use an approach similar to that of the *mergesort*. You start reading the input files side by
side, look at the first key in each file, copy the lowest key (according to the sort order) to the output file, and
repeat. In order to find a particular key in the file, you no longer need to keep an index of all the keys in memory.
Say you’re looking for the key handiwork, but you don’t know the exact offset of that key in the segment file. However,
you do know the offsets for the keys handbag and handsome, and because of the sorting you know that handiwork must
appear between those two. This means you can jump to the offset for handbag and scan from there until you find handiwork
(or not, if the key is not present in the file).

We can now make our storage engine work as follows:

* When a write comes in, add it to an in-memory balanced tree data structure (for example, a red-black tree). This
  in-memory tree is sometimes called a memtable.
* When the memtable gets bigger than some threshold—typically a few megabytes—write it out to disk as an SSTable file.
  The new SSTable file becomes the most recent segment of the database. While the SSTable is being written out to disk,
  writes can continue to a new memtable instance.
* In order to serve a read request, first try to find the key in the memtable, then in the most recent on-disk segment,
  then in the next-older segment, etc.
* From time to time, run a merging and compaction process in the background to combine segment files and to discard
  overwritten or deleted values.

Problem: If the database crashes, the most recent writes are lost. In order to avoid that problem, we can keep a
separate log on disk to which every write is immediately appended, and its only purpose is to restore the memtable after
a crash. Every time the memtable is written out to an SSTable, the corresponding log can be discarded.

Storage engines that are based on this principle of merging and compacting sorted files are often called LSM storage
engines.

Even though there are many subtleties, the basic idea of LSM-trees—keeping a cascade of SSTables that are merged in the
background—is simple and effective. Even when the dataset is much bigger than the available memory it continues to work
well. Since data is stored in sorted order, you can efficiently perform range queries (scanning all keys above some
minimum and up to some maximum), and because the disk writes are sequential the LSM-tree can support remarkably high
write throughput.

#### B-Trees
The most widely used indexing structure is the *B-tree*.

Like SSTables, B-trees keep key-value pairs sorted by key, which allows efficient key-value lookups and range queries.
B-trees break the database down into fixed-size blocks or pages, traditionally 4 KB in size (sometimes bigger), and read
or write one page at a time. This design corresponds more closely to the underlying hardware, as disks are also arranged
in fixed-size blocks.

Each page can be identified using an address or location, which allows one page to refer to another—similar to a
pointer, but on disk instead of in memory.

One page is designated as the root of the B-tree; whenever you want to look up a key in the index, you start here. The
page contains several keys and references to child pages. Each child is responsible for a continuous range of keys, and
the keys between the references indicate where the boundaries between those ranges lie.

The number of references to child pages in one page of the B-tree is called the branching factor. In practice, the
branching factor depends on the amount of space required to store the page references and the range boundaries, but
typically it is several hundred.

If you want to update the value for an existing key in a B-tree, you search for the leaf page containing that key,
change the value in that page, and write the page back to disk (any references to that page remain valid). If you want
to add a new key, you need to find the page whose range encompasses the new key and add it to that page. If there isn’t
enough free space in the page to accommodate the new key, it is split into two half-full pages, and the parent page is
updated to account for the new subdivision of key ranges.

This algorithm ensures that the tree remains balanced: a B-tree with n keys always has a depth of *O(log n)*. Most
databases can fit into a B-tree that is three or four levels deep, so you don’t need to follow many page references to
find the page you are looking for. (A four-level tree of 4 KB pages with a branching factor of 500 can store up to 256
TB.)

##### Making B-trees reliable
In order to make the database resilient to crashes, it is common for B-tree implementations to include an additional
data structure on disk: a write-ahead log (WAL, also known as a redo log). This is an append-only file to which every
B-tree modification must be written before it can be applied to the pages of the tree itself. When the database comes
back up after a crash, this log is used to restore the B-tree back to a consistent state.

An additional complication of updating pages in place is that careful concurrency control is required if multiple
threads are going to access the B-tree at the same time—otherwise a thread may see the tree in an inconsistent state.
This is typically done by protecting the tree’s data structures with latches (lightweight locks).

#### Other Indexing Structures
A secondary index can easily be constructed from a key-value index. The main difference is that keys are not unique;
i.e., there might be many rows (documents, vertices) with the same key. This can be solved in two ways: either by making
each value in the index a list of matching row identifiers or by making each key unique by appending a row identifier to
it. Either way, both B-trees and log-structured indexes can be used as secondary indexes.

##### Multi-column indexes
The most common type of multi-column index is called a concatenated index, which simply combines several fields into one
key by appending one column to another (the index definition specifies in which order the fields are concatenated). This
is like an old-fashioned paper phone book, which provides an index from (lastname, firstname) to phone number. Due to
the sort order, the index can be used to find all the people with a particular last name, or all the people with a
particular lastname-firstname combination. However, the index is useless if you want to find all the people with a
particular first name.

##### Keeping everything in memory
As RAM becomes cheaper, the cost-per-gigabyte argument is eroded. Many datasets are simply not that big, so it’s quite
feasible to keep them entirely in memory, potentially distributed across several machines. This has led to the
development of in-memory databases.

Counterintuitively, the performance advantage of in-memory databases is not due to the fact that they don’t need to read
from disk. Even a disk-based storage engine may never need to read from disk if you have enough memory, because the
operating system caches recently used disk blocks in memory anyway. Rather, they can be faster because they can avoid
the overheads of encoding in-memory data structures in a form that can be written to disk.

Besides performance, another interesting area for in-memory databases is providing data models that are difficult to
implement with disk-based indexes. For example, Redis offers a database-like interface to various data structures such
as priority queues and sets. Because it keeps all data in memory, its implementation is comparatively simple.

### Transaction Processing or Analytics?
In the early days of business data processing, a write to the database typically corresponded to a commercial
transaction taking place: making a sale, placing an order with a supplier, paying an employee’s salary, etc. As
databases expanded into areas that didn’t involve money changing hands, the term transaction nevertheless stuck,
referring to a group of reads and writes that form a logical unit.

An application typically looks up a small number of records by some key, using an index. Records are inserted or updated
based on the user’s input. Because these applications are interactive, the access pattern became known as *online
transaction processing* (OLTP).

However, databases also started being increasingly used for data analytics, which has very different access patterns.
Usually an analytic query needs to scan over a huge number of records, only reading a few columns per record, and
calculates aggregate statistics (such as count, sum, or average) rather than returning the raw data to the user. In
order to differentiate this pattern of using databases from transaction processing, it has been called *online analytic
processing* (OLAP) 

|Property                                          |Transaction processing systems (OLTP)             |Analytic systems (OLAP)                           |
|--------------------------------------------------|--------------------------------------------------|--------------------------------------------------|
|Main read pattern                                 |Small number of records per query, fetched by key |Aggregate over large number of records            |
|Main write pattern                                |Random-access, low-latency writes from user input |Bulk import (ETL) or event stream                 |
|Primarily used by                                 |End user/customer, via web application            |Internal analyst, for decision support            |
|What data represents                              |Latest state of data (current point in time)      |History of events that happened over time         |
|Dataset size                                      |Gigabytes to terabytes                            |Terabytes to petabytes                            |

In the late 1980s and early 1990s, there was a trend for companies to stop using their OLTP systems for analytics
purposes, and to run the analytics on a separate database instead. This separate database was called a data warehouse.

#### Data Warehousing
A data warehouse, is a separate database that analysts can query to their hearts’ content, without affecting OLTP
operations. The data warehouse contains a read-only copy of the data in all the various OLTP systems in the company.
Data is extracted from OLTP databases (using either a periodic data dump or a continuous stream of updates), transformed
into an analysis-friendly schema, cleaned up, and then loaded into the data warehouse. This process of getting data into
the warehouse is known as Extract–Transform–Load (ETL).

#### Stars and Snowflakes: Schemas for Analytics
Many data warehouses are used in a fairly formulaic style, known as a star schema (also known as dimensional modeling).
In this schema, at the center, there is a so-called *fact table*. Each row of the fact table represents an
event that occurred at a particular time. Usually, facts are captured as individual events, because this allows maximum
flexibility of analysis later. However, this means that the fact table can become extremely large.

Some of the columns in the fact table are attributes, other columns are foreign key references to other tables, called
*dimension tables*. As each row in the fact table represents an event, the dimensions represent the who, what, where,
when, how, and why of the event.

Even date and time are often represented using dimension tables, because this allows additional information about dates
(such as public holidays) to be encoded, allowing queries to differentiate between sales on holidays and non-holidays.

The name “star schema” comes from the fact that when the table relationships are visualized, the fact table is in the
middle, surrounded by its dimension tables; the connections to these tables are like the rays of a star.

A variation of this template is known as the snowflake schema, where dimensions are further broken down into
subdimensions. Snowflake schemas are more normalized than star schemas, but star schemas are often preferred because
they are simpler for analysts to work with.


### Column-Oriented Storage
In most OLTP databases, storage is laid out in a row-oriented fashion: all the values from one row of a table are stored
next to each other. Document databases are similar: an entire document is typically stored as one contiguous sequence of
bytes. 

Although fact tables are often over 100 columns wide, a typical data warehouse query only accesses 4 or 5 of them at one
time. A row-oriented storage engine still needs to load all rows (each consisting of over 100 attributes) from disk into
memory, parse them, and filter out those that don’t meet the required conditions. That can take a long time.

The idea behind column-oriented storage is simple: don’t store all the values from one row together, but store all the
values from each column together instead. If each column is stored in a separate file, a query only needs to read and
parse those columns that are used in that query, which can save a lot of work.

The column-oriented storage layout relies on each column file containing the rows in the same order. Thus, if you need
to reassemble an entire row, you can take the corresponding entry from each of the individual column files and put them
together to form the specific row of the table.

#### Column Compression
Column-oriented storage often lends itself very well to compression. Often, the number of distinct values in a column is
small compared to the number of rows, they often are quite repetitive, which is a good sign for compression.

##### Memory bandwidth and vectorized processing
Besides reducing the volume of data that needs to be loaded from disk, column-oriented storage layouts are also good for
making efficient use of CPU cycles. For example, the query engine can take a chunk of compressed column data that fits
comfortably in the CPU’s cache and iterate through it in a loop with no function calls. A CPU can execute such a loop
much faster than code that requires a lot of function calls and conditions for each record that is processed. Column
compression allows more rows from a column to fit in the same amount of cache. Operators can be designed to operate on
such chunks of compressed column data directly. This technique is known as vectorized processing.

#### Aggregation: Data Cubes and Materialized Views
Data warehouse queries often involve an aggregate function, such as COUNT, SUM, AVG, MIN, or MAX in SQL. If the same
aggregates are used by many different queries, it can be wasteful to crunch through the raw data every time. Why not
cache some of the counts or sums that queries use most often?

One way of creating such a cache is a materialized view. In a relational data model, it is often defined like a standard
(virtual) view: a table-like object whose contents are the results of some query. The difference is that a materialized
view is an actual copy of the query results, written to disk, whereas a virtual view is just a shortcut for writing
queries. When you read from a virtual view, the SQL engine expands it into the view’s underlying query on the fly and
then processes the expanded query.

When the underlying data changes, a materialized view needs to be updated, because it is a denormalized copy of the
data. 

A common special case of a materialized view is known as a data cube or OLAP cube. It is a grid of aggregates grouped by
different dimensions.
