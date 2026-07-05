---
title: "Collaborative Text Editing with Eg-walker: Better, Faster, Smaller"
source: https://arxiv.org/abs/2409.14252
author: Joseph Gentle and Martin Kleppmann
company: Cambridge University / Independent
date_posted: 2024-09-22
date_digested: 2026-07-05
---

# Collaborative Text Editing with Eg-walker: Better, Faster, Smaller

## What's new to learn

1. **Event graph representation of edits**: Every edit in a collaborative document can be stored as an immutable, append-only DAG of original, untransformed operations — identical in structure to a git commit graph — rather than as pre-transformed operations (OT) or as per-character metadata (CRDT). This separates "what happened" from "how conflicts get resolved."

2. **Lazy LCA-based replay**: Concurrent edits are reconciled by replaying only the operations since the two branches' lowest common ancestor (LCA), using a transient CRDT built from scratch for that replay and then discarded. The CRDT is an implementation detail for merge time, not a persistent data structure.

3. **The CRDT metadata fallacy**: A traditional CRDT's memory footprint is dominated not by the current document but by tombstones and ordering metadata for characters that were deleted long ago. Eg-walker eliminates this by never persisting per-character metadata — the stored format is just the raw operation log, which shrinks proportionally to the current document, not to all characters ever typed.

## Prerequisites

- **CRDTs (basics)**: A conflict-free replicated data type converges to the same state on all replicas without coordination. Text CRDTs like RGA and FUGUE assign each inserted character a globally-unique ID and resolve concurrent insertions by ordering those IDs. Tombstones represent deleted characters; they stay in memory to allow later delete operations to be applied regardless of order.
- **Operational Transformation (OT)**: The approach used by Google Docs. When Alice and Bob each type simultaneously, each must *transform* the other's operation against their own before applying it. For N concurrent edits, this requires up to O(N²) transformations, and merging two branches that diverged for a long time is expensive.
- **DAG / topological sort**: A directed acyclic graph where edges encode "happened-before" ordering. Topological sort gives a valid linear ordering consistent with all edges.
- **LCA in a DAG**: The lowest common ancestor of two nodes is the most-recent ancestor that both have in common — the point where their histories diverge. Finding LCA in a DAG (not just a tree) requires a BFS/DFS scan but is generally fast if the divergence is recent.

## The core idea

Both OT and CRDTs conflate two orthogonal concerns:

1. **Storing the history of edits** — what did each user type, and when?
2. **Resolving ordering conflicts** — when two users insert at the same position simultaneously, whose character comes first?

OT resolves conflicts **eagerly** at apply time: every incoming operation is transformed against all concurrent operations before it can be used. For a pair of users, that's manageable. For long-running offline branches, the transformation cost blows up.

CRDTs resolve conflicts **statically** at write time: each character gets metadata (a unique ID, a logical timestamp) baked in at creation. This means all replicas can always merge without coordination. But that metadata must be kept forever — even after the character is deleted — because any future replica might still need to apply the delete operation, and the delete references the character by ID.

Eg-walker's insight: **resolve conflicts lazily, only when needed, and discard the resolution state afterward**.

Store history as an append-only event graph (cheap, compact). When two branches diverge and later reconnect, walk from their LCA, build a temporary CRDT in memory just long enough to compute the merged ordering, extract the result, and throw the CRDT away. The persistent format stays the raw operation log.

This is classic **event sourcing + lazy view materialization**: keep the authoritative log of events, derive the materialized view on demand, cache it if hot, don't store it permanently.

## Mechanics

### The event graph

Each operation is a node in a DAG:

```
{
  id:       (agent_id, seq_num),   // globally unique; no coordination needed
  parents:  [(agent_id, seq), ...], // IDs of ops this edit was causally based on
  type:     "insert" | "delete",
  content:  "A",                   // for insert: the character(s) inserted
  position: 42,                    // for insert: index in the *parent* document state
  target:   (agent_id, seq),       // for delete: which insert to undo
}
```

The DAG forms naturally from the causal structure of editing. If Alice makes edit A after seeing Bob's edit B, then A lists B as a parent. If they both edit from some shared state S simultaneously (without seeing each other's work), both list S as a parent and form a "fork" in the graph. When Alice later syncs with Bob, the two branch tips become parents of a notional merge — or, more precisely, both branch tips become part of the materialized document state once both are applied.

No operation is ever modified. Operations are write-once.

### Sequential case (the common case)

When the event graph is a simple chain (no concurrent forks), applying operations is trivial: replay them in topological order. Each insert's `position` field refers to the document state just after all its parents were applied. No transformation needed.

In practice, most editing happens sequentially (one person types, then another), so the vast majority of an editing session's event graph is a single chain. This is the case that benefits most: zero overhead versus a plain string.

### Concurrent case (the interesting case)

When branches fork and then need to be merged:

