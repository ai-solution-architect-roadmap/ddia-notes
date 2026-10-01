# Chapter 5: Replication

_Part II: Distributed Data_

> **Key takeaways**
> - There are three replication approaches: **single-leader**, **multi-leader**, and **leaderless**.
> - Asynchronous replication is fast but causes **replication lag**. Use read-after-write, monotonic-read, and consistent-prefix guarantees where users would notice it.
> - Multi-leader and leaderless systems must handle **write conflicts**. Avoiding conflicts is better than resolving them.
> - Leaderless systems use **quorums** (*w + r > n*) and version vectors to detect concurrent writes.

---

Replication means keeping a copy of the same data on several machines connected by a network. It is used to keep data close to users (lower latency), keep the system running when parts fail (higher availability), and serve more reads (higher read throughput).

If data never changed, replication would just mean copying it once. All the difficulty comes from handling **changes**, which forces trade-offs such as synchronous vs. asynchronous replication and how to handle failed replicas.

## Leaders and Followers

In **leader-based** (single-leader, master–slave) replication:

- All **writes** go to the leader, which sends each change to all **followers**.
- Clients can **read** from any replica, including the leader.

Despite its limitations, this is one of the most widely used replication approaches, both in databases and in message brokers.

### Synchronous vs. asynchronous replication

- **Synchronous** replication guarantees that the follower has an up-to-date copy. But if a synchronous follower doesn't respond, the write can't proceed.
- In practice, usually just **one** follower is synchronous and the rest are asynchronous (sometimes called *semi-synchronous*).
- Synchronous replication is normally fast, but there is no upper bound on how long it can take. That's why fully **asynchronous** replication is widely used, even though a leader failure can lose recently acknowledged writes.

### Setting up new followers

1. Take a consistent **snapshot** of the leader's database.
2. Copy the snapshot to the new follower.
3. The follower connects to the leader and requests every change since the snapshot (identified by a position in the replication log).
4. Once it has caught up, the follower is ready.

### Handling node outages

- **Follower failure (catch-up recovery):** the follower keeps a log of the changes it has received. After restarting, it asks the leader for every change since its last processed position. Failures are usually detected with a **timeout**.
- **Leader failure (failover):** a follower is promoted to be the new leader, clients send writes to it, and the other followers start consuming changes from it.

**Dangers of automatic failover**

- With asynchronous replication, the new leader may not have all of the old leader's writes. Throwing those writes away is dangerous, especially if other systems have already seen them.
- The old leader may come back still believing it is the leader. Two nodes acting as leader is called **split brain**.
- Choosing the right timeout is hard. Too short causes unnecessary failovers, too long delays recovery.

### Implementing replication logs

1. **Statement-based replication:** send the SQL statements themselves. Non-deterministic functions (`NOW()`, `RAND()`), auto-increment columns, and side effects cause replicas to diverge. The leader can replace them with fixed values, but other methods are generally preferred.
2. **Write-ahead log (WAL) shipping:** followers receive the leader's low-level append-only log. This couples replication tightly to the storage engine, so leader and followers usually have to run the same database version.
3. **Logical (row-based) log replication:** a separate log format describes changes at the row level, independent of the storage engine's internals. This allows mixed versions (backward compatibility) and makes it easy for external systems to consume (change data capture).
4. **Trigger-based replication:** application code (triggers or stored procedures) decides what to replicate. Very flexible, but it has more overhead and is more prone to bugs and limitations.

## Problems with Replication Lag

Leader-based replication suits workloads with mostly reads and few writes. To scale reads you need many followers, which in practice means **asynchronous** replication, which means followers can be behind. The system is only **eventually consistent**.

Three guarantees reduce the visible effects of lag:

- **Read-after-write consistency (read-your-writes):** users always see the updates they submitted themselves.
  - Read data the user may have modified from the leader, or track the *logical timestamp* of the user's last write and only read from replicas that are at least that up to date.
  - Complications: requests may have to be routed to the leader's datacenter, and the same user may be on several devices at once.
