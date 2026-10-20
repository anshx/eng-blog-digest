---
title: "What Every Programmer Should Know About Memory"
source: https://lwn.net/Articles/250967/
author: Ulrich Drepper
company: Red Hat
date_posted: 2007-09-21
date_digested: 2026-10-20
---

# What Every Programmer Should Know About Memory

## What's new to learn

- **Cache lines are the unit of memory transfer, not bytes.** A CPU never loads a single integer from RAM; it always loads the 64-byte cache-line block that contains it. Spatial locality is free; random access costs one cache miss per unique cache line touched, regardless of how many bytes you actually need from each one.
- **Cache-set conflicts: a cache can be full and still fail.** An N-way set-associative cache maps each address to exactly one *set* of N slots. Two data objects whose addresses are the same distance apart as a multiple of the cache's stride will compete for the same set — thrashing each other out even when the rest of the cache is empty.
- **False sharing: cores fight over granularity they can't see.** MESI cache coherency has no sub-line resolution. If core A writes byte 0 and core B writes byte 32 of the same 64-byte cache line, every write by either core invalidates the other's copy, even though they're touching disjoint data.

## Prerequisites

- Basic C: struct layout, pointer arithmetic, array indexing.
- Rough mental model of virtual memory: pages, page tables, TLB.
- No hardware background required — Drepper builds up the hardware model from scratch.

## The core idea

Most performance analysis focuses on algorithmic complexity. Drepper's central claim is that for the majority of real programs, **the bottleneck has moved from compute to memory access**. The DRAM wall — the widening gap between CPU clock speed and DRAM latency — means a modern out-of-order core can execute ~100 instructions in the time a single L3 cache miss takes. Algorithmic work is hidden inside that gap.

The implication: a data structure that seems slower by O() analysis can be faster in practice because it happens to be cache-friendly. A B-tree beats a sorted array at *O(log n)* search in theory, yet the B-tree is faster in practice partly because each node fills a cache line, while a sorted-array binary search jumps around unpredictably. **Cache behavior is a first-class property of a data structure, not an implementation detail.**

## Mechanics

### The memory hierarchy

Each level trades size for speed:

| Level | Typical size | Latency | Notes |
|-------|-------------|---------|-------|
| L1 cache | 32–64 KB | ~4 cycles | Per-core; split into I-cache and D-cache |
| L2 cache | 256 KB – 1 MB | ~12 cycles | Per-core; unified |
| L3 cache (LLC) | 8–64 MB | ~40 cycles | Shared across all cores on one socket |
| DRAM | GBs | ~200 cycles | Attached to the memory controller |
| Remote NUMA | GBs | ~400–600 cycles | Other socket's memory; traverses interconnect |

All inter-level transfers are in **cache lines** (64 bytes on all modern x86 CPUs). You never load fewer than 64 bytes from DRAM.

### Cache line lifecycle

1. CPU requests address A.
2. The cache controller extracts A's *index bits* (middle bits of the address) to identify one *set* of N ways.
3. It compares A's *tag bits* (high bits) against all N ways in that set.
4. **Hit:** returns data in ~4 cycles. **Miss:** loads the 64-byte line from the next level; evicts the LRU way in the set.
5. On a write (with write-back policy): the line is marked *dirty*. It's written back to the next level only on eviction, not on every store.

### Cache-set conflicts in detail

An *S*-set, *W*-way cache of size C has sets addressed by bits `log2(C/W) - 1 : 6` of the physical address (the 6 low bits select within a 64-byte line; the top bits are the tag). Two addresses whose physical addresses differ by a multiple of `C/W` bytes map to the **same set**.

Example with a 32 KB 8-way L1:
- `C/W = 32 KB / 8 = 4 KB`
- Any two addresses 4 KB apart map to the same set.
- An array of 4 KB double-precision floats (512 values) accessed in a loop alongside another such array can cause each access to evict the other's line from L1 — even though L1 is mostly empty.

This is why matrix transpose is surprisingly slow with power-of-two matrix dimensions: the row and column access patterns alias to the same cache sets.

### Write combining and non-temporal stores

