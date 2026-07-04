---
title: "Design of a Modern Cache: W-TinyLFU"
source: https://highscalability.com/design-of-a-modern-cache/
author: Ben Manes
company: open-source (Caffeine / Google)
date_posted: 2016-01-25
date_digested: 2026-07-04
---

# Design of a Modern Cache: W-TinyLFU

## What's new to learn

1. **Admission filter as a first-class design axis**: Treating "which entry to admit to the main cache" as a separate decision from "which entry to evict" — a frequency sketch acts as a gate that protects hot entries from being displaced by one-hit wonders.
2. **Count-Min Sketch for compact frequency estimation**: A fixed-width array of 4-bit counters indexed by multiple hash functions estimates per-key access frequency in O(1) time and O(capacity) total space, with frequency aging via periodic halving to weight recent accesses over stale history.
3. **Hill climbing as a self-tuning control loop**: A gradient-following algorithm that periodically measures hit-rate and nudges the window/main cache size split in the direction of improvement — the cache configures itself to the optimal recency/frequency balance for the live workload.

## Prerequisites

- LRU and LFU cache basics (why LRU fails on scans, why LFU fails on cold-start)
- What a Bloom filter is: a bit-array indexed by multiple hash functions, trading exact answers for space
- SLRU (Segmented LRU): a two-segment LRU that promotes entries to a "protected" segment on second hit

## The core idea

Every cache replacement policy is a bet on one of two access patterns: *recency* (you'll access the most recently used entry again soon) or *frequency* (you'll access the most frequently accessed entry again). LRU bets on recency; LFU bets on frequency. Both fail on mixed workloads.

The failure mode matters:
- **LRU under a scan**: A process sequentially reads 200 GB of data. LRU evicts every hot entry to make room for scan pages that will never be touched again.
- **LFU under a burst**: A new viral video gets its first 1,000 views. LFU treats it as cold (frequency = 0 at eviction time) and keeps evicting it in favor of older entries with high historic counts.

W-TinyLFU fixes both by separating the cache into two regions:
- A **window** (small, ~1% of capacity): a pure-LRU region for newly admitted entries. Entries here can build up their access count before facing the frequency gate.
- A **main cache** (large, ~99%): a Segmented LRU divided into a Protected segment (hot entries) and a Probationary segment (newly admitted, at risk of eviction).

When the window fills, a **TinyLFU admission test** decides whether the window's eviction candidate (the least recently used new entry) displaces the main cache's probationary victim. It keeps whichever has higher estimated frequency. Cold entries never graduate from window to main; the main cache is reserved for proven hot entries.

The result: scan resistance from the window (scans fill the window and churn among themselves, leaving the main cache untouched) and recency-burst protection (a new hot entry accumulates frequency in the window before being judged).

## Mechanics

### Cache structure

```
Total capacity: N entries
  ├── Window cache:    ~1% of N  [LRU, recency-focused]
  │     └── on eviction → TinyLFU admission test
  └── Main cache:      ~99% of N [SLRU, frequency-focused]
        ├── Protected: ~80% of main  [hot entries, second-chance]
        └── Probationary: ~20% of main  [newly admitted, eviction candidates]
```

Access path:
1. **Hit in window**: promote to front of window LRU.
2. **Hit in probationary**: promote to head of Protected segment.
3. **Hit in protected**: promote to head of Protected segment.
4. **Miss**: insert into window.

### Count-Min Sketch

The frequency sketch stores access counts without a per-entry counter. It uses d hash functions and a w×d matrix of 4-bit counters:

```
estimate(key):
    return min over all d rows of: counters[row][hash_d(key) % w]

increment(key):
    for each row r:
        counters[r][hash_r(key) % w] += 1  (capped at 15)
```

With d=4 hash functions and w = 10 × capacity, the sketch uses ~8 bytes per cache entry — independent of key size. Estimates overcount (hash collisions inflate counts) but never undercount: `estimate(k) ≥ true_frequency(k)` always.

**Frequency aging (decay)**: Without decay, a one-time thundering-herd event would forever inflate the frequency counts of entries that are no longer hot. W-TinyLFU ages the sketch by halving every counter whenever the total count crosses a threshold (typically `10 × capacity`):

```
if total_increments > 10 * capacity:
    for all counters: counter >>= 1   # right-shift = integer divide by 2
    total_increments >>= 1
```

This is exponential moving average applied to a frequency sketch: old counts lose half their weight every `10 × capacity` accesses, while recent accesses carry full weight.

### Admission test

When the window LRU evicts a candidate (`windowVictim`) and the probationary segment's tail is `mainVictim`:

```
if sketch.estimate(windowVictim) > sketch.estimate(mainVictim):
    admit windowVictim → head of probationary
    evict mainVictim
else:
    evict windowVictim   # "one-hit wonder" rejected
```

The window victim wins ties (bias toward recency avoids thrashing on equal-frequency items).

### Hill climbing for adaptive window sizing

Real workloads oscillate between recency-heavy (streaming, sequential scan) and frequency-heavy (hot keys, Zipf distribution). The optimal window/main split is workload-dependent and changes over time.

