# Chapter 10: Batch Processing

_Part III: Derived Data_

> **Key takeaways**
> - Distinguish the **system of record** (the source of truth) from **derived data** (caches, indexes, views that can be recomputed).
> - Batch jobs read **bounded**, **immutable** input and produce new output. This makes them easy to retry, debug, and roll back.
> - MapReduce brought the Unix philosophy to distributed systems. **Dataflow engines** (Spark, Flink, Tez) do the same work much faster by not writing out every intermediate result.
> - High-level APIs (Hive, Pig, Spark SQL) cut code size and let the engine optimize execution.

---

## Introduction to Part III: Derived Data

Real systems usually combine several databases, indexes, caches, and analytics systems, so they need mechanisms for moving data from one store to another.

Systems that store and process data fall into two broad categories:

- **Systems of record:** the **source of truth**. Each fact is stored exactly once (usually normalized), and the system holds the authoritative version of new data as it arrives.
- **Derived data systems:** the result of transforming or processing data from another system (caches, indexes, materialized views, recommendation models). If lost, they can be **recreated** from the source. They are often denormalized and redundant, but essential for good read performance.

## Three Kinds of Systems

- **Services (online systems):** wait for requests, handle each as quickly as possible, and send a response. Measured by **response time** and availability.
- **Batch processing systems (offline systems):** run a job over a large, **bounded** input and produce output. Usually scheduled periodically, and can take minutes to days, because no user is waiting. Measured by **throughput**.
- **Stream processing systems (near-real-time systems):** consume **unbounded** input shortly after it arrives, process it, and produce output. They sit between online and batch. (See [Chapter 11](Chapter-11-Stream-Processing).)

## Batch Processing with Unix Tools

A lot of data analysis can be done in minutes by combining Unix commands such as `awk`, `sed`, `grep`, `sort`, `uniq`, and `xargs`.

**The Unix philosophy**

- Small programs that each **do one thing well** are composed into powerful pipelines with the shell.
- For composition to work, every program must share the same input/output interface. In Unix, that interface is a **file**, which in practice is just a sequence of bytes. **Pipes** stream those bytes from one program to the next.

**Why Unix tools work so well**

- **Input files are treated as immutable**, so you can run commands as often as you like without damage.
- You can stop the pipeline at any point and inspect the output (for example, pipe it into `less`).
- You can write the output of one stage to a file and restart later stages from there, without rerunning the whole pipeline.

**The biggest limitation:** they run on a **single machine**. That is where tools like Hadoop come in.

## MapReduce and Distributed File Systems

Like Unix tools, a MapReduce job **doesn't modify its input** and has no side effects other than producing output. Unlike Unix tools, it can be spread across **thousands of machines**, and it reads and writes files on a **distributed file system** such as HDFS instead of the local disk.

### How a MapReduce job works

You write two callback functions:

- **Mapper:** called once for every input record. It extracts zero or more **key-value pairs** from the record.
- **Reducer:** the framework sorts and groups the mapper output by key (the *shuffle*). The reducer receives every value for one key and produces output records. The number of reduce tasks is configured by the user.

Mappers and reducers must be **deterministic** (no randomness), so that a failed task can be rerun safely.

### Workflows

A single MapReduce job can solve only a limited range of problems, so jobs are commonly **chained into workflows**. One job's output directory becomes the next job's input, so the chaining happens **implicitly through directory names**. Workflow schedulers (Oozie, Azkaban, Airflow) manage the dependencies.

### Joins in batch processing

Joins are needed whenever you must access records on both sides of an association. MapReduce has no indexes; it reads the **entire** input.

- Querying a remote database for every record would be slow, overload the database, and give **non-deterministic** results if the data changes during the job.
- A better approach is to **take a copy of the other dataset** (for example, a database dump) into the distributed file system and join the two files there.

MapReduce join strategies:

- **Sort-merge join (reduce-side):** mappers emit both datasets keyed by the join key, so the shuffle brings matching records together at the same reducer.
- **Broadcast hash join (map-side):** if one dataset is small enough to fit in memory, every mapper loads it into a hash table and looks up each record in the large dataset.
- **Partitioned hash join (map-side):** if both datasets are partitioned the same way on the join key, each mapper only needs to load one partition of the smaller dataset.

