---
title: "Space/Time Trade-offs in Hash Coding with Allowable Errors"
source: https://dl.acm.org/doi/10.1145/362686.362692
author: Burton H. Bloom
company: Computer Usage Company
date_posted: 1970-07-01
date_digested: 2026-08-13
---

# Space/Time Trade-offs in Hash Coding with Allowable Errors

## What's new to learn

1. **Randomized set membership** — a bit array of size *m* plus *k* independent hash functions can answer "is x in S?" with zero false negatives and a tunable false positive rate ε, using only ~1.44·log₂(1/ε) bits per element regardless of key size.

2. **Optimal hash-function count** — the value of *k* that minimizes false positives for a given m and n has a closed-form solution: k* = (m/n)·ln 2 ≈ 0.693·(m/n), derivable by minimizing the false positive rate expression with calculus.

3. **Probabilistic-hashing-over-fixed-budget** — the pattern of hashing items into multiple positions in a fixed-size structure to answer approximate queries is the unifying principle behind Bloom filters, HyperLogLog (cardinality), Count-Min Sketch (frequency), SFQ (fair scheduling), and Cuckoo Filters — all instances of the same sketch-based approximation trade-off.

## Prerequisites

- **Hash functions**: what it means for a hash function to be "independent" and "uniform" over a range — i.e., that h(x) produces each value in [0, m-1] with equal probability, and that h₁(x) and h₂(x) are uncorrelated.
- **Basic probability**: the complement rule, the Bernoulli trial model, and the approximation (1 − 1/m)^n ≈ e^(−n/m) for large m.
- **LSM trees or Cassandra** is helpful context for the most important application, but not required.

## The core idea

Imagine you have a set S of n elements (say, a list of known-bad URLs), and for every incoming query x you want to answer "is x in S?" The naïve approach stores the set explicitly — but if n is large and keys are long strings or 128-bit identifiers, that costs O(n · |key|) bytes.

Bloom's insight (1970): **you don't need to store the keys; you only need to store evidence that a key was inserted**. Specifically, maintain an *m*-bit array B (initially all zeros) and *k* independent hash functions h₁, …, hₖ, each mapping a key to one position in [0, m−1].

**Insert(x)**: Set B[hᵢ(x)] = 1 for each i = 1, …, k.

**Query(x)**: Return TRUE if and only if B[hᵢ(x)] = 1 for ALL i = 1, …, k.

