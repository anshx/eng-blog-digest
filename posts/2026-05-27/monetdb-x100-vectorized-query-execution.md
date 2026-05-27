---
title: "MonetDB/X100: Hyper-Pipelining Query Execution"
source: https://www.cidrdb.org/cidr2005/papers/P19.pdf
author: Peter A. Boncz, Marcin Zukowski, Niels Nes
company: CWI (Centrum Wiskunde & Informatica) / TU Delft
date_posted: 2005-01-04
date_digested: 2026-05-27
---

# MonetDB/X100: Hyper-Pipelining Query Execution

## What's new to learn

1. **The Volcano iterator model's CPU cache failure mode.** The standard database pipeline model — every operator has a `next()` returning one tuple — processes rows so slowly (one at a time) that data evicts from cache before it's used a second time. The bottleneck is not arithmetic; it's the cost of re-loading the same data repeatedly from DRAM.

2. **Vector-at-a-time (hyper-pipelining) execution.** Each operator call returns a batch of ~1000 column-values instead of a single row. The batch fits in L2 cache; tight inner loops let the compiler emit SIMD instructions (4–8 values per clock cycle) and the CPU hardware prefetcher read ahead without stalls. On TPC-H Q1, this drops from 49 CPU cycles per tuple (MySQL) to **2.2 cycles per tuple**.

3. **Selection vectors: filtering without data movement.** When a predicate eliminates rows, don't compress the survivors into a new array. Record their original positions in a small integer array (the *selection vector*). Downstream operators index into the original arrays via this list — zero memcpy, zero allocation.

## Prerequisites

- CPU cache hierarchy: L1 (~32 KB, ~1 ns), L2 (~256 KB, ~4 ns), L3 (~8–32 MB, ~10 ns), DRAM (~50–100 ns). A cache miss to DRAM costs 50–100× more than an L1 hit.
- Basic SQL query processing: queries compile into trees of relational operators (Scan, Filter, Project, Aggregate, Join).
- Awareness that modern CPUs can execute 4–8 floating-point operations per clock cycle — but only if the data is already in a register or L1 cache.

## The core idea

The **Volcano/iterator model** (Graefe 1994) gives every database operator one method:

```
next() → Tuple
```

The query `SELECT SUM(l_extendedprice * (1 - l_discount)) FROM lineitem WHERE l_shipdate <= '1998-09-02'` compiles into:

```
Aggregate.next()
  → calls Filter.next()
    → calls Scan.next()
      → returns one tuple
  → evaluates predicate on that tuple
  → if it passes, multiplies two fields and adds to running sum
  → calls Filter.next() again ...
```

This is elegant and composable. It is also **catastrophically cache-unfriendly**. On the TPC-H lineitem table with 6 million rows, the engine calls `next()` 6 million times. Each call loads a 128-byte row from somewhere in memory, processes two field values, and discards the row. By the time the engine comes back for the next row, the 6 million rows have long since evicted from cache. Every row access is a cache miss.

X100's answer: make `next()` return a **vector** — an array of ~1000 column-values. One call to `Filter.next(price_vector, discount_vector)` yields 1000 prices and 1000 discounts, both as contiguous arrays. The multiplication loop becomes:

```c
for (int i = 0; i < n; i++)
    result[i] = price[i] * (1.0 - discount[i]);
```

Both arrays (8 KB each at float64) fit in L2 cache together. The loop runs at close to peak memory bandwidth with no function call overhead per element. The compiler auto-vectorizes using SSE or AVX: 4–8 multiplications per clock cycle.

The result on the same Pentium 4 hardware: **2.2 CPU cycles per tuple**, versus 49 for MySQL and ~19 for the original MonetDB (which processed whole columns but caused massive intermediate materialization). X100 is 1–2 orders of magnitude faster on the 100 GB TPC-H benchmark.

## Mechanics

**Column-oriented physical layout.** X100 stores each column as a contiguous typed array (`float64[]`, `int32[]`, etc.). Loading 1000 prices is one sequential read; no row-struct interleaving. Sequential access patterns are what hardware prefetchers optimize for.

**The `Vector` type.** A vector is a lightweight struct:

```c
struct Vector {
    void    *data;       // pointer to the column array
    uint16_t n;          // number of valid entries
    uint16_t *sel;       // selection vector (NULL means "all valid")
    Type     type;
};
```

**Selection vectors for filtering.** When a predicate removes rows, X100 builds a selection vector `sel[] = {2, 5, 6, 9, ...}` listing the indices of rows that passed. Downstream operators iterate:

```c
for (int j = 0; j < sel_count; j++) {
    int i = sel[j];
    result[j] = price[i] * (1.0 - discount[i]);
}
```

Cost: one extra indirection per surviving element, zero data movement. For a 90%-selective predicate (10% of rows survive), this saves copying 900 elements out of every 1000 into a new array. The selection vector itself is tiny (~2 KB at uint16_t for 1000 elements) and stays hot in L1 cache.

There is a crossover: when selectivity is very low (< ~10% survivors), compressing the surviving values into a fresh dense array and dropping the selection vector wins, because the subsequent tight loops benefit from stride-1 access again. This is an adaptive decision later systems make explicitly.

**Late materialization.** Columns flow independently through the operator pipeline. X100 never assembles wide multi-column rows until forced. A filter on `l_shipdate` and an aggregation of `l_extendedprice * l_discount` only ever touch those three columns; the other 14 lineitem columns are never read. The final output operator interleaves columns into result rows as the last step.

**Expression primitives.** Each algebraic primitive (`multiply`, `subtract`, `cast_float_to_int`, `string_like`, ...) is a standalone C function with the signature `op(Vector *out, Vector *a, Vector *b)`. The query compiler selects primitives from a library of ~100 and chains them. Each primitive is a tight inner loop that the compiler can optimize aggressively in isolation — no cross-operator entanglement.