Caffeine uses a **hill-climbing optimizer** that adjusts the split every `10 × capacity` evictions:

```python
# pseudocode
measure hit_rate_now
delta = hit_rate_now - hit_rate_previous
if delta > 0:
    keep adjusting in the same direction (grow or shrink window)
else:
    reverse direction
clamp(window_size, 1%, 50%)   # never let window exceed half the cache
```

If the workload is recency-dominated (hit rate improves when window grows): the optimizer expands the window up to ~50% of capacity — behaving like pure LRU.

If the workload is frequency-dominated (hit rate improves when window shrinks): the optimizer collapses the window to ~1% of capacity — behaving like pure LFU.

### Concurrency design (bonus)

Every cache access would normally require a lock to update the LRU chain or frequency sketch. Caffeine avoids this with **striped ring buffers**: each thread writes access events to a thread-local ring buffer (no contention). A background maintenance thread periodically drains buffers, applies sketch increments, and reorders the LRU chains. On the fast path, a cache hit is a single CAS (compare-and-swap) to mark the ring buffer slot — no lock, no sketch write.

## Where it breaks

**Ghost frequency inflates admission**: Hash collisions mean an entry that was never accessed can inherit counts from other keys sharing its hash cells. A cold entry might show estimated frequency of 3 even if it's truly cold. This can cause a window victim to falsely displace a probationary entry that's genuinely hot.

**Adversarial workloads can pollute the sketch**: A deliberate scan using keys chosen to collide with hot entries can inflate frequency estimates for unrelated hot items and force them out. (Mitigations exist: randomize hash seeds per cache instance.)

**Hill climbing has a short-term memory**: The optimizer measures hit rate over the last `10 × N` evictions. A workload that switches between recency-heavy and frequency-heavy at intervals shorter than this window will cause the optimizer to perpetually lag behind the optimal split.

**Frequency aging is batch, not continuous**: Halving all counters at once means all keys age simultaneously. An entry accessed 5 seconds ago and one accessed 5 months ago get the same halving schedule. True temporal weighting requires per-entry timestamps, which the sketch doesn't store.

**Small caches skip sketching entirely**: When the cache is less than 50% full, Caffeine disables frequency sketching and falls back to pure LRU. This is a pragmatic shortcut (eviction is rare when the cache has room) but it means the admission policy is not uniform across fill levels.

## Why it works

W-TinyLFU approximates **Bélády's OPT replacement algorithm** — the theoretically optimal policy that evicts whichever entry will be accessed furthest in the future.

OPT is equivalent to "evict the entry with the lowest future access frequency." W-TinyLFU's admission test (keep whichever of window-victim or main-victim has higher sketch frequency) is exactly this, using *past* frequency as a proxy for *future* frequency — the same ergodic assumption that makes predictive caching work at all: if an entry was frequently accessed in the recent past, it's likely to be frequently accessed in the near future.

The Count-Min Sketch is an instance of the **same probabilistic space-trading trick as HyperLogLog and Bloom filters**: replace an exact but expensive computation (exact frequency counting = O(N) counters) with an approximate but O(1)-time, fixed-space probabilistic computation (sketch = O(w×d) counters). The core pattern: hash the key into a fixed-size array, use the collision to your advantage (min across rows gives a low-noise estimate; OR across bits gives set membership).

The frequency aging via periodic halving is **exponential smoothing over a counter sketch**: the same technique as EWMA applied to metrics monitoring, or as the periodic clock halving in Linux's VM page scanner. Recency-weighting + frequency-counting in a fixed-width counter = the best approximation of "how recently was this entry very popular?"

The window + admission test design is the same **probationary period** pattern found throughout systems:
- LIRS: HIR (hot-in-recency) and LIR (low-in-recency) segments with a non-resident history list
- ARC: T1 (recency) and T2 (frequency) with ghost entries tracking recently evicted keys
- 2Q: An initial FIFO queue feeds a main LRU; entries graduate only on second hit
- Linux page cache: PG_referenced bit distinguishes single-use from multi-use pages

All of these encode the same insight: **don't trust a single access to predict future hotness — make entries prove their value before committing capacity to them.**

## Going deeper

1. **TinyLFU original paper**: Einziger & Friedman, "TinyLFU: A Highly Efficient Cache Admission Policy" (2014/2017, ACM Transactions on Storage). Derives the formal admission-filter framework and shows that Count-Min Sketch is an optimal implementation. https://arxiv.org/abs/1512.00727

2. **Ristretto design document**: The Go implementation of W-TinyLFU used in DGraph, TiKV, and others — explains how they extended W-TinyLFU with cost-aware eviction (each entry has a cost, the cache respects total-cost budgets instead of entry count). https://github.com/dgraph-io/ristretto/blob/main/DESIGN.md

3. **Part 2 — Design of a Modern Cache (Part Deux)**: Ben Manes' follow-up covers the concurrency model in depth: striped ring buffers, write buffers, draining strategy, and how to measure cache performance without synthetic benchmarks. https://highscalability.com/design-of-a-modern-cachepart-deux/
