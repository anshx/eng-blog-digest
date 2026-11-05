---
title: "How to Do Distributed Locking"
source: https://martin.kleppmann.com/2016/02/08/how-to-do-distributed-locking.html
author: Martin Kleppmann
company: University of Cambridge
date_posted: 2016-02-08
date_digested: 2026-11-05
---

# How to Do Distributed Locking

## What's new to learn

1. **Fencing token** — a monotonically increasing integer returned with every lock acquisition; the protected resource rejects any write whose token is lower than the highest it has seen, making stale lock holders harmless even after they wake from a long pause.

2. **Efficiency lock vs. correctness lock** — the two reasons engineers reach for a distributed lock have fundamentally different safety requirements; conflating them is the root cause of most distributed locking bugs.

3. **The paused-process failure** — a process that holds a TTL-based lock can be paused (GC, OS, VM) for longer than the TTL and wake up still believing it holds the lock; no quorum protocol can prevent this, because the protocol has no way to distinguish "process is slow" from "process has crashed."

## Prerequisites

- **Mutex semantics (single-node)**: a lock grants exclusive access; a TTL means the lock auto-releases after a timeout.
- **Redis basics**: a key-value store with per-key expiry. You don't need to know Redlock specifically.
- **Quorum**: the idea that a majority of replicas agreeing on something provides a stronger guarantee than a single replica.
- **What a GC pause is**: a stop-the-world event in which all application threads freeze for tens of milliseconds to seconds.

No distributed systems theory beyond this is assumed — Kleppmann builds the argument from first principles.

## The core idea

Before reaching for a distributed lock, ask: *what is this lock for?*

**Efficiency locks** prevent two workers from doing the same expensive thing twice — recomputing a cache entry, sending a duplicate notification, re-downloading a large file. Races are possible, but losing one means duplicated work, not corrupted state. A single Redis node with a TTL is usually fine here.

**Correctness locks** prevent two writers from corrupting shared state concurrently — two machines both appending to the same file, two jobs both issuing the same invoice, two nodes both believing they are the leader and writing conflicting records. Races here cause data loss or inconsistency. This is the hard case.

Kleppmann's central claim: **no currently practical distributed lock algorithm — including Redlock with its five nodes and majority quorum — provides the safety guarantee required for correctness, unless the protected resource can itself verify lock freshness using a fencing token.**

The reason is the *paused-process problem*, which no amount of quorum voting can solve:

```
Timeline:
  t=0     Client 1 acquires lock on 3/5 Redis nodes. Gets token 33.
  t=1     Client 1 begins writing to the protected resource.
  t=2     Client 1's JVM GC pause begins. All threads freeze.
  t=10    Lock TTL expires on all Redis nodes. Lock released.
  t=11    Client 2 acquires lock on 3/5 Redis nodes. Gets token 34.
  t=12    Client 2 writes to the protected resource with token 34.
  t=15    Client 1's GC pause ends. It resumes, still believing it holds lock.
  t=15    Client 1 writes to the protected resource — ← DATA CORRUPTION
```

At t=15, Client 1 has stale state: its local variable `lock_held = true` is now a lie. The lock expired, was reissued, and Client 2 is using it. Client 1 cannot detect this because it was frozen during the entire sequence of events. It doesn't know whether 100 ms or 10 seconds have passed.

The fencing token is the fix: at t=15, when Client 1 tries to write with token 33, the resource has already seen token 34 from Client 2. It rejects Client 1's write.

## Mechanics

### Acquiring a lock with a fencing token

The lock service maintains a counter that it increments atomically each time it grants the lock. The counter value is the fencing token.

```
Client 1: ACQUIRE_LOCK(key="invoice-12") → {ok: true, token: 33}
  ... (long pause) ...
Client 2: ACQUIRE_LOCK(key="invoice-12") → {ok: true, token: 34}
```

Client 1 and Client 2 each hold a local integer: 33 and 34, respectively.

### Writing to the protected resource

Every write to the protected storage service includes the fencing token:

```
Client 2: WRITE(resource="invoice-12", data=..., fencing_token=34)
  → Storage: highest_seen_token["invoice-12"] was 33, now 34. Write accepted.

Client 1: WRITE(resource="invoice-12", data=..., fencing_token=33)
  → Storage: highest_seen_token["invoice-12"] is 34 > 33. Write REJECTED.
```

The storage service needs one integer per protected resource. That's the entire state required. The key point: the storage service does not need to contact the lock service to verify the token; it only compares integers.

### Why Redlock's random value doesn't help

Redlock generates a 20-byte random string as the "lock value." You include it in SET commands and use a Lua script to atomically delete the key only if it still matches your random string. This prevents Client 2 from accidentally releasing Client 1's lock.

But a random value is not a fencing token because **you cannot compare random strings for ordering**. The storage service cannot determine which of `a3f9c1...` and `d8b2e7...` was issued more recently. Redlock provides no mechanism for protecting against stale writes.

### Why the clock-skew attack breaks Redlock even earlier

Beyond paused processes, Kleppmann describes an attack that breaks Redlock before any client even becomes stale:

```
  t=0    Client 1 acquires lock on nodes 1, 2, 3. Node 3's TTL set to 10s.
  t=1    Node 3's system clock jumps forward 10+ seconds (NTP correction,
         VM clock skew, misconfigured ntpd).
  t=1    Node 3 expires Client 1's lock.
  t=2    Client 2 acquires lock on nodes 3, 4, 5. 
         Both clients now believe they hold the lock.
```

