---
title: "Keeping CALM: When Distributed Consistency Is Easy"
source: https://cacm.acm.org/research/keeping-calm/
author: Joseph M. Hellerstein, Peter Alvaro
company: UC Berkeley
date_posted: 2020-01-27
date_digested: 2026-07-28
---

# Keeping CALM: When Distributed Consistency Is Easy

## What's new to learn

- **CALM Theorem**: A distributed computation has a consistent, coordination-free implementation if and only if it is *monotone* — the single criterion that cleanly separates which problems need locks/consensus from those that don't.
- **Open-world vs. closed-world reasoning in programs**: Non-monotone operations implicitly assume a closed world (all input facts have arrived), which is exactly what requires coordination; monotone operations work safely under an open-world assumption and can emit output the moment any input arrives.
- **CRDTs as monotone programs**: The join-semilattice structure that all CRDTs share is not arbitrary — it is the algebraic form of monotone computation, unifying the entire CRDT literature under a single logic-theoretic principle.

## Prerequisites

- What coordination means in distributed systems: quorum reads/writes, 2-phase commit, consensus rounds, barriers
- Eventual consistency vs. strong consistency at an intuitive level
- CRDTs are helpful context but not required — a G-Set (grow-only set) and G-Counter are the only examples needed
- One line of Datalog will appear; no prior Datalog knowledge is assumed

## The core idea

Imagine you're building a shopping cart replicated across three nodes. Add-item can be applied locally and synced lazily: if two replicas both add the same item, union is idempotent. No problem. But remove-item is different. If replica A deletes an item and replica B never saw that deletion, a lazy sync might resurrect the deleted item. To prevent this, you need coordination: wait for all replicas to confirm a deletion before it's final.

Why is add safe but delete not? The CALM theorem gives the precise answer.

**Monotonicity**: a computation is monotone if, as the set of known input facts grows (by set inclusion), the set of derived output facts can only grow — never shrink. If you can derive X today from some input I, you can still derive X tomorrow from any I' ⊇ I.

**The CALM Theorem** (Hellerstein & Alvaro, 2020):

> A problem has a consistent, coordination-free distributed implementation if and only if it is expressible in monotone logic.

- "Consistent" means *confluent*: every possible ordering and batching of input messages produces the same final output.
- "Coordination-free" means no component needs to wait for a global barrier before emitting results.

The two directions:

1. **Monotone → coordination-free**: Monotone computations never need to retract output. You can safely produce partial results the moment you have any input. Late-arriving facts add more output, never contradict existing output. So you never need to wait.

2. **Non-monotone → coordination unavoidable**: For any non-monotone computation, there exists an input ordering that causes a naive distributed implementation to produce conflicting outputs. Some barrier is unavoidable.

A quick catalog of where this lands:

| Operation | Monotone? | Coordination needed? |
|-----------|-----------|----------------------|
| Set union (add to cart) | Yes | No |
| Set difference (remove from cart) | **No** | **Yes** |
| MAX / MIN | Yes | No |
| COUNT | **No** | **Yes** |
| EXISTS (is there any X?) | Yes | No |
| NOT EXISTS / HAVING COUNT = 0 | **No** | **Yes** |
| Bloom filter insert | Yes | No |
| Bloom filter with deletion | **No** | **Yes** |
| G-Counter CRDT | Yes | No |
| Bank balance (with overdraft prevention) | **No** | **Yes** |
| Graph reachability | Yes | No |
| Exact shortest path count | **No** | **Yes** |

The insight isn't that these operations are hard. It's that the need for coordination is *structural*, not incidental to implementation choices. You cannot engineer your way around the non-monotone ones without either changing semantics or accepting a coordination cost.

## Mechanics

The formal model uses **Datalog** as a simple notation for distributed programs. In Datalog, a program is a set of rules that derive new facts from existing ones:

```
Reachable(a, b) :- Edge(a, b).
Reachable(a, b) :- Edge(a, c), Reachable(c, b).
```

This program derives the transitive closure of a graph. It is monotone: adding edges can only make more pairs reachable, never revoke reachability.

**Monotone Datalog** = Datalog without negation and without aggregation that can decrease output. All standard relational operations (projection, join, union) are monotone. Negation and counting are not.

Consider the difference:

```
-- Monotone: existential
HasMessage(user) :- Message(user, _, _).

-- Non-monotone: COUNT
MsgCount(user, count) :- count = COUNT(*), Message(user, _, _).

-- Non-monotone: negation
Unread(msg) :- Message(msg), NOT ReadReceipt(msg).
```

**Distributing a monotone Datalog program**: Each node applies rules to its local facts, ships derived facts to neighbors, and repeats. Because monotone rules can only produce new facts (never retract), partial results are always safe. A node that only knows half the input emits a subset of the correct output — never an incorrect output. No synchronization is required.

**Why non-monotone programs break without coordination**: Suppose node A derives `Count = 3` from 3 local messages and emits that count. Later it receives 2 more messages from node B and must revise to `Count = 5`. It has emitted a value it must retract. Two orderings of the same messages produce different output histories — non-confluence. The only fix: don't emit `Count = 3` until you are certain you've seen everything. That certainty requires a barrier — and a barrier requires coordination.

**Finding coordination points in practice**: The Bloom language (a Ruby DSL built on CALM) annotates each rule as monotone or non-monotone. The compiler then flags any non-monotone rule that isn't guarded by an explicit coordination operator. This gives you a list of exactly the places where your program needs quorums, seals, or barriers — and a proof that every other part of your program can run freely.

**The shopping cart reformulation**: 