**Hash join.** The smaller join input is materialized column-by-column into a hash table. The larger input probes the hash table in vectorized batches: gather the join-key column into a vector, compute all 1000 hashes, batch-probe the hash table, collect matching pointers. The hash table must fit in L2/L3 cache for probe performance to stay good; when it doesn't, X100 applies **radix partitioning** to split both inputs into cache-sized chunks first (each chunk fits, so probes are fast), paying one extra sequential pass over both inputs.

## Where it breaks

**Vector size is a fixed approximation.** X100 uses a fixed vector size (~1000 elements). The optimal size depends on how many columns flow simultaneously: 10 active `float64` columns at 1000 elements each is 80 KB, which overflows L1 into L2. Too large and cache reuse drops; too small and per-vector overhead dominates. Later systems (DuckDB uses 2048) pick empirically and accept the approximation.

**Selection vectors have a slow path past a selectivity threshold.** The indirection `a[sel[j]]` defeats auto-vectorization in many compilers because the gather instruction (`_mm256_i32gather_ps`) has higher latency than a stride-1 load. Below ~10% selectivity, compacting survivors into a dense array and running a stride-1 loop is faster. Systems that don't adaptively switch pay a penalty in high-selectivity query branches.

**Large hash joins break cache assumptions.** A hash table for a 100M-row fact-to-dimension join won't fit in L3. Even with radix partitioning (which adds an extra data pass), probe performance is limited by random-access memory latency rather than arithmetic throughput. Vectorizing the probe helps at the margins but doesn't fundamentally solve the memory-bandwidth bound.

**Expression compilation takes it further.** X100's library of hand-written primitives doesn't compose as tightly as native code compiled for a specific query. If the query has complex expressions (`CASE WHEN x > 0 THEN log(x) ELSE 0.0 END` over a price column), chaining multiple primitives re-reads the intermediate vector from memory between steps. LLVM-based JIT compilation (later adopted by HyPer, DuckDB, and ClickHouse's optional JIT) fuses the entire expression into one tight loop and eliminates the intermediate writes. Vectorization sets the ceiling; JIT compilation approaches it.

## Why it works

The deeper principle is that **data movement cost dominates arithmetic cost on modern hardware**. A 2004 Pentium 4 could execute an FP multiply in 1 clock cycle but needed 50 ns (≈100 cycles at 2 GHz) to fetch a cache line from DRAM. The ratio has only widened since: today's server CPUs execute 32+ flops per cycle but still wait ~100 ns for DRAM. Any computation that triggers DRAM accesses is bottlenecked by memory, not compute.

The Volcano model loses because calling `next()` once per row guarantees cache misses: each row is fetched, used briefly, and evicted before it's needed again. It's the database equivalent of the naive nested matrix multiplication:

```python
# Cache-hostile: each C[i][j] += A[i][k] * B[k][j] fetches new rows of A and B
for i in range(N):
    for j in range(N):
        for k in range(N):
            C[i][j] += A[i][k] * B[k][j]
```

Vectorized execution is what you get when you apply **loop blocking (tiling)** to relational query processing:

```python
# Cache-friendly: TILE×TILE submatrices fit in L2; full reuse before eviction
for ii in range(0, N, TILE):
    for jj in range(0, N, TILE):
        for kk in range(0, N, TILE):
            for i, j, k in product(range(ii, ii+TILE), range(jj, jj+TILE), range(kk, kk+TILE)):
                C[i][j] += A[i][k] * B[k][j]
```

Both restructure computation so that a block of data is fully processed within the cache before being evicted. The "oh, X is just Y" insight here:

> **Vectorized database execution is loop tiling applied to relational algebra.**

The same insight appears across systems at different granularities:
- **FlashAttention** tiles GPU attention computation to stay in SRAM (on-chip memory), avoiding round-trips to HBM — the same principle one memory level up.
- **Cache-oblivious algorithms** (Frigo, Leiserson et al., 1999) choose tile sizes that are automatically optimal for any cache level without knowing cache parameters.
- **BLAS / cuBLAS** spend enormous effort on tiling DGEMM to maximize cache reuse — the kernel that HPC is built on.
- **CPU hardware prefetchers** work best with stride-1 access patterns; columnar layout provides exactly this, which is why the hardware helps vectorized DB execution for free.

The general rule: **if you must process N items with operation F, bring all N items into the cache before running F, not one item at a time**.

## Going deeper

1. **"DuckDB: An Embeddable Analytical Database" (SIGMOD 2019) — Mark Raasveldt, Hannes Mühleisen.** The direct descendant of X100, by the same research group. Describes the push-based vectorized pipeline DuckDB uses, adaptive string handling, and how embeddability changes the optimizer's assumptions. Available at [duckdb.org/docs/internals/overview](https://duckdb.org/docs/internals/overview).

2. **"Rethinking SIMD Vectorization for In-Memory Databases" (SIGMOD 2015) — Orestis Polychroniou, Arun Raghavan, Kenneth A. Ross.** Covers the hard frontier: when selection vectors lose to compaction, when AVX-512 scatter/gather helps hash probes, and what the limits of software vectorization are on real hardware. Freely available via ACM DL.

3. **"ClickHouse: Lightning Fast Analytics for Everyone" (VLDB 2024) — Schulze et al.** Shows how X100's ideas look at petabyte scale in production: the MergeTree LSM storage engine, sparse primary indexes (one entry per 8192 rows instead of per row), and optional LLVM JIT compilation to close the gap between interpreted primitives and native code. Available at [vldb.org/pvldb/vol17/p3731-schulze.pdf](https://www.vldb.org/pvldb/vol17/p3731-schulze.pdf).
