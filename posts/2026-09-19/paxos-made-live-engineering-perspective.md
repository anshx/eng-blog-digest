---
title: "Paxos Made Live: An Engineering Perspective"
source: https://research.google.com/archive/paxos_made_live.pdf
author: Tushar Chandra, Robert Griesemer, Joshua Redstone
company: Google
date_posted: 2007-06-01
date_digested: 2026-09-19
---

# Paxos Made Live: An Engineering Perspective

## What's new to learn

- **Multi-Paxos optimization**: Electing a stable leader collapses 2-RTT-per-value down to 1-RTT-per-value by running Phase 1 once across all future log slots instead of per slot — the key performance trick absent from most Paxos descriptions.
- **Master leases for lock-free reads**: A time-bounded exclusivity lease lets the current master answer reads from local state, with no consensus round, as long as the lease is valid — trading lease-duration latency on failover for zero-cost reads in steady state.
- **Algorithm assumptions as engineering obligations**: Disk corruption, group membership changes, and unbounded log growth are each explicitly assumed away in the Paxos proof; every assumption the algorithm elides becomes its own engineering sub-problem when you ship.

## Prerequisites

- Basic Paxos: Phase 1 (Prepare/Promise) and Phase 2 (Accept/Accepted), quorum definitions, the invariant that two quorums always intersect.
- The distinction between safety (nothing bad ever happens) and liveness (something good eventually happens) in distributed protocols.
- Write-ahead logging and log compaction at a conceptual level.

## The core idea

Leslie Lamport's Paxos can be written in about a page of pseudocode. Google's implementation for the Chubby lock service — a production-grade, fault-tolerant, replicated key-value store — runs to several thousand lines of C++. This paper accounts for the gap.

The algorithm is correct under its model: reliable delivery (no drops), durable storage (no corruption), static membership (no adds or removes), and an oracle that says when it's safe to read. None of those hold in a real data center. Each assumption that breaks introduces a failure mode that the algorithm's safety proof cannot protect against, because the proof doesn't cover it. The paper's contribution is a systematic catalog of those failure modes and the specific engineering required to handle each one without undermining the proof.

The takeaway is architectural: an algorithm's model is a **specification of all the ways the real world can surprise you**.

## Mechanics

### Multi-Paxos

Basic Paxos runs two full round trips for every value: Phase 1 (Prepare → Promise from a quorum) then Phase 2 (Accept → Accepted from a quorum). That's 4 messages per committed value, with each pair requiring a full RTT.

Multi-Paxos amortizes Phase 1 across all future log slots. After winning an election, the leader sends a single Prepare that covers every future slot (using a wildcard slot identifier). Acceptors respond with a Promise that covers the entire future. The leader stores the highest accepted value in each slot (if any) and runs Phase 2 for each slot it needs to fill — just one RTT per value in steady state.

Phase 1 only re-runs when a new leader is elected. Between leader elections, the system runs at half the message count of basic Paxos.

**Ballot numbering**: Each Paxos instance tracks the highest ballot seen. A new leader must pick a ballot strictly higher than any it has seen, and must do so globally uniquely. Google's implementation encodes the ballot as (round_number, server_id), making ballots totally ordered without coordination.

### Master Leases

Multi-Paxos elects a master, but reads still require care: can the master serve a read from local state without a full consensus round? The master might not be master anymore — a higher ballot could have been granted elsewhere.

Master leases solve this. When the master wins Phase 1 (or renews its leadership), it also obtains a **time-bounded lease** from a quorum of acceptors: each acceptor promises not to grant a higher ballot for *T* seconds. During a valid lease window, the master is provably unique — no other replica can have won Phase 1 for a higher ballot — so it can answer reads from its local replicated log without any messages.

The clock assumption is critical: this works only if the master's clock and acceptors' clocks diverge by at most ε, where ε ≪ T. If a master thinks its lease still has 200 ms left but its clock runs fast by ε = 300 ms, the lease actually expired already. In practice, NTP-synchronized clocks with bounded drift make this safe. Google chose T = 5–10 seconds with ε well below 1 second.

Lease expiry under leader failure creates a forced wait: the new master cannot start until the old master's lease expires (or the old master explicitly releases it), which introduces up to T seconds of read unavailability on failover. This is a direct trade-off: longer leases mean fewer lease-refresh messages but longer failover delay.

### Disk Corruption

The Paxos safety proof relies on one invariant: an acceptor that sends a Promise for ballot *b* at slot *i* will never send Accept for ballot *b′ < b* at slot *i*. This is only true if the acceptor remembers its promises across restarts. Disks can silently corrupt that persistent state.

