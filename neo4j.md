# Connected and immutable data are better off with Neo4j

[Somewhere, XX-2015]
Migrating from SQL to Neo4j in a new release of TTT, has helped to add time-independent data management,  simplified business logic and open for new connections-in-time features thanks to better expressiveness of highly-connected data in graph database. [bla bla - how much I can say about this project here?]

## Problem:

Our client is providing sophisticated tools to manage projects in big organisations. This software stack was going through a lot of transformation and rapid development to cover new functionality, however using relational database started to be more and more of an issue than a help. It is not like SQL databases are wrong, it is not even about SQL - the thing is SQL is delivering a lot of functionality like constrained schema and types which are only partially relevant to our data model, but has a big hole when data is more connected and our business is in exploring and modifying these connections. Technically, it means a shift so that more and more data processing is done by application code, where database is downgraded to a kind of "simple permanent serialized storage" with synced data from our business logic. This may sound clever and reminds a lot of similar stories with scaling up scenarios, but also brings a log of ugliness - the data model in application logic is quite far from database model and it is getting more and more costly to to add new functionality and fix existing problems.

## Evaluation and rapid prototyping

First phase was to evaluate how this data can be expressed in a better form. The only prototyped database here was Neo4j, simply because our data size was perfectly within Neo4j limits, licensing was Ok, coverage in language drivers/api was good, the same for doc and available help, and the used Cypher query language, somehow similar to SQL, but with a lot of functionality for connected data, was a promise to quickly add more functions when needed. We already had some previous experience with Neo4j, so it was easy to start with.

For prototyping and SQL to Neo4j conversion, we decided to write ETL in Ruby with Neography. The dynamic nature of this language helped a lot with quick changes to how the data model should be transformed and stored. The goal was to have a flexible tool that can convert existing databases, perhaps with some variations about the specific transformation rules depending on different data sets/customers. Neography seemed to be most fit to our needs Neo4j library and was quick to start and complete with.

Having data converted to Neo4j, the next step was to start analyzing it with Cypher. Neo4j comes with a really nice graphical data browser and it was getting better and better between releases - worth to mention as a command-line shell is far from being enough when dealing not only with nodes, simple to show as an equivalent of table or key-value storage, but also with connections between nodes that can have its own set of properties.
[TODO: screenshot of some data vis]

Cypher, as a graph query language, is covering not only what to select as a group of nodes, but also how these nodes are connected, in what direction we would like to traverse our structure and how deeply it should be done. Its crucial part of syntax actually looks simple and quite natural:

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

The business data presented in storage is expressing graph of connections and advancement of executed projects for this moment in time. There was already a functionality to keep previous changes of "node" values, but nothing like this for connections between them. One of the requests for new design was to cover immutability of the whole data structure - e.g. someone would like to know how the state of overall project was looking like exactly 24 hours or 3 months ago, who changed what or do some aggregation in periods of time to see how the progress is going on. The answer was to keep all changes, separating immutable data from mutable with time-stamping all changes.

## Indexing and constraints

Some of the drivers for new graph database model were - how are we going to use this data, how are we going to query it, how database can help us to keep it healthy, unique or just quick to find? Neo4j has indexes, that works on data from nodes - they take node label and node property and can be constrained as a pair of unique values - similar to what we know from other databases. There are also "legacy" indexes, a bit lower-level as they need Java API and they are closer in how they work to Lucene engine. Just by using proper indexing scheme on our data, even on small data sets we could reduce query times 10-fold. With newer version of Neo4j, there also comes profiler and explain syntax to see, how possibly the query could be optimised by Neo engine.

## Delivering new Scala API

This software stack is split into separate parts, UI written with Html5/JS/Angular and backend API written in Scala. The goal was to deliver new changed backend keeping UI api calls intact. As the data source and "data structure engine" has drastically changed, it meant also some additional transformations to get the returned data in exactly the same way as before.

The helpful part of Scala toolbox were type aliases (to better express already used compound structures), packing simple but specific data values into its own case classes and when the processing model was quickly changing - using implicit conversions between types to minimize impact of changes to the existing code in the first step and deliver new functionality without too much refactoring.

Another story is about Scala standard collections, folds, options for optional data, also parts of Scalaz library were very helpful giving simple operators for transforming and merging complex monoid-like data structures.

The crucial part developed from the very beginning were integration tests written in BDD style, covering quickly growing complexity of the new engine and possible operations.

## Scala and Neo4j

The Scala-Neo4j space does not give too much room when picking up the best library, even if some parts of Neo4j are written in Scala. Using native Java connectivity feels like too much boilerplate, FaKod was an interesting option but also forcing to learn "another version of Cypher", which could lead to a game "how I could possibly write such Cypher statement in my Cypher-like narrowed DSL", so well-known when using some SQL/ORM-like libraries. The suitable choice seemed to be AnormCypher, allowing to execute any kind of Cypher queries, but also requiring careful quoting and parsing of returned data. Not to mention constant awareness of Scala-Java conversions in collection types.

## Room for further improvements

There are many places in data model and design, that can be improved further, especially when gathering how the data will be used and where and how quickly it will be growing. Perhaps not all changes should be kept - from time perspective, we are more sensitive about current than historical data, so old records could be partially flatten and erased to keep our storage fit. But this is just next step in the development cycle.

## Overall experience with Neo4j

Neo4j is a quick and easy to start with but complex in its analytical possibilities tool, than can drastically simplify business logic when dealing with more connected data. It opens new area to explore existing data, "a higher kind of abstraction", so well known feeling similar to comparing flat files versus full SQL database. We can quickly write new Cypher queries to prototype and develop them into new raports, getting fresh insights into existing data. More value from existing data - sounds like marketing buzzword, but this is so true here.
