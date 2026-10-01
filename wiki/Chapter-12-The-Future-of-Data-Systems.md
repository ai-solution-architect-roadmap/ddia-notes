# Chapter 12: The Future of Data Systems

_Part III: Derived Data_

> **Key takeaways**
> - No single tool does everything well, so combine specialized systems, with a **log-based** system of record feeding derived views.
> - **"Unbundle the database"**: build database-like behavior from loosely coupled components connected by asynchronous event logs.
> - "Consistency" mixes two requirements: **timeliness** (can be relaxed) and **integrity** (must never be violated).
> - Get correctness from **end-to-end** request IDs, **idempotence**, immutable events, and continuous **auditing**.
> - Engineers are responsible for the consequences of their systems: bias, surveillance, and data retention.

---

## Data Integration

### Combining specialized tools

No single piece of software suits every use case. Trying to do everything in one product almost guarantees a poor implementation, so applications have to **compose several tools** to meet their goals.

### Reasoning about dataflows

When the same data must be kept in several storage systems to serve different access patterns, concurrency problems such as conflicting writes appear.

- The best approach, where possible, is to **funnel all input into a single system of record** that decides the **order of every write** (for example, a log with total order broadcast).
- Other systems then **derive** their data from it by processing the writes in the same order, which is much simpler and more robust.

### Derived data vs. distributed transactions

- **Distributed transactions** are the classic way to keep different data systems consistent. Their main advantage is that they offer **linearizability** (for example, reading your own writes). But they come with significant limitations and overhead (see [Chapter 9](Chapter-09-Consistency-and-Consensus)).
- **Log-based derived data** is asynchronous and offers no timeliness guarantee by default, but it is much more robust and scales better.
- In the absence of widely supported, good distributed-transaction protocols, **log-based derived data is the most promising approach** for integrating different data systems.

### Batch and stream processing

Both batch and stream processing have a strong **functional** flavor: deterministic functions, immutable input, append-only output. This is good for fault tolerance and makes it easier to reason about dataflows across an organization.

**Gradual evolution.** Derived views make it possible to restructure a dataset gradually. You maintain the old and new schemas side by side as two independent derived views, shift users over a little at a time, and always have a working system to fall back to.

**The lambda architecture.** Some systems need fast, **approximate** results (from stream processing) and later **correct, reliable** results (from batch processing). The lambda architecture:

- Records incoming data as **immutable events** in an ever-growing dataset.
- Runs a **batch** system and a **stream** system **in parallel**, each producing its own derived view.

Its main downside is the **operational complexity** of maintaining and debugging the same logic in two different systems. Newer systems that unify batch and stream processing (such as Flink or Beam) reduce the need for it.

## Unbundling Databases

A database is made up of many interacting components (storage, indexes, materialized views, replication, triggers) that we normally take for granted. Keeping such components consistent **across** different systems traditionally needs synchronous distributed transactions, with all their overhead. A more robust and practical approach is an **asynchronous event log with idempotent consumers**. This leads to the idea of **unbundling the database**.

**What unbundling means**

- Build systems that act like a database as a whole, but are actually **loosely coupled components**, each doing one thing well, connected by event logs. (This is the Unix philosophy applied to data.)
- **Robustness:** an outage or slowdown in one component doesn't take everything else down. The log buffers messages until the component recovers.
- **Breadth:** combining several specialized stores gives good performance across a **much wider range of workloads** than any single product can.

**Separating storage from application code**

- It makes sense to have some parts of a system specialize in **durable data storage** and others in **running application code**. The two interact while staying independent.

**Dataflow vs. microservices**

- In a typical microservices design, services call each other through **synchronous request/response** (RPC).
- In a dataflow design, communication is **one-directional and asynchronous**: instead of querying another service over RPC, a service subscribes to its stream of changes and **joins** it with its own events locally. This is faster and more robust, since it has no runtime dependency on the other service.

**All the way to the client.** These ideas aren't limited to the datacenter. Stream processing and messaging can extend to **end-user devices**: clients subscribe to changes and keep their local state up to date (offline-first apps, live-updating UIs).

## Aiming for Correctness

### Transactions aren't the only path

Transactions have been the standard way to build correct applications for over four decades. Some systems have dropped them because of their overhead, but transactions **aren't going away**. Even so, correctness can also be achieved within a **dataflow** architecture.

