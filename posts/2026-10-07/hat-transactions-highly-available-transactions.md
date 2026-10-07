---
title: "Highly Available Transactions: Virtues and Limitations"
source: https://www.vldb.org/pvldb/vol7/p181-bailis.pdf
author: Peter Bailis, Aaron Davidson, Alan Fekete, Ali Ghodsi, Joseph M. Hellerstein, Ion Stoica
company: UC Berkeley / University of Sydney
date_posted: 2014
date_digested: 2026-10-07
---

# Highly Available Transactions: Virtues and Limitations

## What's new to learn

1. **Highly Available Transaction (HAT)**: a transactional guarantee that any single replica can enforce without synchronous communication with other replicas during a transaction — the formalization of "coordination-free transactions."
2. **The achievability boundary**: not all isolation levels cost the same in availability. Read Committed, Monotonic Atomic View, and Causal Consistency are fully HAT-achievable; Snapshot Isolation and Serializability are not, for provable, structural reasons.
3. **Monotonicity as the unifier**: a guarantee is HAT-achievable if and only if it is expressible as a monotone property over a replica's append-only write history — the same condition the CALM theorem gives for coordination-free computation, applied directly to the SQL isolation taxonomy.

## Prerequisites

- Basic SQL isolation levels: Read Uncommitted, Read Committed, Repeatable Read, Serializable
- The CAP theorem at a high level (availability vs. consistency under network partitions)
- What write skew is — the archive's 2026-05-18 Snapshot Isolation entry covers it as background
- Optionally: the 2026-07-28 CALM theorem entry, which HAT directly applies

## The core idea

The CAP theorem is widely misread as "pick consistency or availability." The real claim is narrow: *linearizability* — the strongest consistency model, requiring every operation to appear instantaneous — is incompatible with high availability. Between linearizability and "anything goes" lies a rich spectrum of transaction guarantees, many of which are fully achievable even during a network partition.

Bailis et al formalize this spectrum. A transaction protocol is **Highly Available (HAT)** if, even when network partitions split a cluster into isolated islands, every replica can still commit transactions — with no synchronous cross-partition communication required. The paper then asks: for each classical ANSI SQL isolation level, does there exist a HAT implementation?

The answer divides cleanly into two groups.

**HAT-achievable** (no coordination needed):
- Read Uncommitted, Read Committed
- Monotonic Atomic View (MAV): once you see *any* write from transaction T, you will eventually see *all* of T's writes
- Causal Consistency
- Session guarantees: Read-Your-Writes, Monotonic Reads, Monotonic Writes, Writes-Follow-Reads
- PRAM (Pipeline RAM) consistency

**Not HAT-achievable** (coordination fundamentally required):
- Snapshot Isolation (SI)
- Serializability
- Recency / linearizability guarantees
- Preventing Lost Updates (in the concurrent read-modify-write sense)
- Preventing Phantoms (traditional A3/P3 definition)

The boundary falls exactly where intuition breaks down. Many engineers assume Repeatable Read or SI are "cheap" because they're weaker than Serializability. But both require knowing about *concurrent writes on other partitions* — which is inherently non-local.

## Mechanics

**Why Read Committed is free.** To prevent dirty reads, a replica buffers uncommitted writes in a private staging area and makes them visible only after commit acknowledgment. This is purely local state. No other replica needs to be consulted. Every MVCC implementation already does this: the key insight is that "has this write committed?" is a monotone predicate — once committed, always committed — so answering it requires no inter-replica round trip.

**Why Monotonic Atomic View (MAV) is HAT-achievable.** MAV prevents *fractured reads*: seeing some but not all writes of a committed transaction. The implementation: each transaction carries a unique ID; replicas tag all writes with that ID and refuse to serve reads for those keys until all writes bearing that ID have been applied locally. With multi-version storage this is a simple filter on the version chain. A client connecting to any replica will see a complete transaction or none of it — without the replicas ever needing to coordinate during the transaction.

**Why Causal Consistency is HAT-achievable.** Clients track causal dependencies using vector clocks or per-client logical timestamps. When a client reads a value it causally depends on (its own prior writes, or writes it has observed), it provides its clock to the replica, which delays the response until its local state is at least as advanced. Each replica makes progress independently; the causal ordering is encoded in the clock attached to each request, not in cross-replica locks.

**Why Snapshot Isolation requires coordination.** Under SI, transaction A reads from a snapshot at time T_start and commits only if no *concurrent* transaction wrote to any key A read. "Concurrent" means: the other transaction started before A's T_commit and committed after A's T_start. To enforce this, when A tries to commit on replica 1, it must know whether transaction B on replica 2 is concurrently writing to the same keys. During a network partition, replica 1 cannot reach replica 2. The check is impossible. SI requires an unachievable message.

More concretely: preventing Write Skew (the canonical SI-specific anomaly) requires two transactions to coordinate their read sets — "I won't write X unless no concurrent transaction is writing Y." There is no way to establish this agreement without a synchronous cross-partition message.

