---
title: "Flexible Paxos: Quorum Intersection Revisited"
source: https://arxiv.org/abs/1608.06696
author: Heidi Howard, Dahlia Malkhi, Alexander Spiegelman
company: University of Cambridge / VMware Research
date_posted: 2016-08-22
date_digested: 2026-07-22
---

# Flexible Paxos: Quorum Intersection Revisited

## What's new to learn

1. **Asymmetric quorums are safe in Paxos**: Safety only requires that each Phase 1 (leader-election) quorum intersects each Phase 2 (replication) quorum. Phase 2 quorums from different rounds do *not* need to intersect each other — the standard "majority everywhere" rule is unnecessary.

2. **The only constraint is Q₁ + Q₂ > N**: With N nodes, any Phase 1 quorum size Q₁ and Phase 2 quorum size Q₂ satisfying Q₁ + Q₂ > N yields a correct consensus protocol. This is a strict generalization of majority quorums (where Q₁ = Q₂ = ⌈(N+1)/2⌉).

3. **Safety invariants are causal, not structural**: Paxos's actual safety dependency is a single directed edge in the happens-before order (Phase 1 of round r' must observe Phase 2 of round r), not a clique. Flexible Paxos is the insight that enforcing a clique was never necessary.

## Prerequisites

- **Basic Paxos protocol**: Two phases — Phase 1 (Prepare/Promise) where a leader establishes dominance over a ballot number, and Phase 2 (Accept/Accepted) where the leader replicates a value.
- **Quorum intersection**: Why two majority sets over N items always share at least one member (pigeonhole principle).
- **Single-decree Paxos safety**: Why a value chosen in Phase 2 of round r must be discovered in Phase 1 of any higher round r' — the invariant that prevents two conflicting values from being decided.
- **Multi-Paxos / log replication**: Phase 1 runs once per leader term; Phase 2 runs once per log entry.

## The core idea

Classic Paxos mandates that both phases use majority quorums. For N=5, that means 3 nodes must acknowledge each Prepare message *and* each Accept message. This is intuitive: if two majorities always share a node, then Phase 1 of a new leader will always encounter at least one node that participated in the last Phase 2, learning whatever value was accepted there.

The Flexible Paxos observation is that the two majorities do not need to be the same size. The only thing that matters is that the Phase 1 quorum of any future round can discover what the Phase 2 quorum of any past round accepted. For discovery, you just need the two sets to share at least one member — which happens whenever Q₁ + Q₂ > N.

Concretely, for N=5 you could run Phase 1 with any 4 nodes (Q₁=4) and Phase 2 with any 2 nodes (Q₂=2), because 4+2=6 > 5. Leader elections become harder (need 4/5 nodes reachable), but each log entry only needs 2 acknowledgments instead of 3. Since leader elections are rare and log replication happens on every write, you have moved cost from the common case to the uncommon case.

## Mechanics

**Why Phase 2 quorums don't need to intersect each other**

Suppose leader L₁ ran Phase 2 in round r and got value v accepted by quorum Q₂ = {A, B}. A new leader L₂ runs Phase 1 in round r' > r with quorum Q₁ = {A, C, D, E} (4/5 nodes). Since Q₁ ∩ Q₂ = {A} ≠ ∅, L₂ will hear from node A, which reports "I accepted v in round r." L₂ must then propose v in its own Phase 2 (or something at least as recent). Safety is preserved.

Now suppose L₂'s Phase 2 quorum is Q₂' = {C, D} — which does *not* intersect Q₂ = {A, B}. Does this break safety? No: before L₂ got to Phase 2, L₂'s Phase 1 quorum Q₁ already intersected Q₂ (via A) and learned about v. L₂ could only propose v (or a newer value from a ballot between r and r'). Phase 2 of different rounds can use non-overlapping quorums precisely because cross-round consistency is guaranteed by the Phase 1–Phase 2 intersection, not by Phase 2–Phase 2 intersection.

**The safety proof in one sentence**

The safety invariant is: if v is decided (accepted by Phase 2 quorum Q₂ in round r), then for all higher rounds r' with Phase 1 quorum Q₁, Q₁ ∩ Q₂ ≠ ∅, so Q₁ includes a node that knows v was accepted in round r.

This is satisfied iff Q₁ + Q₂ > N (guaranteed intersection by pigeonhole), which is the *only* quorum constraint Flexible Paxos requires.

**Configuration table for N=5**

| Q₁ (Phase 1) | Q₂ (Phase 2) | Q₁+Q₂ | Leader election | Write quorum |
|---|---|---|---|---|
| 3 | 3 | 6 | 2 failures tolerated | 2 failures tolerated |
| 4 | 2 | 6 | 1 failure tolerated | 3 failures tolerated |
| 2 | 4 | 6 | 3 failures tolerated | 1 failure tolerated |
| 5 | 1 | 6 | 0 failures tolerated | 4 failures tolerated |

The bottom row (Q₁=5, Q₂=1) is pathological but valid: every node must participate in elections, but a single acknowledgment suffices for any write. The top-right corner is the write-optimized sweet spot (Q₁=4, Q₂=2) that Flexible Paxos highlights as the main practical improvement.

**Algorithm change required**: Almost none. Flexible Paxos is a parameterization of Paxos, not a restructuring. The Prepare, Promise, Accept, and Accepted messages are unchanged. The only difference is what counts as "a quorum": a fixed threshold Q₁ for Phase 1 messages and Q₂ for Phase 2, with Q₁ + Q₂ > N. An existing Paxos implementation needs a two-line change to its quorum-check functions.

