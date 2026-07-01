---
title: "Beating hash tables with trees? The ART-ful radix trie"
source: https://www.the-paper-trail.org/post/art-paper-notes/
author: Henry Robinson
company: Paper Trail (independent blog; paper by TU Munich — Leis, Kemper, Neumann)
date_posted: 2018-11-03
date_digested: 2026-07-01
tags: [data-structures, indexing, databases, simd, tries]
---

# Beating hash tables with trees? The Adaptive Radix Tree

## What's new to learn

1. **Adaptive node types**: Instead of allocating a fixed 256-pointer array at every trie node, ART uses four node sizes (Node4, Node16, Node48, Node256) matched to the actual number of children — and promotes or demotes between them on insert and delete. This collapses per-key space from ~2 KB to 2–10 bytes.

2. **SIMD search inside sorted arrays**: A 16-element sorted key array can be searched with a single SSE2 instruction (`_mm_cmpeq_epi8`) that broadcasts one byte to all 16 positions and compares them simultaneously — faster than any hash or binary search at this scale.

3. **Path compression (lazy expansion)**: Single-child chains in the trie are collapsed by storing the skipped key bytes inline in the node header (up to 8 bytes). For longer compressed paths, the full check is deferred until the leaf is reached, so no heap allocation is needed.

## Prerequisites

- **Radix tries**: each node branches on one "character" of the key; paths from root to leaf spell out the full key. Distinguished from B-trees: tries never need rebalancing, and height is proportional to key length (not key count).
- **Hash tables vs sorted structures**: hash tables give O(1) point lookups but can't answer range queries or iterate in key order.
- **CPU cache mechanics**: 64-byte cache lines; L1 ~4 cycles, L2 ~15 cycles, L3 ~40 cycles, DRAM ~200 cycles. Every pointer dereference that misses L1 costs at least one round-trip.
- **SSE2/SIMD basics**: 128-bit registers that operate on 16 bytes simultaneously in a single instruction.

## The core idea

A vanilla radix trie processes keys one byte at a time. Each internal node needs a slot for every possible byte value (0–255). That's 256 × 8 bytes = 2,048 bytes per node — most of which hold NULL for any real-world key set.

ART replaces the one-size-fits-all node with **four node types** that grow as children accumulate:

| Type    | Max children | Lookup method              | Approx. size |
|---------|-------------|----------------------------|--------------|
| Node4   | 4           | Linear scan (4 bytes)      | ~48 B        |
| Node16  | 16          | SIMD compare (16 bytes)    | ~160 B       |
| Node48  | 48          | Byte-indexed indirect      | ~656 B       |
| Node256 | 256         | Direct array index         | ~2,072 B     |

A new node starts as Node4. When it fills up, the node is promoted to the next type (content copied, parent pointer updated). On deletion, a node that falls below a lower threshold is demoted back down.

**The payoff**: For 100 million random 64-bit integer keys, ART uses roughly 4.75 bytes/key — versus ~80–100 bytes/key for a red-black tree and ~40 bytes/key for an in-memory B-tree. Lookup speed is competitive with (or faster than) hash tables while retaining full sorted-order iteration.

## Mechanics

### Node4

Two parallel arrays of 4: `keys[4]` (sorted partial key bytes, one byte each) and `children[4]` (node pointers). To look up byte `b`:

```c
for (int i = 0; i < n; i++)
    if (keys[i] == b) return children[i];
return NULL;
```

Four bytes fit in a single 32-bit word. The loop body is 4 comparisons — well inside one cache line. Linear scan beats binary search here because branching cost dominates at this cardinality.

### Node16

Same two-array layout, now 16 elements. The key optimization: 16 bytes fit in one SSE2 register, enabling a **single-instruction compare**:

```c
__m128i keys_vec = _mm_loadu_si128((__m128i*)node->keys);
__m128i search   = _mm_set1_epi8(byte);          // broadcast search byte × 16
__m128i cmp      = _mm_cmpeq_epi8(search, keys_vec); // compare all 16 at once
uint16_t mask    = _mm_movemask_epi8(cmp);        // bit i = 1 if keys[i] == byte
if (mask == 0) return NULL;
int pos = __builtin_ctz(mask);                    // index of first match
return node->children[pos];
```