Redlock assumes that clock drift between nodes stays small relative to the TTL. In practice, NTP corrections, virtualization, and suspended/resumed VMs can cause clock jumps of seconds or minutes. Redlock's safety proof requires that you can operationally bound these jumps — a condition you cannot enforce in the algorithm itself.

### ZooKeeper as a correct fencing token source

ZooKeeper is built on ZAB, a total-order broadcast protocol. Two of its primitives naturally provide fencing tokens:

- **zxid (ZooKeeper transaction ID)**: a 64-bit counter that ZooKeeper increments with every write. Any client can read the current zxid, and it is monotonically increasing globally across the cluster.
- **Sequential znodes**: when a client creates a znode with the SEQUENTIAL flag, ZooKeeper appends a 10-digit monotonically increasing sequence number to the node name (e.g., `/locks/invoice-12-0000000034`). This sequence number is a fencing token.

The ZooKeeper leader-election recipe is exactly this pattern: clients create sequential ephemeral znodes; the client whose znode has the lowest sequence number is the leader. The sequence number is the fencing token.

### When fencing isn't enough: the storage layer must cooperate

A fencing token only works if the storage layer **enforces it**. This requires modifying the storage service to:
1. Accept a token with every write.
2. Store the highest token seen per resource.
3. Reject writes with lower tokens.

For some storage backends this is easy (databases support conditional writes; ZooKeeper has built-in version checking via `setData(..., version)`). For others it's hard — for example, NFS file systems have no built-in token enforcement. If you can't modify the storage layer, fencing tokens are not an option.

Kleppmann's most provocative conclusion: **if your storage layer already supports conditional writes (CAS, optimistic locking, versioned updates), you may not need a distributed lock service at all.** You can use the storage's own version number as the fencing token and issue writes as conditional updates.

## Where it breaks

**High-performance, low-latency use cases.** ZooKeeper adds ~1 ms per lock operation and requires a quorum write per acquisition. For use cases like hot-path rate limiting or per-request session locking, the overhead may be unacceptable.

**Storage layers that can't be modified.** If you're writing to a third-party API that doesn't accept fencing tokens, or to a filesystem you can't patch, you're stuck. The fencing token approach requires end-to-end cooperation between the lock service and the storage layer.

**Long-running critical sections.** Even with correct fencing, holding a lock for minutes (e.g., a long batch job) means other clients queue indefinitely. The correct fix is usually to redesign the operation as idempotent and use optimistic concurrency instead of a lock.

**Redlock for efficiency locks.** Kleppmann acknowledges that for pure efficiency (deduplicate work, not enforce correctness), Redlock may be acceptable if you're willing to handle occasional duplicates. Even for this case, a single Redis node is simpler and has the same failure modes — Redlock's additional complexity buys no safety.

## Why it works

The fencing token pattern is **the end-to-end argument applied to mutual exclusion.** The end-to-end argument says: a function (here, exclusivity) can only be correctly implemented at the endpoint that owns the resource; intermediate components (the lock service) can assist but cannot substitute for the endpoint check.

The lock service's "you have the lock" is a statement about its own state at the moment it replied. By the time the client reads the lock, transmits a request to the resource, and the resource processes it, the lock may have expired and been reissued. The only component that can enforce current exclusivity is the resource itself — and it does so by tracking fencing tokens.

Deeper: a fencing token is a **Lamport clock** applied to resource access. Lamport's 1978 paper showed that in a distributed system, messages from a single process are totally ordered (Lamport's "process" = the lock service's sequence counter). Fencing tokens encode that the write operation *causally happens after* the N-th lock grant. The resource enforcing `token ≥ highest_seen` is enforcing the happens-before relation: "writes causally before the latest lock grant are rejected."

This is why fencing tokens are so prevalent under different names:
- **MVCC transaction IDs** (Postgres, MySQL, Spanner): a write with a stale transaction ID is rejected — same pattern.
- **Raft term numbers**: a leader message with a stale term is rejected by followers who've seen a newer term.
- **Spanner commit timestamp + TrueTime commit-wait**: waits until the commit timestamp is provably in the past, so any subsequent reader's snapshot timestamp is always ≥ the commit timestamp.
- **Kafka consumer group offsets**: the broker rejects a commit with an offset lower than the current committed position.
- **Optimistic concurrency control in databases**: "UPDATE ... WHERE version = X" is a fencing token check at the storage layer.

The paused-process problem is an instance of the core impossibility from FLP (1985): in an asynchronous system, you cannot distinguish a slow process from a crashed one. A TTL-based lock treats slowness as a crash, releasing the lock — but the slow process may still be running. The fencing token doesn't solve this impossibility; it makes the resource *safe despite the impossibility*: the slow process's stale write is harmlessly rejected.

## Going deeper

1. **"Designing Data-Intensive Applications," Chapter 8-9 (Kleppmann, 2017)** — The book version of this analysis, covering linearizability, consensus, and distributed transactions in depth. The book uses fencing tokens as a running example of the limits of non-linearizable distributed systems.

2. **antirez's response: "Is Redlock safe?" (antirez.com/news/101)** — Salvatore Sanfilippo, Redlock's designer, disputes Kleppmann's analysis. He argues that the paused-process problem affects all lock services and that an incrementing token can be bolted on. Reading both sides in full is the best systems-thinking exercise in this debate.

3. **The Chubby lock service (Burrows, 2006)** — Google's production distributed lock service, built on Paxos. It uses sequencers (essentially fencing tokens, described as "lock holders prove they still hold the lock via a sequencer") as the primitive for resource access. Chubby is the original production validation of Kleppmann's recommendation. https://research.google/pubs/the-chubby-lock-service-for-loosely-coupled-distributed-systems/
