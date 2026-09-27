---
title: "Locality Sensitive Hashing (LSH): The Illustrated Guide"
source: https://www.pinecone.io/learn/series/faiss/locality-sensitive-hashing/
author: James Briggs
company: Pinecone
date_posted: 2022
date_digested: 2026-09-27
---

# Locality Sensitive Hashing (LSH): The Illustrated Guide

## What's new to learn

- **Locality-preserving hashing**: A hash function family where Pr[h(x) = h(y)] ≈ similarity(x, y) — the deliberate reversal of normal hashing's collision-avoidance goal.
- **MinHash as a Jaccard probability estimator**: The minimum value of a randomly permuted set is an unbiased estimator of Jaccard similarity; a single random draw from the union stands in for an exact intersection-over-union computation.
- **AND-OR band amplification**: Splitting a signature into b bands of r rows creates an S-shaped probability curve that concentrates "find" decisions near a target similarity threshold, a general technique for tuning false-positive/false-negative rates without changing the underlying hash family.

## Prerequisites

- **Jaccard similarity**: |A ∩ B| / |A ∪ B|. Ranges from 0 (disjoint) to 1 (identical).
- **Independent events**: Pr[A and B] = Pr[A] × Pr[B] when A, B are independent.
- **Why approximate nearest neighbors**: The HNSW post explains the O(M log n) graph-traversal alternative; LSH is the probabilistic-hashing alternative that predates it.
- **Hashing basics**: A hash function maps an item to a bucket; a good general-purpose hash function tries hard to avoid collisions for distinct items.

## The core idea

Normal hash functions are deliberately locality-*destroying*: SHA-256 produces wildly different digests for "hello" and "helo". That property is a feature for cryptography and hash maps — you want items scattered uniformly across buckets.

LSH inverts the goal. A locality-*sensitive* hash function satisfies:

> Pr[h(x) = h(y)] is high when similarity(x, y) is high, and low when similarity is low.

If you have such a function, approximate nearest-neighbor (ANN) search becomes trivial: hash the query, retrieve all items in the same bucket, and those are your candidates. The question is: which hash families have this property?

For set similarity (Jaccard), the answer is **MinHash**. The algorithm processes three stages:

1. **Shingling** — convert raw items (documents, vectors) into sets.
2. **MinHash signatures** — replace large sets with compact fixed-length signatures that preserve pairwise Jaccard similarity.
3. **Band amplification (banding)** — partition signatures into bands; pairs sharing a full band become candidates.

The deeper insight: the S-curve produced by banding is not specific to MinHash. AND-OR amplification is a general probabilistic technique used wherever a binary test succeeds with probability p and you need to concentrate the success probability around a target threshold.

## Mechanics

### Stage 1: Shingling

Represent a document (or any item) as a set of overlapping k-grams. For k=3, the string "abcde" becomes {abc, bcd, cde}. Two documents with high word overlap will have high Jaccard similarity over their shingle sets.

### Stage 2: MinHash

Choose m independent hash functions h₁, h₂, …, hₘ. For a set S, the MinHash signature is:

```
sig(S) = [min_{x∈S} h₁(x), min_{x∈S} h₂(x), …, min_{x∈S} hₘ(x)]
```

**Why Pr[min_{x∈S_A} h(x) = min_{x∈S_B} h(x)] = Jaccard(A, B):**

A uniform random hash function h is equivalent to choosing a random permutation of the universe. Under a random permutation, which element lands at position 1 (the minimum position)?  It is equally likely to be any element of A∪B. The minimum of A under h equals the minimum of B under h if and only if the element at position 1 is also in A∩B — i.e., it is in both sets. The probability of that event is |A∩B| / |A∪B| = Jaccard(A, B).

So each of the m components of the signature is an independent coin flip that comes up "heads" (equal in both signatures) with probability exactly equal to the Jaccard similarity s. The fraction of signature positions that match across two items is an unbiased estimator of s, with variance O(1/m) — same as any Monte Carlo estimator.

**Practical implementation note:** Rather than storing m distinct hash functions, you can use a single hash function parameterized by (a, b, p) — a linear hash of the form h(x) = (ax + b) mod p — with different (a, b) per row.

### Stage 3: Band amplification (banding)

With a 128-length signature and exact signature matching as the candidate criterion, two items with s=0.7 Jaccard are found with probability 0.7^128 ≈ 0 — almost never. That is too strict.

Partition the signature into b bands of r rows each (b × r = m). Declare two items *candidates* if any band has all r rows equal:

```
Pr[candidate] = 1 - (1 - sʳ)ᵇ
```

- **Within one band**: Pr[all r rows match] = sʳ (AND of r independent events).
- **Across b bands**: Pr[at least one band fully matches] = 1 - (1-sʳ)ᵇ (OR of b independent events).

