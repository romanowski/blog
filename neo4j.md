# Connected and immutable data are better off with Neo4j

[Somewhere, XX-2015]
Migrating data storage from SQL to Neo4j in a new release of TTT, has helped to add time-dependent data management, simplified business logic and open for new connections-in-time features thanks to better expressiveness of highly-connected data in graph database. [bla bla - how much I can say about this project here?]

## Problem:

Our client is providing sophisticated tools to manage multiple projects in big organisations. This software stack was going through a lot of transformation and rapid development to cover new functionality, however using relational database started to be more and more of an issue than a help. It is not like SQL databases are wrong, it is not even about SQL - the thing is SQL is delivering a lot of functionality like constrained schema and types which are not essential to such data model, but has a big hole when data is more connected -- but it is where this business has value, in exploring, modifying and comparing these connections. Technically, it means more and more data processing is done by application code, in memory, where database is downgraded to simple "permanent serialized storage".

This is a usual path when scaling up - taking more and more processing from database and distributing it on appliation nodes - but it also brings a lot of ugliness - the complex data model in application logic is very far from simple stoder database model, it is very hard to validate or cross-check data consistency, do any ad-hoc business analytics, such application complexity also drives up costs of new functionality and bux-fixing/maintance.

## Evaluation and rapid prototyping

First phase was to evaluate how this data can be expressed in a better form. The only prototyped database here was Neo4j, simply because our data size was perfectly within Neo4j limits, licensing was Ok, coverage in language drivers/api was good, the same for doc, community and available help, and the used Cypher query language, somehow similar to SQL, but with a lot of functionality for connected data, was a promise to quickly add more functions when needed. We already had some previous experience with Neo4j, so it was easy to start with.

For prototyping and SQL to Neo4j conversion, we decided to write ETL in Ruby with Neography. The dynamic nature of this language helped a lot with quick changes on how the data model should be transformed and stored. The goal was to have a flexible tool that can convert existing SQL databases, perhaps with some variations about the specific transformation rules depending on different data sets/customers. Neography is a REST client for Neo4j, well documented and in active development, so it was quick to start and complete with. The speed was not too essential for this task but rather proper data conversions, but still there is a room to improve like bigger transaction blocks (batches) between commits.

Having data converted to Neo4j, the next step was to start analyzing it with Cypher. Neo4j comes with a really nice graphical data browser and it was getting better and better between releases - worth to mention as a command-line shell is far from being enough when dealing not only with nodes, simple to show as an equivalent of table or key-value storage, but also with connections between nodes that can have its own set of properties.

[TODO: screenshot of some data vis]

Cypher - the graph query language, is covering not only what to select as a group of nodes, but also how these nodes are connected, what is direction of connections if we care about it and how deeply it should be done. Its crucial part of syntax actually looks simple and quite natural:

```
(node)-[:connection]->(other_node)
```

and could be written SQLish like:

```
MATCH (node)-[:connection]->(other_node)
WHERE node.name="Foo" and other_node.name="BAR"
RETURN node, other_node
```

but can grow to express e.g. what we want to do with returned paths, sum values or aggregate returned nodes.

## Immutable data with time-machine

The business data presented in storage is expressing graph of connections and advancement of executed projects for this moment in time. There was already a functionality to keep previous changes of "node" values, but nothing like this for connections between them. One of the requests for new design was to cover immutability of the whole data structure - e.g. someone would like to know how the state of the overall project was looking like exactly 24 hours or 3 months ago, who changed what or do some aggregation in periods of time to see how the progress is going on. The answer was to keep all changes, separating immutable data from mutable with time-stamping all changes.

## Indexing and constraints

Some of the drivers for new graph database model were - how are we going to use this data, how are we going to query it, how database can help us to keep it healthy, unique or just quick to find? Neo4j has indexes, that works on data from nodes - they take node label and node property and can be constrained as a pair of unique values - similar to what we know from other databases. There are also "legacy" indexes, a bit lower-level as they need Java API and they are closer in how they work to Lucene engine. Just by using proper indexing scheme on our data, even on small data sets we could reduce query times 10-fold. With newer version of Neo4j, there also comes profiler and explain syntax to see, how possibly the query could be optimised by Neo engine.

## Delivering new Scala API

This software stack is split into separate parts, UI written with Html5/JS/Angular and backend API written in Scala/Play/Akka. The goal was to deliver new changed backend keeping UI api calls mostly intact. As the data source and "data structure engine" has drastically changed, it meant also some additional transformations to get the returned data in exactly the same way as before.

The helpful part of Scala toolbox were type aliases (to better express already used compound structures), packing simple but specific data values into its own case classes and when the processing model was quickly changing - using implicit conversions between types to minimize impact of changes to the existing code in the first step and deliver new functionality without too much refactoring.

Another story is about Scala standard collections, folds, options for optional/partial data, also parts of Scalaz library were very helpful giving simple operators for transforming and merging complex monoid-like data structures.

The crucial part developed from the very beginning were integration tests written in BDD style, covering quickly growing complexity of the new engine and possible operations.

## Scala and Neo4j

The Scala-Neo4j space does not give too much room when picking up the best library, even if some parts of Neo4j are written in Scala. Using native Java connectivity feels like too much boilerplate, FaKod was an interesting option but also forcing to learn "another version of Cypher", which could lead to a game "how I could possibly write such Cypher statement in my Cypher-like narrowed DSL", so well-known when using some SQL/ORM-like libraries. The suitable choice seemed to be AnormCypher, allowing to execute any kind of Cypher queries, but also requiring careful quoting and parsing of returned data. Not to mention constant awareness of Scala-Java conversions in collection types.

AnormCypher actually has one processing drawback, that leaded to problems with heap - all data that is red by REST client, is costly transformed in-memory before being given to application. For some queries, even when the whole database was around 15MB in size, the query response data could grow 10-fold and another 10-fold to process it. I hope to find enough time to help fixing it if possible.

But this is a game with many goals - there is not so much business value in valid and complex queries, if the exec+process time is way below expectations. Actually the solution for heap but also speed problems was to go with hybrid data model by adding caching layer - some data is taken from Neo, but some additional operations are optimised by using data from cache.

## Room for further improvements

There are many places in data model and design, that can be improved further, especially when gathering how the data will be used and where and how quickly it will be growing. Perhaps not all changes should be kept - from time perspective, we are more sensitive about current than historical data, so old changes could be partially flatten in time-periods and erased to keep our storage fit.

## Overall experience with Neo4j

Neo4j is a quick and easy to start with but complex in its analytical possibilities tool. It can drastically simplify business logic when dealing with more connected data, giving quick tools to "see" and "touch" it. It opens new area to explore existing data, "a higher kind of abstraction", so well known feeling similar to comparing flat files versus full SQL database and SQL query syntax - or when we put frequently modified files into VCS so we can manage all the changes. It is simple to write Cypher queries to prototype and develop new raports, try some new analytical things, get fresh insights. More value from existing data - sounds like marketing buzzword, but this feeling is so true here.
