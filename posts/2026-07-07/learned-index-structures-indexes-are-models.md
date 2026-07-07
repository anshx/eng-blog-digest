---
title: "The Case for Learned Index Structures"
source: https://arxiv.org/abs/1712.01208
author: Tim Kraska, Alex Beutel, Ed H. Chi, Jeffrey Dean, Neoklis Polyzotis
company: MIT / Google Brain
date_posted: 2017-12-05
date_digested: 2026-07-07
---

# The Case for Learned Index Structures

## What's new to learn

- **An index is a CDF approximator**: Every sorted-key B-Tree is secretly approximating the cumulative distribution function (CDF) of your key distribution — given key k, it predicts *what fraction of keys are ≤ k*, then multiplies by n to get a position. Spelling this out unlocks an entirely new design space.
- **Recursive Model Index (RMI)**: A two-stage model hierarchy — a root model routes a key to one of K specialist sub-models, each sub-model refines the position estimate — compresses 100 M key indexes from gigabytes to kilobytes while reaching 1.5–3× faster lookup.
- **Existence indexes are classifiers**: A Bloom filter answers a binary question (key exists?) that is equivalent to a learned binary classifier, and a trained neural network can achieve the same false-positive rate at a fraction of the bit cost.

## Prerequisites

- How a B-Tree is used as a database index (sorted pages, internal vs. leaf nodes, O(log n) lookup)
- What a cumulative distribution function (CDF) is: F(k) = P(X ≤ k) for a random variable X
- Basic ML vocabulary: model, training, inference, overfitting — no implementation knowledge required

## The core idea

A B-Tree over a sorted column answers one question: *at which position (byte offset or page number) is key k stored?* The answer is simply the **rank** of k — how many keys in the dataset are smaller than k. If you knew the exact CDF of your keys, you could answer this in O(1): `position = round(F(k) * n)`.

A B-Tree doesn't know the CDF analytically, so it builds an implicit **piecewise linear approximation**: each internal node is a split point defining a linear segment; leaf nodes hold the actual data. With n keys and page size B, this approximation has O(n/B) pieces and O(log(n/B)) lookup depth.

The paper's key reframe: *this is a machine learning problem*. You are training a function that maps keys to positions. The B-Tree is one particular model class (axis-aligned piecewise linear). Any more expressive model that better fits your actual key distribution will give a smaller, faster index.

The paper builds the **Recursive Model Index (RMI)**:

1. **Stage 0** — a root model (typically a two-hidden-layer ReLU network with 8–32 neurons per layer) takes a raw key and predicts a coarse position in `[0, n)`.
2. **Stage 1** — the root output is used to pick one of K linear sub-models (one per "bucket"); that sub-model outputs a refined position.
3. **Last-mile error** — every model prediction has a bounded worst-case error ε; a binary search over the window `[pred − ε, pred + ε]` finds the exact record.

The ε bound replaces the full log-depth tree traversal. If the CDF is smooth and monotone — as it is for timestamps, sequential IDs, and many real datasets — a tiny model fits it nearly perfectly and ε shrinks to tens of entries rather than thousands.

## Mechanics

**Training an RMI:**

1. Sort the dataset by key (one-time cost, same as B-Tree construction).
2. Label each key with its rank: `y_i = i / n`.
3. Train the stage-0 model on `(key, rank)` pairs using standard gradient descent.
4. Assign each key to a stage-1 bucket: `bucket = floor(stage0_output * K)`.
5. Train K independent linear regressors, one per bucket, on their assigned keys.
6. Compute the max prediction error ε per bucket; store ε alongside the model.

**Lookup:**
```
pos_estimate = stage1_models[floor(stage0(key) * K)](key)
binary_search(data, key, pos_estimate - ε, pos_estimate + ε)
```
The binary search range is bounded by ε (e.g., 64 entries), so it is O(log ε) — typically 6 comparisons — rather than O(log n).

**Model sizes:** A 100 M key B-Tree index occupies ~2.5 GB. An RMI for the same data can fit in ~100 KB: the stage-0 network has ~200 parameters (floats), and 10,000 stage-1 linear models have two parameters each.

**Learned Bloom filter:** Train a binary classifier on positive keys (label 1) and a sample of the key space (label 0). At query time, keys the classifier labels 0 are *absent* — guaranteed correct for true negatives, probabilistically wrong for true positives. The same false-positive rate requires fewer bits than a standard Bloom filter because the model exploits structural features of the key space (e.g., URLs sharing prefixes, IP ranges) rather than treating keys as opaque hashes.