### The output of batch workflows

- **Search indexes:** Google originally used MapReduce to build its search index. Google later moved away from MapReduce, but it is still a good way to build indexes for **Lucene/Solr**, because the job is a natural fit for full-text indexing.
- **Key-value stores** for serving results, such as machine-learning models or recommendations.
- Writing to a production database **directly from a mapper or reducer**, one record at a time, is a bad idea: it is slow, can overload the database, and breaks the all-or-nothing guarantee if a job fails partway. Instead, **build a brand-new database inside the batch job**, write its files to the distributed file system, and then bulk-load them into read-only servers.

### Benefits of the Unix-like philosophy

- **Easy rollback:** if buggy code produces bad output, roll back the code and rerun the job. The input is unchanged, so the result is correct. This is called *human fault tolerance*.
- **Faster feature development**, because mistakes are cheap to fix.
- **Automatic retries:** the framework transparently retries failed tasks without affecting application logic.
- **Separation of concerns:** the logic is separate from wiring up inputs and outputs.
- **Reuse:** the same set of files can be the input to many different jobs.

### Hadoop vs. distributed databases

Distributed file systems make it possible to **dump data quickly** in any raw format, which can be cleaned up and transformed later (*schema-on-read*, the "data lake" or "sushi principle": raw data is better). Data warehouses and other services can then consume the cleaned-up results.

MapReduce suits **large jobs** that run for a long time and are likely to hit at least one task failure. It writes to disk aggressively so it can recover at fine granularity.

## Beyond MapReduce

### Problems with materializing intermediate state

MapReduce **fully materializes** intermediate state: every job eagerly writes its complete output to files before the next job can read it. This means:

- A job can't start until **all** tasks of the jobs before it have finished.
- Mappers are often redundant, only re-reading what a reducer just wrote.
- Intermediate files are **replicated** across nodes in the distributed file system, which wastes storage and I/O.

Beyond materialization, the raw MapReduce API is hard to use for complex jobs, and it performs poorly for some kinds of processing (such as iterative graph algorithms).

### Dataflow engines

**Dataflow engines** such as **Spark**, **Tez**, and **Flink** address these problems:

- They treat an **entire workflow as one job** instead of many independent sub-jobs.
- They offer more flexible **operators** than just map and reduce. Sorting, for example, only happens where it is actually needed.
- They can pipeline data between operators, schedule work for data locality, and keep intermediate state in memory or on local disk.

They can run the same computations as MapReduce, usually **much faster**.

**Fault tolerance trade-off.** Because they avoid materializing intermediate state, dataflow engines have to **recompute** lost data from earlier stages when a task fails. This works best if operators are **deterministic**. If intermediate data is much smaller than the input, or the computation is CPU-intensive, it can be cheaper to materialize it than to recompute it.

### Graphs and iterative processing

Many algorithms, such as PageRank, iterate until they converge. MapReduce handles this badly. The **Pregel** (bulk synchronous parallel) model, used by Apache Giraph, Spark GraphX, and Flink Gelly, is designed for iterative graph processing.

### High-level APIs and languages

Higher-level languages and APIs such as **Hive**, **Pig**, Cascading, and **Spark SQL / DataFrames**:

- Need **much less code** than raw MapReduce.
- Allow **interactive** use for exploring data.
- Make execution more efficient, because a declarative query lets the engine's optimizer choose join algorithms and use column-oriented formats.

One of the biggest advantages of batch processing over a typical database query is the freedom to run **arbitrary code** in the callback functions, usually in a general-purpose language.

### Applications

Batch processing is increasingly important for **statistical and numerical algorithms**, **machine learning**, **recommendation systems**, and **spatial algorithms** (such as nearest-neighbor search).

---

[← Previous: Chapter 9](Chapter-09-Consistency-and-Consensus) | [Home](Home) | [Next: Chapter 11, Stream Processing →](Chapter-11-Stream-Processing)
