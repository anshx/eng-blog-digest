---
title: "Choosing QuePaxa, an unusual consensus algorithm"
source: https://blog.cloudflare.com/choosing-quepaxa/
author: James Larisch
company: Cloudflare
date_posted: 2026-07
date_digested: 2026-11-21
tags: [distributed-systems, consensus, raft, paxos]
---

# Choosing QuePaxa, an unusual consensus algorithm

Cloudflare built Meerkat — an internal, globally distributed consensus service
for control-plane data — and chose an unusual algorithm called **QuePaxa**
(SOSP 2023) rather than the industry-standard Raft or Multi-Paxos. The post
explains why, and walks through what makes QuePaxa structurally different.

---

## What's new to learn

1. **Hedging instead of electing.** Raft and Paxos rely on a single elected
   leader and use a timeout to detect when it has failed. QuePaxa replaces
   leader elections with *concurrent hedged proposals*: any replica that hasn't
   seen a commit for a configurable delay just starts proposing on its own.
   The timeout no longer affects correctness or liveness — only message cost.

2. **The Interval Summary Register (ISR).** QuePaxa's atomic building block is
   the ISR, a shared register that accepts proposal values alongside a numeric
   priority, tracks all accepted (priority, value) pairs, and returns a
   "summary" that lets later proposers reconstruct what any previous majority
   may have accepted. The ISR is what allows multiple concurrent proposers
   without destructive interference.

3. **Leader as a performance optimization, not a protocol requirement.**
   In Raft/Multi-Paxos the leader *is* the protocol — removing it stalls
   progress until a new one is elected. In QuePaxa a "primary" proposer reduces
   the common case to 1 RTT, but any other replica can step in mid-slot and the
   protocol still converges. The leader is an optimization, not a correctness
   invariant.

---

## Prerequisites

- **Paxos Prepare/Promise/Accept/Decide** — the two-phase structure QuePaxa
  extends. A prepare request establishes a ballot number and blocks lower
  ballots; an accept request commits a value under a majority.
- **Quorum intersection** — why any two majority quorums share at least one
  member, and why that makes it safe for different proposers to pick up each
  other's in-flight proposals.
- **FLP impossibility** — deterministic consensus is impossible in a purely
  asynchronous system with even one crash. QuePaxa, like all practical
  protocols, assumes partial synchrony for efficiency; unlike most, it uses
  randomization to guarantee liveness even under adversarial scheduling.
- **The leader-election bottleneck** — in Raft, election timeout is a
  correctness-adjacent parameter: too short triggers false elections (reduces
  availability), too long delays recovery after a real crash. Under WAN
  conditions with variable RTTs, there is genuinely no "right" value.

---

## The core idea

Raft frames consensus as: *pick one leader, route everything through it,
detect crashes via heartbeat timeouts, elect a replacement*.

The heartbeat timeout is load-bearing: it must be long enough to avoid false
elections under normal jitter, yet short enough to recover quickly when the
leader actually crashes. Under a WAN with RTTs ranging from 5 ms (same
region) to 300 ms (cross-continental), this is an unsolvable tuning problem —
any single value is simultaneously too short for some paths and too long for
others.

