# Chapter 3: Storage and Retrieval

_Part I: Foundations of Data Systems_

> **Key takeaways**
> - There are two families of storage engines: **log-structured** (hash indexes, LSM-trees) and **page-oriented** (B-trees).
> - LSM-trees are usually faster for writes. B-trees are usually faster for reads and have more predictable performance.
> - Transaction processing (**OLTP**) and analytics (**OLAP**) have very different access patterns, so analytics usually runs in a separate **data warehouse**.
> - **Column-oriented storage** suits analytic queries that scan many rows but only a few columns.

---

## Data Structures That Power Your Database

The simplest and most efficient write operation is **appending to a file**.

An index speeds up reads but slows down writes, because every write must also update the index. That is why databases don't index everything by default. You choose indexes based on your query patterns.

### Hash indexes

- An in-memory hash map maps every key to its **byte offset** in the data file.
- New records are appended to the current **segment** file. When a segment reaches a certain size, a new one is started.
- A background thread **compacts and merges** old segments, keeping only the latest value for each key, and then deletes the old segments.
- For simple concurrency and crash recovery, there is a single writer thread, and a snapshot of each segment's hash map is saved to disk regularly.

**Why append-only?**

- Sequential writes are much faster than random writes.
- Concurrency and crash recovery are much simpler.
- Merging old segments prevents fragmentation over time.

**Limitations**

- The hash table must fit in memory.
- Range queries are not efficient.

### SSTables and LSM-trees

A **Log-Structured Merge-Tree (LSM-tree)** builds on the same idea but keeps keys **sorted**.

- Writes go into an in-memory balanced tree (for example, a red-black or AVL tree) called the **memtable**.
- When the memtable grows large enough, it is written to disk as a sorted segment file (an **SSTable**).
- **Reads** check the memtable first, then the most recent segment, then the next-older one, and so on.
- To avoid losing the memtable in a crash, every write is also appended immediately to a separate **write-ahead log** on disk.

**Advantages of sorted segments**

- Merging segments is simple and efficient, similar to mergesort.
- You don't need every key in memory. A sparse index is enough, because you can scan between nearby keys.
- Records can be grouped into blocks and compressed before writing.

**Weakness.** Looking up a key that doesn't exist can be slow, because every segment has to be checked. **Bloom filters** help a lot here.

### B-trees

The B-tree is the most widely used index structure and the standard index in almost every relational database.

- The database is divided into fixed-size **pages** (traditionally 4 KB), matching how disks are organized.
- The tree is **balanced**, so a tree with *n* keys always has depth *O(log n)*.
- The **branching factor** is typically several hundred, so most databases fit in a tree three or four levels deep.
- **Crash resilience** comes from a **write-ahead log (WAL)**. **Concurrency** is handled with lightweight locks called **latches**.

**Common optimizations**

- Copy-on-write: write modified pages to a new location and update the parent's pointer, instead of overwriting in place.
- Store abbreviated keys in interior pages to save space and increase the branching factor.
- Add pointers between sibling leaf pages for fast sequential scans.

### LSM-trees vs. B-trees

| LSM-trees: advantages | LSM-trees: disadvantages |
|---|---|
| Faster writes | Slower reads |
| Higher write throughput | Compaction can interfere with ongoing reads and writes |
| Better compression, smaller files on disk | Less predictable performance, especially at high percentiles |
| Lower write amplification | Compaction can use up disk bandwidth |
| | If compaction can't keep up, unmerged segments grow until the disk fills |
| | The same key can exist in several segments, which makes strong transactional locking harder |

### Other indexing structures

- **Secondary indexes:** besides the primary key, databases can have non-unique secondary indexes, which are often essential for efficient joins. Both B-trees and LSM-trees support them.
- **Clustered indexes:** storing the full row inside the index lets some queries be answered from the index alone. A **covering index** stores only some of the columns.
- **Multi-column indexes:** useful when querying several columns at once, such as latitude and longitude in geospatial data. A plain B-tree or LSM-tree can't answer such range queries efficiently. Options include mapping several dimensions onto one value (a space-filling curve) or using specialized structures such as R-trees.
- **Fuzzy indexes:** let you search for *similar* keys, such as misspelled words, when the exact key is unknown. Useful in full-text search, document classification, and machine learning.
- **In-memory databases:** much faster because they avoid the overhead of encoding data for disk. They are less durable on their own and cost more per gigabyte, so they usually write a log or snapshots to disk asynchronously so they can restart. They are great for smaller datasets and make it easy to offer rich data structures such as queues and sets (for example, Redis).

## Transaction Processing or Analytics?

Businesses usually run two kinds of systems:

- **OLTP (Online Transaction Processing):** the day-to-day operational database. Many small reads and writes, each looking up a few records by key.
- **OLAP (Online Analytic Processing):** analytics over large amounts of history, usually in a **data warehouse**.

A **data warehouse** holds a read-only copy of the data from the OLTP systems, so business analysts and data scientists can query it without hurting transaction performance.

- Data arrives through periodic dumps or a continuous stream of updates. Either way it is cleaned and transformed before loading. This process is called **ETL** (Extract, Transform, Load).
- A big advantage of a separate warehouse is that it can be optimized for analytic access patterns. OLTP indexes don't help much with analytic queries.
- Warehouses are usually relational, because SQL is a good fit for analytic queries. Common schemas are the **star schema** and the **snowflake schema**.

## Column-Oriented Storage

Warehouse tables (especially fact tables) are often very wide, with hundreds of columns, but a typical query touches only a few of them. Storing each **column's** values together, instead of each **row's**, means a query only reads the columns it needs. This works for both relational and non-relational models.

**Benefits**

- **Compression:** columns contain many repeated values, so they compress very well (for example, with bitmap encoding).
- **Vectorized processing:** compressed column data fits in CPU cache and can be processed in tight loops, which uses CPU cycles efficiently.

**Cost: writes are harder.** In-place updates are not practical with compressed, sorted columns. The solution is the LSM-tree approach: collect writes in a sorted in-memory store and merge them into the column files on disk in bulk.

**Materialized aggregates.** Warehouses often cache frequently used aggregates (counts, sums) to avoid recomputing them. In relational databases this is done with **materialized views**. A **data cube** is a special case, holding aggregates grouped by several dimensions.

---

[← Previous: Chapter 2](Chapter-02-Data-Models-and-Query-Languages) | [Home](Home) | [Next: Chapter 4, Encoding and Evolution →](Chapter-04-Encoding-and-Evolution)
