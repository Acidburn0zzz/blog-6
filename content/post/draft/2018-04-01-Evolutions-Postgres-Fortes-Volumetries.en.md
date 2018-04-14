+++
title = "PostgreSQL's developments for high volumes processing"
date = 2018-03-27T09:11:00+02:00
draft = true
summary = "Since few years, postgreSQL received several enhancements to process high volumes databases."

# Tags and categories
# For example, use `tags = []` for no tags, or the form `tags = ["A Tag", "Another Tag"]` for one or more tags.
tags = ["postgres","JIT","Index","BRIN","VACUUM","parallelism", "Bloom" ]
categories = ["Postgres"]

# Featured image
# Place your image in the `static/img/` folder and reference its filename below, e.g. `image = "example.jpg"`.
# Use `caption` to display an image caption.
#   Markdown linking is allowed, e.g. `caption = "[Image credit](http://example.org)"`.
# Set `preview` to `false` to disable the thumbnail in listings.
[header]
image = ""
caption = ""
preview = true

+++

{{% toc %}}

# Introduction

Since few years, postgreSQL received several enhancements to process high volumes databases.

This first post will try to list them. We will see that they can be of different types:

  * Parallelism
  * Improvement of query processing
  * Partitioning
  * Access methods
  * Maintenance tasks
  * SQL features

  In order to maintain clarity, the explanation of each feature will remain brief.

  Note: this article was written during the development phase of version 11.
  I integrated new features of version 11. As long as it is not released,
  these fatures can be removed.

# SQL features

## TABLESAMPLE (9.5)

`TABLESAMPLE` clause (since PG 9.5) allows to make a request on a sample.
This gives an overview of the result. As in statistics, larger is the sample, closer
the result will be to the real result.

## GROUPING SETS (9.5)

