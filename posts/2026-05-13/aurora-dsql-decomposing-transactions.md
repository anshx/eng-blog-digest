---
title: "Decomposing Aurora DSQL"
source: https://brooker.co.za/blog/2025/04/17/decomposing.html
author: Marc Brooker
company: AWS
date_posted: 2025-04-17
date_digested: 2026-05-13
---

# Decomposing Aurora DSQL

## What's new to learn

1. **Transaction decomposition as an analysis tool** — Breaking any distributed transaction protocol into four orthogonal concerns (execution, ordering, validation, persistence) lets you independently ask: *does this step truly need cross-node coordination?* Usually fewer steps require it than you think.

2. **MVCC as a coordination eliminator** — Multi-version concurrency control at a physical timestamp lets every replica serve reads independently. No leader, no lock acquisition, no network round-trip. A read is satisfied whenever the replica has seen all commits up to the requested timestamp.

3. **OCC as "push coordination to commit"** — Optimistic concurrency control (OCC) assumes no conflicts during execution and validates at the end. In the common case (no conflicts), the entire execution phase is local; coordination happens once, at commit time, across only the shards the transaction touched.

## Prerequisites

- SQL transactions and ACID guarantees
- MVCC basics: the idea that a write creates a new version rather than overwriting in place, and a read picks a snapshot by timestamp
- Two-phase locking (2PL) as contrast: locks acquired during execution rather than at commit
- Distributed replication: what it means for a replica to be "behind" and "caught up"
- What a shard is: a horizontal partition of keyspace with its own replica set

## The core idea

Classic distributed databases serialize coordination throughout a transaction. A write acquires a lock on the primary; every replica must hear about it before the lock releases; a read that touches multiple shards must either go through a single global leader or go through a round of voting. The result: coordination is proportional to the *number of operations*, not just the *number of transactions*.

Aurora DSQL flips this by asking a different design question: *which phases of a transaction actually require coordination, and which can be made local?*

Brooker, inspired by Alex Miller's earlier post "Decomposing Transaction Systems," maps a transaction onto four phases:

| Phase | Question | Coordination needed? |
|---|---|---|
| **Execution** | Evaluate reads and writes | No — MVCC + physical time |
| **Ordering** | Assign a position in global history | Minimal — physical clock |
| **Validation** | Check for conflicting concurrent writes | Yes — but only across touched shards |
| **Persistence** | Make the commit durable | No cross-shard coordination |

The punchline: of the four phases, only validation requires coordination, and even then, only among the shards actually touched by that transaction. A transaction touching 2 of 1000 shards pays only for those 2 adjudicators to agree — all other shards are oblivious.

## Mechanics

### Physical layout

DSQL's architecture has three independently scalable tiers:

**Query Processors (QPs)** — Each database connection runs inside a Firecracker microVM (the same hypervisor used by AWS Lambda) containing a customized PostgreSQL engine. The QP executes the SQL, holds the local working set during execution, and presents the transaction's reads and intended writes. QPs are stateless between transactions; a fresh microVM is launched from a snapshot for each new transaction. Because they are isolated and share nothing, they can scale horizontally without coordination.

**Adjudicators** — Partition the keyspace into ranges; each adjudicator owns one range. At commit time, the QP submits the transaction's read set and write set to the adjudicators that own the touched key ranges. The adjudicators run the OCC protocol: they verify that no committed transaction has written any key in the read set since the transaction's read timestamp. If the check passes, the lead adjudicator assigns a commit timestamp; all involved adjudicators write their portion to their associated journal. Cross-shard transactions require the involved adjudicators to coordinate with each other, but only those adjudicators — a 2-shard transaction involves 2 adjudicators, not all of them.

**Journals and storage** — Each adjudicator has its own journal: an append-only, ordered, durably replicated log (an internal AWS component similar in spirit to a Kafka partition, optimized for ordered cross-AZ and cross-region replication). Once the adjudicator writes a commit record to the journal, the transaction is durable. A **crossbar** component then merges the per-adjudicator journal streams in chronological order and fans the merged stream out to the storage shards, where MVCC versions are materialized.

### Read path (coordination-free)

1. The QP picks a read timestamp `τ_read` from its local physical clock.
2. For each row it needs, it asks the relevant storage replica: "give me the version as of `τ_read`."
3. If the replica's applied-up-to point is already ≥ `τ_read`, it answers immediately from its MVCC store.
4. If the replica lags, it waits — briefly — until the journal stream catches it up to `τ_read`, then answers.
5. No lock is acquired, no leader is consulted, any replica in any AZ can serve the read.

This works because MVCC turns the storage into an immutable log of versions. Every past state is still there; the read timestamp selects which state to observe. Replicas can converge asynchronously and still serve consistent reads as long as they are caught up *enough*.

