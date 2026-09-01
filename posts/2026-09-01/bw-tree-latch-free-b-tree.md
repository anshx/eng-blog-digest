---
title: "The Bw-Tree: A B-tree for New Hardware Platforms"
source: https://www.microsoft.com/en-us/research/publication/the-bw-tree-a-b-tree-for-new-hardware/
author: Justin Levandoski, David Lomet, Sudipta Sengupta
company: Microsoft Research
date_posted: 2013-04-01
date_digested: 2026-09-01
---

# The Bw-Tree: A B-tree for New Hardware Platforms

## What's new to learn

- **Page Mapping Table (PMT)**: A shared, lock-free indirection table that maps every B-tree page's logical ID to its current physical location — memory pointer or flash offset — updated via atomic CAS instead of pointer surgery.
- **Delta chaining**: Every write prepends a typed delta record to a page's current state via a single CAS; the chain is the live page, and consolidation is applied lazily when the chain grows too long.
- **Cooperative SMOs**: Structure modifications (splits, merges) are broken into a sequence of single-CAS steps; any thread that encounters a half-finished operation *helps complete it* before proceeding, achieving progress without a coordinator.

## Prerequisites

- **B+ tree basics**: leaf nodes hold data, inner nodes hold routing keys, splits propagate upward.
- **Compare-and-swap (CAS)**: an atomic hardware instruction `CAS(addr, expected, new)` that succeeds only if `*addr == expected` — the primitive underlying all lock-free data structures.
- **Lock-free vs. latch-free**: in database terminology, a "latch" is a short-lived OS mutex; "latch-free" means no OS-blocking synchronization anywhere in the index code path.
- **Log-structured storage**: writing all changes sequentially to an append-only log rather than in-place, then merging (compacting) lazily.

## The core idea

A traditional B-tree holds each node at a fixed physical memory address. When you insert a key, you latch the node, update it in place, and release the latch. The latch prevents torn reads during a write that spans multiple cache lines.

The Bw-Tree eliminates that latch by never modifying a page in place at all. Instead, every update creates a tiny "delta record" — a struct containing just the changed key-value pair and a pointer to the previous page state — and atomically CASes it to the front of the page's delta chain. The mapping table maps the page's *logical ID* to the pointer at the head of that chain.

The result: a reader sees the page as the merge of all delta records in the chain. A writer does a single atomic CAS. Nobody holds a lock. Multiple concurrent CASes for the same page serialise naturally: only one CAS wins; losers retry with the updated chain head.

Think of it as: **the Bw-Tree is a B-tree where every "write" is a git commit, the page is the HEAD of the chain, and consolidation is `git gc --aggressive`.**

## Mechanics

### 1. The Page Mapping Table

The PMT is a flat array indexed by Logical Page ID (LPID). Each slot holds one machine word: either a tagged pointer to a delta chain in memory, or a flash offset. All access — reads, writes, splits — must go through this table.

```
LPID 42 → [delta3 → delta2 → delta1 → base_page]
LPID 43 → [flash:0x1A00000]
LPID 44 → [base_page_ptr]
```

Because every reference to page P is `PMT[P]`, you can atomically relocate the page (to flash, to a new memory address after consolidation) with a single CAS on one word. No other pointer needs updating — the tree's inner nodes hold LPIDs, not physical pointers.

### 2. Delta chaining — reads and writes

**Insert of key K into page P:**
1. Read `head = PMT[P]`
2. Allocate `delta = {type=INSERT, key=K, value=V, next=head}`
3. `CAS(PMT[P], head, delta)` — retry from step 1 if it fails

**Read of key K from page P:**
1. Read `ptr = PMT[P]`
2. Walk the chain: for each delta, check if it covers K (insert/delete with matching key); stop at base page if not found

Chain traversal cost is O(chain_length). Reads get slower as the chain grows — which is why consolidation exists.

**Delta record types:**
| Type | Purpose |
|------|---------|
| INSERT | new key-value pair |
| DELETE | tombstone for a key |
| SPLIT | signals a page split; carries sibling LPID |
| INDEX-TERM | additional routing key posted to parent page |
| MERGE | signals a merge; provides node-merged pointer |

### 3. Structure Modification Operations (SMOs) — splits

Splitting page P at separator key K, creating sibling Q:

**Phase 1 — install split-delta on P:**
1. Allocate LPID Q, populate Q with keys > K
2. Allocate `split_delta = {type=SPLIT, sep=K, sibling=Q, next=PMT[P]}`
3. `CAS(PMT[P], PMT[P], split_delta)`

At this point, any reader of P who wants a key > K follows the side-link to Q. The tree is consistent for reads even though the parent doesn't know about Q yet.

**Phase 2 — update parent with index-term-delta:**
1. Allocate `idx_delta = {type=INDEX-TERM, low=K, child=Q, next=PMT[parent]}`
2. `CAS(PMT[parent], PMT[parent], idx_delta)`

If a thread crashes or stalls between phase 1 and phase 2, the split-delta on P acts as a "pending" marker. Any thread that reads P and encounters the split-delta *completes the parent update* before continuing. This is **cooperative helping**: progress is guaranteed even if the original thread dies.

### 4. Consolidation

