---
title: "The Log: What every software engineer should know about real-time data's unifying abstraction"
source: https://engineering.linkedin.com/distributed-systems/log-what-every-software-engineer-should-know-about-real-time-datas-unifying
author: Jay Kreps
company: LinkedIn
date_posted: 2013-12-16
date_digested: 2026-05-25
---

# The Log: What every software engineer should know about real-time data's unifying abstraction

## What's new to learn

1. **The State Machine Replication Principle** — any two deterministic processes that start in the same state and receive the same inputs in the same order will produce identical outputs; all distributed consensus algorithms are fundamentally implementations of this principle wrapped around an append-only log.

2. **The Tables–Events Duality** — a table (current state) and a log (history of changes that produced that state) are two views of the same information; either can be derived from the other, which unifies databases, event sourcing, CQRS, and change-data capture under one mental model.

3. **The Log as a Data Integration Hub** — routing every write through a central append-only log collapses the N×M point-to-point integration problem into N+M subscriptions, turning heterogeneous distributed systems into a set of independent consumers of a single source of truth.

## Prerequisites

- What a write-ahead log (WAL) is and why databases write to it before applying changes to data pages.
- Basic familiarity with message queues (why you would use Kafka/RabbitMQ at all).
- Rough understanding of database replication (primary → replica).
- What the consensus/consistency problem means in distributed systems (but not the algorithms themselves — those will click into place here).

## The core idea

A log is the simplest storage structure imaginable: an append-only, totally-ordered sequence of records, each stamped with a monotonically increasing offset. Writes are always at the end; reads scan left to right. That's it.

What Kreps argues — and this is the post's big bet — is that this boring structure is actually the *universal primitive* underlying everything from crash recovery to database replication to stream processing to consensus algorithms to real-time ETL. Once you see it, you can't un-see it.

**State Machine Replication.** The central theorem:

> If two identical, deterministic processes begin in the same state and get the same inputs in the same order, they will produce the same output and end in the same state.

To make a distributed system where all nodes agree on state, you don't need to invent some magical protocol for exchanging state. You just need all nodes to consume the *same log in the same order.* Paxos, Raft, Zab (ZooKeeper's protocol) — they all exist to implement exactly one thing: a consistent, fault-tolerant, append-only log. The consensus problem *is* the log problem.

**The Data Integration Insight.** In 2013, LinkedIn had Hadoop for batch analytics, Cassandra for low-latency storage, SOLR for search, and Espresso as a main-line key-value store. Getting data from any one system to any other required custom ETL jobs — N producers × M consumers = N×M bespoke pipelines, each brittle and out of sync. Kreps' fix: make Kafka (a durable, replayable commit log) the single substrate all systems write to and read from. Now you have N pipelines into the log and M pipelines out. Point-to-point collapses into a star.

**Tables and Events.** A table in a database is the *present state* of some data. The WAL is the *history of mutations* that produced that state. These are two representations of the same information. Given the log, you can reconstruct the table by replaying events from the beginning. Given the table plus a changelog trigger, you can reconstruct the log. This duality — which is also the foundation of event sourcing — means that every derived system (a search index, a cache, a replica, a data warehouse) is just a *materialized view* of the log, rebuilt whenever it falls behind.

## Mechanics

**The Anatomy of the Log**

A log segment on disk is a flat file of fixed-size or variable-size records. Each record has a sequence number (Kafka calls it an *offset*). The writer appends; readers seek to an offset and scan forward. Because writes are sequential and readers never block writers, throughput is limited only by sequential I/O speed. On commodity spinning disk, sequential writes run at 600 MB/s vs ~100 IOPS random — a 10,000× difference that makes the log dirt cheap.

**Kafka's Architecture in This Frame**

Kafka exposes *topics* (named logs) split into *partitions* (independent sublogs that can be processed in parallel). Within a partition, order is strictly preserved. Producers append to a partition; consumers track their own offset (where they left off) and poll for new records. Because the offset is consumer-owned, a new consumer can start from offset 0 and replay every message ever written. This makes Kafka both a message queue *and* a replayable audit log.

Crucially, Kafka retains messages for days or weeks rather than discarding them after delivery. That retention is what enables:
- **Catch-up replication** — a replica that was down just resumes from its last offset.
- **Derived view rebuilds** — drop your SOLR index and rebuild it by replaying from offset 0.
- **Stream processing** — Flink/Spark Streaming/Samza treat the topic as a bounded or unbounded input; the same code handles historical backfills and live streaming.

**Physical vs. Logical Logging**

Databases choose what to put in their WAL:

- *Physical log* — the actual byte-level changes to data pages. Easy to apply, hard to interpret across schema versions.
- *Logical log* — the SQL/DML operations (INSERT, UPDATE, DELETE). Portable across storage formats; the basis of MySQL's binlog and PostgreSQL's logical replication.

The tradeoff matters for replication: MySQL's statement-based binlog (logical) is compact but breaks on non-deterministic functions (NOW(), RAND()); row-based binlog (physical-ish) is verbose but exact. PostgreSQL's logical decoding lets external consumers subscribe to a high-level event stream without understanding the internal page format.

