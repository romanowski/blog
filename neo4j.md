# Lambda-like architecture with Neo4j

## Problem:

Our client is providing sophisticated tool to manage projects that is used by some of the CAC-40 (Euronext Paris) listed companies. This software stack was going through a lot of transformations and rapid development to cover new functionality, however using relational database as a persistence layer started to be more and more an obstacle to follow new business needs that should be expressed and adopted in this software. More and more data processing was done by application code, in memory, where database was "downgraded" to a simple key-value storage.

[ simple storage ] -- [ join/split connected data | connected data | queries engine | json/REST ] -- [ UI / Angular ]

While this is a usual path when scaling up - taking more processing from database and distributing it on application nodes - it also brings a lot of ugliness - like no visibility of business facts in your storage, a need to use higher lever tools to "just see" your data, no simple way to query, validate or change them. Any ad-hoc business analytics needs writing additional application features (no simple database queries) -- all these drives up application complexity even further and we end up not only with more expensive development of new functionalities but also with higher bug-fixing/quality/maintenance costs.

There was also a new requirement -- go immutable with all changes to allow queries and comparisons in time. That was something the existing model was not prepared for -- while fixing it to have "undo" values was an easy step, fixing it for "undo relations" would mean a snapshot of the whole structure every time such changed occurred. That way in few weeks we would have to manage hundreds of DB instances, available for querying. A complete no-go without hiring a new DevOps team (and another budget for it).

## Graph DB evaluation and prototyping

First phase was to evaluate how such data can be expressed in a more usable form. At that time there were 2-3 graph database options available on the market (OrientDB and perhaps Titan), but after quick check and with a very constrained budget, we decided to go prototyping with Neo4j. It was the most mature solution for this type of data and size, with good coverage in language drivers/api/doc, active community and ok licensing. The Cypher query language used by Neo4j is similar to SQL but covers functionality for connected data points. All that was a promise for a project with acceptable risks levels that will bring new, expected features. We'd already had some previous experience with Neo4j, so it was easy to start with.

### How do you query your data?

The important thing about modelling your graph data in Neo4j is about getting the model right. You are not going to get it for the first time, as what you really need you will see when you translate existing queries into Cypher or db-api calls and execute them, getting some benchmark numbers. A small ETL in Ruby/Neography helped a lot with quick changes on how the data model should be transformed from SQL and stored in Neo. That way many customisations already existing in SQL data didn't need to go to new Scala logic as one-offs for different customers.

When we got a useful model, we could start analysing it with Cypher. Neo4j comes with a nice graphical data browser/visualisation tool, however you can still use a command-line shell when you need it.

[TODO: screenshot of some data vis]

In Cypher - the graph query language used in Neo4j, we describe what to select as a group of nodes and how these nodes are connected. Its crucial part of syntax actually looks simple and quite natural:

```
(node1)-[:connection]->(other_node)
```

and could be written in a way similar to SQL:

```
MATCH (node1)-[:connection]->(other_node)
WHERE node1.name="Foo" and other_node.name="BAR"
RETURN node1, other_node
```

but can grow to express e.g. what we want to do with returned paths, sum values or aggregate returned nodes.

## Time-machine with immutable data

The business data represents graph of connections and advancement of executed projects for this moment in time. There was already a functionality to keep previous changes of "node" values, but nothing like this existed for connections between them. One of the requests for the new design was to cover immutability of the whole data structure - e.g. someone would like to know how the state of the overall project was looking like exactly 24 hours or 3 months ago, who changed what or to do some aggregation over periods of time to see how the progress has been going on. The answer was to separate mutable changes from immutable "cores", time-stamping all the changes - where the change could be not only in a typical data row but also in connection.

The idea is quite simple, as shown in this example graph:

[pic-state relations]

P1 and P2 are "immutable cores of data entities". P1 got 4 changes, so they are stored as a sequence of states S1-S4. P2 got two changes S1-S2. At some point of time the properties of relation r1 connecting P1 and P2 have changed, creating another relation r2. 

## Indexing and constraints