One cache line load, one SIMD compare, one bitmask scan. This is measurably faster than a 16-entry hash table because there are no collision chains and no second cache line.

### Node48

A 256-byte `index` array (one byte per possible key byte value, value = slot number or 0xFF for absent) plus a `children[48]` array. Lookup:

```c
uint8_t slot = node->index[byte];
if (slot == 0xFF) return NULL;
return node->children[slot];
```

Two array dereferences: one to convert the partial key byte to a slot, one to load the child pointer. No scan needed. The `index` array fits in 4 cache lines; at high access rates the top few nodes are hot in L1.

### Node256

A flat `children[256]` array. One array access:

```c
return node->children[byte];  // NULL if absent
```

Used only when a node has more than 48 children. At near-full occupancy (≥49 children), the 2,048 bytes of pointer storage have acceptably low waste.

### Path compression

If every node in a subtree has exactly one child — a "compressed path" — ART collapses the entire chain into the parent's **prefix field**:

```
Node header:
  uint8_t prefix_len;     // how many bytes are compressed
  uint8_t prefix[8];      // up to 8 compressed bytes stored inline
```

Example — keys `database`, `dataflow`, and `data` share the prefix `data`:

```
Root [prefix="data", len=4]
 ├── 'b' → leaf "ase"
 ├── 'f' → leaf "low"
 └── (null terminator) → leaf ""   (for the key "data" itself)
```

Without compression, `d`, `a`, `t`, `a` would be four separate Node4s each with one child — four pointer dereferences and four cache misses instead of one.

**Lazy expansion**: If the prefix is longer than 8 bytes, only the first 8 are stored. When lookup reaches a leaf, the search key is compared against the actual stored key byte-by-byte to catch cases where the 8-byte stored prefix matches but the remainder diverges. No heap allocation is needed; the leaf pointer is always sufficient to recover the full original key.

### Key encoding

To traverse byte-by-byte while preserving sort order, keys must be in **big-endian byte order** (MSB first). On x86 (little-endian):

- Unsigned integers: `bswap64(key)` before insert/lookup
- Signed integers: flip the sign bit, then `bswap64`
- IEEE 754 floats: flip sign bit (negatives get full bit-reversal); this maps float bit patterns to their signed sort order
- Null-terminated strings: already in sort order byte-by-byte

This transformation is O(1) and invisible to callers of an ordered-map API.

### Promotion and demotion

```
Node4  full (n=4) on insert  → allocate Node16, copy 4 keys+pointers, free Node4
Node16 full (n=16) on insert → allocate Node48, build index[], copy children
Node48 full (n=48) on insert → allocate Node256, expand to full array
Node48 sparse (n≤16) on delete → shrink to Node16
Node16 sparse (n≤4) on delete  → shrink to Node4
```

Each promotion/demotion requires one allocation, one memcpy, and one parent pointer update. No rebalancing — the structure is always correct by construction after the parent update.

## Where it breaks

**1. Variable-length string keys without a length bound.**
Trie height is proportional to key length. A 1,000-byte URL produces up to 1,000 trie levels even with aggressive path compression — far worse than an O(log n) B-tree for a modest key set. ART is best suited to fixed-width keys (integers, UUIDs) where height is strictly bounded.

**2. Concurrent writes.**
Promoting a Node4 to Node16 is not atomic: allocate, copy, then update the parent pointer. Under concurrent reads and writes, a reader can observe the old pointer after the new node is initialized but before the parent pointer is updated. The published solution (ART-OLC, "The ART of Practical Synchronization," DaMoN 2016) uses **optimistic lock coupling**: each node carries a version counter; readers take no locks but validate the version after reading, retrying if the node was modified mid-traversal. Correct but substantially more complex than a single-writer implementation.