**Why Phantom prevention requires coordination.** Preventing phantoms requires holding a predicate lock: "no new row matching this WHERE clause appeared between my read and my commit." Predicates are open-world. You cannot know that a row you haven't seen doesn't exist on another partition without asking that partition. This is structurally non-local.

**A concrete implementation — Eiger (Bailis et al's prototype).** Write transactions collect all reads, send all writes in one round trip to the local primary for each key; primaries broadcast writes tagged with a transaction ID and vector-clock timestamp. Read transactions are served entirely from the local replica using MVCC with causal-timestamp checking; fractured reads are avoided by verifying that all writes for a given transaction ID are locally present before returning. Result: transactions complete in 1–2 WAN round trips versus 4+ for 2PC-backed SI or Serializable systems.

## Where it breaks

**Write skew is the critical gap.** Two doctors on call: each transaction reads "is anyone else on call?" and writes "I'm going off call." Both see "yes, someone else is on call," so both commit. Result: nobody is on call. HAT systems cannot prevent this without coordination, because each replica's decision depends on what the other replica *hasn't done yet* — a non-local, non-monotone predicate.

**Lost updates in concurrent read-modify-write.** If two transactions read a counter's current value and both increment it, a HAT system lets both succeed with the same starting value, losing one increment. Systems like Cassandra that use Last-Write-Wins semantics lose updates deterministically based on timestamp ordering.

**Recency is unachievable.** HAT systems cannot bound staleness. A replica that has been partitioned for an hour can still serve reads from its stale state. If your application requires "read the value written within the last second," that is a linearizability requirement and requires coordination.

**Fractured reads across replicas at connection boundaries.** If a client reads key K from replica A (which has transaction T's writes) and then reads key K′ from replica B (which hasn't received T's writes yet), it sees inconsistent state. MAV prevents this only eventually — a client that migrates between replicas mid-transaction may observe a window of inconsistency.

**Complexity in practice.** Building correct MAV over an eventually consistent store is subtle. Most production "NoSQL" systems don't implement the full HAT taxonomy. Cassandra's base layer provides something closer to Read Committed + Causal Consistency for single-key operations; its Lightweight Transactions (LWTs) layer Paxos on top precisely for the cases — CAS, conditional updates — that are not HAT-achievable.

## Why it works

The deepest insight: **HAT-achievable guarantees are exactly the monotone properties of a replica's append-only write log.**

The CALM theorem (in this archive, 2026-07-28) states: a distributed computation is coordination-free if and only if it is monotone — meaning once a fact is established, it stays established as more data arrives.

Apply this lens to isolation guarantees:

| Guarantee | Predicate | Monotone? | HAT-achievable? |
|---|---|---|---|
| Read Committed | "this write has been committed" | Yes — committed is permanent | Yes |
| MAV | "I have all of transaction T's writes" | Yes — set only grows | Yes |
| Causal ordering | "A causally precedes B" | Yes — causal order is permanent | Yes |
| No Write Skew (SI) | "no concurrent transaction is writing key K" | No — new concurrent writes appear | No |
| No Phantom (SI) | "no row matching predicate P exists" | No — new rows can appear | No |
| Recency | "this is the latest value" | No — later writes supersede | No |

Every isolation guarantee that requires ruling something *out* — "no concurrent writes," "no phantom rows," "not stale" — is non-monotone and therefore fundamentally requires coordination. Every guarantee that only requires ruling something *in* — "has been committed," "is causally ordered," "was seen at least once" — is monotone and achievable for free.

This is why HAT is CALM applied to SQL: the CAP theorem tells you *that* a limit exists; the CALM theorem tells you *where* the limit is; and the HAT paper maps that limit onto the specific isolation levels in every database's transaction documentation.

The practical implication is counterintuitive: engineers who want "strong consistency" often actually want only Read-Your-Writes plus Causal Consistency — achievable at full availability. The CAP framing makes these sound like sacrifices on the "weak" side of a binary choice. The HAT framework shows they are not sacrifices at all: they are first-class guarantees that any replica can provide, unconditionally, even during a total network partition.

## Going deeper

1. **"Coordination Avoidance in Database Systems"** — Bailis et al, VLDB 2015. Extends HAT with *I-confluence*: a formal condition for when an application's specific business invariants (not just generic isolation levels) permit coordination-free execution. Answers "for *my* application, do I need coordination?" rather than "for this isolation level, do I need coordination?"

2. **"A Critique of ANSI SQL Isolation Levels"** — Berenson, Bernstein, Gray, Melton, O'Neil, O'Neil, SIGMOD 1995. The foundational taxonomy HAT builds on. Formally defines every anomaly (dirty write, dirty read, fuzzy read, phantom, lost update, read skew, write skew) and shows the original ANSI SQL definitions are ambiguous. The paper that introduced Snapshot Isolation as a named concept.

3. **Eiger source code and design doc** — the reference implementation of MAV over Cassandra. Reading how MAV is implemented on top of an eventually consistent KV store concretizes exactly where the boundary between "achievable" and "requires coordination" falls in real code.