Uncacheable writes (e.g., to video memory) are buffered in a small **Write Combining Buffer** (WCB) — typically 8 entries of one cache line each. Writes accumulate until a full cache line is complete, then are flushed as one burst. If you interleave writes to more than 8 distinct lines, you'll flush half-full buffers, cutting write bandwidth drastically. Sequential writes to uncached regions require maintaining spatial locality just to fill the WCB.

For normal cacheable writes, use **non-temporal stores** (`movntps`, `_mm_stream_ps` in C intrinsics) when you're writing a large buffer sequentially and never reading it back — they bypass the cache entirely, saving cache pollution and letting the WCB do its burst work.

### The TLB

Before the cache is even consulted, every virtual address must be translated to a physical address. This translation is cached in the **Translation Lookaside Buffer** (TLB). A typical L1 TLB holds 64 entries (Skylake: 64 4KB entries, 32 2MB entries). Each entry covers one 4 KB page.

A TLB miss costs ~10–50 cycles for a hardware page-table walk plus fills one TLB entry. An access pattern that touches many distinct 4 KB pages (linked lists, pointer-chasing) will exhaust the TLB before exhausting the cache. **Huge pages** (2 MB on x86) reduce TLB pressure by 512×: 64 TLB entries covering 2 MB each span 128 MB, vs 256 MB for 64 × 4 KB entries.

### NUMA: rack awareness for RAM

On a multi-socket server, each CPU socket has its own memory controller. DRAM slots physically attached to socket 0 are *local* to cores on socket 0 (~100 ns latency), but *remote* to cores on socket 1 (~300–400 ns latency, mediated by QPI/Infinity Fabric).

The OS Linux NUMA scheduler attempts to keep a process's memory on the same socket as its CPU thread (`NUMA_BALANCE`), but it's imperfect under migration. Critical real-time paths should use:
- `numactl --localalloc` to force allocations to the local node
- `mbind()`/`set_mempolicy()` to control NUMA policy per allocation
- `MADV_SEQUENTIAL` hints for streaming-access patterns

### Multi-core false sharing (MESI protocol)

Modern CPUs maintain a per-core L1/L2 and a shared L3, kept coherent by the **MESI protocol** (Modified / Exclusive / Shared / Invalid). The unit of coherency is the cache line (64 bytes). Two cores can read the same line simultaneously (Shared state). The moment one core writes, it broadcasts an *invalidation* to all other cores holding that line, forcing them into Invalid state — they must reload it next time.

**False sharing** is when two logically independent variables share a cache line:

```c
struct counters {
    volatile long hits;    // bytes 0–7
    volatile long misses;  // bytes 8–15
};
// If thread 0 increments hits and thread 1 increments misses,
// they're in the same 64-byte cache line → constant cross-core invalidations.
```

Fix: pad to force each counter into its own cache line:

```c
struct counter {
    volatile long value;
    char _pad[64 - sizeof(long)];  // align to 64 bytes
};
```

This is the same technique as LMAX Disruptor's sequence padding — and the same insight, just at the intra-machine level.

### Hardware prefetching

Modern CPUs detect regular access patterns and speculatively issue loads before they're demanded. The hardware prefetcher typically handles:
- **Stride-1** (sequential scan): always prefetched, up to 4 cache lines ahead.
- **Constant stride** (e.g., every 64 bytes, every 128 bytes): detected and prefetched after a few repetitions.
- **Complex or irregular strides**: not prefetched; every access is a cache miss.

For unpredictable patterns, use **software prefetch** hints inserted ahead of the access:

```c
// Prefetch the cache line containing addr for a read; hint = 3 means "keep in L1"
__builtin_prefetch(addr, /* rw */ 0, /* locality */ 3);
```

Insert the prefetch ~200–300 cycles (several iterations) before the corresponding use.

### AoS vs SoA and struct layout

The standard "Array of Structs" layout:

```c
struct Particle { float x, y, z, mass; };
Particle particles[N];  // AoS: x0 y0 z0 m0 | x1 y1 z1 m1 | …
```

If you only ever need `x` and `y` in an inner loop, you load 16 bytes of useful data per 64-byte cache line (25% efficiency). Struct of Arrays solves this:

```c
struct Particles { float x[N], y[N], z[N], mass[N]; };  // SoA
```

