---
title: "Skip Lists: A Probabilistic Alternative to Balanced Trees"
source: https://dl.acm.org/doi/10.1145/78973.78977
author: William Pugh
company: University of Maryland
date_posted: 1990-06-01
date_digested: 2026-07-12
---

# Skip Lists: A Probabilistic Alternative to Balanced Trees

## What's new to learn

1. **Probabilistic balancing**: Assigning each node a random height via coin flips achieves O(log n) expected search, insert, and delete without any rotation or rebalancing — the tree organizes itself at insertion time through randomness alone.
2. **Concurrency-by-structure**: Skip lists are the only ordered data structure where lock-free concurrent writes are practical at production scale, because insertions never trigger multi-node restructuring that must be made atomic.
3. **Expected-case reasoning**: In a randomized algorithm the random choices belong to the algorithm, not the adversary — no input sequence can reliably trigger worst-case behavior, making "expected O(log n)" a stronger real-world guarantee than it first appears.

## Prerequisites

- Singly linked lists (pointer chasing, insertion by relinking)
- What a balanced BST (e.g., AVL or red-black tree) is and why rotations exist
- Basic probability: geometric distribution, expected value, linearity of expectation
- Big-O and Ω notation

Non-obvious prerequisite: the difference between worst-case and expected-case bounds, and why the latter is sound when the randomness is internal to the algorithm (not dependent on input).

## The core idea

A skip list is a sorted linked list augmented with multiple "express lanes" layered on top. Every node lives in lane 0 (the base list). At insertion time, the node flips a fair coin: with probability *p* (usually 1/2) it also appears in lane 1; with probability *p²* it appears in lane 2; and so on.

Searching starts at the highest occupied lane and scans right, making big jumps. Whenever the next node would overshoot the target, it drops down one lane and continues. By the time it reaches lane 0, it has narrowed down to the exact position.

The structure converges to:
- ~log₁/ₚ(n) lanes (≈ log₂(n) for p = 1/2)
- O(1) comparisons expected per lane traversal
- O(log n) expected total comparisons — the same asymptotic cost as a balanced BST, with no balancing code written by the programmer.

## Mechanics

### Node layout

```
       | key | val | forward[0] | forward[1] | forward[2] | ... |
```

`forward[k]` is a pointer to the next node at lane k. A node with height h occupies lanes 0 through h.

### Visual example (p = 1/2, keys 7 23 31 47 58)

```
Lane 3: [-∞] -----------------------------------------------[+∞]
Lane 2: [-∞] ----------> [23] --------------------------> [+∞]
Lane 1: [-∞] -> [ 7] -> [23] -----------> [47] --------> [+∞]
Lane 0: [-∞] -> [ 7] -> [23] -> [31] -> [47] -> [58] -> [+∞]
```

### Level assignment at insert

```python
def random_level(p=0.5, max_level=32):
    level = 0
    while random() < p and level < max_level:
        level += 1
    return level
```

The height is geometrically distributed: P(height ≥ k) = pᵏ, so the expected height is 1/(1-p). With p = 1/2, 50% of nodes are height 0, 25% height 1, 12.5% height 2, and so on.

### Search

```python
def search(sl, target):
    node = sl.head
    for lane in range(sl.max_lane, -1, -1):   # top to bottom
        while node.forward[lane].key < target:
            node = node.forward[lane]
    return node.forward[0]                      # candidate at lane 0
```

### Insert

1. Run the search loop above but record `update[lane]` — the rightmost node visited at each lane (this is where the new node needs to be spliced in).
2. Draw a random level `h`.
3. For lanes 0 through h: splice the new node between `update[lane]` and `update[lane].forward[lane]`.

### Delete

Same as insert: track `update[]`, then remove the target node from each lane it occupies by relinking `update[lane].forward[lane] = target.forward[lane]`.

### Probabilistic analysis

Let C(k) = expected comparisons to climb from lane 0 to lane k in a list of n nodes.

A backward induction argument (following Pugh 1990 §4): before each step up, you're at some node; the probability that its predecessor is also in the higher lane is p. This gives a recurrence whose solution is:

```
Expected comparisons to search a skip list with n keys ≈ (log n / log(1/p)) / p + 1/(1-p)
```

For p = 1/2 this is approximately 2 log₂(n), so the constant is roughly 2 — comparable to a red-black tree (≤ 2 log₂(n) comparisons).

**Tail bounds are tight**: with p = 1/2 and n = 65,536 nodes, the probability that any single search costs more than 3× the expected value is less than one in a million. The coin flips are independent across inserts, so no adversarial key sequence can force a bad structure.

### Space

Expected total pointers = n × E[height] = n × 1/(1-p). For p = 1/2: 2n pointers — the same asymptotic space as a BST with two child pointers per node.

## Where it breaks