- *Add-only cart*: `Cart(item) :- AddEvent(item)` — monotone, CRDT-compatible, zero coordination.
- *Cart with removes*: `Cart(item) :- AddEvent(item), NOT RemoveEvent(item)` — non-monotone. Must coordinate on deletions OR choose a semantics that re-encodes deletion monotonically (add-wins: a later add overrides a concurrent remove, implemented via a vector-clock tombstone that delays the deletion decision to read time).

The add-wins CRDT doesn't eliminate the non-monotone computation; it pushes the coordination from write time to read time. The CALM theorem says the coordination cost has to go somewhere — the design choice is only where.

## Where it breaks

**Confluence ≠ semantic correctness**: A G-Set cart is CALM-confluent under add-wins, but a user who adds and then removes an item in the same session may find the item still in the cart. The program satisfies CALM's consistency definition; it violates the user's expectation. CALM tells you when your distributed computation is self-consistent, not when it's correct by external criteria.

**Open systems and the "all facts arrived" problem**: CALM assumes a program's input set is bounded and eventually complete. In real systems, inputs are open-ended (sensor streams, user activity). Knowing "all relevant facts are in" requires a mechanism like watermarks or epoch markers — and establishing those watermarks is itself a coordination problem. CALM characterizes the cost; it doesn't eliminate it.

**Restructuring isn't always possible**: Sometimes a non-monotone query can be reformulated into a monotone one. `COUNT(x) ≥ k` (threshold) is monotone; `COUNT(x) = k` is not. But there's no general rewrite algorithm. Expert knowledge of CALM is required to find monotone reformulations, and for some computations none exist.

**Datalog as a simplified model**: Datalog has no mutable state, no I/O side effects, no failures, no timeouts, and no transactions. Real distributed programs have all of these. The theorem applies to the logical semantics of what a program computes, not to the implementation substrate. Mapping a real system to the Datalog model requires care and sometimes awkward encodings.

## Why it works

The deep principle is the **open-world vs. closed-world distinction** from knowledge representation and logic.

Under the **closed-world assumption (CWA)**: "X is not in the database" means "X is definitively false." This is the assumption SQL uses — `NOT EXISTS (SELECT ...)` asserts not just that X hasn't been seen but that X doesn't exist. Establishing that assertion in a distributed system requires knowing that every node has reported in and none of them have X. That's coordination.

Under the **open-world assumption (OWA)**: "X is not yet in the database" means only that it hasn't been observed yet. You cannot conclude X is false. Monotone programs operate entirely under OWA: they assert only what's provably derivable; they never assert the absence of facts.

**Every coordination mechanism in distributed systems is implementing the CWA for some non-monotone operation.** This unifies a wide range of prior results:

*Connection to CRDTs* (in archive, May 20): Every CRDT is a join-semilattice: merge(a, b) ≥ a, b under the lattice order. This is exactly the algebraic statement of monotonicity — the merged state is always an upper bound of either input. CALM explains *why* CRDTs don't need coordination: they are monotone programs. The "hard parts" of CRDTs (tree moves, intent preservation, interleaving prevention) are precisely the non-monotone operations that CRDTs approximate with opinionated heuristics (add-wins, last-write-wins, tombstones).

*Connection to watermarks and the Dataflow model* (in archive, June 20): Watermarks are the mechanism for establishing the CWA for a finite time window. "All events before time T have arrived" is the closed-world precondition for computing COUNT over the window. The coordination cost of watermarks is exactly the CALM cost of the non-monotone aggregation they guard.

*Connection to FLP impossibility* (in archive, July 13): Consensus is non-monotone. "Has a majority agreed on value V?" requires knowing that the agreement state won't be retracted if more votes arrive later. FLP shows this is unsolvable in purely async systems — CALM shows why: consensus is a closed-world computation (does a majority exist *among all possible voters?*), and no coordination-free protocol can implement CWA in an asynchronous system.

*Connection to Chandy-Lamport snapshots* (in archive, July 10): A consistent global snapshot is precisely a mechanism for establishing a coherent CWA point — a moment where all nodes agree on what "now" means. The barrier token that propagates through channels establishes the cut; the cut establishes the closed world for downstream non-monotone analysis.

The meta-insight: **if you see a lock, quorum, barrier, watermark, 2PC protocol, or seqno in a distributed system, you are looking at a closed-world enforcement mechanism for some non-monotone sub-computation**. CALM tells you which computation it is. And CALM's contrapositive says: if you want to eliminate coordination, the only path is to redesign the computation to be monotone — either by changing semantics (add-wins instead of remove-wins), by restricting to monotone queries (threshold instead of exact count), or by accepting a coordination point and placing it where it's least expensive.

## Going deeper

1. **"Consistency Analysis in Bloom: a CALM and Collected Approach"** (Alvaro et al., CIDR 2011) — the Bloom language paper that operationalizes CALM. Annotates distributed Ruby programs, measures what fraction of real Cassandra-backed Twitter code is CALM-compliant, and shows that the coordination points the compiler flags match the places engineers added explicit barriers by intuition.

2. **"Keeping CALM: When Distributed Consistency Is Easy"** full paper (Hellerstein & Alvaro, CACM 2020, preprint: arxiv.org/abs/1901.01930) — the definitive formal treatment with full proofs of both directions, a systematic catalog of monotone vs. non-monotone SQL operators, and extensions to programs with mutable state and non-Datalog constructs.

3. **"CRDTs: The Hard Parts"** (Martin Kleppmann, Hydra 2020) — already in this archive (May 20) — re-read with CALM in mind: every counterintuitive CRDT behavior (the cycle in concurrent tree moves, the interleaving in collaborative text) corresponds exactly to a non-monotone operation the CRDT is approximating. The heuristic chosen (add-wins, last-write-wins) is a policy for handling the CWA at read time instead of write time.