Now the `x` loop touches only `x[]`, getting 16 floats per cache line (100% efficiency). SoA is also SIMD-friendly — `x[0]`…`x[15]` fit in one 512-bit AVX-512 register without any gather overhead.

The general principle: **separate hot fields from cold fields**. Put all the data a tight loop actually accesses into a tightly-packed sub-struct, and keep rarely-accessed fields in a separate allocation. This is the same insight as columnar vs row storage in databases.

## Where it breaks

- **Model-specific effects:** Cache sizes, associativity, prefetcher capabilities, NUMA topology all vary by CPU microarchitecture. A layout optimized for Skylake may regress on Zen 4. Profile on the target machine.
- **Indirect structures are harder to fix:** Linked lists, trees, and hash tables with open-addressing on heaps are fundamentally pointer-chasing workloads. TLB and cache effects are real but hard to eliminate without architectural change (arena allocation, pool allocation, ART instead of hash maps).
- **False sharing heuristics:** Padding structs to 64 bytes bloats memory by 8× or more for small objects (one long → 64 bytes). The fix for thread-local counters is often per-thread sharding (like mimalloc's local free list), not padding.
- **NUMA is invisible to most tooling:** `perf stat` shows cache misses but not NUMA remote-access latency. Use `numastat -n`, `pcm-numa`, or Intel VTune's memory access analysis.
- **2026 hardware has gotten kinder:** Deeper LLC hierarchies, larger TLBs, smarter prefetchers, and on-die memory (HBM) have partially reduced the raw impact of some of Drepper's worst-case scenarios. The principles remain; the exact numbers require re-measurement.

## Why it works

### The deep principle: the memory hierarchy is explicit, programmable, and predictable

Drepper's contribution is not the hardware description — that's in any textbook. It's the argument that **these parameters are stable enough to optimize against**. Cache line size is 64 bytes and has been for 20 years. L1/L3 latency ratios are stable across generations. NUMA topology is discoverable via `hwloc` and stable during a process's lifetime.

This makes the memory hierarchy programmable: you can treat it as a set of contracts, not as black-box magic.

### Spatial/temporal locality is the CPU's bet

Every level in the memory hierarchy is a bet: "if you just accessed address A, you'll soon access address A+64 or address A again." Hardware caches are the mechanism that cashes in on that bet. This is not a CPU-specific insight — it's the same assumption that underlies:
- Database buffer pools (if you read page P, you'll likely read it again)
- Kafka consumers (sequential offset access makes HDDs behave like SSDs)
- OS readahead (read 128 KB when 4 KB is requested)
- HTTP/2 push (send the CSS with the HTML, before it's requested)

The programs that break these bets (random access, pointer chasing, OLTP working sets larger than RAM) are exactly the programs that have memory as their bottleneck.

### Cache lines = distributed system write amplification

False sharing is cache-line write amplification: a write to one byte forces a full 64-byte transfer across the interconnect. This is structurally identical to write amplification in LSM trees (a small key-value update triggers a full-page rewrite), or to the "write amplification" of narrow updates in a wide-column database row. The "unit of I/O" is always larger than the "unit of computation," and the programmer must align writes to that larger unit.

### Associativity conflict = hash table bucket overflow

A cache set filling up and evicting useful data is structurally the same problem as a hash table bucket that has too many collisions. The solution in both cases: more ways (larger caches, more buckets), or probing/hashing strategies that distribute load evenly. The **skewed caches** described in some literature use multiple independent hash functions per level — directly analogous to cuckoo hashing.

## Going deeper

1. **Agner Fog's *Optimizing Software in C++* series** (agner.org/optimize) — CPU microarchitecture specifics: branch misprediction costs, instruction latency/throughput tables, SIMD optimization. The practical complement to Drepper's memory focus.
2. **"Basics of Futex"** by Ulrich Drepper (lwn.net/Articles/360699/) — the same author's follow-up on how kernel/user synchronization interacts with the memory hierarchy and why `FUTEX_WAIT` can be fast.
3. **"An Analysis of Linux Scalability to Many Cores"** by Boyd-Wickizer et al. (OSDI 2010) — empirical study of what happens when NUMA effects and false sharing at the kernel level dominate scaling; directly extends Drepper's theory to real system code.