**For range indexes (B-Tree replacement):** the RMI works directly because predicting the rank of a key gives you the start position for a range scan. For hash indexes, a learned function maps key → bucket and can eliminate collision chains when the data has exploitable structure.

## Where it breaks

**Updates invalidate predictions.** B-Trees handle inserts by splitting pages; an RMI has no equivalent mechanism. After enough inserts the predicted rank diverges from actual rank, widening ε until performance degrades to O(n). The follow-on work ALEX (SIGMOD 2020) addresses this with gapped arrays and dynamic re-training, but write-heavy workloads remain a weakness.

**Adversarial keys break the model.** An attacker who reverse-engineers the stage-0 model can craft keys that cluster in one bucket, blowing up that bucket's ε and forcing linear scans. B-Trees have a fixed O(log n) guarantee regardless of key distribution.

**Distribution shift requires retraining.** If the application bulk-loads 100 M historical rows and then starts inserting present-day rows that follow a different distribution, the model's ε explodes. B-Trees adapt locally; RMIs do not.

**Small datasets see no benefit.** The B-Tree page hierarchy is cheap; the neural network inference (matrix multiplications) has a fixed overhead that only amortizes at 10 M+ keys.

**No tail-latency advantage.** The RMI's worst case is `binary_search(ε)` per lookup; B-Trees occasionally incur deeper traversals. But RMIs consistently run the neural network overhead on *every* lookup, while B-Trees often hit in caches. Latency variance can be *higher* for learned indexes at low load.

## Why it works

The deeper principle is **"any algorithm that fits data to answer a query is a model, and making that model explicit opens the design space."**

A B-Tree implicitly chooses a model class: piecewise constant over pages, then binary search. This model class has a known capacity (O(n/B) pieces) and a known sample complexity (it works for any key distribution). It is the *universal* choice that requires no knowledge of the data.

Learned indexes trade universality for specialization: if you *do* know the data distribution, a more expressive model fits it with fewer parameters and smaller prediction error. The tradeoff surface is:

```
universality ←→ specialization
update speed  ←→ query speed
parameter count ←→ prediction accuracy
```

This is the same tradeoff that appears in:
- **Roaring Bitmaps** (this archive, 2026-06-30): choosing sorted arrays vs. bitsets vs. RLE based on measured density, rather than one fixed encoding for all cases.
- **Adaptive Radix Trees** (this archive, 2026-07-01): node types that adapt to actual child-count density; Node4 vs. Node256 are different model classes for the same "map key fragment to child pointer" function.
- **W-TinyLFU** (this archive, 2026-07-04): a Count-Min Sketch as a learned model of access frequency, used to admit or reject cache candidates.
- **Compilers' loop unrolling and vectorization**: the compiler models the loop's iteration count and data layout to choose an execution strategy.

The meta-principle: **hand-crafted data structures are implicit models trained by algorithm designers on worst-case inputs.** When you can replace "worst-case" with "this actual dataset," the model can be simpler, smaller, and faster — at the cost of generality and update agility.

Learned indexes also expose an important *negative* lesson: the regime where this is safe is read-heavy, static (or slow-changing) datasets with smooth key distributions. These happen to be common in analytics and OLAP, uncommon in OLTP. Knowing *why* B-Trees are general-purpose is as useful as knowing when you can escape them.

## Going deeper

1. **ALEX: An Updatable Adaptive Learned Index** (Ding et al., SIGMOD 2020, https://arxiv.org/abs/1905.08898) — adds gapped arrays and cost-model–driven node splitting so the RMI handles inserts. Reaches within 2× of a B-Tree on write-heavy workloads while maintaining 3–4× speed on reads.
2. **PGM-Index** (Ferragina and Vinciguerra, PVLDB 2020, https://pgm.di.unipi.it/) — provably space-optimal: the index size equals the minimum number of linear segments needed to approximate the CDF within error ε, and it supports range queries and inserts in O(log n / log(ε)) amortized.
3. **Are Learned Indexes a Replacement for B-Trees?** (Kipf et al., aiDM Workshop @ SIGMOD 2019, https://arxiv.org/abs/1907.05452) — adversarial evaluation showing that data sets without smooth CDFs (random UUIDs, adversarial keys) cause RMI performance to degrade toward or below B-Trees, providing the boundary conditions under which the technique applies.