### The end-to-end argument

- Even systems with strong safety properties (such as serializable transactions) **can't guarantee** freedom from data loss or corruption, because application bugs and human error still happen.
- Recovery is much easier if faulty code **can't destroy good data**. Immutable, append-only data helps here.
- **Idempotence** is one of the most effective tools: make operations safe to repeat.
- Database-level guarantees (such as 2PC or TCP-level duplicate suppression) **aren't enough** to ensure an operation happens exactly once, because duplicates can come from the client, such as a user resubmitting a form. You need to think about the **end-to-end flow** of the whole request:
  - Generate a **unique request ID** on the client.
  - Pass it all the way to the database, and use it to **suppress duplicates**.
  - There is no standard abstraction for this yet, so applications must do it themselves.

### Enforcing constraints

- **Uniqueness** constraints need consensus. The usual way is to send all requests for a given value through **a single leader** or partition.
- In an **unbundled** system, a **partitioned log** achieves the same thing: route all requests for a given username to the same partition, and process them sequentially there.

**Multi-partition operations without atomic commit.** Traditionally, a transaction across several partitions (such as a payment that debits one account and credits another) needs atomic commit. Equivalent correctness can be achieved with **partitioned logs**:

1. The client gives the request a **unique ID**, and the request is atomically appended to a log partition chosen by that ID.
2. A stream processor reads the request log and emits messages (with the request ID) to the **output streams** for each affected account, such as a debit instruction and a credit instruction.
3. Further processors consume those streams, apply the changes, and **deduplicate** by request ID.

### Timeliness and integrity

The word "consistency" mixes up two different requirements:

- **Timeliness:** users see the system in an up-to-date state. Violating timeliness gives *eventual consistency*. It is temporary and can be fixed by waiting and retrying.
- **Integrity:** no corruption, no lost or contradictory data, derived data that correctly reflects its source. Violating integrity gives **perpetual inconsistency**, which can be catastrophic.

**Integrity matters much more than timeliness.**

Event-based dataflow systems **decouple** the two. They don't guarantee timeliness unless you build consumers that wait, but they can **preserve integrity** by:

- Representing the content of a write operation as a **single message**, which can be written atomically.
- Deriving all other state updates from that message using **deterministic** derivation functions.
- Passing a **client-generated request ID** through every stage of processing, enabling end-to-end duplicate suppression.
- Keeping messages **immutable**, so derived data can be reprocessed when needed.

Many applications can also use **loosely interpreted constraints**: allow a temporary violation (such as overbooking) and fix it afterwards with an apology or compensation. This often makes business sense and avoids coordination.

### Trust, but verify: auditing

- Don't trust software guarantees blindly, however widely the software is used. Bugs always creep in, and hardware can silently corrupt data.
- We need ways to find out **automatically and continuously** whether data has been corrupted, so we can fix it and trace the cause. This is **auditing**.
- **Event-based systems** are easier to audit than transaction-based systems. The event log explains **why** each change was made, and a **deterministic, well-defined dataflow** makes it easy to debug and trace execution, for example by rerunning derivations and comparing results.
- Ideally, check that the **entire derived-data pipeline is correct end to end**. That gives confidence in every disk, network, service, and algorithm along the way.
- Cryptographic tools such as Merkle trees and transparency logs may become more widely used for verifiable integrity.

## Doing the Right Thing

- Every system is built for a purpose, and has both **intended and unintended consequences**. Engineers are responsible for thinking these through carefully.
- **Predictive analytics** (often based on machine learning) can be badly misleading. If the input contains **systematic bias**, the system will most likely learn and **amplify** that bias in its output, and an algorithm's decisions can be hard to appeal.
- **Privacy and tracking.** Collecting large amounts of behavioral data can turn into surveillance. Users should keep meaningful control over their data, and consent should be informed.
- **Data retention.** Don't keep data forever. **Purge it** as soon as it is no longer needed. This conflicts with the idea of immutability, but a promising approach is to enforce access control through **cryptographic protocols** (for example, deleting a key so the data becomes unreadable), not just through policy.

---

[← Previous: Chapter 11](Chapter-11-Stream-Processing) | [Home](Home)
