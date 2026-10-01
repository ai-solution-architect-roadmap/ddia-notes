# Chapter 9: Consistency and Consensus

_Part II: Distributed Data_

> **Key takeaways**
> - **Linearizability** makes a replicated system behave as if there were a single copy of the data. It is a recency guarantee, and it is *not* the same as serializability.
> - **Causal consistency** is the strongest model that stays available during network faults. Lamport timestamps and **total order broadcast** provide ordering.
> - **Two-phase commit (2PC)** gives atomic commit across nodes, but it blocks if the coordinator fails.
> - Fault-tolerant **consensus** (Raft, Paxos, Zab) is usually used indirectly, through services like **ZooKeeper**.

---

The simplest way to handle faults is to let the whole service fail and show an error. A better way is to find **general-purpose abstractions with useful guarantees**, implement them once, and let applications rely on them.

**Consensus**, getting all nodes to agree on something, is one of the most important and trickiest of these abstractions.

## Consistency Guarantees

- Most replicated databases provide at least **eventual consistency**: if writes stop, all replicas eventually converge. This is a weak guarantee. Be aware of its limits and don't assume too much, because problems only appear during faults or high concurrency, which makes them hard to test.
- Systems with **stronger guarantees** usually perform worse or are less fault-tolerant, but they are easier to use correctly.

## Linearizability

**Linearizability** (also called *atomic*, *strong*, or *immediate* consistency) makes a system look as if there were **only one copy of the data** and every operation on it were **atomic**.

- **Recency guarantee:** once any read returns a new value, every later read, on any client, must also return that value (or a newer one).
- It says nothing about transaction isolation. It applies to individual operations on single objects.
- A system can be **tested** for linearizability by recording the timing of every request and response and checking whether they can be arranged into a valid sequential order.

### Linearizability vs. serializability

These two are often confused, but they are different guarantees:

| | Serializability | Linearizability |
|---|---|---|
| What it is | An **isolation** property of **transactions** | A **recency** guarantee on reads and writes of a **single object** |
| Promise | Transactions behave as if executed in *some* serial order | Every read sees the most recent write |

- A database may provide both. That combination is called **strict serializability** (or strong one-copy serializability).
- **Two-phase locking** and **actual serial execution** are typically linearizable.
- **Serializable snapshot isolation is not linearizable**, by design: it reads from a consistent snapshot, which may not include the very latest writes.

### When linearizability is needed

- **Locking and leader election:** all nodes must agree on who holds the lock.
- **Constraints and uniqueness guarantees:** such as unique usernames, or a bank balance never going negative.
- **Cross-channel timing dependencies:** for example, a file is written to storage and a message telling someone to process it is sent through a queue. The reader must see the file.

Without a recency guarantee, race conditions are possible, and linearizability may be the simplest way to avoid them. Some constraints, such as foreign-key or attribute constraints, can be enforced without it.

### Implementing linearizable systems

- A system with a **single copy** of the data is linearizable, but not fault-tolerant.
- **Single-leader replication** is potentially linearizable, if reads go to the leader or to synchronously updated followers.
- **Consensus algorithms** are linearizable, and they are how fault-tolerant linearizable storage (like ZooKeeper or etcd) is built.
- **Multi-leader replication** is generally **not** linearizable.
- **Leaderless (Dynamo-style)** replication is **probably not** linearizable. It can be made linearizable in limited cases (synchronous read repair, read-before-write), at a significant performance cost.

### The cost of linearizability

- **CAP theorem:** during a network partition you must choose between linearizability (consistency) and availability. Applications that **don't need** linearizability can tolerate network problems better, because each replica can keep serving requests on its own.
- Even without network faults, linearizability is **slow**: response times are at least proportional to the uncertainty in network delays. That is why few systems are linearizable in practice. It is often dropped for **performance**, not just for fault tolerance.

## Ordering Guarantees

Ordering preserves **causality** (a question comes before its answer, a write comes before the read that sees it). A system that respects the order imposed by causality is **causally consistent**. Snapshot isolation is one example: each snapshot is consistent with causality.

### Causal order vs. total order

- **Causality defines a partial order.** Two operations are ordered if one happened before the other. Concurrent operations aren't ordered and may be processed in any order.
- **Linearizability defines a total order.** Any two operations can be compared, because there are no truly concurrent operations: everything behaves like a single timeline.
- Linearizability **implies** causality, so it is one way to preserve causal order, but it is stronger than necessary.
- **Causal consistency is the strongest consistency model that doesn't slow down because of network delays and stays available during network failures.**

### Sequence numbers and Lamport timestamps

- Causal consistency means tracking causal dependencies across the **whole database**, not just per key. **Version vectors** can do this, but tracking every dependency can be impractical.
- A more compact approach is to order events with **sequence numbers** or **logical timestamps**, which are small and give a **total order** consistent with causality.
- The best-known way to generate them is **Lamport timestamps**:
  - Each timestamp is a pair: *(counter, node ID)*.
  - Every node and every client remembers the **maximum** counter value it has seen and includes it in every request.
  - When a node receives a request or response with a higher counter, it immediately raises its own counter to that value.
- **Lamport timestamps vs. version vectors**
  - **Version vectors** can tell whether two operations are **concurrent** or whether one **causally depends** on the other.
  - **Lamport timestamps** always enforce a **total order**, so you **can't** tell from them whether two operations were concurrent. In exchange, they are much more compact.
- Limitation: a total order of timestamps is only known **after the fact**. It can't, for example, decide on the spot whether a username is already taken. That needs total order broadcast.

### Total order broadcast

