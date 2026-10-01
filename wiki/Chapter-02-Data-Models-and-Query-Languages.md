# Chapter 2: Data Models and Query Languages

_Part I: Foundations of Data Systems_

> **Key takeaways**
> - The data model shapes how you think about the problem, so choose it deliberately.
> - **Document** models suit self-contained, tree-shaped data (one-to-many). **Relational** models suit data with many-to-one and many-to-many relationships. **Graph** models suit highly interconnected data.
> - Document databases are not schemaless. They are **schema-on-read**, not **schema-on-write**.
> - **Declarative** query languages (SQL, Cypher) let the database optimize execution, and they parallelize more easily than imperative code.

---

## Data Models

Data models may be the most important part of developing software, because they deeply affect how we think about the problem we are solving.

### The relational model

The relational model is the best-known data model today. It hides implementation details behind a clean interface, and it has generalized very well as computers have been used for more and more purposes.

### The rise of NoSQL

NoSQL databases were adopted quickly because they offered:

- Better scalability, including very high write throughput.
- Free, open-source implementations.
- Better support for some specialized query operations.
- More dynamic and expressive data models.

Relational databases will keep being used alongside a wide range of non-relational stores. This mix is called **polyglot persistence**.

### The object-relational mismatch

A common criticism of relational databases is the awkward translation layer between objects in application code and tables, rows, and columns in the database. ORM frameworks reduce this boilerplate but cannot remove it entirely.

### One-to-many relationships

A relational database can store one-to-many data in three ways:

1. **Normalized** (the common approach): put the "many" values in a separate table with a foreign key to the "one".
2. **Structured columns:** later versions of SQL support multi-valued data (such as JSON or XML columns) in a single row, and allow queries inside them.
3. **Encoded text** (least preferred): store the values as a JSON or XML string and leave the application to interpret it.

Document databases support one-to-many relationships natively. Because a JSON document is self-contained, they also give better **locality**: everything about the object can be loaded in one read.

### Many-to-one and many-to-many relationships

Offering standardized lists for users to pick from (instead of free text) gives consistent style and spelling, avoids ambiguity, makes updates easier, helps localization, and improves search. These lists should be stored by **ID**, because anything meaningful to humans may need to change one day.

- Relational databases handle **many-to-one** relationships easily: refer to rows by ID and join.
- Document databases handle them poorly because joins are weak or missing. The application has to emulate the join with extra queries, although it can cache small, slow-changing lists in memory.

### Why the relational model won historically

Older models, the **hierarchical** and **network** (CODASYL) models, lost out over time largely because of the relational **query optimizer**. The optimizer chooses access paths automatically, so adding new features and queries to an application became much easier.

### Relational vs. document today

| | Document model | Relational model |
|---|---|---|
| Strengths | Schema flexibility, better performance through locality, closer match to application data structures | Better join support, handles many-to-one and many-to-many relationships well |
| Schema | **Schema-on-read**: structure is implicit and interpreted when data is read | **Schema-on-write**: structure is explicit and enforced by the database |

Notes:

- Schema-on-read is not enforced by the database, but it makes changing the data format easier.
- Locality only helps if documents stay fairly small, because the whole document is usually loaded and rewritten on every update.
- The two models are converging. Relational databases now support JSON, and some document databases support joins. A hybrid is likely the future.

## Query Languages

**Declarative vs. imperative.** SQL is attractive because it is **declarative**: you describe the shape of the result you want, not the steps to compute it. This lets the database optimize the query, and declarative code is also easier to parallelize across machines.

**MapReduce** is a fairly low-level programming model for distributed execution. It is one option, not the only way to query data in a distributed system.

**Graph data models** are usually the best fit when data has many many-to-many relationships (social networks, road networks, web links).

- Many well-known algorithms operate on graphs, such as shortest path and PageRank.
- Good declarative graph query languages exist, such as **Cypher**, SPARQL, and Datalog.
- Unlike the old network model, graph databases don't fix the access paths in advance, so applications can evolve much more flexibly.

---

[← Previous: Chapter 1](Chapter-01-Reliable-Scalable-and-Maintainable-Applications) | [Home](Home) | [Next: Chapter 3, Storage and Retrieval →](Chapter-03-Storage-and-Retrieval)
