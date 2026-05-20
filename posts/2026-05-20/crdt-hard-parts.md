---
title: "CRDTs: The Hard Parts"
source: https://martin.kleppmann.com/2020/07/06/crdt-hard-parts-hydra.html
author: Martin Kleppmann
company: University of Cambridge (independent researcher)
date_posted: 2020-07-06
date_digested: 2026-05-20
---

# CRDTs: The Hard Parts

## What's new to learn

- **Convergence ≠ semantic correctness.** A CRDT guarantees that all replicas reach the *same* state — but that state can be semantically wrong (characters interleaved, cycles in a tree, violated invariants). The convergence proof and the correctness proof are two separate things.
- **The move-to problem requires undo/redo.** A move operation on a replicated tree (e.g., "drag folder A into folder B") cannot be made conflict-free using pure commutativity; concurrent moves can create cycles. The only known general solution requires an undo-log and replay — which is structurally closer to Operational Transformation than to a classical CRDT.
- **Conflict-free ≠ interleaving-free.** Naive position-based text CRDTs can interleave characters from concurrent edits character-by-character (HWeolrlldo instead of HelloWorld), even though all replicas converge to the *same* interleaved garbage. Correct algorithms must encode ordering intent into identifiers.

## Prerequisites

- What a CRDT is at the introductory level: grow-only sets (G-Set), counters (G-Counter/PN-Counter), OR-Set for add-and-remove.
- Why commutativity solves distributed convergence: if `f(g(x)) = g(f(x))` for all operations f, g, you can apply them in any order and always reach the same state.
- What Operational Transformation (OT) is at a high level: operations are transformed against each other before application so that intent is preserved, but this requires a central server or complex diamond transform protocols.
- Basic familiarity with logical clocks / Lamport timestamps.

## The core idea

The standard CRDT pitch goes: "operations commute, so you can apply them in any order, and replicas automatically converge — no conflicts, no coordination needed." This is mathematically true and genuinely powerful. But it proves only *one* thing: all replicas reach *the same state*. It says nothing about whether that state is what users intended.

Kleppmann's talk systematically identifies the cases where the gap between convergence and correctness is large enough to break real applications:

1. **Text editing:** Two users type at the same position. Replicas converge — to an interleaved mess.
2. **Tree moves:** Two users move folders concurrently. Replicas converge — to a cycle if not handled carefully.
3. **Application invariants:** Two users concurrently perform actions that are individually valid but jointly violate a business rule (e.g., both claim the same unique username).

In each case, the answer is *not* "CRDTs are broken." The answer is that CRDT convergence is a weaker guarantee than most engineers intuitively assume, and each hard case requires additional design work — sometimes work that looks nothing like a traditional CRDT.

## Mechanics

### Hard Part 1: Text interleaving

Consider a shared document where Alice types "Hello" and Bob types "World" at the same empty position, concurrently. After both edits merge, what should appear?

The *intended* result is either "HelloWorld" or "WorldHello" — contiguous blocks. What many naive CRDTs produce is "HWeolrlldo" — characters interleaved one-by-one.

Why? Position-based CRDTs (Logoot, LSEQ) assign each character a fractional position identifier between its neighbors. When two characters are inserted at the same position, their identifiers are compared to determine order. If Alice's 'H' and Bob's 'W' get assigned identifiers in an alternating pattern, they interleave.