**3. Skewed key distributions (clustered keys).**
If all keys share a very long common prefix (e.g., Unix filesystem paths under `/home/user/...`), path compression stores only 8 bytes of that prefix inline. The lazy-expansion check at the leaf adds one extra comparison per lookup — minor in isolation but non-trivial under billions of lookups.

**4. Very small key sets.**
For fewer than ~100 keys, a sorted array with binary search outperforms ART: no pointer chasing, perfect cache utilization, and trivial implementation. ART's overhead per node only pays off at scale.

**5. Pointer-heavy value storage.**
ART gives key lookups in O(key_length) time, but values are pointed to from leaves. If values are stored in an unsorted heap (as in DuckDB's ART index pointing into a row store), range scan performance degrades to O(k) random reads — similar to a B-tree secondary index. A clustered index (values stored in trie leaf nodes themselves) would fix this at the cost of variable leaf sizes.

## Why it works

The core principle: **adapt the internal representation to the density of the data**.

A plain radix trie allocates 256 child slots everywhere because any node *might* have up to 256 children. In practice, the root and top few levels are sparse (a small fraction of bytes actually appear as the first byte of a key), while deep nodes near the leaves are denser. ART's four node types pay only for what's needed at each level.

This is exactly the same insight as:

- **Roaring bitmaps** (archive 2026-06-30): 65,536-element chunks stored as sorted arrays (sparse), flat bitsets (dense), or run-length lists (very dense) — the crossover is at the 4,096-element threshold where all three have equal memory cost.
- **Protocol Buffers varints**: 1 byte for values 0–127, 2 bytes for 128–16,383, etc. — code length matches value magnitude.
- **Huffman coding**: short bit sequences for frequent symbols, long sequences for rare ones — code length is inversely proportional to probability.

In each case, the trick is having a **cheap O(1) dispatch** that picks the right representation at runtime — `node->type` in ART, `container->type` in Roaring, the high bit in protobuf varints, the code-table lookup in Huffman.

ART adds a second, distinct insight: **for fixed-width keys, trie height is O(1) in the number of keys**. An 8-byte integer key produces exactly 8 trie levels regardless of whether the table has 1,000 or 1,000,000,000 entries. The lookup cost is bounded by the key width, not the key count — the same property that makes radix sort O(n) rather than O(n log n): when the key domain is fixed, "sorting by key" is the same as "distributing into buckets by key prefix."

The SIMD insight at Node16 is a third independent lesson: **broadcast-and-compare converts sequential search into parallel search**. The pattern `_mm_set1_epi8(b); _mm_cmpeq_epi8(...)` appears in virtually every high-performance string library (`memchr`, `memmem`, PCRE's JIT) and in vectorized database predicates (the MonetDB/X100 archive entry, 2026-05-27). Here it is applied not to a large array but to a tiny 16-element one — demonstrating that SIMD pays even at small scales when the comparison is on the critical path of every lookup.

## Going deeper

1. **The original paper**: Viktor Leis, Alfons Kemper, Thomas Neumann, "The Adaptive Radix Tree: ARTful Indexing for Main-Memory Databases," ICDE 2013 — https://db.in.tum.de/~leis/papers/ART.pdf. Full pseudocode, micro-benchmarks against Judy, STX B+-tree, and red-black trees across four real-world key distributions.

2. **Concurrent ART**: Leis et al., "The ART of Practical Synchronization," DaMoN@SIGMOD 2016 — https://db.in.tum.de/~leis/papers/artsync.pdf. Introduces optimistic lock coupling (OLC): version counters on each node let readers proceed without acquiring locks, validating atomicity after the fact. The same versioned-read pattern underlies SeqLock in the Linux kernel and MVCC snapshot validation in databases.

3. **ART in DuckDB production**: "Persistent Storage of Adaptive Radix Trees (ART) in DuckDB," DuckDB Blog, 2022 — https://duckdb.org/2022/07/27/art-storage. How DuckDB serializes ART to disk for primary key and unique constraint enforcement, with lazy subtree loading that avoids materializing the entire index on startup.