### Write/commit path (coordination at commit only)

1. **Execution** (in QP): The transaction runs SQL, reading via MVCC and buffering writes locally. No coordination, no locks, no network calls except to storage for reads.
2. **Commit** (adjudicators): The QP sends the read set `R`, write set `W`, and `τ_read` to the adjudicators for all touched shards. Each adjudicator checks: "did any write to a key in `R` happen after `τ_read`?" If any adjudicator says yes, the transaction aborts (OCC conflict). If all say no, one lead adjudicator assigns `τ_commit > τ_read` and the group commits.
3. **Persistence** (journal): The commit record enters the journals of the touched adjudicators. The client receives acknowledgment as soon as the journals confirm durability. The crossbar applies the writes to storage asynchronously.

### MVCC version lifecycle

Because QPs are bounded-lifetime (transactions cannot run longer than 5 minutes), DSQL can bound MVCC garbage collection with a simple rule: discard any version older than 5 minutes. No need for PostgreSQL's VACUUM machinery tracking open transaction IDs. The 5-minute bound also means the system has a hard upper limit on the age of a read timestamp, keeping storage replicas from needing to maintain arbitrarily old versions.

## Where it breaks

**High-contention workloads**: OCC performs well when conflicts are rare. If many transactions compete to write the same rows (e.g., a single "counter" row), abort rates rise and throughput collapses. 2PL-based systems hold locks and queue waiters; OCC aborts and retries, which is expensive under sustained contention.

**Cross-shard transaction cost**: A transaction that writes to *k* shards must synchronously involve *k* adjudicators at commit time. Wide transactions (touching many shards) pay proportionally more latency and create more coordination surface. DSQL's design favors narrow, frequent transactions over wide, rare ones.

**5-minute transaction limit**: Long-running analytical queries or interactive transactions with a human in the loop will be killed. This is a feature for MVCC GC and against holding OCC "locks" (read sets), but it rules out several workload patterns.

**Regional clock skew**: The read path assumes physical clocks are close to synchronized. If a QP's clock is significantly ahead of the storage replicas' views, reads might block waiting for replicas to catch up. DSQL relies on tight NTP/PTP discipline across the fleet (similar assumptions underlie Google's TrueTime, though DSQL uses a probabilistic rather than bounded approach).

**Read-your-own-writes in multi-region**: A write committed in one region propagates to other regions via the journal. A subsequent read in another region that arrives before the write has replicated will not see it — unless the client's application explicitly waits for replication. DSQL provides mechanisms for session-level consistency, but the tradeoff is explicit.

## Why it works

The deeper principle: **physical time can substitute for coordination wherever you only need "monotonically increasing" agreement, not "exactly this value" agreement**.

In a 2PL system, a read must check with the lock manager: "is this row available?" That is coordination — you must contact someone. In MVCC at a physical timestamp, you never ask "is this row available?" You just say "I want the version at time T." If the local replica has seen T, it answers without consulting anyone. The distributed agreement on T happened implicitly via the physical clock, which every node observes independently.

The same principle explains OCC's effectiveness: execution is purely local because nothing is "locked" for others. The only moment you need agreement is at commit, and even then, you only need the subset of the system that holds data you touched to agree.

This is the same architectural bet as Google Spanner — use physical time to order transactions — but DSQL makes the separation of concerns more explicit. Spanner's TrueTime gives bounded uncertainty on wall-clock order; DSQL uses a softer version (replicas wait for the timestamp to become safe) with the same goal: avoid per-read leader round-trips.

The "X is just an instance of Y" insight: **Aurora DSQL's coordination-free reads are just MVCC reads in a distributed storage system. There is no magic. The replica is just a persistent snapshot; the timestamp is just a cursor into that snapshot. Every distributed read-only SQL database that scales reads already does this. DSQL's contribution is doing it while *also* maintaining full ACID write semantics via adjudicators.**

## Going deeper

1. **Alex Miller, "Decomposing Transaction Systems"** (the direct inspiration for this post) — Miller's framework for systematically classifying transaction systems by which phases they coordinate. Brooker applies this lens to DSQL.

2. **Marc Brooker's DSQL Vignette series** (December 2024, brooker.co.za) — Five short posts that walk through reads and compute, transactions and durability, multi-region design, the CAP tradeoff, and Brooker's personal story building DSQL. More detail on the mechanics than the decomposition post.

3. **Corbett et al., "Spanner: Google's Globally Distributed Database"** (OSDI 2012) — The seminal paper on using physical time (TrueTime with atomic clocks) to eliminate coordination from read paths. Spanner's "external consistency" is the same property DSQL targets; comparing the two illuminates the cost of tight vs. loose clock bounds.