Correct algorithms (RGA — Replicated Growable Array; Yjs; Automerge's peritext) solve this by making each character's identifier encode its *causal predecessor* rather than just a position. An insertion says "I come immediately after character with ID X," and concurrent insertions at the same predecessor are broken by replica ID or timestamp — but crucially, all of Alice's characters share the same predecessor chain, so they stay contiguous.

The fix requires careful identifier design, not just commutativity. The convergence proof doesn't change; what changes is that the converged state is now semantically meaningful.

### Hard Part 2: Move in a replicated tree

This is the deepest problem in the talk. Consider a file system tree:

```
root/
├── A/
│   └── (empty)
└── B/
    └── (empty)
```

Alice does: **move A under B** (making B the parent of A).
Bob simultaneously does: **move B under A** (making A the parent of B).

Each operation is locally valid. But if you apply both naively:
- Apply Alice's op → B/A (B contains A)
- Apply Bob's op → A/B/A or a cycle

The problem: there is no commutative function that makes "move A under B" and "move B under A" produce the same correct result regardless of application order. Cycles violate the fundamental tree invariant.

**The algorithm (Kleppmann et al., 2021):**

Every move operation is stamped with a logical timestamp. The merge rule is:

1. Maintain an **undo-log**: a per-replica record of the previous parent of every node before each move was applied.
2. When a remote move arrives with timestamp `t`, **undo** all local moves with timestamp `> t` (replaying them backwards via the undo-log).
3. **Apply** the remote move (now guaranteed to be in causal order).
4. **Redo** the undone local moves, skipping any that would now create a cycle (they are silently dropped).

The cycle check is cheap: before applying any move of node N to parent P, walk up the ancestor chain of P; if you encounter N, the move would create a cycle and is rejected.

This is decidedly *not* a classical CRDT: it requires seeing other replicas' operations before finalizing your own, and it maintains mutable history. Structurally, it resembles OT's undo/redo protocol. The CRDT property is preserved (all replicas applying the same set of operations converge to the same tree), but the *mechanism* crosses into OT territory.

Reference implementation: [github.com/trvedata/move-op](https://github.com/trvedata/move-op)

### Hard Part 3: Application-level invariants

Some invariants cannot be maintained without coordination, period. Examples:

- **Unique usernames:** Alice and Bob both claim `@alice` concurrently. Last-write-wins gives the name to one, but both believed they owned it during the window.
- **Bounded counters:** A shared item quantity cannot go below 0. PN-Counter additions and subtractions commute, but you can't enforce `value ≥ 0` without knowing what other replicas are simultaneously doing.

For bounded invariants, the only principled options are:
- Accept violations and reconcile (showing the user an error after the fact).
- Partition the invariant budget across replicas (each replica gets an allocation it can safely consume without coordination).
- Accept coordination when you truly need a global invariant.

## Where it breaks

- **Performance cost of the move algorithm.** Undoing and redoing operations is O(depth of undo-log × cost per operation). For deeply nested undo histories (long offline editing sessions), this can be expensive.
- **Undo-log garbage collection.** The log can grow unboundedly. Trimming it requires knowing that all replicas have seen all operations up to some point — which requires coordination (e.g., a distributed GC protocol).
- **The algorithm handles trees, not DAGs.** If your structure is a DAG (a node can have multiple parents), the cycle-detection invariant is ill-defined. New algorithms are needed.
- **Application invariants require out-of-band design.** CRDTs don't help you with uniqueness or bounded quantities. You need either coordination, pre-allocation, or accepting temporary violations.
- **Interleaving-free text editing is still an open research area** for rich text with overlapping formatting marks (bold, italic, comments) — Automerge's Peritext paper (2022) addresses a subset of this.

## Why it works

The deeper principle is a clean separation of two levels of correctness:

> **Level 1 (Data-structure level):** Do all replicas converge to the same bit-pattern? This is what CRDT convergence proofs guarantee.
>
> **Level 2 (Semantic level):** Does the converged state reflect user intent? This is what CRDTs say nothing about.

Most distributed consistency mechanisms — serializability, linearizability — protect both levels simultaneously by preventing concurrent access. CRDTs give up coordination to gain availability, but in doing so, they only protect Level 1. The burden of Level 2 correctness moves to the algorithm designer.

This maps onto a well-known CS principle: **specification completeness**. A proof that an implementation matches a spec is only as good as the spec. CRDT convergence proofs specify "all replicas reach the same state"; they don't specify "the state users wanted." The hard parts are all about closing this specification gap.

The move-to algorithm's need for undo/redo also reveals something important: **commutativity is not closed under all operations.** G-Set additions commute. PN-Counter increments commute. But tree-restructuring operations do not commute in general, because their preconditions reference shared mutable state (the tree structure). Any operation whose precondition can be invalidated by a concurrent operation needs either coordination or a richer merge rule — exactly what the undo-log provides.

This generalizes: **whenever your operation's *legality* depends on state that another replica can change concurrently, pure commutativity cannot make it conflict-free.** You need either coordination at that invariant boundary, or an explicit merge rule that defines which concurrent operations "win" and how the losers are replayed.

Seen from this angle, the CRDT design space is: pick a set of operations whose *result* commutes even when their *preconditions* might conflict. For most structural operations on graphs or trees, the only way to achieve this is to broaden "result" to include an undo-log entry — making the operation itself richer, not simpler.

## Going deeper

1. **"A Highly-Available Move Operation for Replicated Trees and Distributed Filesystems"** (Kleppmann et al., 2021) — the peer-reviewed paper formalizing the undo/redo algorithm with full correctness proofs. [PDF](https://martin.kleppmann.com/papers/move-op.pdf)
2. **"Peritext: A CRDT for Collaborative Rich Text Editing"** (Litt et al., 2022) — extends conflict-free text editing to overlapping character-level formatting marks; the same distinction between convergence and intent applies. [ACM Digital Library](https://dl.acm.org/doi/10.1145/3512897)
3. **Automerge source code** (specifically `automerge/src/columnar_doc/encoding.rs` and the tree-move logic) — a production CRDT library that implements these algorithms; reading the implementation is the fastest way to internalize how the undo-log is structured and garbage-collected. [github.com/automerge/automerge](https://github.com/automerge/automerge)
