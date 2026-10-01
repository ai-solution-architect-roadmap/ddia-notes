# Chapter 6: Partitioning

_Part II: Distributed Data_

> **Key takeaways**
> - Partition (shard) data when it is too big or too busy for one machine. The aim is an even spread of data and load, with no **hot spots**.
> - **Key-range** partitioning keeps range queries efficient but risks hot spots. **Hash** partitioning spreads load evenly but loses range queries.
> - Secondary indexes are partitioned either **by document** (local; reads go to every partition) or **by term** (global; faster reads, slower writes).
> - Never rebalance with `hash mod N`. Use a fixed number of partitions, dynamic partitioning, or partitioning proportional to nodes.

---

When a dataset doesn't fit on one machine, or the query load is too high for one machine, the answer is **partitioning**, also called **sharding**. Each piece of data (record, row, or document) usually belongs to exactly one partition. Some operations still touch several partitions, so complex queries may be **parallelized across many nodes**.

## Partitioning and Replication

Partitioning and replication are usually combined: each partition is stored on several nodes for fault tolerance. The choice of partitioning scheme is mostly independent of the choice of replication scheme.

## Partitioning of Key-Value Data

The goal is to spread data and query load **evenly** across nodes. If partitioning is unfair, some partitions get more data or queries than others. This is called **skew**, and a partition with disproportionately high load is a **hot spot**.

There are two main approaches.

**Partitioning by key range**

- Assign each partition a continuous range of (sorted) keys, like volumes of an encyclopedia.
- Range queries are efficient.
- The boundaries must adapt to the data, and some access patterns still create hot spots. For example, timestamp keys send all of today's writes to one partition.

**Partitioning by hash of key**

- Apply a hash function to the key and give each partition a range of **hash values** instead of keys.
- The hash function doesn't need to be cryptographically strong (Cassandra and MongoDB use MD5, for example), but it must distribute keys evenly.
- Keys are spread much more evenly.
- You **lose efficient range queries**, because neighboring keys end up in different partitions.
- A middle ground is a **compound primary key**: hash only the first column to choose the partition, and keep the remaining columns sorted within it. This supports efficient queries for one-to-many data, such as all posts by one user sorted by time.

### Skewed workloads and relieving hot spots

Hashing can't help when most reads and writes go to the **same key**, such as a celebrity's user ID. Most databases can't fix this automatically, so the **application** has to:

- Add a random prefix or suffix to the hot key, spreading its writes across several partitions.
- Accept the cost: extra bookkeeping to know which keys are split, and reads that must query every split and combine the results.

## Partitioning and Secondary Indexes

Secondary indexes are central to relational databases and common in document databases, but they don't map neatly onto partitions. There are two approaches.

**Partitioning secondary indexes by document (local index)**

- Each partition keeps its own secondary index, covering only the documents in that partition.
- Writes only touch one partition.
- A query on the secondary index must be sent to **all partitions** and the results combined (**scatter/gather**), which can be expensive and amplifies tail latency.

**Partitioning secondary indexes by term (global index)**

- One global index covers all data, and the index itself is partitioned **by term** (the indexed value).
- **Reads are efficient:** a query only goes to the partition holding that term.
- **Writes are slower and more complex:** one document write may update index entries in several partitions.
- In practice, global index updates are often applied **asynchronously**. They are usually fast, but the index may be briefly out of date.

## Rebalancing Partitions

Over time, data and load move between nodes. Rebalancing should meet these minimum requirements:

- Afterwards, load is shared fairly among nodes.
- During rebalancing, the database keeps accepting reads and writes.
- No more data is moved than necessary.

### Strategies

**Don't use `hash mod N`.** It seems obvious, but when the number of nodes *N* changes, most keys have to move to a different node, which makes rebalancing extremely expensive.

**Fixed number of partitions**

- Create many more partitions than nodes (for example, 1,000 partitions for 10 nodes) and assign several to each node.
- When a node is added, it takes whole partitions from existing nodes. Only partition assignments change, never the key-to-partition mapping.
- More powerful nodes can be given more partitions.
- Choosing the number is hard: partitions that are too large make rebalancing and recovery expensive, and partitions that are too small add overhead.

**Dynamic partitioning**

- Works like the top level of a B-tree. When a partition grows beyond a configured size it **splits** in two, and when it shrinks below a threshold it **merges** with a neighbor.
- Partitions can then move between nodes to balance load.
- The number of partitions **adapts to the total data volume**. One caveat: an empty database starts with a single partition, unless you pre-split.

**Partitioning proportionally to nodes**

- Keep a fixed number of partitions **per node**.
- A new node picks a fixed number of existing partitions at random, splits them, and takes half of each.
- Randomness can produce unfair splits, but averaged over many partitions it evens out.

**Automatic vs. manual rebalancing.** Fully automatic rebalancing is convenient but unpredictable. Combined with automatic failure detection, it can cause cascading failures. Keeping a **human in the loop** to approve rebalancing is often wiser.

## Request Routing

How does a client know which node to send a request to? This is a case of the general **service discovery** problem. Routing logic can live in:

1. **Any node:** the client contacts any node, which forwards the request if needed.
2. **A routing tier:** a partition-aware load balancer.
3. **The client:** the client knows the partition assignment and connects directly.

Many distributed databases use a separate coordination service such as **ZooKeeper** to track cluster metadata, including the mapping of partitions to nodes. Routing tiers or clients subscribe to it for updates.

---

[← Previous: Chapter 5](Chapter-05-Replication) | [Home](Home) | [Next: Chapter 7, Transactions →](Chapter-07-Transactions)
