# Chapter 7: Transactions

_Part II: Distributed Data_

> **Key takeaways**
> - A transaction groups several reads and writes into one unit that either **commits** entirely or **aborts** entirely.
> - **Read committed** prevents dirty reads and dirty writes. **Snapshot isolation** also prevents read skew. Neither prevents **write skew** or **phantoms**.
> - **Serializable** isolation prevents all race conditions. It is implemented by actual serial execution, two-phase locking (2PL), or serializable snapshot isolation (SSI).
> - Many databases that call themselves "ACID" use weak isolation by default. Know which level you actually have.

---

A **transaction** is a way for an application to group several reads and writes into one logical unit. The whole unit either succeeds (**commit**) or fails (**abort**, **rollback**). If it fails, the application can safely retry. Transactions exist to simplify the programming model by letting the database handle failures and concurrency.

## The Slippery Concept of a Transaction

Two opposite claims are both exaggerations:

- "Any large-scale system must abandon transactions for performance and availability."
- "Transactional guarantees are essential for every serious application."

Transactions have both benefits and limitations, and the right choice depends on the application.

### ACID

- **Atomicity:** the transaction can't be broken into smaller parts. If a fault happens partway through a series of writes, the transaction is **aborted** and all its writes are discarded, so the client can safely retry. ("Abortability" would be a better name.)
- **Consistency:** the database is in a "good state" because certain invariants always hold. This is mainly a property of the **application**, not the database: the database can enforce some constraints, but the application defines what "valid" means.
- **Isolation:** concurrently running transactions are isolated from each other. Ideally each behaves as if it were the only transaction running (serializability).
- **Durability:** once a transaction commits, its data won't be lost. On a single node that means writing to non-volatile storage (with a write-ahead log). In a replicated database it means copying the data to some number of nodes.

### Single-object and multi-object operations

- Storage engines almost always provide atomicity and isolation for **a single object**: atomicity through a log for crash recovery, isolation through a lock on each object.
- **Multi-object transactions** are needed when:
  - A row has a foreign key referencing a row in another table.
  - Denormalized data or secondary indexes must be updated together with the main record.
- Many distributed datastores dropped multi-object transactions because they are hard to implement across partitions.

### Handling errors and aborts

- Leaderless-replication datastores work on a "best effort" basis: they won't undo what has already been done, so **recovering from errors is the application's job**.
- Retrying an aborted transaction is useful but not perfect:
  - If the transaction succeeded but the acknowledgment was lost on the network, a retry performs it **twice** (unless there is application-level deduplication).
  - If the error is permanent (such as a constraint violation), retrying is pointless.
  - If the failure was caused by overload, retrying makes things worse. Use exponential backoff.

## Weak Isolation Levels

Concurrency bugs are hard to find through testing. Databases have long tried to hide concurrency problems through **transaction isolation**, ideally **serializable** isolation. Because serializability costs performance, many systems use **weaker isolation levels**, which protect against *some* concurrency problems but not all. Many popular relational databases described as "ACID" use weak isolation by default.

### Read committed

The most basic isolation level. It guarantees:

- **No dirty reads:** a transaction never sees another transaction's **uncommitted** writes.
  - Why it matters: if a transaction updates several objects, others might otherwise see some updates and not others. And if it later aborts, others would have seen data that never really existed.
  - Implementation: locking out readers would hurt response times, so databases remember **both the old committed value and the new uncommitted value**, and give readers the old value until the writer commits.
- **No dirty writes:** a transaction never overwrites another transaction's **uncommitted** write.
  - Why it matters: concurrent multi-object updates could otherwise interleave and produce inconsistent results.
  - Implementation: most databases use **row-level locks**.
  - It still doesn't prevent every race condition (see lost updates below).

### Snapshot isolation and repeatable read

Read committed still allows **read skew** (a *non-repeatable read*): a transaction reads different parts of the database at different points in time and sees an inconsistent picture. This is a serious problem for:

- **Backups**, which take a long time while writes continue.
- **Analytic queries and integrity checks**, which scan large parts of the database.

The solution is **snapshot isolation**. Each transaction reads from a **consistent snapshot** of the database, as if it had been frozen at the moment the transaction started.

- It is implemented with **write locks** to prevent dirty writes. **Readers take no locks**, so readers never block writers and writers never block readers.
- It uses **multi-version concurrency control (MVCC)**. Where read committed needs only two versions of an object (committed and uncommitted), snapshot isolation keeps **several committed versions**, one for each in-progress snapshot.
- **Indexes** under snapshot isolation: either have the index point to *all* versions of an object and filter out invisible ones, or use an append-only/copy-on-write B-tree that never overwrites pages, so each root is a consistent snapshot.

### Preventing lost updates

A common pattern is the **read-modify-write** cycle (read a value, change it, write it back). If two transactions do this concurrently, one update can overwrite the other and be **lost**. Solutions:

