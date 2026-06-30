---
title: "Better bitmap performance with Roaring bitmaps"
source: https://arxiv.org/abs/1402.6407
author: Samy Chambi, Daniel Lemire, Owen Kaser, Robert Godin
company: Université du Québec à Montréal / University of New Brunswick
date_posted: 2016-02-01
date_digested: 2026-06-30
---

# Better bitmap performance with Roaring bitmaps

## What's new to learn

1. **Two-level chunked indexing**: any 32-bit integer set can be split into 65,536 independent "containers" keyed by the high 16 bits, each holding only the low 16 bits — so each container spans exactly one chunk of the integer space.

2. **Adaptive container dispatch**: within each chunk, the data structure independently chooses the cheapest representation (sorted array, flat bitset, or run-length list) by measuring how many integers are present, switching at a threshold derived from equal memory cost.

3. **Heterogeneous set operations**: AND/OR/NOT between Roaring bitmaps work container-by-container and pick the correct output container type from the input types — bitset × bitset → SIMD AND; array × bitset → probe; array × array → galloping intersect — with no decode step needed.

## Prerequisites

- Bitmap (bitset) basics: a flat byte array where bit `i` records whether integer `i` is in the set.
- Why naive bitmap indexes are wasteful: a full 32-bit bitmap needs 512 MB even if the set has two elements.
- Run-length encoding: represent `[3,3,3,4,4,5]` as `(start=3, len=3), (start=4, len=2), (start=5, len=1)`.
- SIMD: single instructions that operate on 128–512 bits at once (SSE2/AVX2 on x86).

## The core idea

Suppose you want to store the set `{0, 1, 100000, 4294967295}` — four 32-bit integers. A naive bitset needs 512 MB. WAH and EWAH (word-aligned hybrid) apply run-length encoding to the bitset words, which compresses well for clustered data but requires decode before any bitwise operation.

Roaring takes a different path: it observes that a sorted array of 16-bit integers costs 2 bytes each, while a flat bitset costs 8 bytes per potential element (`65536 / 8 = 8192 bytes` for a 16-bit range). **At exactly 4096 integers, both representations use 8192 bytes. Below 4096 the array wins; above 4096 the bitset wins.** So you pick the cheaper one automatically.

This is the whole idea: split the 32-bit space into 65,536 chunks (each chunk = all integers that share the same top 16 bits), store each chunk in whichever of three container formats is cheapest, and do all set operations container-by-container with SIMD where possible.

A third container type — the run container — was added in the follow-up paper (arXiv 1603.06549) for highly structured data: it stores `(start, length)` pairs and can represent the set `{0,1,2,...,65535}` in just 4 bytes.

## Mechanics

### The two-level index structure

A Roaring bitmap is a sorted array of `(key, container)` pairs, where each key is a uint16 (the high 16 bits of every integer in that chunk) and each container holds only the corresponding low 16 bits.

```
32-bit integer = [key: uint16][value: uint16]
                     ↑              ↑
               container ID      stored in container
```

Looking up whether integer `n` is in the set:
1. Extract key = `n >> 16`, value = `n & 0xFFFF`.
2. Binary-search the key array → find (or miss) the container.
3. Search the container for `value`.

Iteration over all set members: visit each container in key order, emit `key << 16 | value` for each member.

### The three container types

**Array container** — `cardinality ≤ 4096`:
- `uint16[]` stored sorted, using `2 × cardinality` bytes.
- Lookup: binary search, O(log k).
- Set operations: sort-merge, or galloping search if one operand is much smaller.

**Bitset container** — `cardinality > 4096`:
- Flat 8192-byte array (65,536 bits, one per possible uint16 value).
- Lookup: `bitset[value >> 6] >> (value & 63) & 1`, O(1).
- Set operations: 128 SIMD `AND`/`OR`/`NOT` instructions cover all 8192 bytes at 64 bytes each (AVX2 does it in 64 instructions at 256 bits).
- Cardinality: hardware `POPCNT` summed over 64-bit words.

**Run container** — variable (added in 2018 paper):
- `(start: uint16, length: uint16)[]` sorted by start, using `4 × num_runs` bytes.
- Lookup: binary search over starts, O(log r).
- Optimal when: `4 × num_runs < min(2 × cardinality, 8192)`.
- Set operations: merge two sorted run lists in O(|runs_A| + |runs_B|).

### Container type selection

At construction time, the library inserts elements into an array container. When cardinality crosses 4096, it converts to a bitset container in O(k). After a set operation, the result container is converted to the optimal type based on its cardinality.

For run containers, an explicit `runOptimize()` call scans each container and replaces it with a run container if doing so saves space.

### Set operations across container types

