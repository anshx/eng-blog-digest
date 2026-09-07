---
title: "DiskANN: Fast Accurate Billion-point Nearest Neighbor Search on a Single Node"
source: https://www.microsoft.com/en-us/research/publication/diskann-fast-accurate-billion-point-nearest-neighbor-search-on-a-single-node/
author: Suhas Jayaram Subramanya, Devvrit, Rohan Kadekodi, Ravishankar Krishnaswamy, Harsha Simhadri
company: Microsoft Research
date_posted: 2019-12-08
date_digested: 2026-09-07
---

# DiskANN: Fast Accurate Billion-point Nearest Neighbor Search on a Single Node

## What's new to learn

- **Vamana graph construction**: A single-layer directed proximity graph built by alternating greedy beam search with a degree-bounding `RobustPrune` operation. The `α > 1` relaxation parameter explicitly preserves long-range "highway" edges that strict MRNG pruning would remove — giving the graph global navigability without a hierarchy.

- **Product quantization (PQ) as a two-level memory tier**: Decompose each vector into M sub-vectors, train a k-means codebook per sub-vector, and store only codebook indices (32 bytes instead of 512). PQ distances live in RAM; full-precision distances live on SSD. This is a lossy compression scheme whose error is bounded by the codebook distortion, not random noise.

- **Page-aligned co-location**: Pack each node's full-precision vector *and* its neighbor list into one 4 KB SSD sector. Every graph traversal step costs exactly one SSD read — no separate index vs. data files, no two I/Os to get what you need. The layout bets that the unit of access and the unit of storage should be the same.

## Prerequisites

- What approximate nearest neighbor (ANN) search is: given a query vector, find the k dataset vectors with smallest distance, accepting ~5% miss rate to gain 100-1000× speed.
- HNSW (hierarchical navigable small world) — the in-memory graph-based ANN the archive covered on 2026-05-21. DiskANN is the disk-resident answer to the same problem.
- SSD I/O model: NVMe SSDs have ~100 µs latency and can serve many parallel reads; sequential vs. random access gap is <10× (vs. >1000× for spinning disks).
- Product quantization is explained below; no prior knowledge needed.

## The core idea

HNSW stores the entire index in RAM. For a billion 128-dimensional float32 vectors, that's ~500 GB. Almost nobody has that. DiskANN's insight is that ANN graph search only visits O(L) nodes out of billions per query, so most of the index is cold on any given query. Cold data belongs on SSD, not RAM.

The trick is making each visit cheap: when the search algorithm lands on a node and needs to evaluate "which of its neighbors should I explore next?", it only needs the neighbor list and the node's vector. DiskANN stores both in one 4 KB page. One NVMe read (~100 µs), and you have everything. Multiply by the beam width W and the search depth, and a query issues on the order of 100–300 SSD reads total — achieving <5 ms latency even on commodity hardware.

To avoid reading full-precision vectors for every candidate considered (most of which will be discarded), DiskANN keeps PQ-compressed vectors in RAM. PQ distances are approximate but good enough to rank candidates and decide whom to visit next. Only after search finishes are the top-k candidates loaded from SSD and re-ranked by exact distance.

The graph itself is built by the Vamana algorithm, which constructs a *flat* (single-layer) directed proximity graph with bounded out-degree R. Flat is critical for disk: HNSW's hierarchical design means a handful of top-layer hub nodes are accessed on virtually every query — they become memory-or-bust hot spots. A flat graph distributes access evenly across all nodes.

## Mechanics

### Step 1: Product quantization compression

Given n vectors of dimension d, split each into M sub-vectors of dimension d/M. Train a k-means codebook of size K (typically K=256, so each sub-vector maps to 1 byte). Now each original vector is represented as M bytes (e.g., M=32 → 32 bytes vs. 512 bytes for float32 d=128). The codebook is shared (typically a few hundred KB), so PQ-compressed codes for 1B vectors fit in ~30 GB — feasible in RAM.

At query time, precompute a distance table: for each of M sub-codebooks and each of the K centroids, compute the distance from the query sub-vector to that centroid. A PQ lookup is then M table reads + M additions — cheap enough to do millions of times per second.