QuePaxa's insight: **don't elect a new leader when the old one is slow —
just let other replicas propose concurrently**. If the primary's proposal
reaches a majority first, it wins in 1 RTT. If a non-primary also started
proposing (because it hadn't seen a commit for a hedge interval), both
proposals run concurrently, and the protocol converges on exactly one value
without stalling. The primary's "leadership" is dissolved per-slot: for each
log slot, whoever gets a majority first wins.

Critically, two simultaneous proposals don't corrupt the system. The ISR
ensures that if either proposal has been accepted by any quorum member, later
proposals will adopt the already-accepted value instead of competing with it —
the same quorum-intersection safety guarantee Paxos has, but now preserved
under concurrency.

---

## Mechanics

### The Interval Summary Register

Each acceptor stores an ISR per log slot. An ISR supports two operations:

```
propose(slot, value, priority) → (highest_seen_priority, committed_value?)
summary(slot) → (highest_seen_priority, all_accepted_entries)
```

When a proposer calls `propose`, the recorder atomically:
1. Rejects the call if a higher-priority proposal has already arrived.
2. Otherwise records (priority, value) and returns the highest priority it has
   ever seen for this slot.

The `priority` is a randomly chosen integer freshly generated per proposer
per proposal attempt. This is what breaks symmetry between concurrent
proposers — the one with the highest priority at any recorder "wins" that
recorder's vote.

### Phase 0: The Leader Fast Path

Each slot has a designated "primary" proposer (configurable; can rotate via
round-robin, a fixed leader, or a multi-armed bandit that optimizes on
observed commit latency). In the normal case:

1. The primary picks a random priority `p`, sends `propose(slot, v, p)` to
   all recorders.
2. If a majority of recorders return success (no higher priority seen), the
   slot is committed in **1 RTT**.
3. The primary broadcasts the commit, and all replicas apply the value.

This is structurally identical to Multi-Paxos Phase 2 (the leader's prepare is
done once during initialization, not per slot).

### Phases 1–3: Concurrent Slow Path

A non-primary replica starts its own proposal for a slot after a *hedge
interval* elapses with no observed commit. This is not an election — it fires
unconditionally and runs concurrently with any existing primary attempt.

**Phase 1 — Prepare:** The replica broadcasts a prepare message to claim the
slot. Each recorder replies with a summary of all (priority, value) pairs it
has accepted. This is the classic Paxos Phase 1; its purpose is to discover
any value that might already have been accepted by a quorum.

**Phase 2 — Accept:** The proposer examines all summaries. If any value
appears with a priority high enough that it may have been accepted by a
previous majority, the proposer must adopt that value (Paxos safety rule). The
proposer then broadcasts an accept for that value (or its own if none was
found), along with its own random priority.

**Phase 3 — Decide:** Once a majority of recorders accept in Phase 2, the
proposer knows the value is committed. It broadcasts the decision.

If two proposers are running Phase 2 concurrently, one of them will see that
the other's proposal was accepted by some recorders and adopt the
higher-priority value. Because priorities are random and uncoordinated, the
two proposals converge to one value within O(1) additional round trips with
high probability.

Cloudflare reported that **>95% of decisions use Phase 0** (the fast path).
The end-to-end implementation benchmarks show 250,000 ops/sec with p50 7.2 ms
and p99 14 ms on loopback with fsync-per-write.

### Hedging, Not Electing

The hedge interval is the duration a non-primary replica waits before starting
its own proposal. Setting it short means more redundant proposals (and wasted
bandwidth); setting it long means higher tail latency when the primary is slow.
But unlike a Raft election timeout:

- Setting it too short never causes a correctness issue (no split-brain
  possible, no "term" corruption).
- Setting it too long only increases tail latency; the system does not stall.
- It can be tuned independently per replica, per region, or dynamically via
  a multi-armed bandit optimizer.

This is the structural difference from Raft: Raft's timeout is a
*safety-adjacent* parameter whose misconfiguration can cause availability loss.
QuePaxa's hedge interval is a pure *performance* parameter.

### Membership Changes and Reads

Membership changes go through the log and take effect 64 slots after their
commit — a simple, verifiable window that ensures any majority elected under
the old membership also overlaps with any majority under the new one. This is
more conservative than Raft's joint consensus but considerably easier to
reason about.

Linearizable reads require one round trip to a majority (read returns only
when a majority of recorders confirm no higher-slot commit is in-flight) — no
lease or clock tricks needed.

---

## Where it breaks

**Higher protocol complexity.** The slow path has up to 4 phases; Raft under
normal conditions has 1–2. The ISR is a novel data structure with no
off-the-shelf battle-tested implementations. Debugging a multi-proposer
protocol under partial failures is significantly harder.

**More per-slot state.** The ISR must track all accepted (priority, value)
pairs, not just the single highest-ballot accepted value that classical Paxos
recorders need. Under high concurrency this grows, but bounded by the number of
concurrent proposers.

**Still needs partial synchrony for efficiency.** FLP says a purely
asynchronous protocol cannot guarantee termination. QuePaxa's randomized
priorities provide liveness asymptotically — eventually any two concurrent
proposals converge — but under an adversarial scheduler that always delays the
higher-priority message, convergence can be slow. In practice, this scenario
does not arise; under normal partial synchrony, the fast path dominates.

**Not Byzantine fault tolerant.** QuePaxa is designed for crash faults only.
Under Byzantine failures, a malicious recorder could lie about its ISR state,
breaking safety. HotStuff or PBFT are needed for Byzantine settings.

**Still experimental at Cloudflare.** Meerkat was not in production as of the
post's date. It is planned for internal control-plane data only (small
write volume, need strong consistency), not for general-purpose databases.

---

## Why it works

The deeper principle is that **QuePaxa separates two concerns that standard
Paxos fuses together**:

1. *Who is allowed to propose in this slot?* (Classic Paxos: exactly one
   proposer with the highest ballot/term. Everyone else must wait or lose.)
2. *Which value wins if two proposals conflict?* (Paxos: whichever proposal
   ran Phase 1 first.)

By using random priorities rather than ballot numbers to answer question 2,
QuePaxa can allow any number of proposers to run simultaneously and still
converge — because "which one wins" is settled by the randomly-drawn
priority, not by ordering concurrent Phase 1s through a serialization point.

This is the same move randomized algorithms always make to escape impossibility
results: FLP impossibility applies to *deterministic* protocols.
Ben-Or's 1983 randomized Byzantine agreement paper was the first to show that
randomization escapes FLP. QuePaxa applies the same insight to practical,
high-performance state machine replication.

The mental model: **QuePaxa is to Raft what OCC is to strict two-phase locking**.
In strict 2PL, you must acquire a lock (elect a leader) before writing; anyone
who doesn't hold the lock must wait. In OCC, you optimistically write (propose)
and handle conflicts at commit time (Phase 3 priority resolution). The
"commit abort" rate is low in both cases when contention is low — which is why
both OCC and QuePaxa's fast path work well in practice.

Concretely, the invariant that makes this safe is the same quorum intersection
property Paxos has always relied on: any majority accepting a value at priority
`p` guarantees that any later Phase 1 reading a majority will see at least one
record of that acceptance. The ISR just extends this to allow *multiple*
accepted values per slot, sorting them by priority at read time.

---

## Going deeper

1. **QuePaxa paper (SOSP 2023):** Tennage, Basescu, Kokoris-Kogias, Syta,
   Jovanovic, Estrada-Galinanes, Ford. The original paper with full proofs,
   ISR formalization, and performance evaluation:
   https://doi.org/10.1145/3600006.3613150

2. **Cloudflare's "Introducing Meerkat"** — the companion post describing the
   deployment context, why Cloudflare needed stronger availability than Raft
   provides at global WAN scale, and what they plan to build on top:
   https://blog.cloudflare.com/meerkat-introduction/

3. **dedis/quepaxa** — the original EPFL reference implementation in Go,
   with the Proposer, Recorder, and ISR code that maps directly to the paper's
   pseudocode: https://github.com/dedis/quepaxa

4. **Ben-Or 1983, "Another Advantage of Free Choice"** — the paper that first
   showed randomization escapes FLP, and whose core technique (randomized
   binary agreement as a sub-protocol) is QuePaxa's theoretical ancestor.