**Total order broadcast** (atomic broadcast) is a protocol for exchanging messages between nodes that guarantees:

- **Reliable delivery:** no messages are lost. If a message reaches one node, it reaches all nodes.
- **Totally ordered delivery:** every node delivers messages in **the same order**.

Uses include:

- **Database replication** (state machine replication): every replica applies the same writes in the same order.
- **Serializable transactions:** every node processes transactions in the same order.
- A **log** of messages, such as a replication log, transaction log, or write-ahead log.
- **Locks** that issue **fencing tokens**: the sequence number serves as the token.

Total order broadcast can be used to build linearizable storage, and the reverse is also true. Both are equivalent to consensus.

## Distributed Transactions and Consensus

The goal of consensus is to get several nodes to **agree on something**. It is essential for:

- **Leader election:** all nodes must agree on the leader, to avoid split brain.
- **Atomic commit:** in a transaction spanning several nodes, all must either commit or abort.

In theory, consensus is **impossible** in an asynchronous system where a node may crash (the FLP result). In practice it is possible, because real systems can use **timeouts** and other ways of detecting crashes.

### Atomic commit and two-phase commit (2PC)

- On a single node, atomic commit is easy: it depends on the order in which data is durably written to disk (the commit record comes last).
- Across several nodes it is hard. Sending a commit request to each node independently isn't enough, because some may commit and others fail.
- Most NoSQL datastores don't support distributed transactions. Various relational databases do.

**How 2PC works**

1. The application reads and writes data on several database nodes (**participants**) as usual, under a global transaction ID.
2. When it is ready to commit, a **coordinator** (transaction manager) starts **phase 1, prepare**: it asks every participant whether it can commit.
3. If **all** participants vote "yes", the coordinator records its decision in its log and starts **phase 2**, sending **commit** to everyone. If any participant votes "no" or times out, it sends **abort**.

**Two points of no return**

- When a participant votes **yes**, it promises it will definitely be able to commit later, whatever happens (crashes, power failures, and so on).
- Once the coordinator **decides**, the decision is irrevocable. A committed transaction can only be undone by a separate **compensating transaction**.

**Coordinator failure**

- If the coordinator fails **before** the prepare phase, participants can safely abort.
- If it fails **after** a participant voted yes, that participant is **in doubt**. It can neither commit nor abort on its own, and must **wait** for the coordinator to recover and read its log.
- **Three-phase commit (3PC)** avoids this blocking in theory, but it assumes bounded network delay and response times, so it doesn't work in practice.

### Distributed transactions in practice

Distributed transactions provide important safety guarantees, but they are criticized for causing operational problems and hurting performance (for example, MySQL's distributed transactions are reported to be over 10 times slower than single-node ones). There are two kinds:

- **Database-internal** distributed transactions, where all nodes run the same software. These can use optimizations and often work well.
- **Heterogeneous** distributed transactions, where participants are different technologies (two different databases, or a database and a message broker) implementing a common standard such as **XA**.

**Holding locks while in doubt.** An in-doubt transaction can't release its locks, which can make large parts of the application unavailable. **Orphaned** in-doubt transactions (whose coordinator lost its log) need either:

- **Manual intervention** by an administrator, or
- **Heuristic decisions:** a participant decides on its own to commit or abort. This can break atomicity and should only be an emergency escape hatch.

### Fault-tolerant consensus

Formally, a consensus algorithm must satisfy:

1. **Uniform agreement:** no two nodes decide differently.
2. **Integrity:** no node decides twice.
3. **Validity:** a decided value must have been proposed by some node.
4. **Termination:** every node that doesn't crash eventually decides some value.

Termination is a liveness property, and it requires **a majority of nodes to be working**.

**Algorithms.** The best-known fault-tolerant consensus algorithms are **Viewstamped Replication (VSR)**, **Paxos**, **Raft**, and **Zab**. Most implement **total order broadcast** directly, which is equivalent to repeated rounds of consensus. (Multi-Paxos is the total-order variant of Paxos.)

**Epochs and quorums**

- All of these protocols use a **leader** internally, but they don't guarantee a unique leader at all times. They guarantee only one leader **per epoch** (called a *ballot number* in Paxos, *view number* in VSR, *term number* in Raft).
- This means **two rounds of voting**: one to elect a leader for a new epoch, and one to vote on that leader's proposals. The two quorums must overlap.

**Limitations of consensus**

- It needs a **strict majority** to operate. Tolerating *f* failures takes 2*f* + 1 nodes.
- Most algorithms assume a **fixed set** of voting nodes, so changing membership is hard.
- It relies on **timeouts** to detect failed nodes, which causes unnecessary leader elections when network delays vary.
- It is sensitive to network problems, and can spend a lot of time electing leaders instead of doing useful work.

### Membership and coordination services

Tools like **ZooKeeper** and **etcd** implement fault-tolerant consensus and expose it through a small, useful API. They provide:

- **Linearizable atomic operations** (such as compare-and-set, used for locks).
- **Total ordering of operations** (such as fencing tokens through transaction IDs).
- **Failure detection** (through session heartbeats and ephemeral nodes).
- **Change notifications** (watches).

They are typically used for **allocating work to nodes** (leader election, partition assignment), **service discovery**, and **membership**. Applications usually use them **indirectly**, through other systems such as Kafka, HBase, or Hadoop YARN.

---

[← Previous: Chapter 8](Chapter-08-The-Trouble-with-Distributed-Systems) | [Home](Home) | [Next: Chapter 10, Batch Processing →](Chapter-10-Batch-Processing)