### Step 2: Vamana graph construction

```
Initialize: connect each node to R random neighbors
Pass 1 (α = 1.0):
  For each node p (shuffled):
    candidates = BeamSearch(graph, medoid, p, L=75)
    p.neighbors = RobustPrune(p, candidates, R=64, α=1.0)
    For each q in p.neighbors:
      if |q.neighbors| > R: q.neighbors = RobustPrune(q, q.neighbors ∪ {p}, R, α)
      else: q.neighbors.add(p)
Pass 2 (α = 1.2):
  Same loop, now α > 1 — relaxed pruning preserves longer edges
```

**RobustPrune(p, candidates, R, α):**
```
Sort candidates by d(p, candidate) ascending
result = []
while candidates not empty and |result| < R:
    p* = argmin distance from p         # select closest remaining
    result.append(p*)
    # remove any candidate p' that p* already "covers" well enough
    remove from candidates all p' where: α · d(p*, p') ≤ d(p, p')
return result
```

The removal condition `α · d(p*, p') ≤ d(p, p')` means: "p' is close enough to p* that we don't need a direct edge from p to p' — we can reach p' from p* with at most α× overhead." When α=1 this is the strict MRNG condition (no edge is redundant). When α=1.2, you tolerate 20% detour before removing an edge — so more long-range edges survive, giving the graph "expressways" between distant clusters.

### Step 3: Disk layout

```
Sector size: 4096 bytes (one NVMe page)
Per node layout within one sector:
  [n_neighbors: 4 bytes]
  [neighbor_ids: R × 4 bytes]
  [full_precision_vector: d × 4 bytes]

Multiple nodes may share a sector if n_neighbors is small, or a
large node spans multiple contiguous sectors. Either way, each
node maps to exactly one I/O request.
```

The graph file is a flat array of these fixed-size records. Node i lives at offset `i * node_size`, rounded to 4 KB alignment. Given a node ID, you compute the disk offset in O(1) and issue one `pread()`.

### Step 4: Query algorithm

```
Input: query q, beam width W, search list size L, num results k
Precompute: PQ distance table for q against all M sub-codebooks

frontier = priority queue, ordered by PQ distance
  (initialized with the graph's medoid node)
visited = set()
results = []

while frontier not empty and |visited| < max_iterations:
    # select W unvisited candidates with smallest PQ distance
    batch = frontier.pop_W_unvisited()
    visited.update(batch)

    # issue W parallel async SSD reads (one per node)
    records = async_read_disk(batch)

    for record in records:
        exact_d = distance(q, record.full_vector)
        results.update((record.id, exact_d))   # exact distance now known
        for neighbor_id in record.neighbor_list:
            if neighbor_id not in visited:
                pq_d = pq_distance(q, neighbor_id)  # cheap, from RAM
                frontier.push((neighbor_id, pq_d))

return top_k(results, k)   # already have exact distances
```

The beam width W controls I/O parallelism: issuing W reads simultaneously keeps the NVMe queue deep, maximizing device throughput. L is like HNSW's `ef_search` — larger L visits more nodes, increasing recall at the cost of more I/Os.

### Parameters and results

| Parameter | Typical value | Effect |
|-----------|--------------|--------|
| R (max out-degree) | 64–128 | Graph quality vs. node size |
| L (build search list) | 75–100 | Index quality (build time) |
| α | 1.2 | Long-range edge retention |
| W (beam width) | 4–16 | I/O parallelism at query |
| L (search) | 50–200 | Recall-latency tradeoff |

On SIFT1B (1 billion 128-dim float32 vectors):
- **Memory**: 64 GB RAM (vs. ~500 GB for HNSW in-memory)
- **QPS**: >5,000 queries/second on a 16-core machine
- **Latency**: <3 ms mean, <5 ms P99
- **Recall@1**: 95%+
- **Index size**: ~400 GB on SSD

HNSW at equivalent recall on 1B points requires the full index in RAM, which costs 10–50× more hardware per node. DiskANN achieves 5–10× more vectors per node at the same latency/recall.

## Where it breaks