- **Atomic write operations:** usually the best option when the change can be expressed this way, for example `UPDATE counters SET value = value + 1 WHERE key = 'foo';`. Not every update fits this form, and ORMs make it easy to write unsafe read-modify-write code by accident.
- **Explicit locking:** the application locks the rows it is about to update: `SELECT * FROM ... WHERE ... FOR UPDATE;`
- **Automatic lost-update detection:** let transactions run in parallel, and have the database **detect** a lost update and abort the offending transaction, forcing it to retry. The database can do this efficiently together with snapshot isolation, and because it is automatic it is less error-prone. (PostgreSQL's repeatable read does this; MySQL/InnoDB's does not.)
- **Compare-and-set:** only allow the update if the value hasn't changed since it was read: `UPDATE ... SET content = 'new' WHERE id = 1 AND content = 'old';`. This can be unsafe if the database lets the `WHERE` clause read from an old snapshot.
- **Conflict resolution (replicated databases):** locks and compare-and-set assume a single up-to-date copy, so they don't work with multi-leader or leaderless replication. Instead, allow concurrent writes to create conflicting versions (*siblings*) and resolve or merge them in application code. Commutative atomic operations, such as incrementing a counter, work well here. Last-write-wins is the default in many systems, and it loses updates.

### Write skew and phantoms

**Write skew** happens when two transactions read the same objects, make a decision based on what they read, and then update **different** objects. Each is fine on its own, but together they break an invariant that serial execution would have kept. (Classic example: two doctors both go off call at the same time, leaving nobody on call.)

- Atomic single-object operations and automatic lost-update detection **don't help**, because different objects are written.
- Options:
  - Use **serializable isolation** (the best option).
  - Enforce the invariant with constraints, triggers, or materialized views where the database supports it. Multi-object constraints are rarely supported.
  - Explicitly lock the rows the transaction depends on with `SELECT ... FOR UPDATE`.

**Phantoms** occur when a write in one transaction changes the result of a **search query** in another transaction. If the check is for the *absence* of rows (such as "is this room free at 2 pm?"), `FOR UPDATE` has nothing to lock, because the rows don't exist yet.

- **Materializing conflicts** works around this by creating rows that act as locks (for example, a table of every room and time slot). It is error-prone and lets concurrency control leak into the data model, so **serializable isolation is much preferred**.

## Serializability

Isolation levels are hard to understand, it's hard to tell from looking at code whether it is safe at a given level, and there are few good tools to help. The simple answer is **serializable isolation**: the strongest level, which guarantees that the result is the same as if transactions ran **one at a time**. It prevents **all** of the race conditions above.

There are three main implementations.

### 1. Actual serial execution

Literally execute one transaction at a time, on a single thread.

- This only became feasible fairly recently, because:
  - RAM got cheap enough to keep the active dataset in memory.
  - Designers realized that OLTP transactions are usually short and make few reads and writes. Long-running analytic queries can run on a consistent snapshot outside the serial loop.
- A single thread can outperform concurrent designs because it avoids locking overhead, but **throughput is limited to one CPU core**.
- Transactions must be submitted as **stored procedures** (the whole transaction code sent ahead of time), because the loop can't wait for interactive back-and-forth with the application. This performs well, especially in databases that support general-purpose languages for stored procedures.
- **Partitioning** lets you scale to several cores or nodes, as long as most transactions touch only **one partition**. Cross-partition transactions are much slower.

### 2. Two-phase locking (2PL)

For about 30 years, 2PL was essentially the only widely used algorithm for serializability.

- It is like the lock used to prevent dirty writes, but much stricter: **writers block readers and readers block writers**.
- There are two lock modes: **shared** (for readers) and **exclusive** (for writers). Locks are held until the transaction commits or aborts.
- **Predicate locks** or **index-range locks** also lock rows that match a search condition, including rows that don't exist yet. This is how 2PL prevents phantoms.
- **Deadlocks** are common. The database detects them automatically and aborts one of the transactions.
- **Big downside: performance.** Throughput is much worse than with weak isolation, latency is unstable, and high percentiles can be very slow.

### 3. Serializable snapshot isolation (SSI)

A relatively new, **optimistic** concurrency control technique (first described in 2008). It is fast enough that it may become the new default.

- It builds on snapshot isolation. Transactions run **without blocking** even when something potentially dangerous happens.
- At **commit time**, the database checks whether the transaction's reads were affected by concurrent writes (stale premises). If so, it aborts the transaction, which must then retry.
- Compared with 2PL, readers don't block writers, so performance is much more predictable.
- Compared with serial execution, it can use **many cores and partitions**.
- It performs poorly under **high contention**, where many transactions abort. Commutative atomic operations can reduce contention.

---

[← Previous: Chapter 6](Chapter-06-Partitioning) | [Home](Home) | [Next: Chapter 8, The Trouble with Distributed Systems →](Chapter-08-The-Trouble-with-Distributed-Systems)