Some of the questions that we needed to consider while modeling new structures were - how are we going to use this data, how are we going to query it, how database can help us to keep it healthy, unique or just quick to find? Neo4j has indexes that work on data from nodes - they take node label and node property and can be constrained as a pair of unique values - similar to what we know from other databases. There are also "legacy" indexes, a bit lower-level and closer to Lucene engine, but give us possibility to index practically any property in graph. Just by using proper indexing scheme on our data we could reduce query times 10-fold, even on small data sets. With newer versions of Neo4j, there also comes profiler and query-explain syntax to see how possibly a query can be optimised by Neo engine and that way we have feedback on how to rebuild queries for the expected data.

## Delivering new Scala API

Our software stack is split into separate parts, with UI written in Html5/JS/Angular and backend APIs written in Scala/Play/Akka. The goal was to deliver new backend while keeping UI api calls mostly intact. As the data source and "query engine" have drastically changed, it meant also some additional fiddling with returned data to get them in exactly the same way as before. Additional complexity here was coming from the fact that it was already mimicking previous version with more SOAP-like queries, while one of the sub-goals was to introduce simple REST calls.

While developing this stack the helpful part of Scala toolbox were type aliases (to better express already used compound structures), packing simple but specific data values into its own case classes and when the processing model was quickly changing - using implicit conversions between types to minimize impact of changes to the existing code in the first step and deliver new functionality without too much refactoring.

Another story is about Scala standard collections, folds, options for optional/partial data. Also parts of Scalaz library were very helpful by providing simple operations for transforming and merging complex monoid-like data structures. 

I hope to write another blog post just focusing on these techniques - while such Scalaz-monoid functionalities can be found in libraries in other languages too, implicit type conversions are not that common.

The crucial part developed from the very beginning were integration tests written in BDD style, covering quickly growing complexity of the new engine and possible operations. To sum it up from higher perspective, another layer of functional tests were covering http API calls and JSON structures/transformations, with help of Scalatest.

## Scala and Neo4j

The Scala-Neo4j space does not give too much room when picking up the best library, even if some parts of Neo4j are written in Scala. Using native Java connectivity feels like too much boilerplate. FaKod library was an interesting option with its own DSL but also forcing to learn "another version of Cypher" which could lead to a game "how I could possibly write such Cypher statement in my Cypher-like narrowed DSL", so well-known when using some SQL/ORM-like libraries. The suitable choice seemed to be AnormCypher, allowing to execute any kind of Cypher queries, but also requiring careful quoting and parsing of returned data. It is worth to mention constant awareness of Scala-Java conversions in collection types, as they can be sometimes quite tedious to spot when "leaking" to other parts of code with quite surprising error messages.

AnormCypher actually has one processing drawback that led to problems with heap - all data read by REST client is transformed in-memory before being handed out to an application. For some queries, even when the whole database was around 15MB in size, the query response data could grow 10-fold with another 10-fold to process it. I hope to find some time to help fixing it, as it was quite annoying to see JVM breaking with 3-4 GBs memory pool, while processing so little in terms of data size.

But processing such data is a game with many goals - there is not so much business value in perfectly valid and complex queries if the execution and processing time is way below expectations. Actually the solution for heap but also for speed problems was to go with hybrid data model by adding caching layer - some data is taken from Neo4j but some additional operations are optimised/filled up by using data from cache.

## Room for further improvements

There are many places in data model and design that can be improved further, especially when it comes to gathering how the data will be used and where and how quickly it will be growing. Perhaps not all changes should have been kept - from time perspective, we are more sensitive about current than historical data, so old changes could have been partially flattened in time-periods with the other erased to keep our storage fit.

## Overall experience with Neo4j

Neo4j is a tool that is quick and easy to start with but complex in its analytical possibilities. It can drastically simplify business logic when dealing with more connected data, giving quick tools to "see" and "touch" it. It opens up new area to explore existing data, "a higher kind of abstraction", giving  a feeling similar to when someone compares flat files versus full SQL database with SQL queries - or when we put files modified by a team of people into VCS so we can easily manage and see all changes. It is simple to write Cypher queries to prototype and develop new reports, try some new analytical things, get fresh insights. There is a hidden complexity that can sporadically appear --  Neo4j approach to graph structure, graph indexing, queries with optional matches or differences between REST or write-your-own server extension. There are also some "grey" places like Neo4j simple backup that actually creates files that cannot be directly consumed by Neo4j import tool. On the other hand - by easier manipulation, navigation and visualisation, Neo4j adds new value to existing data.

Perhaps it can simplify your project-of-connected-data too?