A corrupted acceptor that loses its phase-1 promise may re-grant Phase 1 for an old ballot, allowing two different leaders to successfully commit two different values at the same slot — a split-brain violation of the fundamental consensus safety property.

**Detection**: every piece of Paxos persistent state is checksummed. On restart, all state is checksum-verified. A failed checksum means the state must be treated as lost.

**Recovery**: a replica that has lost its state is demoted to a **non-voting observer** — it participates in the catch-up protocol (receives committed log entries from the current master) but does not respond to Phase 1 or Phase 2 messages until it has fully replicated all committed slots. This is safe: a non-voting replica can never grant a quorum to an old ballot because it doesn't vote at all.

**The marker file trick**: the replica must know *that* it lost state — otherwise it might erroneously believe its state is valid. They write a marker to GFS (Google's distributed file system) after successfully persisting each checkpoint. On restart, if the marker exists but the local state fails its checksum, the replica knows it lost committed data and must catch up before voting.

### Group Membership Changes

Paxos assumes the set of replicas (the group) is fixed. In practice, you add replicas for capacity, remove failed ones, and perform rolling upgrades. How do you change the group without a window where two quorums can be formed from incompatible configs?

The safe approach: **treat group config changes as values in the replicated log**. To add replica *R*, the master proposes a special config-change command at slot *N*. When that command is committed (learned by a quorum), all replicas apply it by updating their view of the quorum. The new config takes effect at log position *N* atomically — every replica applies it at the same point in the log.

The subtle requirement: configs must change one at a time. You cannot jump from {A,B,C} to {A,B,D,E} — you must go through {A,B,C,D} or some other intermediate that preserves quorum intersection with both the old and new config. Skipping creates a window where an old-config quorum and a new-config quorum do not intersect.

The paper acknowledges that formal verification of this protocol (that it is actually safe in combination with disk corruption and Multi-Paxos) was not fully resolved; they worked from informal arguments and extensive testing.

### Snapshots and Log Compaction

Without compaction, the replicated log grows unboundedly. Snapshots solve this: serialize the application state (the key-value store contents) at a specific log sequence number *S*, then discard all log entries before *S*.

The engineering challenge is consistency: the snapshot must reflect exactly the application state after applying log[0..S−1] and before applying log[S]. You cannot snapshot mid-command.

**Mechanism**:
1. The master picks a snapshot sequence number *S* and inserts a `Snapshot` marker at slot *S*.
2. Each replica, when its application layer processes the marker at position *S*, serializes its application state to disk and records the sequence number *S* in the snapshot header.
3. Log entries before *S* are eligible for deletion.

**Recovery**: load the most recent snapshot, note its sequence number *S*, then replay the log starting from *S*. Replicas that were down during the snapshot miss it; they receive the snapshot via GFS transfer from a peer, then resume log replay from *S*.

**The snapshot + group membership interaction**: if the group config changes between snapshots, the snapshot must record the config at time *S*, not just the application state. Otherwise a recovering replica would apply replayed log entries with the wrong quorum.

### Testing: SimAct

The implementation has thousands of lines of code and a large number of internal invariants. Testing the correctness of a consensus system under network partitions, disk corruption, and leader elections with only integration tests is impractical — the interesting failure modes require precise timing that is hard to reproduce.

Their testing framework, **SimAct**, replaces all I/O with simulated, deterministic equivalents:
- Network sends are captured; SimAct decides when (and whether) to deliver them.
- Disk operations are intercepted; SimAct can inject checksum failures.
- Clocks are virtualized; SimAct advances simulated time.

All non-determinism comes from a single seed value. A test that fails with seed *X* always fails with seed *X*, on any machine, at any speed. The bugs SimAct found before production:
- A race where a lease expiry and a new election fired simultaneously, allowing a brief overlap where two masters were active.
- A catch-up bug where a rejoining replica sent stale Phase-1 responses before fully loading its snapshot, transiently corrupting the master's slot map.
- A multi-step membership change where the transition config briefly had a quorum intersection smaller than majority-of-new-config.

This is architecturally identical to FoundationDB's simulation framework (which the FoundationDB team developed independently, ~7 years later): cooperative actors + deterministic I/O + seeded PRNG = reproducible distributed system bugs.

### MultiOp: Transactions on Top of Paxos

The replicated key-value store needs multi-key transactions. The authors built **MultiOp**, a compound command consisting of:
1. A list of **guards**: `(key, comparison, value)` read-then-check predicates.
2. A list of **operations**: `(key, write/delete, value)` mutations.

A MultiOp command is a single Paxos value. The application executes the guards atomically (no other command interleaved); if all pass, it applies the operations. If any guard fails, the entire MultiOp aborts. Since Paxos guarantees a single linearized order for all committed commands, MultiOp provides multi-key compare-and-swap with linearizability — the same primitive later formalized as "conditional writes" in systems like Percolator, FoundationDB, and etcd.

## Where it breaks

**Failover latency under leases**: the new master cannot start until the old master's lease expires. Lease duration T is a direct knob on read performance (larger = fewer renewals) vs. failover latency (larger = longer unavailability). There is no setting that optimizes both.

**Group membership is still under-specified**: the paper explicitly states their membership protocol is unproved correct. This ambiguity persisted in the Paxos literature for years; Raft's joint-consensus approach is partially a response to this. Formal verification of Paxos with dynamic membership remains hard.

**Snapshot complexity approaches algorithm complexity**: the snapshot subsystem (coordination, GFS transfer, application serialization, position tracking, config embedding) is several hundred lines of code — comparable to the core Paxos algorithm itself. Log compaction is not a simple add-on.

**Clock skew vulnerabilities in leases**: any environment with unreliable NTP — VMs that pause and wake, hypervisor-scheduled CPU time — can push clock skew past the safety margin. The algorithm silently produces incorrect behavior (two simultaneous masters) rather than crashing, making these bugs difficult to detect.

**No formal correctness proof for the full system**: the paper is an engineering report, not a proof. The combination of Multi-Paxos + leases + disk corruption + group membership + snapshots has never been formally verified end-to-end. TLA+ model checking (as AWS later used for DynamoDB; see 2026-09-02 post) was not applied here.

## Why it works

The deeper principle: **an algorithm's assumptions are an exact list of the ways the real world will differ from its model, and each difference is an engineering sub-problem of comparable complexity to the original algorithm.**

Paxos on paper assumes:
| Assumption | What breaks it | Fix |
|---|---|---|
| Durable storage | Disk corruption | Checksums + non-voting catch-up |
| Static membership | Operational changes | Config-as-log-value |
| Unbounded log | Finite disk | Snapshots + compaction |
| Only consistency, reads don't matter | Tail latency in production | Master leases |

The same pattern appears everywhere in systems:
- **Mutual exclusion algorithms** assume atomic shared-memory operations; real hardware requires memory barriers and cache flush instructions — each assumption about the memory model creates a platform-specific engineering burden.
- **Database ACID** assumes serializable isolation; production systems pay 10–100× cost to provide it, so they offer weaker levels (RC, SI) and document the anomalies — the isolation level taxonomy is a map of which assumptions they chose to relax.
- **Cryptographic protocols** (TLS) assume a trustworthy CA hierarchy; production deployments add certificate pinning, HPKP, and Certificate Transparency logs — each accounting for specific ways the CA hierarchy breaks.

The pattern is: **algorithm → idealized model with correctness proof; production system → same model + explicit handling of every assumption.**

One meta-lesson stands out: the paper is 14 pages of engineering challenges for a one-page algorithm. The ratio is roughly constant across CS: the theoretical contribution and the production engineering live at orthogonal levels of complexity, and you need both.

Connecting this archive to itself: FoundationDB's simulation (2026-05-19 post) independently arrived at the same deterministic-testing insight. Flexible Paxos (2026-07-22 post) later generalized the quorum intersection requirement. AWS's use of TLA+ (2026-09-02 post) is the formal answer to "how do you verify that your production implementation doesn't break the algorithm's invariants?" Raft (2026-06-12 post) was explicitly designed to be more understandable, partly because production Paxos implementations had been so hard to reason about correctly — this paper is one reason why.

## Going deeper

1. **"Paxos Made Simple" — Leslie Lamport (2001)**: the one-page algorithm that this paper's thousands of lines are built on. Read this first to understand exactly which algorithm all the engineering is implementing. Available at lamport.azurewebsites.net.
2. **"The Chubby Lock Service for Loosely-Coupled Distributed Systems" — Burrows (OSDI 2006)**: the actual production system built on this Paxos implementation. Describes the API, the caching model, and how client sessions interact with a replicated lock service.
3. **etcd's Raft internals and design doc**: the open-source project that most directly absorbed the lessons from this paper, replacing Paxos with Raft for understandability while keeping most of the production engineering described here (membership changes, snapshots, leases). The implementation is one of the most-read consensus codebases today.
