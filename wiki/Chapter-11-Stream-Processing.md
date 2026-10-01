# Chapter 11: Stream Processing

_Part III: Derived Data_

> **Key takeaways**
> - A stream is an **unbounded** sequence of small, immutable **events**. Producers and consumers are connected by a messaging system.
> - **Log-based brokers** (Kafka) combine durable storage with low-latency delivery and keep messages in order within a partition.
> - **Change data capture** and **event sourcing** turn database writes into a stream, so derived systems (indexes, caches) stay in sync.
> - Time is hard in streams. Distinguish **event time** from **processing time**, and choose the right **window** type.
> - **Exactly-once** semantics come from micro-batching or checkpoints, atomic commits, and **idempotent** writes.

---

Batch processing has two main drawbacks: it must read the **entire input** before producing results, and changes in the input only show up in the output after a long delay, sometimes a day or more. **Stream processing** solves both by processing each event shortly after it happens, usually within a second.

## Transmitting Event Streams

- A stream is a sequence of **events**: small, self-contained, **immutable** objects, each recording something that happened at a point in time (with a timestamp).
- An event is produced once by a **producer** (publisher) and may be processed by several **consumers** (subscribers).
- Related events are grouped into a **topic** (or stream).

A database could connect producers and consumers, but **continuous polling** is expensive. It is better for consumers to be **notified** when new events arrive, which usually requires specialized tools: **messaging systems**.

### Messaging systems

A messaging system lets **many producers** send messages to the same topic and **many consumers** receive messages from it. Two questions help compare them:

1. **What happens if producers send faster than consumers can process?**
   - **Drop** messages.
   - **Buffer** them in a queue (and decide what happens when the buffer fills up).
   - Apply **backpressure**: block the producer until there is room.
2. **What happens if nodes crash or go offline?**
   - For **durability**, combine writing to disk with replication, at the cost of lower throughput and higher latency.

### Direct messaging (no broker)

Examples: **UDP multicast**, broker-less libraries such as ZeroMQ, webhooks, or direct HTTP/RPC calls.

- Fast and simple.
- **Biggest drawback:** the application must be aware that messages can be lost, and handle it.

### Message brokers

The more common option is a **message broker** (message queue): a server that producers and consumers both connect to. Traditional (JMS/AMQP-style) brokers:

- **Delete a message** once it has been successfully delivered.
- Support subscribing to topics, sometimes to a subset of messages by pattern.
- **Notify** consumers when new data arrives.

With several consumers on one topic, a broker can:

- **Load-balance:** each message goes to one consumer, which spreads the work.
- **Fan out:** every message goes to all consumers.
- Combine both, using consumer groups.

Brokers use **acknowledgments** to make sure a message was processed before removing it. Redelivery after a crash, combined with load balancing, can change the order of messages.

### Log-based message brokers

A **log-based broker** (such as Apache **Kafka**, Amazon Kinesis, or DistributedLog) combines the **durable storage** of a database with the **low-latency notifications** of messaging.

- The producer appends messages to a **log**. Consumers read the log sequentially, each tracking its own **offset**.
- The log is **partitioned** across machines for high throughput and **replicated** for fault tolerance.
- Messages are **ordered within a partition**.
- Messages aren't deleted on consumption, so consumers can **replay** old messages.

**When to use which**

- **Log-based brokers:** high message throughput, fast processing per message, and **ordering matters**. They are ideal for stream processors that build derived state or output streams.
- **Traditional JMS/AMQP brokers:** messages are **expensive to process**, you want to parallelize message by message, and ordering matters less.

## Databases and Streams

A database write is itself an **event** that can be captured, stored, and processed. Seeing a database as a stream opens up powerful ways to **integrate systems**.

### Keeping systems in sync

The same data often lives in several places (a database, a search index, a cache, a warehouse), and they must stay in sync.

- **Dual writes** (the application writes to each system separately) are fragile:
  - Concurrent writes can arrive in different orders at different systems (**race conditions**), leaving them permanently inconsistent unless you add concurrency detection.
  - If one write fails and another succeeds, the systems diverge. Preventing that requires **atomic commit**.

### Change data capture (CDC)

**Change data capture** observes every change written to a database and extracts it in a form that can be replicated to other systems, such as a search index.

- The database becomes the **leader**, and derived systems become its **followers**.
- It is usually implemented by **parsing the database's replication log**.
- Start a new consumer from a consistent **snapshot** of the database, then apply the changes after it.
- **Log compaction** (keeping only the latest value for each key) stops the log from growing forever, while still letting a new consumer rebuild the full state.

### Event sourcing

**Event sourcing** is similar to CDC but works at a different level:

- The application records the **user's actions** as immutable events (for example, "seat reserved"), not just their effect on the database (such as "row updated").
- New side effects and views can easily be **chained off existing events**.
- The application must turn the event log into current state **deterministically**. Snapshots help avoid replaying the full history.
- Log compaction is harder, because later events don't simply overwrite earlier ones.
- **Commands vs. events:** a command is a request that may still be rejected. Once validated, it becomes an immutable event (a fact).

### State, streams, and immutability

Both CDC and event sourcing are built on **immutability**. Mutable state and an append-only log of immutable events don't contradict each other: the current state is the result of applying the events in the log. Benefits:

