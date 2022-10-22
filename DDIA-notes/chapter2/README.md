# Chapter 2 - Data Models and Query Language


## Table of Contents
1. [Mind Map](#mind-map)


## Mind Map
![mindmap](/DDIA-notes/chapter2/DDIA%20Chapter%202.jpg)


![Untitled](/DDIA-notes/chapter2/datamodelflow.jpg)

<aside>
💡 each layer hides the complexity of the layer below
</aside>

# Relational Model vs Document Model

relational database is dominant for **transaction processing** and **batch processing** in the early days. There were a lot of competition but they never lasted. 

## The Birth of NoSQL

> Not Only SQL
> 
- the need for greater **scalability**, very **large dataset** and **high throughput**
- the need for free and open source software
- the need for removing **restrictiveness** of relation **schemas**
- the need for specialized query operations

## The Object-Relational Mismatch

- there is an awkward translation layer between OOP and SQL database model which is called **impedance mismatch**. ORM frameworks are helpful but not perfect.
- one-to-many
    - normalization to separate tables and join using FK, not **self-contained, less locality**
    - encode multi-value data into JSON/XML and store within a single row. Some DB support internal indexing if the data is structured.

## Many-to-One and Many-to-Many

- IDs vs plain text - dropdown selection vs free input
    - consistent style
    - avoid ambiguity (same or similar names)
    - ease of updating - single place and we can keep the same ID since it machine meaningful only
    - localization and translation support
    - better search - if we know the relationship
    - duplicating human meaningful information
- many-to-one and many-to-many do **NOT** fit well in document model since join is weakly supported - this need to be performed in the application layer
- even these relation might not be needed when first designing a DB, data has a tendency of becoming more and more **interconnected** as features are added into the application.
- to support many-to-many, the network model: unlike hierarchy model, each node can have **multiple parent**. the link is not FK, it’s pointers.  dev needs to track the access path
- query optimizer in hierarchy model automatically decide how to execute a query, so any changes in the DB is easy, dev does NOT need to track the access path

## Relational vs Document DB Today

- Comparison
    - schema flexibility - **can be both good or bad,** similar to strong/weak type programming languages (JS vs TS)
        - schema-on-read : **runtime**
            - adding/updating columns is easy
        - schema-on-write : **static**
            - schema migration & backward/forward compatibility
    - better **performance** due to locality
        - fast read since less queries
        - need to update the entire document
        - some SQL support locality - materialized view01
    - closeness to data structure in the application - JS and JSON
    - support joins, many-to-one and many-to-many relationship - you can do it in application code but it will be slower compared to specialized code in DB
    - refer to nested items in documents
- **convergence** of document and relational DB
    - XML JSON support in SQL - Oracle blob
    - some MongoDB driver support join
- we can use a **hybrid** of relational and document DB if needed

# Query Language for Data

- SQL is a **declarative** language, compared to **imperative.** you just need to specify the pattern, not how to achieve that. query optimizer will handle that part. it hides the complexity of the database engine, even parallel execution.
- MapReduce is neither a declarative nor imperative language, but somewhere in between: the logic of the query is expressed with snippets of code ( mapper and reducer) which is called repeatedly by the processing framework. it is a fairly low-level programming model for distributed execution on a cluster of machines. High-level query language can be implemented as a pipeline if MapReduce operations.

# Graph-Like Data Models

- if many-to-may relationships are very common
- store completely different types of object - non homogenous
    
    ![Untitled](/DDIA-notes/chapter2/graphdb.png)
    
- use of vertices (node) and edges
    - social graph
    - web graph
        - PageRank
    - road networks
        - navigation, Dijkstra

## Property Graphs

- Node contains:
    - ID
    - set of outgoing edges
    - set of incoming edges
    - collection of properties (k-v pairs)
- Edges contains:
    - ID
    - starting node
    - ending node
    - relation label
    - collection of properties (k-v pairs)
    
    ```sql
    CREATE TABLE vertices (
    	vertex_id integer PRIMARY KEY,
    	properties json
    );
    CREATE TABLE edges (
    	edge_id integer PRIMARY KEY,
    	tail_vertex integer REFERENCES vertices (vertex_id),
    	head_vertex integer REFERENCES vertices (vertex_id),
    	label text,
    	properties json
    );
    ```
    
- schema-less, great flexibility for data modeling
- Graphs are good for evolvability

## The Cypher QL

- declarative language for Graph DB - Neo4j
    
    ```sql
    MATCH
    	(person) -[:BORN_IN]-> () -[:WITHIN*0..]-> (us:Location {name:'United States'}),
    	(person) -[:LIVES_IN]-> () -[:WITHIN*0..]-> (eu:Location {name:'Europe'})
    RETURN person.name
    ```
    
- There will always be more than one way to traverse a graph and get your query result. As a declarative QL, you do not need to specify the **execution details**. Query optimizer will choose the most **efficient** strategy.

## Graph QL in SQL

- put Graph in a relational DB is feasible, but with one challenge, **the number of join is not fixed** in advance since we might need to traverse a variable number of edges to get the data.
- Some SQL support recursive expression, but it’s complex and too verbose

## Triple-Stores and SPARQL

- equivalent to the property graph model, using different words to describe the same ideas. (subject, predicate, object)

# Summary

- **Document** databases target use cases where data comes in self-contained documents and relationships between one document and another are rare.
- **Graph** databases go in the opposite direction, targeting use cases where anything
is potentially related to everything.
- R**elational** DB is somewhere in between
- All three are used widely today