1. **Find the LCA** of the two branch tips in the event graph.
2. **Collect all operations** from both branches since the LCA — these are the "concurrent" operations.
3. **Build a transient CRDT** (e.g., RGA or FUGUE) from scratch, starting at the LCA document state.
4. **Apply all collected operations** from both branches in topological order. The CRDT's conflict-resolution rules (ordering by unique IDs) handle simultaneous insertions at the same position.
5. **Extract the merged document** from the CRDT.
6. **Discard the CRDT state** — only the event log (with both branches' ops now appended) is persisted.

Because every replica performs this same deterministic replay from the same LCA using the same ops in the same topological order, they all arrive at the same merged state — satisfying the CRDT convergence guarantee.

### File format and loading

The serialized format is just the operation log: a flat, topologically-sorted array of operations. Each op is ~30–50 bytes. No per-character metadata. Deleted characters appear once in the log as an insert op, and once as a delete op — no tombstone hanging around in a live data structure.

Loading a saved document means reading the log and applying ops sequentially. Because the stored log has no forks (previous merges have already been computed and linearized), loading is a single pass with no CRDT needed — O(n) in the number of operations.

Compare with loading an RGA or Automerge document: you must reconstruct the full CRDT state, including all tombstones for every character ever deleted, all unique IDs, all ordering metadata. A 50 KB text file might require 10–100 MB of CRDT memory.

### Performance numbers from benchmarks

The paper runs against seven real editing traces (from projects like the Linux kernel, Automerge itself, and a 260,000-word novel). Against RGA (Automerge):

- **Memory**: 10–100× less per document
- **Load time**: 10–100× faster
- **Apply remote ops**: comparable or better than CRDTs for typical workloads; orders of magnitude faster than OT for long offline branches

## Where it breaks

**Log growth is unbounded.** Every insert and delete ever made stays in the log forever. For a long-lived document with millions of edits, the log grows large even if the document itself is small. Compacting the log requires all peers to acknowledge a snapshot (analogous to database checkpointing), which requires coordination — the very thing CRDTs were designed to avoid.

**Expensive replay for pathological concurrency.** If ten users all work offline for a week and then sync simultaneously, Eg-walker must replay all ten branches from their LCA and run CRDT conflict resolution across all of them at once. The complexity is roughly O(k × d) where k is the total number of concurrent operations and d is the depth of the CRDT's internal work per insertion. In practice this is rare, but degenerate.

**Still requires a sequence CRDT underneath.** Eg-walker is a framework around a pluggable sequence CRDT (it uses RGA or FUGUE by default). The internal algorithm's quirks — for example, RGA's behavior when two users insert at exactly the same position — become Eg-walker's quirks too. Changing the underlying CRDT changes the resolved output.

**Semantic correctness is out of scope.** Like all CRDTs, Eg-walker guarantees that all replicas converge to the same string, but says nothing about whether that string is semantically meaningful. Two concurrent edits to nearby code can produce syntactically invalid code. This is the problem domain covered by "CRDTs: The Hard Parts" and remains unsolved.

**No built-in GC.** Long-running P2P systems need a garbage collection protocol to prune old history. This is a separate mechanism that must be built on top of Eg-walker.

## Why it works

The deep principle: **separate the log from the view, and materialize the view lazily**.

This is the same principle underlying:

- **Write-Ahead Logging (WAL)**: The append-only log is the authoritative record. The heap file is a cached materialization that can always be reconstructed by replaying the log. Crash recovery replays the log. Replicas stream the log and build their own heap files.
- **Git's object model**: Commits form a DAG; no commit is ever modified. The working tree is materialized on demand by walking from a commit to its trees and blobs. Merging two branches means finding their LCA and applying both sets of changes. Git does not store a permanent "merged CRDT state" — it just stores commits.
- **Event sourcing**: The canonical pattern in DDD. Store `OrderPlaced`, `ItemShipped`, etc. events. Derive the current order state by replaying events. Rebuild any read model from the log.
- **Kafka**: The append-only log is the truth. Every consumer builds its own materialized view at its own pace. Old data is not deleted until it ages out — the same unbounded-log problem Eg-walker has.

The specific efficiency gain comes from the observation that **the expensive part (CRDT conflict resolution) is only needed for the concurrent portion of the history** — and concurrent editing is rare. In a typical editing session, 99%+ of the event graph is a linear chain. Eg-walker pays O(1) per operation for that chain (just append to the log), and pays the CRDT cost only for the rare forks.

Traditional CRDTs pay the CRDT cost **on every operation**, even sequential ones, because they maintain CRDT state at all times in case a concurrent edit arrives. Eg-walker amortizes that cost: pay it when you merge, not when you write.

This is structurally identical to the **lazy vs. eager materialized view** trade-off in databases: eager materialized views are always up-to-date but pay update cost on every write; lazy views are cheaper to write but pay reconstruction cost at read time. Eg-walker chooses lazy for the CRDT state: free on sequential writes, paid at merge time.

The "oh, so X is just Y" moment: **Eg-walker is git's merge algorithm applied to fine-grained character-level editing**. Git stores commits as DAGs of snapshots and merges branches by finding their LCA and applying diffs. Eg-walker stores character-level ops as a DAG and merges branches by finding their LCA and applying those ops through a CRDT. The structural insight is identical.

## Going deeper

1. **Eg-walker paper + reference implementation**: [arxiv.org/abs/2409.14252](https://arxiv.org/abs/2409.14252) and the TypeScript reference at [github.com/josephg/eg-walker-reference](https://github.com/josephg/eg-walker-reference). The production Rust implementation is [diamond-types](https://github.com/josephg/diamond-types) — benchmarks in the repo reproduce all paper figures.

2. **FUGUE**: "FUGUE: Sorting Out Determinism in Collaborative Text Editing" (Weidner and Kleppmann, 2023). This is the sequence CRDT that Eg-walker uses internally for concurrent resolution. FUGUE fixes subtle interleaving bugs in RGA by using a tree-based tombstone structure — understanding it explains *why* the internal CRDT is needed at all.

3. **CRDTs: The Hard Parts** (Kleppmann, Hydra 2020 talk): Covers the semantic-correctness problems that Eg-walker explicitly does not solve — concurrent tree moves, list-insertion interleaving at higher semantic granularity. A good companion to this post showing where the next frontier of difficulty lies.