- **Monotonic reads:** a user never sees data go "back in time" after seeing a newer version.
  - Have each user always read from the same replica (for example, chosen by a hash of the user ID). If that replica fails, reads must be rerouted.
- **Consistent prefix reads:** if writes happen in a certain order, anyone reading them sees them in that same order.
  - Make sure causally related writes go to the same partition, so they are applied in order.

If the application can't tolerate lag that may grow to several minutes, design for a **stronger guarantee** instead of pretending replication is synchronous.

## Multi-Leader Replication

With several datacenters, you can have a **leader in each datacenter**. Each leader replicates its changes to the others asynchronously.

**Advantages**

- **Performance:** writes are handled locally, so inter-datacenter latency is hidden from users.
- **Tolerance of datacenter outages:** each datacenter keeps working independently.
- **Tolerance of network problems:** asynchronous replication copes better with an unreliable link between datacenters.

**Main problem:** the same data may be changed concurrently in two datacenters, causing **write conflicts** that must be resolved.

Features such as auto-incrementing keys, triggers, and integrity constraints are often retrofitted onto multi-leader setups and can behave badly. For this reason **multi-leader replication is often considered dangerous and should be avoided when possible**. (Offline-capable clients and collaborative editing are, in effect, multi-leader systems too.)

### Handling write conflicts

The recommended approach is to **avoid conflicts**, for example by routing all writes for a given record through the same leader. When conflicts do happen, converge to a consistent state by:

- Giving each write a unique ID (such as a timestamp) and letting the highest ID win (**last write wins**). This loses data.
- Giving each replica a unique ID and letting writes from the higher-numbered replica win. This also loses data.
- **Merging** the conflicting values (for example, concatenating them).
- **Recording** the conflict in an explicit data structure and resolving it later, in application code or by asking the user.

### Topologies

Common topologies are **all-to-all**, **circular**, and **star**.

- In circular and star topologies, one failed node can interrupt replication for others (a single point of failure).
- All-to-all avoids this, but some network links may be faster than others, so writes can arrive out of order and cause **causality** problems (for example, an update arriving before the insert it depends on). Version vectors can help.

## Leaderless Replication

In write-heavy systems, a leader can become a bottleneck. **Leaderless** (*Dynamo-style*) replication removes the leader: clients send writes, and reads, to **several replicas in parallel**.

### Catching up on missed writes

- **Read repair:** a client reading from several replicas can spot stale values and write the newest value back to the stale replicas.
- **Anti-entropy:** a background process continuously compares replicas and copies missing data across, asynchronously.

### Quorums

With *n* replicas, every write must be confirmed by *w* nodes and every read must query *r* nodes. If

> ***w + r > n***

then at least one of the nodes read must have the latest value.

- Smaller *w* or *r* gives lower latency and higher availability, but a greater chance of reading **stale** values.
- Even with quorums there are edge cases (concurrent writes, sloppy quorums, partial failures), so Dynamo-style databases are generally optimized for **eventual consistency**. Stronger guarantees require transactions or consensus.
- Leaderless replication also suits **multi-datacenter** operation.

### Detecting concurrent writes

If each node simply overwrote a key with whatever value it received last, nodes could end up permanently inconsistent. Ways to converge:

- **Last write wins (LWW):** attach a timestamp to each write and keep the most recent one. This achieves convergence **at the cost of durability**, because concurrent writes are silently discarded.
- **The happens-before relation:** two operations are *concurrent* if neither knows about the other. Only concurrent writes are true conflicts that need resolving. A write that happens after another can simply overwrite it.
- **Merging concurrent values ("siblings"):** keep all concurrent values so nothing is lost, and let the application merge them (for example, a union for a shopping cart, using tombstones to mark deletions).
- **Version vectors:** each replica keeps its own version number and increments it on every write it processes. The collection of version numbers from all replicas is sent with reads and writes, which lets the database tell whether two writes are concurrent or one follows the other.

---

[← Previous: Chapter 4](Chapter-04-Encoding-and-Evolution) | [Home](Home) | [Next: Chapter 6, Partitioning →](Chapter-06-Partitioning)
