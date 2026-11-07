---
title: "Ideal Hash Trees"
source: https://lampwww.epfl.ch/papers/idealhashtrees.pdf
author: Phil Bagwell
company: EPFL
date_posted: 2001-01-01
date_digested: 2026-11-07
---

# Ideal Hash Trees

## What's new to learn

1. **Hash Array Mapped Trie (HAMT)**: A trie where each internal node stores a 32-bit bitmap plus a *compact* array of only the populated children. Looking up a child is a two-instruction operation: test a bit in the bitmap, then index into the array using `popcount(bitmap & prefix_mask)`.

2. **Structural sharing via path copying**: "Updating" an immutable tree means copying only the O(log₃₂ N) ≤ 7 nodes on the root-to-leaf path and reusing all other nodes verbatim. Old versions remain valid — GC reclaims only what becomes unreachable.

3. **Popcount indexing**: Population count (counting set bits in a word) is the bridge from a sparse per-slot bitmap to a dense array. It converts a logical slot number into a physical array offset in one hardware instruction on modern CPUs (`POPCNT` on x86; `vcnt` on ARM).

## Prerequisites

- Hash tables: how a hash function maps a key to an integer index in a flat array.
- Trie (prefix tree): a tree where the branching at each level is determined by one chunk of the key.
- Immutable collections in a GC language are helpful context but not required — the paper itself is implementation-agnostic.
- Basic bitwise operations (AND, shift, popcount).

## The core idea

A standard hash table maps a key to a single integer slot in a flat array. That works, but it has two problems: (1) you must size the array up front or periodically rehash, and (2) any "copy" is O(N) — you must copy the whole array.

HAMT solves both by organizing the same key-value pairs as a *trie*, where the traversal key at each level is a 5-bit slice of the hash. Each internal node spans all 2⁵ = 32 possible children, but instead of always allocating 32 pointers it stores only the occupied ones, using a 32-bit bitmap as a directory and a compact array for the values.

For persistence (immutability), the crucial observation is: a "new version" of the map only differs from the old version along one root-to-leaf path. That path is at most ⌈32/5⌉ = 7 levels deep (for a 32-bit hash; ⌈64/5⌉ = 13 for 64-bit). So creating a new version costs O(depth) pointer-sized copies regardless of how many total entries the map has. Every other node is shared between the old and new version.

Explain it to a colleague at a whiteboard like this: *"Hash your key, split it into 5-bit nibbles. Walk down the trie one nibble at a time. When you hit the target leaf, create new copies of that leaf and every ancestor on the way back up — that's O(log₃₂ N) copies. Point the new root at the new copies; the old root still points to the old copies. The two versions share everything except the spine you just walked."*

## Mechanics

**Node layout**

```
struct Node {
    uint32_t bitmap;    // one bit per possible 5-bit slot (0–31)
    Value    children[popcount(bitmap)];  // only occupied slots stored
};
```

A `Value` is either a key-value leaf or a pointer to a child `Node`. The array is dense: slot 0 may be at `children[0]`, but only if the 0th bit of `bitmap` is set.

**Lookup**

```
fn get(node, hash, shift):
    slot   = (hash >> shift) & 0x1F          // 5 bits at current level
    if bit slot is NOT set in node.bitmap:
        return NOT_FOUND
    idx    = popcount(node.bitmap & ((1 << slot) - 1))   // how many set bits before slot
    child  = node.children[idx]
    if child is a leaf and child.key == key:
        return child.value
    return get(child, hash, shift - 5)       // recurse one level deeper
```

The `(1 << slot) - 1` mask selects all bits to the *right* of the target slot; popcount on that gives the number of occupied slots before it, which is exactly the dense-array index.

**Insertion (persistent path copy)**

```
fn insert(node, key, value, hash, shift):
    slot = (hash >> shift) & 0x1F
    idx  = popcount(node.bitmap & ((1 << slot) - 1))
    new_node = copy of node           // O(popcount(bitmap)) — small
    if bit slot is NOT set:
        new_node.bitmap |= (1 << slot)
        insert new leaf at position idx in new_node.children
    else:
        child = node.children[idx]
        if child is a leaf with same key:
            new_node.children[idx] = new leaf(key, value)   // replace
        else:
            new_child = insert(child, key, value, hash, shift - 5)  // recurse
            new_node.children[idx] = new_child
    return new_node   // new root of this subtree
```

Callers receive a new root node. The old root is unchanged. A GC language collects the old root when nothing holds a reference to it.

**Complexity**

