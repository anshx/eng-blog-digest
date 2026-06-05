---
title: "HyperLogLog in Practice: Algorithmic Engineering of a State of the Art Cardinality Estimation Algorithm"
source: https://research.google/pubs/hyperloglog-in-practice-algorithmic-engineering-of-a-state-of-the-art-cardinality-estimation-algorithm/
author: Stefan Heule, Marc Nunkesser, Alexander Hall
company: Google
date_posted: 2013-03-18
date_digested: 2026-06-05
---

# HyperLogLog in Practice: Algorithmic Engineering of a State of the Art Cardinality Estimation Algorithm

## What's new to learn

1. **Probabilistic bit-pattern sampling as a cardinality estimator** — The maximum number of leading zeros observed across a stream of hashed values grows as log₂(n) for n distinct elements. Inverting this gives an O(log log n)-space estimator for cardinality.

2. **Stochastic averaging with harmonic mean** — Splitting the hash space into m independent "register" buckets and combining their geometric estimates via harmonic mean (not arithmetic) reduces variance by √m and suppresses the heavy right tail that makes the arithmetic mean unreliable.

3. **Mergeable synopses** — A HyperLogLog sketch is closed under set union via element-wise register maximum, making it a commutative monoid homomorphism that distributes across nodes without coordination and enables exact `COUNT(DISTINCT)` over partitioned data.

## Prerequisites

- Hash functions: assume uniformly distributed outputs, inputs mapped to integers
- Basic probability: expected value, variance, law of large numbers
- Why approximate answers sometimes suffice (the cost of exact counting in distributed systems)
- Integer binary representation (leading zero count, bit position)

## The core idea

The fundamental problem: count distinct elements in a data stream using constant memory. An exact hash set costs O(n) space—untenable for a billion-row web log. HyperLogLog solves this with a probabilistic shortcut.

**The leading-zeros insight.** Hash each element to a uniformly random 64-bit integer. The probability that such a hash has at least k leading zeros is exactly 1/2^k. In a stream of n distinct elements, the expected maximum leading-zero run observed concentrates sharply around log₂(n)—because the probability that at least one of n independent hashes achieves k or more leading zeros is:

```
P(max leading zeros ≥ k) = 1 − (1 − 1/2^k)^n ≈ 1 − e^(−n/2^k)
```

This crosses 50% at n ≈ 2^k. So if you observe a maximum leading-zero run of k, your best estimate for cardinality is 2^k. This single observation uses O(log log n) bits of memory—you only store the count, not the hash itself.

**The noise problem.** A single estimate is too noisy—log₂(n) steps in integer increments, and one lucky hash with 20 leading zeros would cause a 4× overestimate. The fix: use the first b bits of each hash to route each element into one of m = 2^b independent registers, each tracking its own maximum leading-zero count. You now have m independent estimators, each covering a 1/m slice of the space. Averaging m independent estimates reduces variance by √m (central limit theorem). With m = 2^14 = 16,384 registers at 6 bits each, the structure fits in 12KB and achieves 0.81% relative error for any n, from 1 to 10^18.

**The harmonic mean twist.** The "average" of independent estimates 2^M[0], 2^M[1], ..., 2^M[m-1] (geometric estimates from each register) has a heavy right tail: occasional registers with very high values dominate the arithmetic mean. HyperLogLog uses the harmonic mean instead—the reciprocal of the average of reciprocals. This is the only power mean below the arithmetic mean, and it is insensitive to outliers. The HyperLogLog paper proves this choice, with a calibration constant α_m, makes the estimator unbiased.

## Mechanics

**Data structure:** m registers M[0..m-1], each initialized to 0. b = log₂(m) bits from each hash select the register; the remaining 64-b bits feed the leading-zero computation.

**Add(element):**
```
h = hash64(element)               # uniform 64-bit hash
j = h >> (64 - b)                 # top b bits → register index
w = h & ((1 << (64 - b)) - 1)    # remaining bits
ρ = count_leading_zeros(w) + 1   # ρ = position of leftmost 1-bit
M[j] = max(M[j], ρ)
```

Each register tracks the maximum ρ it has ever seen for elements hashed to its bucket. That maximum is a sufficient statistic for the bucket's cardinality estimate.

**Count():**
```
Z = 1.0 / sum(2^(-M[j]) for j in 0..m-1)   # inverse harmonic sum
E = α_m * m² * Z                             # raw estimate
```

Where `α_m ≈ 0.7213 / (1 + 1.079/m)` is the calibration constant derived analytically.

Then apply range corrections:

- **Small-range correction** (n ≪ m): When E < 5m/2, many registers are still 0. Let V = count of zero registers. Return `m * ln(m/V)` instead — this is "linear counting," which is near-exact when most registers are empty.

- **Large-range correction** (32-bit hashes only): When E > 2^32/30, hash collisions bias estimates low. Return `-2^32 * ln(1 - E/2^32)`. With 64-bit hashes this threshold is unreachable in practice.

**Merge(sketch_A, sketch_B):**
```
M_result[j] = max(M_A[j], M_B[j])   for all j
```

This is set union: an element's ρ contribution ends up in whichever sketch it was added to, and taking the element-wise max combines them correctly. The result estimates COUNT_DISTINCT(A ∪ B).

**HLL++ improvements from the Google paper (what makes this paper interesting):**

*Sparse representation.* For small cardinalities (n ≪ m), most registers stay at 0. Instead of allocating all 12KB upfront, store only (register_index, ρ) pairs in a sorted list using 4-byte encodings. Memory scales with actual distinct count until it exceeds a threshold, then converts to dense form. This gives near-zero memory overhead for sketches that never grow large.

