---
title: "Time, Clocks, and the Ordering of Events in a Distributed System"
source: https://dl.acm.org/doi/10.1145/359545.359563
author: Leslie Lamport
company: SRI International (later Microsoft Research)
date_posted: 1978-07-01
date_digested: 2026-08-24
---

# Time, Clocks, and the Ordering of Events in a Distributed System

## What's new to learn

1. **The happens-before relation (→)**: A formal partial order that captures which events causally precede which others, defined entirely through message passing — no physical clocks required. Two events are either causally related by → or they are *concurrent*.

2. **Lamport timestamps**: A two-line counter rule (increment before each event; on receive, take max+1) that assigns integers to events such that a → b implies C(a) < C(b). This is sufficient to impose a total order on all distributed events.

3. **Vector clocks** (Fidge / Mattern, 1988 extension): A per-process counter array that restores the converse — C(a) < C(b) implies a → b — enabling true discrimination between causally related events and concurrent events, at the cost of O(n) message overhead.

## Prerequisites

- Familiarity with processes communicating by message passing (you do not need formal CSP or Pi-calculus)
- Intuition for why wall-clock timestamps are unreliable across machines (NTP drift, clock skew)
- The archive entries on Raft (2026-06-12), Chandy-Lamport snapshots (2026-07-10), and Spanner TrueTime (2026-06-06) are excellent motivation: this paper is the formal foundation every one of those posts assumes without stating

## The core idea

Physical clocks are liars. Two servers can both report "14:00:00.001 UTC" for events that are causally related — the first event genuinely caused the second via a network message — but if their clocks are 2 ms apart the timestamps say nothing useful. You cannot use wall-clock time alone to reconstruct causal order in a distributed system.

Lamport's 1978 insight is to **decouple ordering from time**. You do not need to know *when* things happened. You only need to know *what caused what*. And causality has a clean, message-based definition:

The **happens-before** relation → (also written "causally precedes"):

1. **Same process**: If event *a* executes before event *b* in a single process, then a → b.
2. **Message delivery**: If *a* is the sending of a message and *b* is the receipt of that same message, then a → b.
3. **Transitivity**: If a → b and b → c, then a → c.
4. **Concurrency**: If neither a → b nor b → a, then *a* and *b* are concurrent, written a ∥ b.

The relation → is a strict partial order — irreflexive, asymmetric, transitive. It is not total: concurrent events are simply incomparable.

A **Lamport clock** assigns a non-negative integer C to every event such that the **Clock Condition** holds:

> If a → b, then C(a) < C(b).

The algorithm to achieve this is remarkably simple:

1. Before executing any event, the process increments its local counter: L ← L + 1.
2. When sending a message, the process stamps the message with its current L.
3. When receiving a message stamped T, the process sets L ← max(L, T) + 1, then assigns C(receive) = L.

That is the entire algorithm. The max+1 rule propagates causality across processes: if P1 sends at L=42 and P2's counter is at 7 when it receives the message, P2 jumps to L=43. All of P2's subsequent events are guaranteed to have a higher timestamp than P1's send, preserving the Clock Condition.

## Mechanics

**Full Lamport clock algorithm for process Pi with counter Li:**

```
local event:
    Li ← Li + 1
    C(event) ← Li

send message m:
    Li ← Li + 1
    m.timestamp ← Li
    send(m)

receive message m:
    Li ← max(Li, m.timestamp) + 1
    C(receive) ← Li
```

**Total order**: To convert the partial order → into a total order (needed for replicated state machines, distributed mutex), break timestamp ties by process ID. Define event *a* at process *i* as "before" event *b* at process *j* if C(a) < C(b), or if C(a) = C(b) and i < j. Every pair of events is now ordered, consistently across all processes.

**Distributed mutual exclusion** — the paper's killer application: Lamport shows you can implement a fair, starvation-free distributed lock across n processes using only the Lamport clock total order, with zero central coordinator:

1. **Request**: Pi broadcasts REQUEST(Ti, i) and adds itself to its local queue sorted by (Ti, i).
2. **Acknowledge**: When Pj receives a REQUEST, it queues the request and replies ACK(Tj, j).
3. **Enter**: Pi enters the critical section when (a) its own REQUEST(Ti, i) is first in its local queue, AND (b) Pi has received a message from every other process with a timestamp strictly greater than Ti.
4. **Release**: Pi broadcasts RELEASE(Ti, i); everyone dequeues the request.

Since every process sorts by (Ti, i) identically, the queues are always in the same order at all processes. No coordinator, no leader, no Paxos needed — just counters and sorted queues.

**Vector clocks** (Fidge 1988, Mattern 1988):

Lamport clocks satisfy C(a) < C(b) ⟸ a → b but NOT the converse. You cannot tell from Lamport timestamps alone whether two events are causally related or concurrent. Vector clocks fix this.

Each process i maintains a vector Vi[1..n] (n = total number of processes):

```
local event at i:
    Vi[i] ← Vi[i] + 1

send from i:
    Vi[i] ← Vi[i] + 1
    include Vi in message

receive from j with vector W at i:
    Vi[k] ← max(Vi[k], W[k])  for all k
    Vi[i] ← Vi[i] + 1
```

**Comparison**: V < W iff every component V[k] ≤ W[k] and at least one is strictly less. Concurrent: V ∥ W iff neither V < W nor W < V.

