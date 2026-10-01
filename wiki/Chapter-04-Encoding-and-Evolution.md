# Chapter 4: Encoding and Evolution

_Part I: Foundations of Data Systems_

> **Key takeaways**
> - Old and new versions of code and data coexist during rolling upgrades, so you need both **backward** compatibility (new code reads old data) and **forward** compatibility (old code reads new data).
> - Avoid language-specific serialization. Prefer JSON, XML, or schema-based binary formats (Protocol Buffers, Thrift, Avro).
> - Data flows between processes in three main ways: **through databases**, **through service calls** (REST/RPC), and **through asynchronous messages**.
> - RPC tries to make a network call look like a local function call. That abstraction is fundamentally flawed.

---

When an application's features change, the data it stores usually changes too. In a large system, the code can't all be upgraded at once (rolling upgrades on servers, users who don't update client apps), so **multiple versions of code and multiple data formats can exist at the same time**. Two kinds of compatibility must be maintained:

- **Backward compatibility:** newer code can read data written by older code.
- **Forward compatibility:** older code can read data written by newer code.

## Formats for Encoding Data

Data held in memory (objects, lists, hash maps) must be **encoded** (serialized) into a sequence of bytes before it is written to a file or sent over the network, and **decoded** (deserialized) on the other side.

**Language-specific formats** (such as Java's `Serializable` or Python's `pickle`) are convenient but should be avoided:

- They tie you to one programming language.
- Decoding can instantiate arbitrary classes, which is a security risk.
- Versioning is often an afterthought.
- Performance is often poor.

**Textual formats (JSON, XML, CSV)** are widely supported and human-readable, but they are ambiguous about numbers and don't support binary strings well.

**Binary formats** can save a lot of space on large datasets. For small datasets, the savings may not be worth losing human readability.

**Schema-based binary formats** (Thrift, Protocol Buffers, Avro):

- They come with code-generation tools that produce classes from a schema.
- They are much more compact than textual formats, because field names don't need to be stored.
- The schema is a useful form of documentation that is always up to date.
- They have clear rules for backward and forward compatibility when the schema evolves.

## Modes of Dataflow

The most common ways data flows between processes are through **databases**, **service calls** (REST, RPC), and **asynchronous message passing**.

### Dataflow through databases

Writing to a database is like sending a message to your future self. It needs both backward and forward compatibility, because newer and older code may read and write the same records.

- One workaround is to **migrate** (rewrite) the whole database into the new schema whenever it changes, but this is expensive for large datasets.
- Watch out: old code reading a record written by new code must not drop fields it doesn't recognize when it writes the record back.
- *Data outlives code.* A five-year-old record is still in its original encoding unless you explicitly rewrite it.

### Dataflow through services: REST and RPC

A server can itself be a client of another service. This idea led to **service-oriented architecture (SOA)**, more recently called **microservices**: services are easier to change because each can be deployed and evolved independently.

- A service exposes only a specific API, unlike a database, which can be queried for any data it holds.
- Again, expect old and new versions of services and clients to run at the same time.

**REST vs. SOAP**

- **REST** is a design philosophy built on HTTP. It favors simple data formats and uses URLs to identify resources.
- **SOAP** is an XML-based protocol. Clients access the remote service through generated local classes and method calls.

**The problems with RPC.** A **Remote Procedure Call** tries to make a request to a remote service look like a local function call. This is fundamentally flawed, because a network call is very different from a local one:

- A network request is unpredictable. It may be lost, or the remote machine may be slow or down, so clients must handle retries.
- A request may time out without returning any result, leaving you unsure whether it succeeded.
- Retrying may perform the action twice, unless you build in **idempotence**.
- Network latency varies wildly.
- Arguments must be encoded into bytes. Passing references to objects in local memory doesn't translate to the network.
- Client and server may use different languages, so data types must be translated.

**In practice**, REST is the dominant style for public APIs, while RPC frameworks are often used for requests between services owned by the same organization, usually within the same datacenter.

**Compatibility.** Assuming servers are upgraded before clients, RPC needs backward compatibility on **requests** and forward compatibility on **responses**. A service usually can't force its clients to upgrade, so it often has to maintain **several API versions** side by side.

### Dataflow through asynchronous messages

Here, an intermediary called a **message broker** (or **message queue**) stores messages temporarily. Compared with direct RPC, a broker:

- Acts as a **buffer** if the recipient is unavailable or overloaded, which improves reliability.
- Can **redeliver** messages to a process that crashed, so messages aren't lost.
- Means the sender doesn't need to know the recipient's IP address and port.
- Can deliver one message to **several recipients**.
- **Decouples** the sender from the receiver.

The downside: messaging is usually **one-way**. If the recipient needs to reply, it typically sends the reply on a separate channel.

Brokers organize messages into **topics** (or queues). When a message is published to a topic, the broker delivers it to that topic's subscribers. A consumer can in turn publish messages to other topics, which allows flexible chains of processing.

---

[← Previous: Chapter 3](Chapter-03-Storage-and-Retrieval) | [Home](Home) | [Next: Chapter 5, Replication →](Chapter-05-Replication)
