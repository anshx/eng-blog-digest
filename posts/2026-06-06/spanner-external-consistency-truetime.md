---
title: "Spanner under the hood: Understanding strict serializability and external consistency"
source: https://cloud.google.com/blog/products/databases/strict-serializability-and-external-consistency-in-spanner
author: Doug Judd
company: Google
date_posted: 2023-04-07
date_digested: 2026-06-06
---

# Spanner under the hood: Understanding strict serializability and external consistency

## What's new to learn

- **TrueTime** — A clock API that returns a bounded uncertainty interval `[earliest, latest]` instead of a single timestamp, backed by GPS receivers and atomic clocks on dedicated time-master servers; typically sub-millisecond uncertainty at p99.
- **Commit-wait** — A write-transaction step that blocks until `TrueTime.now().earliest > commit_timestamp`, converting bounded clock uncertainty into latency and guaranteeing a transaction's timestamp is provably in the past before any client can observe its effects.
- **External consistency** — A property stronger than strict serializability: if transaction T1's `Commit()` call returns before T2's `Commit()` call is made, then T1's commit timestamp is strictly less than T2's — the database's logical ordering matches real wall-clock ordering, with no global coordinator required.

## Prerequisites

- **MVCC / snapshot reads** — databases serve reads from versioned row snapshots keyed by timestamp; a reader at timestamp *s* sees the most recent committed version ≤ *s*.
- **Paxos/Raft basics** — writes are appended to a replicated log; a quorum of replicas must acknowledge a write before it is considered durable.
- **Serializability** — transactions execute as if they ran one at a time in some serial order; *strict* serializability also requires that serial order is consistent with real time.
- **Why clocks drift** — NTP keeps servers within ~1–10 ms of each other but makes no hard guarantees; GPS-synchronized atomic clocks shrink this to tens of microseconds, but even then a software API cannot return the *exact* current time — it can only bound the uncertainty.

## The core idea

Picture two transactions touching completely different shards in different datacenters. T1 commits in us-east; T2 commits in us-west, half a second later in real time. If each datacenter just uses its local clock to pick timestamps, clock skew between the two can make T2's timestamp *smaller* than T1's. Now any client that reads both shards sees T2's write but not T1's — the database looks causally broken, even though both transactions were serializable in their own region.

The classical fix is global coordination: before committing, contact a central timestamp authority (like a Paxos leader) to get a monotonically increasing global timestamp. This works, but it means every write requires a cross-datacenter round trip to the coordinator.

Spanner's fix is to lean on physics rather than coordination. It observes that you don't need to *ask* someone what time it is — you need to *prove* that your timestamp is in the past. TrueTime gives you that: `TT.after(s)` returns `true` once it can guarantee that real time has passed *s*. Spanner's commit protocol picks a timestamp *s* and then *waits* until `TT.after(s)` is true before acknowledging the commit to the client. By the time the client learns "T1 committed with timestamp *s*", the universe has already advanced past *s*. Any T2 that begins after this acknowledgment will pick a TrueTime timestamp that starts at a later moment — so T2.commit\_ts > T1.commit\_ts, guaranteed, with no coordinator involved.

The tradeoff is explicit: you convert clock uncertainty into latency (typically 7–14 ms per write transaction) instead of into coordination cost. Because GPS-backed TrueTime keeps uncertainty very small, that latency tax is small enough to be acceptable for most workloads.

## Mechanics

### The TrueTime API

```
TT.now()    → TTinterval { earliest, latest }
TT.after(t) → bool   // true if t is definitively in the past
TT.before(t)→ bool   // true if t is definitively in the future
```

The uncertainty bound ε = (latest − earliest) / 2 is typically < 500 µs at p99 in production Google datacenters. Each machine runs a time daemon that polls time-master servers; each datacenter's time masters carry GPS receivers (absolute time reference) and atomic clocks (stable oscillators that drift slowly between GPS syncs). The daemon models its own clock error and advertises ε conservatively — it widens ε rather than risk underestimating uncertainty.

### Read-write transaction commit (6 steps)

```
1. Acquire row locks for all cells being read/written (2PL)
2. Choose commit timestamp:
       s = max(TT.now().latest, s_maxprev + 1)   ← monotonicity invariant
3. *** COMMIT WAIT ***
       while !TT.after(s): sleep
       // blocks until TT.now().earliest > s
4. Write s and the mutation set to the Paxos log on each participant group;
   wait for quorum acknowledgment
5. Release all locks
6. Return "committed at timestamp s" to the client
```

Step 2's formula has two parts: `TT.now().latest` ensures *s* is not in the past relative to this machine's clock, and `s_maxprev + 1` ensures timestamps are monotonically increasing within a Paxos group (in case a leader re-election replayed an old timestamp).

Step 3 is the key insight: the commit is *not acknowledged* until real time has definitively passed *s*. The wait is typically a few milliseconds (≈ 2ε). Because steps 3 and 4 can overlap (the Paxos log write can proceed in parallel with the wait), the effective latency addition is just the tail of the two.

### Lock-free read-only transactions

A read-only transaction needs no locks and requires no coordination with other transactions:

