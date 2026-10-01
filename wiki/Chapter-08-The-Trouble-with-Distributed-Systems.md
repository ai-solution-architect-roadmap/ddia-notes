# Chapter 8: The Trouble with Distributed Systems

_Part II: Distributed Data_

> **Key takeaways**
> - In a distributed system, **partial failures** are normal and **nondeterministic**. Assume anything that can go wrong will.
> - Networks can lose, delay, or duplicate packets. **Timeouts** are the only practical way to detect faults, and they can't tell a crash from a slow network.
> - Clocks are unreliable. Don't rely on time-of-day clocks to order events. Use **logical clocks** or fencing tokens instead.
> - A node can't trust its own judgment. Decisions are made by **quorum**. Use the *partially synchronous, crash-recovery* system model.

---

The main difference between a program on one computer and a distributed system is that in a distributed system there are **many more ways for things to go wrong**. We should assume they *will* go wrong.

## Faults and Partial Failures

- A program on a single computer is **deterministic**: it usually either works or fails completely.
- In a distributed system, some parts can break in **unpredictable, nondeterministic** ways while other parts keep working. This is a **partial failure**.

To make distributed systems work, accept that partial failures will happen and build fault tolerance into the software:

- Decide what behavior you expect from the software when a fault occurs.
- Consider a wide range of possible faults, including unlikely ones.
- Create such faults deliberately in testing to see what happens (*chaos engineering*).

## Unreliable Networks

**Shared-nothing** systems (machines that communicate only over the network) have become the dominant way to build internet services, because they use commodity cloud machines and get high reliability through redundancy. Their only means of communication, though, is the network.

Most networks are **asynchronous packet networks**, and many things can go wrong:

- The request may be lost.
- The request may wait in a queue and be delivered late.
- The remote node may have crashed, or paused temporarily (for example, for garbage collection).
- The response may be lost or delayed.

From the sender's side, these cases look the same. The usual way to handle them is a **timeout**. After a timeout you don't have to recover automatically: simply showing an error message to the user can be a valid choice.

### Detecting faults

Many systems need to detect faulty nodes automatically. For example, a load balancer must stop sending requests to a dead node, and single-leader replication must promote a new leader. Some signals can help:

- If no process is listening on the port, the OS may reply with a TCP `RST` or `FIN` packet.
- A script can notify other nodes when a process crashes.
- Network switches or the datacenter's management interface may report a link or machine as down.

None of these are strong guarantees. A positive response from the **application itself** is the only reliable proof that a request succeeded.

### Choosing a timeout

- **Short timeouts** detect faults faster, but carry a higher risk of wrongly declaring a node dead when it was only slow. That can cause an action to be performed **twice**, or load to move to nodes that are already busy, causing **cascading failures**.
- In theory, a good timeout is **2d + r**, where *d* is the maximum packet delay and *r* is the maximum processing time. In practice, neither is bounded.
- So it is usually better to **choose timeouts experimentally**: measure the distribution of round-trip times continuously and adjust automatically (as Phi Accrual failure detectors do).

### Network congestion

Queueing happens at switches, in the OS, in the hypervisor, and at TCP flow control. This is the main cause of variable delays. Some latency-sensitive applications (such as video calls and online games) use **UDP** instead of TCP, because they would rather drop delayed data than wait for retransmission.

## Unreliable Clocks

Time is hard to define in a distributed system, because each machine has its own clock that may be slightly faster or slower than the others. The **Network Time Protocol (NTP)** is commonly used to keep clocks roughly in sync.

### Two kinds of clocks

- **Time-of-day clock:** returns the current date and time (wall-clock time). It is usually synchronized with NTP and may **jump backward or forward** when corrected, so it is **not suitable for measuring elapsed time**.
- **Monotonic clock:** guaranteed to always move forward, so it is **suitable for measuring durations** such as timeouts. Its absolute value is meaningless. NTP may adjust how fast it moves (its *rate*), but it never jumps.

### Clock accuracy

