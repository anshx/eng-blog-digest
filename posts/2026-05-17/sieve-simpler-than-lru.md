---
title: "SIEVE is Simpler than LRU: an Efficient Turn-Key Eviction Algorithm for Web Caches"
source: https://cachemon.github.io/SIEVE-website/blog/2023/12/17/sieve-is-simpler-than-lru/
author: Juncheng Yang, Yazhuo Zhang, Ymir Vigfusson, Yao Yue, Rashmi Vinayak
company: Carnegie Mellon University / Emory University (NSDI '24)
date_posted: 2023-12-17
date_digested: 2026-05-17
---

# SIEVE is Simpler than LRU: an Efficient Turn-Key Eviction Algorithm for Web Caches

## What's new to learn

- **Lazy promotion**: deferring "this object is popular" bookkeeping from the moment of a cache hit to the moment of eviction is both simpler and yields more accurate decisions, because by eviction time you have seen the full access history since last check.
- **Quick demotion**: new objects that are never re-accessed should be expelled fast; a single pass of a "hand" pointer through the insertion-order queue achieves this without any frequency counter.
- **No-promotion concurrency gain**: because SIEVE never moves an object in the data structure on a cache hit — only flips a single bit — concurrent readers don't contend on the list lock, producing >2× throughput over optimized LRU at 16 threads.

## Prerequisites

- **LRU (Least Recently Used)**: on a cache hit, the accessed object moves to the front of a doubly-linked list; on eviction, the tail object is removed. This "promotion" is the key cost SIEVE avoids.
- **CLOCK / Second-chance algorithm**: a circular buffer with a "hand" pointer and a reference bit per slot. On eviction, the hand scans; if bit=1, it resets to 0 and advances; if bit=0, it evicts. LRU approximation without actual movement, but CLOCK re-inserts retained objects at the head, mixing them with fresh arrivals.
- **Power-law (Zipf) access distribution**: in web caches, a small fraction of objects receive most requests. The vast majority of objects are fetched exactly once. Any good cache eviction policy should rapidly discard these one-hit wonders.

## The core idea

LRU's promotion-on-hit is expensive: every cache hit rearranges pointer pairs in a doubly-linked list, which under contention requires holding a global lock or per-bucket lock. But promotion is also unnecessary during the hit — all you really need is a single bit: "was this object accessed since I last considered evicting it?"

SIEVE defers that question entirely. Its data structure is:

1. A **FIFO queue** (insertion order, head = newest, tail = oldest).
2. A **`visited` bit** per cached object (initially 0).
3. A **`hand` pointer** that starts at the tail and moves toward the head.

**On a cache hit**: set `visited = 1`. Nothing else. No lock, no pointer shuffle.

**On a cache miss** (must evict one object):
- Look at the object the `hand` currently points to.
- If `visited == 1`: set `visited = 0`, advance `hand` one step toward the head. Repeat.
- If `visited == 0`: evict this object. Insert the new object at the head with `visited = 0`.
- If `hand` reaches the head without finding a `visited == 0` object, wrap around to the tail.

The net effect: objects that were accessed at least once since the hand last passed them survive; objects that were never accessed since then are evicted in FIFO order of their insertion.

## Mechanics

### Step-by-step trace

Suppose the cache holds objects A, B, C, D, E (D is newest, A is oldest). The hand starts at A.

```
queue (tail→head): A  B  C  D  E
visited:           0  1  0  1  0
hand: ↑ (at A)
```

Eviction request arrives:
- A: `visited=0` → evict A. Insert F at head. Advance hand to B.

```
queue:   B  C  D  E  F
visited: 1  0  1  0  0
hand: ↑ (at B)
```

Next eviction:
- B: `visited=1` → set 0, advance hand to C.
- C: `visited=0` → evict C. Insert G at head.

The key difference from **CLOCK**: CLOCK reinserts retained objects (like B) at the head of the queue, mixing them with freshly arrived objects. SIEVE instead leaves B exactly where it is. This means the "old, popular" objects form a distinct region (tail-side) that the hand has already swept, while "new, untested" objects accumulate at the head. New objects that aren't re-accessed are evicted in a single pass of the hand — **quick demotion**.

### Pseudocode

```python
def evict(cache):
    obj = hand
    while obj.visited:
        obj.visited = False
        obj = obj.prev  # move toward head
        if obj is None:  # wrap
            obj = tail
    evict(obj)
    hand = obj.prev or tail

def access(key):
    if key in cache:
        cache[key].visited = True   # no structural change
        return cache[key]
    new_obj = insert_at_head(key, visited=False)
    if len(cache) > capacity:
        evict(cache)
    return new_obj
```

### Performance numbers (from the NSDI'24 paper)

| Scenario | SIEVE vs. LRU |
|---|---|
| Single-threaded throughput | +16% |
| 16-thread throughput | +2× |
| Miss ratio vs. LRU (average) | lower on majority of traces |
| Miss ratio vs. ARC (best case) | up to −63.2% lower |
| Miss ratio vs. FIFO (average) | −21% |

Evaluated on **1,559 cache traces** from 7 sources: Meta, Wikimedia, X (Twitter), and four others. SIEVE has a lower miss ratio than all 9 state-of-the-art algorithms on **>45%** of traces, while the next-best algorithm leads on only 15%.

The paper won the NSDI '24 Community Award, recognizing that the algorithm, dataset, and code were fully open-sourced.

### Implementation cost

Five production cache libraries (C++, Go, JavaScript, Python, Rust) had their LRU replaced with SIEVE in **fewer than 20 lines of code on average**. The main change is: on a hit, mark a bit instead of moving a node.

## Where it breaks

- **Uniform random workloads**: if every object has equal probability of re-access (no Zipf skew), the `visited` bit gives SIEVE no advantage over CLOCK or LRU; all algorithms perform similarly.
- **Single-bit frequency**: SIEVE cannot distinguish "accessed 2 times" from "accessed 200 times". An object that is moderately popular but sitting at the front of the hand's path can be evicted if the hand sweeps through during a quiet period. ARC and LFU-based policies (like W-TinyLFU) track frequency more richly and may outperform SIEVE on workloads with stable hotsets and frequent hand wraps.
- **Scan resistance**: a sequential scan over a large dataset sets `visited=1` on every object in the cache, locking them all in for one full eviction cycle. LRU, CLOCK-Pro, and ARC have explicit mechanisms for scan detection; SIEVE does not.
- **Object size heterogeneity**: the paper focuses on object-count-based caches, not byte-size-based. Evicting a small unpopular object when the cache needs to make room for one large object requires a size-aware variant.
- **Cache warm-up**: after a restart, all objects start with `visited=0`. The hand sweeps through the entire working set before popular objects accumulate their bit, which can cause higher miss rates during warm-up than steady-state LRU.

## Why it works

**The deeper principle is that promotion is a premature optimization.**

LRU promotes on every hit because it wants to answer "which object was used *most recently*?" But for eviction, you only need to answer "was this object used *at all* since I last considered evicting it?" That is strictly less information — and a single bit captures it perfectly.

Lazy promotion means: when the hand arrives at an object, you have the full picture of whether that object has been useful since the hand's last visit. Eager promotion (LRU) makes this decision with partial information (only the last access time of a single object, not relative access patterns across all objects).

Quick demotion is a direct consequence of the power law: the probability that a brand-new object will be re-accessed before the hand completes one sweep is low. SIEVE bets against new objects surviving their first sweep — and wins on most real-world workloads.

**Connecting to broader systems ideas:**

- **Lazy evaluation in programming languages**: defer computation until the value is actually needed. SIEVE defers "is this object worth keeping?" to the moment the hand arrives, when the decision is cheapest and most accurate.
- **Batching and amortization**: LRU pays a promotion cost on every hit. SIEVE batches all hit-time bookkeeping into a single bit flip and amortizes the real decision to eviction time, which is O(1) amortized per eviction across all hits since the last sweep.
- **Optimistic concurrency**: SIEVE's hit path (flip a bit) needs only an atomic store, not a list lock. This is the same insight behind optimistic read locks in database MVCC: avoid acquiring an exclusive lock on the common path; do the real work only when necessary (eviction, commit).
- **Separation of concerns**: SIEVE separates hit-tracking (cheap, concurrent, one bit) from eviction policy (hand pointer walk). LRU conflates the two — the list *is* both the tracking structure and the eviction order — which is why it needs locking on hits.

The family of algorithms this comes from — lazy promotion + quick demotion — was first formalized in S3-FIFO (a companion paper from the same group), where it is shown to be near-optimal for skewed workloads. SIEVE is the simplest instantiation of those two principles that still beats LRU in practice.

## Going deeper

1. **S3-FIFO paper (SOSP '23)**: the predecessor that formally establishes the lazy-promotion / quick-demotion design principles SIEVE embodies. Shows a three-queue design (small FIFO → main FIFO → ghost FIFO) that is even more miss-ratio-efficient. https://dl.acm.org/doi/10.1145/3600006.3613147
2. **CLOCK-Pro (USENIX ATC '05)**: extends CLOCK with a "hot"/"cold" classification and a ghost list to handle scans and varying object lifetimes. Useful to understand what SIEVE gives up in exchange for simplicity. https://www.usenix.org/legacy/events/usenix05/tech/general/full_papers/jiang/jiang.pdf
3. **Caffeine's W-TinyLFU design post (Ben Manes, High Scalability)**: shows how a frequency sketch (Count-Min) can be layered on top of CLOCK-like eviction to get both quick demotion and frequency-awareness, at the cost of a probabilistic sketch structure. https://highscalability.com/design-of-a-modern-cache/
