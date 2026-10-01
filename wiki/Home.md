# Designing Data-Intensive Applications: Chapter Notes

Welcome! This wiki is a chapter-by-chapter summary of **_Designing Data-Intensive Applications_** by Martin Kleppmann (O'Reilly, 2017). The book is one of the most important in the software industry because it connects distributed-systems theory with everyday engineering practice.

The notes focus on **what to do** more than on **how it works**, though they explain enough of the mechanics for the recommendations to make sense. You can use them in two ways:

- **Quick lookup:** jump straight to a topic when you need to remember a detail.
- **Full recap:** read the whole book's highlights in under an hour.

> 📖 The book is available from [O'Reilly](https://www.oreilly.com/library/view/designing-data-intensive-applications/9781491903063/).

---

## Contents

### Part I: Foundations of Data Systems
How a single-machine data system works: what we want from it, how data is modeled, stored, and encoded.

| # | Chapter | In one line |
|---|---------|-------------|
| 1 | [Reliable, Scalable, and Maintainable Applications](Chapter-01-Reliable-Scalable-and-Maintainable-Applications) | The three goals every data system design trades off. |
| 2 | [Data Models and Query Languages](Chapter-02-Data-Models-and-Query-Languages) | Relational, document, and graph models, and when each fits. |
| 3 | [Storage and Retrieval](Chapter-03-Storage-and-Retrieval) | Log-structured storage vs. B-trees, OLTP vs. analytics, and column stores. |
| 4 | [Encoding and Evolution](Chapter-04-Encoding-and-Evolution) | Data formats, schema evolution, and how data flows between processes. |

### Part II: Distributed Data
What changes when data is spread across many machines.

| # | Chapter | In one line |
|---|---------|-------------|
| 5 | [Replication](Chapter-05-Replication) | Keeping copies of the same data on several machines. |
| 6 | [Partitioning](Chapter-06-Partitioning) | Splitting a large dataset across machines. |
| 7 | [Transactions](Chapter-07-Transactions) | ACID, isolation levels, and the race conditions they prevent. |
| 8 | [The Trouble with Distributed Systems](Chapter-08-The-Trouble-with-Distributed-Systems) | Unreliable networks, unreliable clocks, and partial failure. |
| 9 | [Consistency and Consensus](Chapter-09-Consistency-and-Consensus) | Linearizability, ordering, two-phase commit, and consensus. |

### Part III: Derived Data
How to build systems out of several data stores that derive data from one another.

| # | Chapter | In one line |
|---|---------|-------------|
| 10 | [Batch Processing](Chapter-10-Batch-Processing) | Unix pipes, MapReduce, and dataflow engines. |
| 11 | [Stream Processing](Chapter-11-Stream-Processing) | Message brokers, change data capture, event sourcing, and windowing. |
| 12 | [The Future of Data Systems](Chapter-12-The-Future-of-Data-Systems) | Unbundling databases, end-to-end correctness, and ethics. |

---

## Credits

These pages are an edited and restructured version of the reading notes by [Ahmed Hammad](https://github.com/ahmedhammad97/Designing-Data-Intensive-Applications-Notes). The wording has been polished, a few factual slips have been corrected, and each chapter now has its own page with key takeaways. All ideas belong to the book's author, Martin Kleppmann.