**Build time**: Constructing a Vamana index over 1B points takes many hours, sometimes days, on a 16-core machine. This makes dynamic updates expensive — DiskANN is best for mostly-static workloads. (FreshDiskANN extends this with streaming inserts at some recall cost.)

**PQ recall ceiling**: Quantization error means some neighbors that should rank in the top-k beam are scored wrong and dropped before disk reads happen. Increasing M helps but raises RAM cost. In practice, recall@10 of 95%+ is achievable, but 99%+ requires much larger L (more I/Os).

**SSD dependence**: NVMe SSDs (100–200 µs, high queue depth) are required. On SATA SSDs or spinning disks, the 4–5 ms per I/O assumption breaks and latency becomes seconds. DiskANN is not a general storage-conscious algorithm; it assumes specific SSD characteristics.

**Cold-start caching**: High-centrality nodes (near-medoids) are visited by almost every query. Without an explicit hot-node cache in RAM, the SSD sees repeated reads for the same sectors. DiskANN's implementation caches the top ~5% most-accessed nodes in RAM to handle this.

**Filtered ANN**: When queries add metadata predicates ("only return vectors where category = shoes"), simple graph traversal visits many filtered-out nodes before finding k valid candidates. Extensions like FilteredDiskANN pre-label edges with reachable filters, but this is a research-active problem not solved by the base algorithm.

**High dimensionality**: PQ compression quality degrades for d > 768 because codebook distortion increases. Recent embedding models (3072-dim) require many more sub-quantizers (M ≫ 64), raising RAM cost.

## Why it works

**The deeper principle: match your data structure's unit of access to your storage hierarchy's unit of transfer.**

B-trees pack many keys into one disk block so each seek delivers a fanout's worth of index. Parquet groups column values into row groups that fit in one HDFS block scan. CPU caches transfer data in 64-byte cache lines, not bytes. DiskANN does the same thing at the NVMe layer: its 4 KB "node record" is exactly one SSD page, so the algorithm pays the minimum possible I/O cost per graph edge explored.

HNSW's design fought this principle: a tiny hierarchy of hub nodes created a de facto hot-node bottleneck. Vamana's flat graph distributes load evenly, so no single node becomes an I/O choke point.

The PQ-in-RAM / full-vectors-on-SSD split is the same "approximate guidance, precise confirmation" pattern that appears everywhere:
- **Bloom filter + disk read**: check a probabilistic filter (cheap, in memory) before doing the expensive lookup
- **Column-store predicate pushdown**: scan compressed/encoded column data to filter rows, decompress only survivors
- **CPU cache tag arrays**: the tag array (small, in L1) answers "is this address cached?" before touching the data array (large, in L2/L3)

In each case, a lossy or compact representation in fast memory decides whether to pay for the expensive operation against slow memory. The "miss penalty" — fetching from disk when PQ said this was promising — is the cost of the approximation.

The α > 1 relaxation in RobustPrune maps onto a principle from expander graph theory: maintaining some redundant long-range edges is what makes greedy routing on graphs polynomial rather than exponential. Without them, the graph can develop local minima where greedy descent stalls far from the query. α lets you explicitly control how many such "shortcut" edges to preserve — it's the Vamana analogue of HNSW's layer structure, but implemented as a parameter rather than a separate data structure.

## Going deeper

1. **DiskANN paper (NeurIPS 2019)**: Subramanya et al., "DiskANN: Fast Accurate Billion-point Nearest Neighbor Search on a Single Node" — the original algorithm, proofs that Vamana produces navigable graphs, and benchmarks against HNSW/IVF. Available via Microsoft Research.

2. **ParlayANN (arXiv 2023)**: Carnegie Mellon's parallel Vamana implementation. Explains how to construct the index in parallel (the incremental node-update loop is embarrassingly parallel with careful concurrent neighbor updates), and benchmarks on the 2023 NeurIPS ANN competition.

3. **HNSW paper (2016-05-21 in this archive)**: Malkov & Yashunin, "Efficient and robust approximate nearest neighbor search using Hierarchical Navigable Small World graphs." Read this alongside DiskANN to understand exactly why hierarchical vs. flat matters for disk access patterns, and how the two algorithms make opposite bets on graph topology.