**Primary-Backup vs. State Machine Replication**

Two replication architectures exist:

- *Primary-backup* — one node accepts all writes, applies them, then ships state deltas (pages, rows) to replicas. Replicas are passive.
- *State machine replication* — all nodes receive the same input log and process it independently. Every replica is an equal peer.

Primary-backup is simpler and lower-latency for individual writes (one hop). State machine replication tolerates more failures (no single writer) and enables synchronous multi-master. Paxos/Raft are state machine replication protocols: they agree on a log entry before any node acts on it, ensuring all nodes see the same sequence.

**The N×M → N+M Collapse**

Before the central log, the integration graph looks like:

```
MySQL ──────→ Hadoop
    ╲─────→ SOLR
     ╲────→ Cassandra
Postgres ──→ Hadoop
         ╲─→ SOLR
...
```

Each arrow is a custom ETL job that breaks when either end changes schema. After introducing Kafka as the hub:

```
MySQL ─→ [Kafka] ─→ Hadoop
Postgres ─→       ─→ SOLR
...                ─→ Cassandra
```

N sources write to Kafka; M consumers read from it. Adding a new consumer costs one pipeline, not N. Adding a new source costs one pipeline, not M. The log also decouples producers from consumers in time — if the Hadoop cluster is down, messages accumulate in Kafka; when it recovers, it catches up automatically.

## Where it breaks

**Partitioned total order.** Within a Kafka partition, order is guaranteed. Across partitions, it is not. This forces producers to decide which partition each record belongs to (usually by key hash). Cross-partition ordering requires external coordination — which negates much of the simplicity.

**Exactly-once is hard.** Consumer failures can cause re-delivery; producer retries can cause duplication. "Exactly-once" processing requires either idempotent consumers, transactional producers (Kafka's `enable.idempotence`), or distributed transactions — all complex. The log gives you *at-least-once* for free; *exactly-once* is extra work.

**Schema evolution.** A durable log is great until you need to change the event schema. Old consumers break on new messages; new consumers break on old messages. Avro with a schema registry or Protobuf with careful field management helps, but adds operational overhead.

**Retroactive corrections.** The log is immutable. If you discover that 10,000 past events were written with the wrong customer ID, you cannot rewrite them. You must emit a compensating event and reason about corrections in all consumers.

**Write amplification for derived views.** Every consumed log event may write to multiple downstream stores (search index, cache, warehouse). A single Kafka message can fan out to many writes, creating amplification spikes during rebuilds or catch-up replication.

**The log is not a database.** Kafka does not support random reads by key, secondary indexes, or ad-hoc queries. Consuming the log to answer "what is the current value of X?" requires materializing the log into a queryable store — you're back to needing a database as a view.

## Why it works

The deeper principle is **immutability as the foundation of distributed coordination**.

Every hard problem in distributed systems boils down to: "nodes have different views of shared state, and we need to reconcile them." The log sidesteps this by *never mutating shared state in place.* All updates are appends. If two nodes disagree, neither has to be "rolled back" — you just compare their offsets and the one with the lower offset reads forward to catch up.

This is the same principle behind:

- **Git** — commits are immutable; branches are just pointers to commit hashes. Merging is reconciling two logs of immutable snapshots.
- **Merkle trees / blockchains** — append-only chains where each record commits to its history via a hash; tampering is detectable because it breaks the chain.
- **LSM trees** — writes go into an in-memory log (memtable) and get compacted offline; the in-place B-tree is replaced by an append-only structure (see the 2026-05-23 digest on SSDs and databases).
- **HDFS / S3** — write-once blocks or objects; no random write API, which makes replication straightforward (copy the block once, it never changes).
- **Event sourcing** — the domain model is stored as a log of past events (the ledger), never as mutable current state. The "current state" is a derived, reconstructible view.

In every case, the reason immutability works is the same: **reads never conflict with writes**, so replicas can always catch up by reading forward, recovery always works by replaying, and the history is always auditable.

The log also reveals why consensus algorithms feel like they should be simpler than they are: they are. Paxos/Raft are just building reliable total order on top of an unreliable network. Once you have that, every distributed system problem that requires agreement reduces to: "subscribe to the log."

## Going deeper

1. **Jay Kreps, *I ♥ Logs* (O'Reilly, 2014)** — the full book expansion of this post, covering exactly-once semantics, stream processing patterns, and the Kappa architecture (stream-only, no batch layer).

2. **Martin Kleppmann, "Using logs to build a solid data infrastructure"** (martin.kleppmann.com, 2015) — companion post arguing concretely why dual writes (writing to a database *and* to a cache simultaneously) are fundamentally unsafe and why a log is the only correct solution.

3. **Chapter 11 of *Designing Data-Intensive Applications* (Kleppmann, 2017)** — the definitive textbook treatment of stream processing, log-based messaging, change data capture, and the events/state duality, with formal reasoning about what guarantees each architecture provides.