**Multi-Paxos amplification**

In multi-Paxos (the actual mechanism behind Raft, Zookeeper, etcd), Phase 1 runs once per leader epoch and Phase 2 runs for every committed log entry. Switching from Q₁=Q₂=3 (N=5) to Q₁=4, Q₂=2 saves one network round-trip acknowledgment on *every* write, forever, across all entries in the log, while adding one acknowledgment to leader elections, which happen at most every few minutes (or never, in a stable cluster). The savings dwarf the cost.

**Geographical configurations**

Consider a 5-node cluster with 3 nodes in region A and 2 in region B. A traditional 3/5 majority quorum can form within region A without talking to region B — writes are fast. But Flexible Paxos with Q₁=5, Q₂=1 would put the election cost on a cross-region round-trip (rare) and the write cost on one local acknowledgment (extreme). More practically, Q₁=4, Q₂=2 lets writes complete with 2 local nodes (within one region) while ensuring elections require at least one cross-region node.

## Where it breaks

1. **Election availability degrades linearly with Q₁**: With Q₁=4/5, losing *two* nodes makes leader election impossible even though three nodes are running. Traditional 3/5 quorums elect a leader after any two failures. This is a real operational cost — if you want zero-downtime rolling upgrades, you cannot take two nodes down simultaneously with Q₁=4.

2. **The "right" Q₁ and Q₂ are workload-dependent**: A cluster with frequent leader churn (aggressive failure detectors, flapping nodes) should use a smaller Q₁. A cluster with bursty writes should optimize Q₂. These are not observable without measuring, and choosing wrong wastes the flexibility.

3. **Static quorum sizes**: Flexible Paxos as described picks Q₁ and Q₂ at cluster-configuration time. Dynamic reconfiguration (adding/removing nodes) requires re-deriving Q₁ and Q₂ for the new N and carefully transitioning. This is no harder than normal Paxos reconfiguration, but it's easy to accidentally violate Q₁ + Q₂ > N during a resize.

4. **Doesn't reduce message counts**: Both phases still broadcast to all N nodes. The quorum only controls when the leader considers the phase complete. For latency under partial loss, reducing the threshold helps; for total bandwidth, it's identical to standard Paxos.

5. **Doesn't generalize to Byzantine failures without care**: In Byzantine fault-tolerant (BFT) consensus, you need Q₁ + Q₂ > N + f (where f is the Byzantine threshold) because adversarial nodes can lie. The Flexible Paxos insight extends but the minimum viable quorum sizes are larger.

## Why it works

**Safety constraints are edges in the happens-before DAG, not cliques.**

Traditional Paxos guarantees safety by making every quorum intersection with every other quorum — a complete graph of "these sets overlap." But the safety proof only ever uses one kind of intersection: Phase 1 of round r' must overlap with Phase 2 of rounds 1, 2, …, r'-1. That's a directed bipartite relationship (Phase-1 nodes → Phase-2 nodes), not an all-pairs mesh.

This is the same structural insight that shows up across distributed systems:

- **MVCC snapshot isolation**: A snapshot at time T only needs to be consistent with all transactions that *committed before T* — not with concurrent snapshots. Safety is a one-directional temporal relationship, not pairwise consistency across all snapshots.
- **Chandy-Lamport snapshots** (covered in this archive): A marker injected into a channel creates a causal cut in the happens-before order. The cut only needs to be consistent with the message ordering in one direction — not with every other concurrent cut.
- **Raft's election check**: A candidate must be "at least as up-to-date" as a voter's log. This is a Phase-1-reads-from-Phase-2 dependency. Raft enforces it, but because Raft inherits majority quorums for both phases, it misses the asymmetry Flexible Paxos identifies.

The general principle: **find the actual causal dependency and enforce only that, not a conservative over-approximation.** Paxos was originally proven correct with the strong "all quorums intersect" property because the proof is simpler to write. Flexible Paxos peeled back to the minimal structural requirement.

This is the same intellectual move as: "locks protect against concurrent access to shared state — but do you need a lock, or do you need the reads to see the writes?" (the motivation for read-copy-update and seqlocks). The minimal synchronization requirement is often much weaker than the obvious one.

## Going deeper

1. **Heidi Howard's PhD thesis, "Distributed Consensus Revised" (2019)** — Comprehensive treatment of flexible quorums, extending the result to crash-and-recovery models, Byzantine failures, and multi-leader protocols. The canonical deep treatment.
   URL: https://www.cl.cam.ac.uk/techreports/UCAM-CL-TR-935.pdf

2. **EPaxos: There Is More Consensus in Egalitarian Parliaments (Moraru et al., SOSP 2013)** — Multi-leader Paxos where *non-interfering* commands (e.g., writes to different keys) can commit with a single Phase 2 round using fast quorums of size ⌈(3N/4)⌉. A related but distinct application of asymmetric quorum logic.
   URL: https://www.cs.cmu.edu/~imoraru/epaxos/tr.pdf

3. **Fast Flexible Paxos (Malkhi and Spiegelman, ICDCN 2021)** — Extends Flexible Paxos to the "Fast Paxos" setting where clients write directly to acceptors (skipping the leader in Phase 2). Proves the optimal fast quorum constraint is Q₁ + 2·Q₂_fast > 2N.
   URL: https://dl.acm.org/doi/10.1145/3427796.3427815