**Delete**: NOT supported. (Bits are shared — you can't un-set a bit without potentially erasing evidence of another key.)

The guarantee:
- **No false negatives**: if x was inserted, every bit hᵢ(x) was set to 1, so query returns TRUE.
- **False positives possible**: if x was NOT inserted, its k hash positions might all happen to be 1 already (set by other insertions). That's the only error mode, and its probability is tunable.

You could now explain this to a colleague at a whiteboard: the filter is a lossy summary — it remembers *which positions were touched*, not *what touched them*. A query succeeds only if every one of x's positions was touched by something; that might have been x itself (true positive) or accidental collisions (false positive).

## Mechanics

### False positive rate derivation

After inserting n elements, each touching k positions:

**Step 1.** Probability a specific bit is still 0 after one hash-function application:

    P(bit stays 0 | one insert) = (1 − 1/m)

After k hash applications per insert, and n inserts:

    P(bit is 0 after n inserts) = (1 − 1/m)^(kn) ≈ e^(−kn/m)

**Step 2.** A false positive on query x requires ALL k of x's bit positions to be 1. These events are approximately independent (for large m), so:

    FPR ≈ (1 − e^(−kn/m))^k

This is the fundamental formula. Every Bloom filter design lives on this surface.

**Example**: m = 10 bits/element, n elements, k = 7 hash functions:
    FPR ≈ (1 − e^(−7·1/10))^7 ≈ (1 − e^(−0.7))^7 ≈ (0.503)^7 ≈ 0.0082 ≈ 0.8%

### Optimal k

For fixed m and n (bits per element = m/n = b), minimize FPR over k.

Let p = e^(−kn/m) = probability a bit is zero. Then FPR = (1−p)^k.

Minimize ln[(1−p)^k] = k·ln(1−p) over k, subject to k·n/m = −ln(p).

Setting d/dk [k·ln(1−p)] = 0 (using the chain rule through the constraint):

    k* = (m/n) · ln 2 ≈ 0.693 · (m/n)

Substituting back, at the optimum p = 1/2, i.e., exactly **half the bits are set** — a surprising fact. The optimal Bloom filter has 50% bit-density.

The minimum FPR becomes:

    FPR* = (1/2)^(k*) = 2^(−(m/n)·ln 2)

Solving for bits-per-element needed to achieve target FPR ε:

    m/n = log₂(1/ε) / ln 2 ≈ 1.44 · log₂(1/ε)

| Target FPR  | Bits/element | Example: 1M items |
|-------------|--------------|-------------------|
| 1%          | ~9.6 bits    | ~1.2 MB           |
| 0.1%        | ~14.4 bits   | ~1.8 MB           |
| 0.01%       | ~19.2 bits   | ~2.4 MB           |

This is remarkable: **the space cost depends only on the desired accuracy, not on the key size**. A Bloom filter storing URLs (each ~100 bytes) takes the same space as one storing 256-bit hashes, for the same FPR. The key is thrown away; only the shadow of its hash positions remains.

Compare to alternatives:
- **Perfect hash set** (store compressed keys): O(n · log |U|) bits for universe U
- **Sorted array + binary search**: O(n · |key|) bytes, O(log n) lookup
- **Bloom filter**: ~9.6·n bits for 1% FPR, O(k) lookup (≈ O(1) in practice)

### Algorithm walk-through for 1% FPR on n = 1,000,000 items

Pick: m = 9,585,059 bits (~1.14 MB), k = 7 hash functions.

Insert "example.com":
  - h₁("example.com") = 3,021,417 → set bit 3,021,417
  - h₂("example.com") = 7,893,214 → set bit 7,893,214
  - ... (5 more)

Query "unknown.net":
  - Check h₁("unknown.net") = 1,024,832 → bit is 0 → return FALSE immediately (definitive non-member)

Query "malware.xyz":
  - h₁ through h₇ all happen to be set by other insertions → return TRUE (false positive, prob ≈ 1%)

## Where it breaks

1. **No deletion**: The standard Bloom filter cannot remove elements. Clearing bit hᵢ(x) might erase evidence for another key y with hᵢ(y) = hᵢ(x). The fix — **counting Bloom filters** (replace each bit with a 4-bit counter, increment on insert, decrement on delete) — costs 4× space and overflows if an element is inserted >15 times.

2. **FPR degrades under overload**: The formula assumes n ≤ design capacity. Filling a Bloom filter past its target occupancy causes FPR to blow up rapidly. An m/n = 10 filter designed for 1% FPR reaches ~15% FPR at 2n elements and ~35% at 3n.

3. **Cache behavior**: Each query touches k random bit positions across the m-bit array. For m > L1 cache size, each probe is likely a cache miss — k cache misses per query. For k = 7 and m = 10 MB, that's 7 DRAM fetches per lookup. This is why Blocked Bloom Filters (confine all k bits to one cache line) trade slightly higher FPR for k → 1 cache misses.

4. **Only exact-match membership**: Bloom filters cannot answer range queries, enumerate members, count distinct members, or tell you *which* element matched. They answer exactly one question: "have I seen this exact key before?"

5. **Shared-bit collisions create dependencies**: The approximation (1 − 1/m)^(kn) treats bit-setting events as independent, but they are not — the same bit can be set by multiple elements. This independence assumption is accurate for large m, but the formula underestimates FPR slightly for small m.

## Why it works

The Bloom filter is the purest instance of the **sketch paradigm**: represent a large dataset as a compact, lossy summary that preserves the ability to answer specific queries with bounded error.

The key insight is that k independent hash functions act as k independent "witnesses." For a true member, all k witnesses agree (all positions set). For a non-member, the probability that all k independent witnesses agree by accident is (probability one agrees)^k — exponentially falling in k.

This pattern appears throughout the archive:

| Structure          | Fixed budget | Query answered     | Error model          |
|--------------------|-------------|--------------------|----------------------|
| Bloom filter       | m bits      | Membership         | False positive rate ε |
| HyperLogLog        | m registers | Cardinality        | ±1.04/√m relative    |
| Count-Min Sketch   | w×d cells   | Frequency          | Additive error n/w   |
| SFQ (stochastic fair queuing) | B buckets | Flow fairness | Collision probability |

All four structures share the same recipe: **hash to multiple positions → aggregate → answer query**. The aggregation function changes (OR for Bloom, MAX for HLL, MIN for CMS, round-robin for SFQ), but the underlying trick — using hash functions as a randomized projection into a compact space — is identical.

Mathematically, these are all **linear sketches** — functions F(data) such that F(A ∪ B) can be computed from F(A) and F(B) without touching the original data. This merge-ability is what makes them ideal for distributed systems (you can compute local sketches and merge them).

The deeper principle: **hashing is the universal approximate encoder**. A perfect encoder would compress data to its entropy; a hash encoder compresses it to a fixed budget at the cost of collision probability. The Bloom filter, HyperLogLog, Count-Min Sketch, and cuckoo filters are all calibrated trade-offs on the accuracy-vs-space curve of randomized hashing.

## Going deeper

1. **"Network Applications of Bloom Filters: A Survey"** — Broder & Mitzenmacher (Internet Mathematics, 2004). The definitive survey covering 30+ applications in networking and databases: IP routing, spam detection, synchronization, collaborative caching. https://www.eecs.harvard.edu/~michaelm/postscripts/im2005b.pdf

2. **"Cuckoo Filter: Practically Better Than Bloom"** — Fan, Andersen, Kaminsky, Mitzenmacher (CoNEXT 2014). Shows how storing fingerprints in a compact cuckoo hash table supports deletion, uses 30% less space than Bloom at the same FPR, and has better cache behavior (2 cache lines vs. k). https://www.cs.cmu.edu/~dga/papers/cuckoo-conext2014.pdf

3. **"Xor Filters: Faster and Smaller Than Bloom and Cuckoo Filters"** — Graf & Lemire (VLDB 2020). The current state-of-the-art for static sets: construction using 3-wise independent XOR maps achieves ~1.08·log₂(1/ε) + 2.08 bits per element — near the information-theoretic lower bound of log₂(1/ε) bits. https://arxiv.org/abs/1912.08258