1. Choose read timestamp: `s_read = TT.now().latest` (or a slightly older "safe" timestamp to avoid waiting on in-progress transactions).
2. For each shard, pick any up-to-date replica and read the latest version ≤ `s_read`.
3. A replica can serve a read at timestamp *t* only when its **safe time** `t_safe ≥ t`. A replica's safe time is bounded by:
   - The timestamp of the last applied Paxos write, and
   - The minimum timestamp of any prepared-but-not-yet-committed transaction on that replica (a 2PC participant that's holding a lock but hasn't committed is conservatively assumed to be below its prepare timestamp).

Because commit-wait guarantees that every write with timestamp ≤ `s_read` was *already committed* before any client was told about it, a replica only needs to wait for in-flight transactions to resolve — it never needs to lock or coordinate with writers. In practice, most reads don't wait at all.

### Multi-shard transactions and 2PC

When a transaction spans multiple Paxos groups (e.g., debiting account A in one shard and crediting account B in another), Spanner uses two-phase commit with one shard elected as the coordinator. Each participant writes its portion of the mutation to its own Paxos log; the coordinator collects all "prepared" votes, chooses the final commit timestamp *s* (the maximum of all participant-proposed timestamps, then commit-waited), and broadcasts the commit record. This adds two cross-datacenter round trips to the commit path.

## Where it breaks

**Requires specialized hardware.** TrueTime's <1 ms uncertainty comes from GPS receivers and atomic clocks co-located in every datacenter. Running Spanner on commodity hardware (with NTP) degrades uncertainty to ~100–500 ms, making commit-wait costs prohibitive. CockroachDB approximates TrueTime by adding a configurable "maximum clock offset" (default 250–500 ms) as a pessimistic bound — it works, but every cross-shard write waits up to twice that offset.

**Write latency floor.** Even with atomic clocks, every read-write transaction pays ≈ 2ε ≈ 7–14 ms for commit-wait. This is the irreducible cost of external consistency. If your application already serializes writes through a single coordinator, you can get strict serializability without this tax.

**2PL contention.** Spanner uses strict two-phase locking (locks held until after commit). Long-running transactions block other transactions for the entire commit duration. Systems like CockroachDB use optimistic concurrency control to avoid this, at the cost of aborting on conflict.

**Cross-shard write overhead.** A write touching *k* shard groups requires *k* Paxos-round-trip write operations plus a 2PC coordination round trip. For writes that touch many shards, this stacks up.

**External consistency scope.** The guarantee "T1 commits before T2 starts → ts(T1) < ts(T2)" holds only for *non-overlapping* transactions (T2 starts after T1 finishes). Concurrent transactions are merely serializable; the external ordering between them is not defined.

## Why it works

The deeper principle: **convert uncertainty to latency, not coordination.**

Most distributed systems handle clock uncertainty in one of two ways:

1. **Ignore it** (bugs happen when clocks disagree).
2. **Use a global coordinator** to generate timestamps (adds a round-trip per write).

TrueTime introduces a third way: **quantify the uncertainty exactly, then wait it out**. The bound ε is small (~7 ms), the wait is cheap (sleep a few milliseconds), and the result is that every subsequent *read* operation becomes lock-free and coordination-free — you paid the coordination cost once at write time, in time units rather than in message-passing overhead.

This pattern recurs across systems:

- **Lease-based lock management**: a new leader doesn't act until it is certain the old leader's lease has expired (`now > lease_expiry`). Same commit-wait, different context: wait until the uncertainty window closes before taking an exclusive action.
- **CDN cache invalidation with TTL**: baking a pessimistic TTL into cached objects means "I know everyone has fetched a fresh copy by now." Converting temporal uncertainty into a time budget.
- **Hybrid Logical Clocks (HLC)**: advance the logical clock to `max(physical_clock, last_seen_ts + 1)`. HLC bounds the drift between physical and logical time the same way TrueTime bounds clock uncertainty — by making the uncertainty a first-class, reasoned-about quantity rather than an invisible source of bugs.
- **Read-your-writes via causal tokens** (see the Zanzibar zookie in this archive): a causality token encodes "safe to read after timestamp t," which is another form of "wait until real time passes t before returning."

The unifying principle is: *physics gives you a monotonic, forward-moving signal for free; most distributed consistency problems collapse to "prove you are after time t," and the cheapest proof is to wait*. The expensive thing — coordination — is only needed when you can't tolerate latency. TrueTime quantifies exactly how much latency you must tolerate, and it turns out to be very little.

## Going deeper

1. **Spanner: Google's Globally-Distributed Database** (Corbett et al., OSDI 2012) — the original paper with the full TrueTime API specification, the formal proof of external consistency, and production latency/throughput numbers: https://research.google.com/archive/spanner-osdi2012.pdf
2. **Living Without Atomic Clocks** (CockroachDB blog) — how CockroachDB approximates TrueTime using NTP with a bounded offset, and the concrete places where that approximation costs them: https://www.cockroachlabs.com/blog/living-without-atomic-clocks/
3. **Use of Time in Distributed Databases** (Murat Demirbas, January 2025) — a four-part series comparing TrueTime, HLC, and Logical Clocks in production databases; part 4 focuses on synchronized clocks in CockroachDB, YugabyteDB, and Spanner: http://muratbuffalo.blogspot.com/2025/01/use-of-time-in-distributed-databases.html