| Operation | Time              | Space per version |
|-----------|-------------------|-------------------|
| Lookup    | O(log₃₂ N) ≈ O(1) | 0                |
| Insert    | O(log₃₂ N) ≈ O(1) | O(log₃₂ N) nodes copied |
| Delete    | O(log₃₂ N) ≈ O(1) | O(log₃₂ N) nodes copied |

For N ≤ 10⁶, depth is ≤ 4. For N ≤ 10⁹, depth is ≤ 6. The practical constant is so small that library benchmarks show HAMT insert at 100–300 ns per operation on modern hardware, only 3–5× slower than a mutable hash table.

**Hash collisions at max depth**

If two different keys hash to the same 32-bit value, the trie can't distinguish them at depth 7. The standard fix (used by Clojure) is a *collision node* — a small linear list that stores all key-value pairs sharing the same full hash. This occurs with probability 1/2³² per pair, so it is negligible in practice.

## Where it breaks

**Mutable-workload performance.** When you don't need immutability, a standard hash table is 10–50× faster. Each HAMT update allocates O(depth) small heap objects, which thrashes the allocator and kills spatial locality. Never use a HAMT for a hot mutable path.

**Cache unfriendliness.** Walking down depth-7 indirection chains chases pointers across the heap. Each level is typically a cache miss on a cold map. A flat open-addressing hash table with linear probing packs everything into one or two cache lines.

**Write amplification.** Updating a map with 10 M entries still copies 7 nodes. If you batch 10 M sequential updates (not using bulk-build operations), you create 70 M new nodes total. Persistent bulk-build requires a different algorithm (level-order insertion, not individual inserts).

**No ordering guarantee.** HAMT is a hash-keyed structure; keys come out in hash order, not insertion order or sort order. For sorted-order iteration or range queries, use a persistent balanced BST (red-black tree, AVL tree) instead.

**Large hash collisions.** The design assumes the hash function is well-distributed over 32 bits. A poor or adversarial hash function can cause one subtree to reach depth 32, degrading to a linear linked list.

## Why it works

The deeper principle is that **any tree update that only touches one root-to-leaf path can be made persistent in O(depth) time and space**, regardless of tree width or total node count. HAMT just picks the branching factor that minimizes depth in practice:

- Branching factor 2 (binary tree) → depth = log₂ N ≈ 20 for N=10⁶. Too many copies.
- Branching factor 32 (HAMT) → depth = log₃₂ N ≈ 4 for N=10⁶. Just right.
- Branching factor N (flat array) → depth = 1, but the entire array must be copied. Worse again.

The sweet spot is 32 (or 64) because it matches a CPU register width, makes popcount natural, and limits depth to ≤ 7 for all 32-bit hash spaces.

This is the same mechanism git uses for its object store: a git commit stores a tree object; that tree object shares every blob and sub-tree it hasn't touched with the previous commit. A commit that changes one file copies only the file blob and the O(depth) tree objects on the path from root to that file. Practically this is usually 2–4 node copies for a typical repository.

It's also the same mechanism as MVCC in databases: when Postgres updates a row, it creates a new heap tuple (the new version) and marks the old one with `xmax`. Only the changed tuple is new; every other tuple in the page is shared between the old and new transaction snapshots. The SST files in an LSM tree are the same idea at a coarser granularity — a compaction creates a new generation of immutable files, sharing nothing at the byte level, but the semantic invariant (old readers see old data) is structurally the same.

The unifying pattern: **immutability via path copying turns "modify" into "create a new root that shares all unchanged structure with the old root."** The cost is proportional to the path length, not the total structure size.

## Going deeper

1. **Clojure's PersistentHashMap** (`clojure/src/jvm/clojure/lang/PersistentHashMap.java`): Rich Hickey's original Java implementation, which introduced the CHAMP-precursor idea of merging data nodes and trie nodes. Reading it shows how the collision node is handled and how transient (temporarily mutable) versions work for bulk construction.

2. **CHAMP — "Optimizing Hash-Array Mapped Tries for Fast and Lean Immutable JVM Collections"** (Steindorfer & Vinju, ECOOP 2015): Separates the concept of a data node from a trie node, reducing node size and enabling efficient equality testing and iteration. CHAMP is what Scala 2.13+, Clojure 1.12+, and Pony's json package use internally.

3. **Okasaki, "Purely Functional Data Structures"** (Cambridge University Press, 1998): The foundational textbook on making data structures persistent. Covers persistence via path copying, amortization with lazy evaluation, and persistent queues, deques, and heaps. HAMT builds directly on the path-copying technique Okasaki formalizes.