- Our ways of keeping clocks correct are much less reliable or accurate than you might expect: quartz drift, NTP failures, firewalls, leap seconds, virtualization, and misconfigured devices all cause problems.
- Good accuracy is possible with **GPS receivers**, the **Precision Time Protocol (PTP)**, and careful deployment and monitoring.
- **Monitor clock offsets** between machines, so that a node whose clock drifts too far is removed from the cluster before it causes damage.
- Robust software must be prepared for incorrect clocks.

### Ordering events with clocks

- Even with NTP, timestamps from different nodes **can't reliably order events**. With last-write-wins, a node with a fast clock can silently overwrite newer data.
- **Logical clocks** (such as version vectors or Lamport timestamps), which track causality using counters, are a safer way to order events.
- A clock reading is really a **range** (a confidence interval), not a point. In practice the best accuracy is roughly tens of milliseconds over the public internet. The uncertainty bound depends on the time source; Google's TrueTime API, for example, reports it explicitly.

### Process pauses

A thread can be paused for a long time for many reasons: garbage collection, virtual machine suspension, context switches, swapping to disk, or the process being stopped. While paused, it has no idea that time has passed.

So a node must assume its execution can be paused **at any point, even in the middle of a function**, and must be designed so this doesn't break correctness. For example, a lease may expire while the node is paused, but the node still thinks it holds the lease.

## Knowledge, Truth, and Lies

A node in a distributed system **can't know anything for certain**. It only knows what it learns from messages. We can state the assumptions we make about the system's behavior (the **system model**) and prove algorithms correct within that model.

### Truth is defined by the majority

- A node can't trust its own judgment about whether it is alive or still the leader. It must abide by the decision of a **quorum** (a majority) of nodes, even when that decision affects the node itself.
- When a **lock or lease** protects a resource, use **fencing tokens**: a number that increases each time the lock is granted. The storage service rejects writes that carry an older token. This stops a node that wrongly believes it still holds the lock from corrupting data.
- It is unwise for a service to assume its clients will always behave well.

### Byzantine faults

- If nodes may **lie** (send faulty or malicious messages), the problems get much harder. These are **Byzantine faults**. A system is **Byzantine fault-tolerant** if it keeps working correctly even when some nodes malfunction or are under malicious attack.
- Byzantine fault-tolerant algorithms are complicated and expensive to deploy. They are usually impractical when all nodes run in your own datacenter, but they make sense in **peer-to-peer** networks such as blockchains.
- Even trusted nodes can show weak forms of "lying" through hardware faults, software bugs, or misconfiguration. Simple safeguards help:
  - **Checksums** at the TCP or application level, to detect corrupted packets.
  - **Input validation** and sanity checks on values.
  - Querying several NTP servers and ignoring outliers.

### System models

**Timing assumptions**

- **Synchronous model:** network delay, process pauses, and clock error are all bounded. Unrealistic for most systems.
- **Partially synchronous model:** the system behaves synchronously *most* of the time but sometimes exceeds the bounds. **Realistic.**
- **Asynchronous model:** no timing assumptions at all, not even clocks. Very restrictive.

**Node failure assumptions**

- **Crash-stop faults:** a node fails only by crashing, and is then gone forever.
- **Crash-recovery faults:** a node can crash at any moment and may come back after some unknown time. Data on stable storage survives, but in-memory state is lost.
- **Byzantine (arbitrary) faults:** nodes can do anything, including trying to trick other nodes.

The most useful model for real systems is **partially synchronous with crash-recovery faults**.

### Safety and liveness

- **Safety:** "nothing bad happens" (for example, uniqueness of fencing tokens). Once violated, the damage can't be undone. Safety properties must **always** hold.
- **Liveness:** "something good eventually happens" (for example, a request eventually gets a response). Liveness can come with caveats, such as only holding once the network recovers.

We have to make some assumptions about which faults can happen. But real implementations may still need to handle cases the model says are impossible, even if only by logging an error and stopping.

---

[← Previous: Chapter 7](Chapter-07-Transactions) | [Home](Home) | [Next: Chapter 9, Consistency and Consensus →](Chapter-09-Consistency-and-Consensus)