Still with version 9.5, PostgreSQL has new clauses allowing
to do multiple aggregations called [GROUPING SETS](https://www.postgresql.org/docs/current/static/queries-table-expressions.html#QUERIES-GROUPING-SETS).
New aggregates are: `GROUPING SETS`, `ROLLUP`, `CUBE`.

See this article by Depesz: [Waiting for 9.5 – Support GROUPING SETS, CUBE and ROLLUP](https://www.depesz.com/2015/05/24/waiting-for-9-5-support-grouping-sets-cube-and-rollup/)

Note that version 10 brings very significant gains thanks to improvement in
hash functions.


## Foreign table inheritance (9.5)

Since version 9.5 it is possible to declare foreign tables as child tables.
It is thus possible to distribute data on different servers and to
access from a single instance. This technique is similar to sharding.

See this article by Michael Paquier: [Postgres 9.5 feature highlight - Scale-out with Foreign Tables now part of Inheritance Trees](http://paquier.xyz/postgresql-2/postgres-9-5-feature-highlight-foreign-table-inheritance/)

# Parallelism

PostgreSQL being multi-process, the processing of a query was only done
on one core. There are many posts where users complain about this operation:

  * [Query parallelization for single connection in Postgres](https://stackoverflow.com/questions/32629988/query-parallelization-for-single-connection-in-postgres?rq=1&utm_medium=organic&utm_source=google_rich_qa&utm_campaign=google_rich_qa)
  * [Any way to use >1 Core in PostgreSQL for a single Connection/Query?](https://stackoverflow.com/questions/18268130/any-way-to-use-1-core-in-postgresql-for-a-single-connection-query)

Since 9.6, Postgres is able to call multiple processes (called *workers*)
for the processing of a request. This major breakthrough was the culmination of
several years of work. It took a lot of infrastructure
to allow Postgres to use multiple processes. See this article from
Robert Haas: [Parallelism Progress](http://rhaas.blogspot.fr/2013/10/parallelism-progress.html)

## Sequential Scan (9.6)

Since 9.6, PostgreSQL is able to use multiple processes to parallelize read
operation. This process matches to node *Parallel Seq Scan*.


## Index Scan (10)

Version 10 has made it possible to extend the parallelization to index scans.

So Postgres is now able to use several processes for these types of scan (only with BTree):

  * *Index Scan*
  * *Bitmap heap scan*
  * *Index Only Scan*

## Joins (9.6, 10, 11)

In addition to the parallelization of the sequential scan, Postgres 9.6 brought the
possibility to parallelize join operations to the following nodes:

  * *Nested-loop*
  * *Hash join*

Postgres 10 allowed to extend the parallelization to the *merge join* node.

Finally, version 11 brings a big change with the *parallel hash join*.
With previous versions, every worker had to build his own hash table.
There was a big loss of efficiency:

   * Several workers were doing the same operation
   * The same hash table existed several times in memory

The *parallel hash join* allows workers to parallelize the creation of this
hash table and share a single hash table.

The main author of this feature has written an excellent article: [Parallel Hash for PostgreSQL ](https://write-skew.blogspot.fr/2018/01/parallel-hash-for-postgresql.html)

## Aggregation (9.6)

Still with version 9.6, Postgres is able to use multiple workers
to perform aggregation operations (`COUNT`, `SUM`...).

In fact, each worker makes a partial aggregation (*Partial Aggregate*), then,
a parent node is responsible for finalizing (*Finalize Aggregate*).


## Union of sets (11)

Version 11 provides the ability to parallelize unions (node *append*).
For example, when using `UNION` or when tables are inherited.


# Access methods

Indexes and access methods are often confused by regularly using the term "index".

In fact, in the broad sense, in the domain of databases, an index is an access method.


## BRIN Indexes (9.5)


Since version 9.5, PostgreSQL offers a particular kind of index: BRIN for Block Range INdexes.

This type of index contains the summary of a set of blocks. They are very compact and can easily fit in RAM.

They are particularly suitable to high volumes with queries manipulating
a large amount of data. Be careful, it is very important that there is a strong correlation between the data and their location.

I presented the operation of this index during the PGDay France 2016 in Lille: [BRIN indexes - How they works and usages?](https://blog.anayrat.info/en/talk/2016/05/31/brin-indexes---how-they-works-and-usages/)

## BLOOM filters (9.6)

Since version 9.6 it is possible to use [Bloom filters](https://en.wikipedia.org/wiki/Bloom_filter).
Without going into details, this type of data structure used to say with certainty
that the information sought is not in a set. Conversely, information *may*
(with some probability) be in another set.

The advantage of bloom filters is that they are very compact and allow to respond
to multi-column searches from a single index.

See this article by Depesz : [Waiting for 9.6 – Bloom index contrib module](https://www.depesz.com/2016/04/11/waiting-for-9-6-bloom-index-contrib-module/)

# Partitioning

Partitioning existed as a table inheritance. However this approach
had the disadvantage of requiring to set up triggers to route writes in the right tables.
The impact on performance was important.

Version 10 includes native partitioning management. So, it is no longer
necessary to set up triggers. Maintenance operations are facilitated
and performance is improved.

## Forms of partitioning (10, 11)

PostgreSQL supports partition by:

  * List - `LIST` (10)
  * Interval - `RANGE` (10)
  * Hash - `HASH` (11)

See theses articles by  Depesz :

  * [Waiting for PostgreSQL 10 – Implement table partitioning](https://www.depesz.com/2017/02/06/waiting-for-postgresql-10-implement-table-partitioning/)
  * [Waiting for PostgreSQL 11 – Add hash partitioning](https://www.depesz.com/2017/11/10/waiting-for-postgresql-11-add-hash-partitioning/)

## Indexes (11)

Version 11 also provides easier index management on partition tables. Index creation
on a partitioned table will also be created on all partitions.
Similarly, it is possible to have a unique index (only if they include partition key).
It is also possible to have a foreign key on a partitioned table.



## Partition exclusion (11)

*partition pruning* consists of excluding unnecessary partitions. Postgres relies
on exclusion constraints to exclude partitions from planning.

The algorithm was not designed to handle a large number of partitions. His
complexity is linear depending on the number of partitions ([How many table partitions is too many in Postgres?](https://stackoverflow.com/a/6131446))

To fix this, version 11 incorporates a new, more efficient search algorithm: *Faster Partition Pruning*

Finally, Postgres could exclude partitions only during the planning.
With version 11, the engine is able to exclude a partition during execution.
This feature is called *Runtime Partition Pruning*.

## Joins and aggregates (11)


Version 11 introduces new join and aggregation algorithms.
The idea is to perform join operations and aggregates partition by
partition during a join between two partitioned tables  (See [Partition and conquer large data with PostgreSQL 10](https://www.pgcon.org/2017/schedule/events/1047.en.html)).

# Internal improvements

## Hashing functions (10)

Hash functions have been improved in version 10. Thus, aggregate (`GROUP BY`,` GROUPING SETS`, `CUBE` ...)
as well as the nodes type *bitmap scans* benefit from these improvements. The execution time of some queries has been halved!

## Executor improvements (10)

The executor has been improved in version 10, it is now more efficient expressions processing.
This thread mentions very significant gains: [Faster Expression Processing](https://www.postgresql.org/message-id/20170314065259.ffef4tfhgbsaieoe%40alap3.anarazel.de)

## Sorts improvements

### Abbreviated keys (9.5)

Sorting algorithm has been revised with version 9.5, which makes better use of processor's cache.
Gains were reported between 40 and 70% (see [Use abbreviated keys for faster sorting of text datums](http://pgeoghegan.blogspot.fr/2015/01/abbreviated-keys-exploiting-locality-to.html)).

### External sorts improvements (10)

Version 10 also brings very significant gains when sorts are made on disk. Some queries saw their execution time halved.

## Just-In-time (11)

Version 11 includes an infrastructure for [Just-In-Time (JIT)](https://en.wikipedia.org/wiki/Just-in-time_compilation).
JIT compile the query to generate [bytecode](https://en.wikipedia.org/wiki/Bytecode) which will be executed.

Once again, announced gains are impressive as evidenced by these slides: [JITing PostgreSQL using LLVM](http://anarazel.de/talks/fosdem-2018-02-03/jit.pdf)

# Maintenance tasks

## VACUUM FREEZE (9.6)

Before version 9.6, a `VACUUM FREEZE` read the whole table. Even if lines had already
been frozen". Version 9.6 adds additional information to the *visibility map*
to see if a block has already been frozen. This information allows Postgres to skip blocks already frozen.

## Reduce index scan during VACUUM (11)

During a simple vacuum (where postgres will clean the old lines). Postgres was able
to skip blocks where he knew there was no line to deal with. However, he still had
to scan the index, which can be expensive with a large table. Version 11 makes it
possible to avoid unnecessary index scan.

A new parameter appeared: *vacuum_cleanup_index_scale_factor*.

Postgres can avoid index scan if two conditions are met:

   * No block should be deleted
   * Statistics are up to date

Postgres considers that the statistics are stalled if more than:
[number of inserted lines] > vacuum_cleanup_index_scale_factor * [number of rows in the table]

## Parallel create index (11)

When creating an index, Postgres can use multiple processes to perform sort operation.
Depending on the number of processes, the creation time of the index can be divided from 2 to 4 approximately.

# Foreign Data Wrapper pushdown (9.6, 10)

A reminder, Foreign Data Wrapper provides access to external data.
This is the Postgres implementation of the SQL / MED standard for "Management of external Data".

A FDW allows access to any type of external data when a FDW exists (see [Foreign data wrappers](https://wiki.postgresql.org/wiki/Foreign_data_wrappers)).

The community maintains a FDW to connect to a Postgres instance: *postgres_fdw*.

But that's not all, some operations can be "pushed" to the remote server. This is called *pushdown*

Prior to Version 9.6, a sort or join resulted in the download of all the data and the server performed sort and/or join locally.

Version 9.6 includes sort and join pushdown. Thus, sort or join operation can be
performed by the remote server. On the one hand it unloads the server running the
query, on the other hand, the remote server will be able to use appropriate join
algorithm or an index for sorting the data.

Finally, version 10 also includes the execution of `FULL JOIN` aggregations and
joins on the remote server.

# Future

Feature freeze of version 11 ended on April 7: [PostgreSQL 11 Release Management Team & Feature Freeze](https://www.postgresql.org/message-id/AA141CD1-19CB-414F-98CB-87A32F397295%40postgresql.org)

This means that some features have not been implemented, some of which are considered
not mature enough to be integrated. Others are still in the state of discussion
or demonstration.

Note that some features can be removed after the feature freeze if the developers
consider that they are not stable or their the implementation must be revised.

Luckily, the developers of the various companies that contribute to the development
of Postgres, communicate about their roadmap. This gives an overview of the trends
of the future features.

## Extension of the storage system

Community works to make modular storage system (*[pluggable storage](https://www.postgresql.org/message-id/flat/20160812231527.GA690404%40alvherre.pgsql#20160812231527.GA690404@alvherre.pgsql)*).
Thus, Postgres could have different storage engines. In the work in progress, we can list:

  * Columnar storage (associated with vectorization)
  * Table compression
  * *In-memory* storage
  * [zheap](https://github.com/EnterpriseDB/zheap) : this engine would update records
  directly in the heap. Without duplicating tuples in the tables according to MVCC.
  And so, get rid of fragmentation and vacuum.

## Extension of JIT

The author of the JIT plans to extend it to the rest of Postgres (aggregation, hashing, sorts ...).

In the mentioned tracks, there would also be caching and sharing the bytecode between
sessions. This would make it possible to use the JIT even in the case of OLTP traffic.

## Vectorization

The idea of vectorization would be to process data in batch to use processors's
[SIMD instructions](https://en.wikipedia.org/wiki/SIMD).

Coupled with a column storage, gains can be impressive:

  * [Vectorized Postgres (VOPS extension)](https://pgconf.ru/media/2018/02/20/%D0%9A%D0%BD%D0%B8%D0%B6%D0%BD%D0%B8%D0%BA_Vectorized%20Postgres%20(VOPS%20extension).pdf)

## FDW and asynchronous execution

Sharding comes back regularly. In a way, the use of FDW partly meets this need.
However, PostgreSQL still has some progress to make in this area. For example,
if it performs an aggregate query on multiple remote tables, he must request each
remote server sequentially. An improvement track would be to query all remote servers
asynchronously. Thus, the operation would be parallelized on all remote servers.
See this presentation: [FDW-based Sharding Update and Future](https://fr.slideshare.net/masahikosawada98/fdwbased-sharding-update-and-future).