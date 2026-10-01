# Chapter 1: Reliable, Scalable, and Maintainable Applications

_Part I: Foundations of Data Systems_

> **Key takeaways**
> - A *fault* is one component deviating from its spec. A *failure* is the whole system stopping. Design so that faults don't become failures.
> - Describe load with concrete parameters, and measure performance with response-time **percentiles**, not averages.
> - Most of a system's lifetime cost is maintenance, so design for operability, simplicity, and evolvability.

---

For most applications today, CPU power is rarely the limiting factor. The bigger problems are the amount of data, its complexity, and how fast it changes.

There are many tools to choose from. The job is to pick the right tools and approaches so that the data stays correct and complete, performance stays good, and the system can handle more load, even when parts of it fail or degrade.

## Reliability

> ***The system should keep working correctly, even when faults and human errors occur.***

A **fault** is one component of the system deviating from its specification. A **failure** is when the system as a whole stops providing the required service to the user. It is impossible to prevent every fault, so the aim is to stop faults from turning into failures by building **fault-tolerance** mechanisms.

**Hardware faults.** Redundancy (RAID disks, dual power supplies, backup generators) is the first line of defense. For a long time it was enough. As data volumes and computing demands grew, systems moved toward tolerating the loss of *entire machines* by adding software fault tolerance on top of hardware redundancy.

**Software errors.** These usually come from assumptions about the environment that hold most of the time, until one day they don't. There is no quick fix, but software can check itself continuously while it runs and raise an alert when it finds a discrepancy.

**Human errors.** Ways to build reliable systems despite unreliable human actions:

- Design abstractions and APIs that make the right thing easy to do, without being so restrictive that people work around them.
- Provide full-featured sandbox environments with real data, where people can experiment without affecting real users.
- Test thoroughly at every level, from unit tests to whole-system integration tests.
- Make it quick to roll back configuration changes, and provide tools to recompute data if something goes wrong.
- Set up clear monitoring (telemetry) that gives early warning signs of faults.

## Scalability

> ***As the system grows, there should be reasonable ways to deal with that growth.***

**Describe load first.** Before discussing growth, define the system's **load parameters**, for example requests per second, the ratio of reads to writes, or the number of concurrently active users.

**Describe performance.**

- For batch processing systems, the key metric is usually **throughput** (records processed per second).
- For online systems, the key metric is **response time**.

Response time is a distribution, so report it as **percentiles**. "p99 = 200 ms" means 99% of requests are faster than 200 ms.

- High percentiles (*tail latencies*) matter because the slowest requests often come from your most valuable customers, who have the most data (for example, the most purchases).
- Over-optimizing extreme percentiles (such as p99.99) can cost far more than it is worth.
- Measure response times on the **client side**, under realistic traffic.

**Coping with load.**

- *Scaling up* (a more powerful machine) and *scaling out* (more machines) are both valid. Real systems often mix them.
- **Elastic** systems that add resources automatically help when load is very unpredictable. Manually scaled systems are simpler and have fewer operational surprises.
- In an early-stage startup, being able to iterate quickly on product features matters more than scaling to some hypothetical future load.

## Maintainability

> ***Everyone who works on the system over time should be able to work on it productively.***

Most of the cost of software is in ongoing maintenance, not initial development. Three design principles help.

**Operability: make life easy for operations.** Routine tasks should be easy. Good systems provide:

- Good monitoring and visibility into runtime behavior.
- No dependence on individual machines, so machines can be taken down for maintenance.
- Good documentation.
- Sensible default behavior, with the option for administrators to override it.
- Self-healing where appropriate, while still giving administrators manual control.

**Simplicity: manage complexity.** Reducing complexity doesn't mean removing features. It means removing *accidental* complexity, and the best tool for that is a good **abstraction**. Simple, easy-to-understand systems are also easier to change.

**Evolvability: make change easy.** Requirements always change. Agile practices (such as test-driven development and refactoring) help keep systems adaptable.

---

[Home](Home) | [Next: Chapter 2, Data Models and Query Languages →](Chapter-02-Data-Models-and-Query-Languages)