The key property: **a → b if and only if V(a) < V(b)**. Vector clocks are therefore a faithful representation of the happens-before partial order.

Example: process P1 sends to P2 after local event a, which P2 receives before event b:
- C(a) = (1, 0) after P1's local increment
- C(send) = (2, 0), timestamp on message = (2, 0)
- C(b) = (2, 1) after P2 receives and does max+1

V(a) = (1,0) < (2,1) = V(b), confirming a → b. If instead P2 had already been at (3, 0) before the receive, b would have been at (3,1), and (1,0) < (3,1) still holds. The ordering is preserved regardless of interleaving.

## Where it breaks

**Lamport clocks do not detect concurrency**: C(a) < C(b) does not mean a caused b. You cannot use Lamport timestamps alone to discover that two writes are concurrent. Dynamo needed vector clocks to surface conflicting writes to the application. Systems that use only a single global counter (like many databases' transaction IDs) cannot distinguish "these two writes happened concurrently" from "write A happened before write B."

**Vector clocks do not scale**: Every message carries O(n) integers. At n = 1000 processes this is 4–8 KB of per-message metadata — unacceptable at scale. Real systems use approximations: Riak's *dotted version vectors* (track only causality from the storage layer's perspective), *interval tree clocks* (dynamic process participation), CockroachDB's *hybrid logical clocks*.

**Physical clocks can win if you pay the latency**: Spanner (in the archive, 2026-06-06) does not use Lamport counters at all. It uses GPS receivers and atomic oscillators (TrueTime) accurate to ±ε ms, then imposes a *commit-wait* of ε ms per commit to guarantee that a transaction's commit timestamp is truly in the past when it becomes visible. TrueTime converts the Clock Condition from a logical guarantee into a physical one, at the cost of 7–14 ms per write.

**The Clock Condition is one-directional**: The paper's Clock Condition says C(a) < C(b) is *necessary* for a → b, not sufficient. Any system that treats a lower Lamport timestamp as *evidence of causality* (rather than just ordering) is making an error.

## Why it works

The deeper principle: **time is just causality, and causality is just message reachability**.

The happens-before relation is the *transitive closure of the message-delivery graph*. If there is a directed path of messages from event *a* to event *b*, then a → b. Lamport clocks and vector clocks are efficient bookkeeping schemes for this one fact: rather than storing the entire message graph, you summarize reachability as a counter (Lamport) or a counter-per-process (vector).

This explains why every distributed system "timestamp" is secretly a Lamport clock:

- **Raft log indices**: each committed entry at index k was causally prior to all entries at k+1, k+2, ... The log index is a Lamport clock for the replicated state machine.
- **Kafka partition offsets**: offset k was written before offset k+1. Lamport clock per partition. Cross-partition ordering requires transactions (exactly the multi-partition extension of the problem Lamport solved with his distributed mutex).
- **MVCC transaction IDs (xid)**: every transaction that commits at xid T serialized before any transaction at xid T+1. Lamport clock for the serialization order. The Percolator archive entry (2026-07-18) is a detailed engineering of this.
- **ZooKeeper's zxid**: a monotonically increasing 64-bit transaction identifier issued by the ZAB leader. Lamport clock for the total broadcast order. Everything in the archive entry (2026-08-12) about ZAB is an implementation of the Lamport distributed mutex.
- **Dapper/Jaeger trace spans**: the parent-child relationship between spans encodes the → relation directly. The trace DAG *is* the happens-before graph, restricted to one request's causal chain.
- **Spanner's commit-wait**: instead of a logical counter, Spanner uses a physical clock accurate to ±ε ms, then stalls the commit by ε ms. The stall IS the Clock Condition enforced by physics instead of counting.
- **Git commit hashes**: a commit's SHA depends on its parent's SHA, making the hash a cryptographic Lamport clock. You cannot fabricate an event that appears to have "happened before" an existing commit without invalidating all its descendants.
- **The Chandy-Lamport barrier (archive entry 2026-07-10)**: the barrier token flows through the same FIFO channels as data, so receiving the barrier always happens-after the data that preceded it. The barrier defines a "cut" in the → graph — a set of events such that for every pair (a in snapshot, b not in snapshot), b → a is impossible.

One sentence encapsulates the paper's legacy: **once you stop trying to measure time and start tracking causality instead, all of distributed systems becomes the question "did a message path exist between these two events?"**

## Going deeper

1. **Hybrid Logical Clocks** — Kulkarni, Demirbas, et al. 2014. Combines Lamport counters with wall-clock timestamps so that HLC values are interpretable as physical time while always satisfying the Clock Condition. The practical default for databases that want both human-readable timestamps and causal correctness (CockroachDB, TiKV). Available at https://cse.buffalo.edu/tech-reports/2014-04.pdf

2. **"Timestamps in Message-Passing Systems That Preserve the Partial Ordering"** — Colin Fidge, 1988 (and Mattern's independent concurrent work). The original vector clock paper. Short, readable, directly extends the Lamport 1978 result to the bidirectional case.

3. **"Don't Settle for Eventual: Scalable Causal Consistency for Wide-Area Storage with COPS"** — Lloyd et al., SOSP 2011. Shows how to use vector clocks at scale (one entry per datacenter, not per key) to provide causally consistent reads across geographies. A direct engineering translation of Lamport clocks into production systems.