*Empirical bias correction.* The raw HyperLogLog estimate is systematically biased upward for cardinalities in the range 1×m to 5×m. Google experimentally measured the bias at 200 cardinality checkpoints for each b value by averaging estimates over 10,000 random hash seeds. These biases are stored in a lookup table and subtracted from the raw estimate. This alone cuts the maximum error by a factor of 2–3× in the formerly problematic region.

*64-bit hashing throughout.* Using 64-bit hashes eliminates the large-range correction and makes the algorithm correct for cardinalities up to ~10^14—enough for any practical dataset.

## Where it breaks

**Error floor.** The irreducible standard error is ~1.04/√m. At m = 2^14, that's 0.81%—no amount of data or clever implementation can push it lower without using more registers. For applications that need sub-0.1% error, HyperLogLog is the wrong structure.

**Small-cardinality regime.** For n < 10, the linear counting fallback or the empirical bias correction can't fully compensate, and results are noisy. Use an exact hash set for single-digit cardinalities.

**Cannot answer membership.** HyperLogLog only answers "how many distinct?" not "did I see X?" You cannot reconstruct the elements from the sketch. (Use a Bloom filter for membership.)

**No intersection or difference.** `merge(A, B)` gives COUNT_DISTINCT(A ∪ B). There is no analog of sketch subtraction for set difference. Intersection requires inclusion-exclusion: |A ∩ B| ≈ |A| + |B| - |A ∪ B|, which combines three noisy estimates, amplifying error as more sets are involved.

**Merge is one-way.** The element-wise max is monotone—you can only add elements to a register, never lower it. A sketch that has seen element X cannot "forget" X. To count distinct elements over a sliding time window, you need separate sketches per time bucket.

**Hash function matters.** HyperLogLog assumes a universal hash function with uniformly distributed outputs. CRC and polynomial hashes are not uniform and inflate errors. MurmurHash3 and xxHash work well.

## Why it works

The leading-zero insight has a clean information-theoretic basis. A 64-bit uniform hash of n distinct elements gives n independent samples from a geometric distribution: the probability of drawing a value with leading-zero run ≥ k is 1/2^k. The maximum of n such samples concentrates sharply—by extreme value theory for geometric variables, the maximum is the value of k where the expected count of items exceeding k transitions from "less than 1" to "more than 1": that's k = log₂(n). You are essentially reading off log₂(n) from the empirical extreme of a known distribution.

The harmonic mean is not arbitrary. Each register's estimate 2^M[j] is right-skewed (log-normally distributed around the true n/m). For a log-normal random variable X, the arithmetic mean E[X] systematically overshoots the true median. The harmonic mean 1/E[1/X] corrects for exactly this bias: it is the maximum-likelihood estimator for the median of a log-normal distribution. The derivation of α_m in the original Flajolet et al. paper is the exact integral that makes the harmonic-mean-based estimator unbiased.

**The deep principle: HyperLogLog is a mergeable synopsis.**

A synopsis (or sketch) is a fixed-size summary of a dataset that supports specific queries approximately. A synopsis is *mergeable* if the synopsis of the union of two datasets can be computed from the synopses of the individual datasets without touching the original data:

```
synopsis(A ∪ B) = merge(synopsis(A), synopsis(B))
```

HyperLogLog's `merge` is element-wise max. This makes it a homomorphism over set union — the same structural property as:

| Structure        | Query               | Merge operation   |
|------------------|---------------------|-------------------|
| HyperLogLog      | COUNT(DISTINCT)     | element-wise max  |
| Bloom filter     | membership test     | bitwise OR        |
| Count-Min Sketch | frequency query     | element-wise sum  |
| MinHash          | Jaccard similarity  | element-wise min  |
| T-Digest / KLL   | quantile query      | sorted merge      |

Each synopsis defines mergeability based on the underlying set lattice operation (union, intersection). Mergeability is precisely what makes HyperLogLog work in distributed systems like ClickHouse, Spark, or BigQuery: each worker computes a local sketch over its shard, the coordinator merges all sketches in O(m × num_workers) time, and runs Count() once. No inter-node coordination is needed during ingestion—a property the ClickHouse parallel aggregation architecture relies on directly. The 9,000-core GROUP BY post in this archive exploits the same monoid homomorphism; HyperLogLog is a specific instance of it applied to cardinality.

This also connects to the FoundationDB simulation testing post: deterministic replay is possible only when aggregate functions are deterministic and commutative over their inputs. HyperLogLog's commutativity (max is commutative) makes it replay-safe.

## Going deeper

1. **Original HyperLogLog paper** — Flajolet, Fusy, Gandouet, Meunier (2007): the mathematical derivation of why harmonic mean yields a 30% accuracy improvement over the earlier LogLog algorithm, and the proof that α_m is the unique unbiasing constant. The integral over the Poisson-limit distribution is the core of the analysis. Available at http://algo.inria.fr/flajolet/Publications/FlFuGaMe07.pdf

2. **"Mergeable Summaries"** — Agarwal, Cormode, Huang, Phillips, Wei, Yi (PODS 2012): the general theory of which aggregates admit bounded-space mergeable synopses, and lower bounds on their size. Proves that HyperLogLog's cardinality estimation is essentially optimal — no structure using fewer bits can merge correctly and maintain bounded error.

3. **Redis HyperLogLog source** (src/hyperloglog.c in the Redis codebase): ~1,200 lines of annotated C implementing HLL++ with a sparse/dense representation switch, a 6-bit register packing scheme using 8 registers per 48-bit word, and the full bias correction table. The most accessible production implementation of the ideas in this paper.