The library maintains a dispatch table over the nine pairs (array × array, array × bitset, bitset × bitset, etc.). Key cases for AND (intersection):

| Left       | Right      | Algorithm                                          | Output |
|------------|------------|----------------------------------------------------|--------|
| array      | array      | galloping intersect: O(min · log(max/min))        | array  |
| array      | bitset     | probe each array element in bitset: O(\|array\|)   | array  |
| bitset     | bitset     | SIMD AND across 8192 bytes, then popcount          | bitset or array |
| run        | run        | merge two run lists                                | run    |

For bitset × bitset AND, if `popcount(result) ≤ 4096` the output is converted to array. This is a single pass: accumulate bits with SIMD, count with `POPCNT`, convert if needed.

**Critical efficiency**: containers for chunks not present in both operands are skipped entirely. If bitmap A has elements in chunk 7 and bitmap B has none, chunk 7 produces an empty intersection with zero work — the same skip-absent-chunk principle as predicate pushdown in columnar storage.

### Performance numbers from the papers

On typical OLAP workloads:
- Intersection (AND): 900× faster than WAH/EWAH on sparse data.
- Union (OR): 2–10× faster.
- Serialized size: 2–5× smaller than plain bitset, 1.5–2× smaller than EWAH.
- In-memory footprint: 10–100× smaller than full 32-bit bitset for real-world distributions.

## Where it breaks

**Only 32-bit integers natively.** For 64-bit keys, variants exist (split into three levels) but add complexity and the container tricks become less clean.

**Not thread-safe by default.** All set operations produce a new container; mutation requires external locking. The Java implementation has a `ImmutableRoaringBitmap` class but mutation requires explicit management.

**Run containers add a code-path cost.** The nine-pair dispatch table becomes a 25-pair table with run containers, adding branch overhead in the common case even when no run containers are present.

**Adversarial cardinalities near 4096 cause thrashing.** If insertions and deletions keep cardinality oscillating around the threshold, the library repeatedly converts between array and bitset (O(k) each way). Real workloads rarely hit this; synthetic benchmarks can.

**Assumes integer keys.** If your identifiers are not dense 32-bit integers (e.g., UUIDs), you need to map them first. The quality of the compression depends heavily on how clustered the integer space is.

## Why it works

The deep principle is **adaptive representation dispatch**: measure a quantity that predicts which encoding is cheapest, then switch representations at the crossover point. This is not specific to bitmaps:

- **Adaptive Radix Tree (ART)**: inner nodes come in four sizes (4, 16, 48, 256 children), switching as fan-out grows. The switch point is where one node size's storage overhead equals the next's.
- **Parquet column encodings**: dictionary encoding for low-cardinality columns, delta for timestamps, plain for high-cardinality floats — chosen by column statistics at write time.
- **jemalloc size classes**: small objects use thread-local slabs, large objects use mmap. The switch near 64 KB matches TLB and page-size economics.
- **HNSW layer structure**: a node appears at level `⌊-log(uniform(0,1)) × m⌋`, giving exponentially fewer nodes per layer — adaptive density across levels.

In each case, the key insight is: **don't commit to one global representation; measure per-shard and dispatch locally**. Roaring implements this at the chunk (container) granularity, and the economics are unambiguous because the 4096 crossover is exact, not empirical.

A second principle is **operation-time type propagation**: the output of an AND between two bitset containers might be sparse enough to fit in an array. By checking cardinality after the SIMD pass, the library produces the optimal output type without a separate conversion step. This is analogous to how a JIT compiler picks narrow integer representations after tracking value ranges through a computation — don't widen until you have to.

Finally, skipping empty chunks during set operations is **block-level predicate pushdown**: the same insight that makes ClickHouse's granule skipping and Parquet's page-level statistics effective. When one operand has no data in a chunk, no work is done for that chunk, regardless of how sparse or dense the other operand is.

## Going deeper

1. **"Consistently faster and smaller compressed bitmaps with Roaring"** (Lemire et al., 2018, arXiv 1603.06549) — the follow-up that adds the run container and benchmarks across 65 real-world datasets. Explains when `runOptimize()` pays off and how run × run operations work.

2. **"Roaring Bitmaps: Implementation of an Optimized Software Library"** (Lemire et al., 2018, Software: Practice and Experience) — covers the SIMD implementation details, cache-line alignment, and the `POPCNT`-over-AVX2 trick for simultaneous bitwise-AND and cardinality counting.

3. **"The Adaptive Radix Tree: ARTful Indexing for Main-Memory Databases"** (Leis et al., ICDE 2013, db.in.tum.de/~leis/papers/ART.pdf) — the canonical paper on the other major adaptive-representation data structure, for readers who want to see how the same "switch at fan-out crossover" principle plays out for ordered indexing rather than set membership.