- Easier to **diagnose bugs** and recover from mistakes, because history is never overwritten.
- Captures **more information** than the current state alone (for example, items added to a cart and then removed).
- Easier to **evolve** the application over time.
- Separating the format in which data is **written** (the event log) from the format in which it is **read** (derived views) lets you have **many read-optimized views** of the same data. This is the idea behind CQRS (Command Query Responsibility Segregation).

**Downsides and limitations**

- Consumers of the event log are usually **asynchronous**, so a user may not **read their own writes** right away. Options:
  - Update the read view **synchronously** with the log append (this needs a transaction).
  - Implement linearizable storage using **total order broadcast**.
  - If the event log and application state are **partitioned the same way**, a single-threaded consumer per partition needs no concurrency control for writes.
- Immutable history can grow **very large**, especially with frequent updates and deletes, which hurts performance.
- Sometimes data must be **truly deleted** for legal or administrative reasons (such as privacy regulations). In an immutable system this is surprisingly hard.

## Processing Streams

There are three things you can do with a stream:

1. **Write it to a datastore** (a database, cache, or search index) so clients can query it later.
2. **Push it to users** (alerts, emails, real-time dashboards).
3. **Process it into another output stream**. This is a stream processor (operator or job).

Stream processing is like batch processing in that it reads its input **read-only** and writes its output **append-only**. But a stream **never ends**, so:

- Operations such as **sorting** the whole input are impossible. Sort-merge joins don't apply.
- **Restarting from the beginning** after a failure isn't an option for a job that has run for years.

### Uses of stream processing

- **Monitoring:** fraud detection, financial trading, factory machine status, and alerting on system metrics.
- **Complex event processing (CEP):** queries are stored for the long term, and the engine continuously **matches incoming events** against them, emitting a "complex event" when a pattern is found.
- **Stream analytics:** aggregations and statistics over windows (rates, rolling averages).
- **Maintaining materialized views:** keeping caches, search indexes, and other derived data up to date.

### Reasoning about time

Analytics usually aggregate over a period of time, called a **window**.

- Many frameworks use the processor's **local clock** (processing time) to assign events to windows. This is simple, but it breaks down whenever there is significant **processing lag**, such as a backlog after a restart, because events end up in the wrong windows.
- Using the **event time** (when the event actually happened) is more accurate, but raises a new problem.

**When is a window complete?** You can never be sure that no more events for a window are still on their way.

- A common approach is to close the window after a **timeout**: no new events for that window for a while.
- Events that arrive later (**stragglers**) can either be **ignored** (and counted as a metric) or trigger a published **correction** to the result.

**Whose clock?** Device clocks can be wrong. To estimate the true event time, log three timestamps:

1. When the event **occurred**, by the device clock.
2. When the event was **sent** to the server, by the device clock.
3. When the event was **received**, by the server clock.

The difference between (2) and (3) estimates the **offset** between the device clock and the server clock, which can then be applied to (1).

### Types of windows

| Window | Length | Overlap | Example |
|---|---|---|---|
| **Tumbling** | Fixed | None. Each event belongs to exactly one window | 10:00–10:05, 10:05–10:10 |
| **Hopping** | Fixed | Yes. Windows advance by a fixed "hop" | 5-minute windows every 1 minute |
| **Sliding** | Fixed interval between events | Yes, by definition | All events within 5 minutes of each other |
| **Session** | Variable | None | Events from one user close together in time, ending after inactivity |

### Stream joins

Like batch jobs, stream processors need joins. There are three types:

- **Stream–stream join (window join):** match related events from two streams within a time window, such as a search and a click on its results.
- **Stream–table join (stream enrichment):** enrich each event with data from a database table. The processor keeps a local copy of the table, kept up to date with CDC.
- **Table–table join (materialized view maintenance):** both inputs are changelogs, and the output is a continuously updated view of the join.

All three require the processor to keep **state** based on one input and query it for messages from the other. That makes **ordering** important: if events from different streams arrive in a different order, the join result can differ (a *slowly changing dimension* problem). A common fix is to give each version of the joined record a **unique identifier**.

### Fault tolerance

The goal is **exactly-once** (or *effectively-once*) semantics: the output is as if every event were processed once, even if some were processed more than once internally.

- **Micro-batching:** split the stream into small blocks (around one second) and treat each as a mini batch job (Spark Streaming).
- **Checkpointing:** periodically save rolling state checkpoints to durable storage, and restart from the latest one after a crash (Flink).
- **Atomic commit:** make sure every output and side effect of processing an event happens **if and only if** processing succeeds, so side effects don't happen twice. The overhead can be **amortized** by processing several messages in one transaction.
- **Idempotent writes:** an alternative to distributed transactions. An idempotent operation has the same effect whether it runs once or many times. Many operations can be made idempotent with a little extra **metadata**, for example by storing the offset of the message that produced each write.
- **Rebuilding state after failure:** keep processor state in a remote store, or keep it local and replicate or periodically snapshot it.

---

[← Previous: Chapter 10](Chapter-10-Batch-Processing) | [Home](Home) | [Next: Chapter 12, The Future of Data Systems →](Chapter-12-The-Future-of-Data-Systems)