When a thread finds `chain_length(PMT[P]) > threshold` (typically 8):
1. Build a fresh, sorted base page C by replaying all deltas onto the old base
2. `CAS(PMT[P], old_head, C)`
3. If CAS wins, schedule the old chain for epoch-based garbage collection

**Epoch-based GC** prevents use-after-free in the absence of locks:
- Each thread registers its current epoch on entry
- A freed chain is actually released only when all registered epochs have advanced past the epoch when the CAS happened
- Same as Linux RCU's "wait for grace period before freeing" — but implemented explicitly

### 5. LLAMA — the durable storage layer

Below the in-memory Bw-Tree sits LLAMA (Log-structured Lock-free Access for Durable Storage):

- The PMT slot for a page can hold a *flash offset* instead of a memory pointer
- Flushing: consolidated base pages are written sequentially to a flash log; the PMT is updated with the flash address
- Reading from flash: on miss, load from flash offset, cache in memory, CAS the PMT slot to the memory pointer
- Sequential flash writes = exploits SSD's bandwidth asymmetry and avoids random-write write amplification

## Where it breaks

**1. Delta chain traversal is cache-unfriendly.** Each delta is a separate heap allocation; traversing a chain of 8 means 8 pointer dereferences, each likely a cache miss. Traditional B-trees fit a node's keys in one cache line (with careful layout); Bw-Tree read latency scales with chain length.

**2. Consolidation is still a bottleneck.** Only one thread wins the consolidation CAS. Under heavy write load, many threads attempt consolidation concurrently; most retry, burning CPU. The effective consolidation rate is serialized per page.

**3. The paper's GC protocol had bugs.** Wang et al. (SIGMOD 2018 — "Building a Bw-Tree Takes More Than Just Buzz Words") implemented the Bw-Tree faithfully and found multiple correctness issues in the original epoch-based GC and SMO help-completing logic. Implementing this correctly requires significant additional machinery.

**4. Merges are much harder than splits.** The paper describes the merge protocol in one paragraph; in practice, merge-delta, remove-delta, and parent cleanup interact with concurrent splits and consolidations in ways the paper doesn't fully specify. Most open-source implementations skip merges entirely.

**5. PMT is a shared-memory bottleneck at high core counts.** At 128+ cores, CAS contention on heavily-accessed PMT entries (hot pages near the root) becomes significant. The PMT is fundamentally global state.

**6. Real-world adoption has been narrow.** The Bw-Tree powers SQL Server's Hekaton in-memory engine — a successful production deployment — but saw little adoption elsewhere. The complexity of correctly implementing SMOs and GC, combined with the read-path overhead, has limited uptake vs. lock-based B-trees with fine-grained latching or skip lists.

## Why it works

The Bw-Tree is built on one fundamental insight: **every B-tree operation that previously needed a latch needed it because it touched more than one pointer simultaneously**. By introducing an indirection layer (the PMT), you reduce every multi-pointer update to *one CAS on one word*.

This is the **"indirection breaks the atomicity requirement"** principle, appearing in:

| System | Indirection Layer | What it enables |
|--------|-------------------|-----------------|
| OS virtual memory | Page table (VA → PA) | Move physical pages without updating pointers in code |
| git | Object store (SHA → blob) | Share objects, rewrite history without pointer surgery |
| inode-based filesystems | Inode table (ino → disk block) | Move data blocks without updating directory entries |
| Percolator / TiKV | Write-intent columns as extra KV pairs | Distributed 2PC without a single lock table |
| **Bw-Tree** | PMT (LPID → ptr) | Lock-free structural changes to a B-tree |

Delta chaining is **functional / persistent data structure updates** applied to a mutable index. Instead of destructive mutation, you prepend to an immutable linked list. This is how Clojure's persistent vectors, Haskell's Data.Map, and purely functional red-black trees are implemented. The delta chain *is* the version history of the page; consolidation is discarding old versions (like garbage-collecting old git commits).

The two-phase SMO with cooperative helping is **optimistic two-phase commit without a coordinator**. Phase 1 makes the split logically visible (via the split-delta side-link) before the parent is updated. Any concurrent reader can complete the operation — there's no "rollback" because Phase 1 is safe to replay. This is the same insight behind the Serf/raft "read-only follower can complete pending operations" pattern and Percolator's lock-column-as-coordinator approach.

## Going deeper

1. **"Building a Bw-Tree Takes More Than Just Buzz Words"** (Wang et al., SIGMOD 2018) — the CMU group implemented the Bw-Tree from scratch, found bugs in the original, and documented exactly what it takes to build a correct, high-performance version. The most important follow-up to this paper.

2. **"FASTER: A Concurrent Key-Value Store with In-Place Updates"** (Chandramouli et al., SIGMOD 2018) — Microsoft Research's next-generation key-value store, which evolved the hybrid-log idea from LLAMA into a cleaner design with a "log-structured update record" separated from a B-tree for point lookups.

3. **"Open-sourced Bw-Tree implementation"** (wangziqi2013/BwTree on GitHub) — the CMU reference implementation that accompanied the 2018 paper; real code with the GC and SMO bugs fixed, benchmarks, and comments explaining subtle invariants.