This is an S-curve. For b=20 bands, r=5 rows (m=100 total):
- s=0.3: Pr[candidate] ≈ 0.047 (almost never — correct, these are dissimilar)
- s=0.5: Pr[candidate] ≈ 0.47 (boundary zone)
- s=0.8: Pr[candidate] ≈ 0.999 (almost always — correct, these are similar)

The inflection point of the curve sits near s* = (1/b)^(1/r). By adjusting r and b (keeping b×r constant), you can shift the inflection point and steepen the curve:
- Larger r: steeper curve, lower inflection
- Larger b: steeper curve, higher inflection

In practice, choose s* to be your target similarity threshold, then solve for r and b.

### Query execution

```
1. Shingle the query document → set Q
2. Compute sig(Q) = [min_{x∈Q} h₁(x), …, min_{x∈Q} hₘ(x)]
3. For each of the b bands:
     hash that band's r rows into a bucket table
     if any stored item is in the same bucket → add to candidates
4. Re-rank candidates by exact Jaccard similarity
5. Return top-k
```

Steps 1–3 are O(m × |Q|) time. Step 4 is O(|candidates| × |Q|). For a well-tuned system the candidate set is a tiny fraction of the corpus.

## Where it breaks

**High-dimensional dense vectors (Euclidean / cosine similarity):** Jaccard MinHash is only valid for set (Jaccard) similarity. For dense float vectors you need *random projection LSH*: project onto a random hyperplane and hash based on which side the point falls on. Collision probability = 1 - arccos(cosine_similarity)/π. This requires completely different hash families and does not compose with shingling.

**Memory**: Storing m hash functions per item costs O(m × N) memory for N items. For large corpora this can dwarf the raw data. FAISS solves this with Product Quantization on top of LSH, but that adds complexity.

**Parameter sensitivity**: The S-curve shape is very sensitive to the (r, b) split. A poor choice — e.g., b=1, r=100 — degrades to exact signature matching (no benefit). There is no automatic tuning; you must estimate the Jaccard distribution of your data to pick good parameters.

**Fundamental recall gap**: LSH misses *some* true neighbors (false negatives) by design. HNSW's graph-based approach achieves recall > 99% at comparable precision; LSH typically achieves 80–95% recall at practical parameters. When recall matters more than memory, HNSW wins.

**Not updatable without rebuilding**: Adding a new document requires computing its signature and inserting into all b band tables — O(m) per item. That is cheap per item, but deletes require re-hashing or tombstone logic; there is no "remove from bucket" primitive.

## Why it works

**The unifying principle: hash functions as Monte Carlo estimators.** MinHash belongs to a family of ideas where random hash functions estimate a hard-to-compute property:

| Technique | Hash to | Estimates |
|---|---|---|
| Bloom filter | k bits | set membership |
| HyperLogLog | max leading zeros | cardinality |
| Count-Min Sketch | k row×column | frequency |
| MinHash | minimum hash value | Jaccard similarity |
| SimHash | sign of dot product | cosine similarity |

The pattern is always: replace an exact, expensive computation over all elements with a cheap random probe that is correct in expectation. Error shrinks as O(1/√m) by the central limit theorem.

**AND-OR amplification** is the probability-theory equivalent of error-correcting codes. A single bit test that succeeds with probability p is unreliable. Take r independent bits AND-ed together: Pr[all succeed] = pʳ — this suppresses low-p pairs. Take b such groups OR-ed: Pr[at least one] = 1-(1-pʳ)ᵇ — this recovers high-p pairs. The same structure appears in:
- Quorum systems (require k-of-n servers to agree)
- Reed-Solomon codes (any k of k+m shards suffice)
- Redundant disk writes (RAID-1: write to 2, read from either)

**LSH vs HNSW in one sentence**: LSH is a *pre-index* (hash into buckets offline, query instantly) with recall limited by probability; HNSW is a *graph* (traverse greedily at query time) with recall bounded by graph connectivity. LSH uses O(m) space per item and O(m) query time regardless of corpus size; HNSW uses O(M × N) edges and O(M log n) query time. When data fits in RAM and recall must be > 99%, HNSW dominates. When data is on disk or recall can be 85–95%, LSH is competitive.

## Going deeper

1. **Indyk & Motwani, 1998** — "Approximate Nearest Neighbors: Towards Removing the Curse of Dimensionality" (STOC). The original paper introducing LSH for Euclidean spaces. The proof that random projection satisfies the locality-sensitive property is the core theorem.

2. **Pinecone's companion post on random projection LSH** — https://www.pinecone.io/learn/series/faiss/locality-sensitive-hashing-random-projection/ — extends these ideas to dense float vectors and cosine similarity, showing how the same banding machinery applies with a different hash family.

3. **Google's SimHash for near-duplicate web detection** — Charikar 2002, "Similarity Estimation Techniques from Rounding Algorithms." SimHash is a streaming O(1)-update LSH for cosine similarity used by Google to deduplicate the web crawl. The core trick (sign of random dot product = 1-bit LSH for cosine) is a beautiful instantiation of the random projection idea.