**Cache behavior** is the main weakness relative to B-trees. Each lane hop follows a pointer to a potentially cold cache line. A B-tree with fanout 128 needs log₁₂₈(n) = 3 levels for 2 million keys, keeping nearly the entire tree in L3 cache; a skip list with the same data makes ~21 pointer-chase hops, many of them to cold nodes. For in-memory workloads skip lists are within 2–3× of B-trees; for disk-backed indexes B-trees dominate.

**Memory layout is hard to optimize**. Nodes are heap-allocated with variable-length forward arrays, scattered across the address space. You can mitigate with a slab allocator (as RocksDB does), but you cannot easily pack keys contiguously the way a B-tree can store all keys in one flat array per node.

**No worst-case bound**. If every node flips all heads and lands at height 0, you have a plain linked list. This probability is (p)^n — negligible in practice — but a hard real-time system that needs deterministic bounds cannot use a randomized structure.

**Tuning p and max-level is non-obvious**. The optimal p trades between space (fewer high-level nodes) and expected search time (more levels → less work per level). p = 1/4 halves pointer overhead but adds ~40% to comparison count. max-level = ⌈log₁/ₚ(N)⌉ for the expected maximum size; setting it too low degrades to O(n).

## Why it works

### The randomized balance insight

The deeper principle: **randomization can substitute for deterministic invariants while preserving expected performance**.

In a red-black tree, maintaining balance requires 5 invariants, 2 rotation types, and careful case analysis. The invariant is a global constraint: every path from root to leaf has the same black height. This must be repaired after every mutation.

In a skip list, the "balance" is implicit in the distribution of heights across all nodes. No single node's height depends on its neighbors. Because each node's height is drawn independently, the expected structure of the list is always tree-like — Pugh proves that the expected number of nodes at lane k is n·pᵏ, which gives a geometric decay exactly like the number of nodes at depth k in a perfectly balanced tree.

This is the same insight that makes **quicksort** with a random pivot O(n log n) in expectation: the key is that the randomness belongs to the algorithm (the pivot choice or the coin flip), not to the input. No adversary who controls the keys can force consistently bad behavior.

The connection to **treaps** is exact. A treap assigns each key a random priority, then maintains BST order on keys and max-heap order on priorities. The resulting tree is a random BST — provably equivalent in expected height to a skip list. A skip list is a treap expressed as a multi-level linked list: the height of a skip list node is the log of the inverse of its treap priority.

### The concurrent programming insight

Skip lists are the only comparison-based ordered structure that admits practical lock-free implementations. Here is why.

A red-black tree rotation modifies three nodes (the rotated node, its child, and its subtree root) atomically. Doing this lock-free requires a multi-word CAS (MWCAS), which is expensive and complex.

A skip list insertion modifies forward pointers one-by-one, bottom-up. Each `update[lane].forward[lane] = new_node` is a single pointer write. You can execute them with ordinary atomic stores, or with single-word CAS for the bottom lane (to serialize concurrent inserts at the same position), and then lazily update higher lanes with simple stores that readers tolerate seeing stale.

This is why:
- **RocksDB** uses a lock-free skip list (`InlineSkipList`) as its default MemTable when concurrent writes are enabled — trees were replaced precisely because the rebalancing required locks.
- **Java** provides `ConcurrentSkipListMap` (since Java 6) but no concurrent AVL or red-black tree in the standard library.
- **HBase** and **Apache Cassandra** both use skip-list based MemStores/Memtables for the same reason.
- **Redis** uses a skip list to implement `ZSET` (sorted sets), backing O(log n) `ZADD`, `ZRANGE`, and `ZRANGEBYSCORE` operations.

The general principle: **data structures with local, independent update operations compose better with concurrency than structures with cascading invariant repairs**. This is the same reason LSM trees beat in-place B-trees for write-heavy workloads — write isolation (level by level) beats shared-state mutation.

## Going deeper

1. **The original paper** — Pugh, W. (1990). "Skip Lists: A Probabilistic Alternative to Balanced Trees." *Communications of the ACM*, 33(6), 668–676. Twelve pages, including the full probabilistic analysis, implementation guide, and measured performance vs. AVL trees. Still the clearest exposition of the coin-flip argument.

2. **RocksDB's lock-free skip list** — `db/skiplist.h` and `db/inlineskiplist.h` in the RocksDB source tree. The inline variant avoids one pointer dereference per node by storing the key directly in the node allocation. The concurrent variant uses a CAS on the bottom pointer and then non-atomic stores upward, with a subtle "publish" barrier. Reading the ~600-line file teaches more about lock-free programming than most textbooks.

3. **The Ubiquitous Skiplist survey** — Vadrevu, Xing & Aref (2024), arxiv 2403.04582. Covers 20+ skip list variants (deterministic, bounded-height, disk-resident, LSM-integrated), traces their use across RocksDB, HBase, Cassandra, Redis, and blockchain systems, and surveys the theoretical landscape up to 2024. A comprehensive map of the design space if you want to go beyond the basics.
