---
title: "Hierarchical Navigable Small Worlds (HNSW)"
source: https://www.pinecone.io/learn/series/faiss/hnsw/
author: James Briggs
company: Pinecone
date_posted: 2022
date_digested: 2026-05-21
---

# Hierarchical Navigable Small Worlds (HNSW)

## What's new to learn

1. **Proximity-graph search**: instead of comparing a query against every stored vector, build a graph where nearby vectors are pre-linked, then navigate that graph greedily at query time — turning O(n) brute-force into O(M log n) expected hops.

2. **The skip-list decomposition of metric space**: assign each node to a randomly chosen maximum layer (exponentially distributed), giving a hierarchy where the top layer has few nodes with long-range connections and the bottom layer has all nodes with short-range connections — the same trick skip lists use on a sorted line, now applied to an arbitrary embedding space.

3. **The diversity-aware neighbor selection heuristic**: when wiring a new node into the graph, reject candidates that are geometrically "shadowed" by an already-selected neighbor, forcing the M chosen links to span different directions and keeping the graph navigable across cluster boundaries.

## Prerequisites

- **Vector embeddings**: how text, images, or audio can be represented as points in R^d where semantic similarity correlates with geometric proximity.
- **The curse of dimensionality**: why exact nearest-neighbor tree structures (k-d trees, ball trees) degrade to O(n) in high dimensions — their partitions stop pruning branches.
- **Brute-force ANN**: the baseline of scanning all n vectors and keeping the k smallest distances — O(n × d) time, 100% recall, unusably slow at millions of vectors.
- **Basic graph concepts**: nodes, directed/undirected edges, adjacency list.
- **Skip lists** (optional but illuminating for the key insight): a probabilistic sorted linked list with multiple levels, achieving O(log n) search by "skipping" over elements via sparser express lanes.

## The core idea

Given a collection of n high-dimensional vectors and a query q, HNSW finds the k approximate nearest neighbors in O(M log n) expected time by maintaining a **layered proximity graph**:

- **Layer 0** contains every vector. Each is connected to its ≤ Mmax0 nearest neighbors in the graph (typically 2M nodes).
- **Layer 1** contains a random ~1/M fraction of all vectors, each connected to ≤ M neighbors *at that layer*.
- **Layer 2** contains ~1/M² of all vectors, and so on.

Search starts at the entry point in the topmost layer. At each layer, the algorithm moves greedily toward q (taking big, coarse hops with few choices), then drops one layer and repeats (taking finer, more precise hops with more choices). Because each layer has exponentially fewer nodes, the total work is logarithmic.

The skip-list analogy is exact: a skip list runs this same multi-level descent on a sorted 1-D number line. HNSW generalizes it to "closest in metric space."

## Mechanics

### Layer assignment

When inserting element q, first sample its maximum layer l:

```
l = floor(-ln(uniform(0, 1)) * mL)    where mL = 1 / ln(M)
```

With M = 16 (a common default):

| Layer | Probability of reaching it |
|-------|---------------------------|
| 0     | 100% (all nodes)           |
| 1     | ~6.25% (= 1/16)            |
| 2     | ~0.39% (= 1/256)           |
| k     | (1/M)^k                    |

The exponential decay mirrors skip-list level assignment (each list level halves the node count). The factor mL = 1/ln(M) normalizes so that the expected maximum layer grows as log_M(n).

### Insertion algorithm

**Phase 1 — descend to insertion layer (layers L_top down to l+1):**
At each layer above l, run a greedy search with ef = 1 (track only the single best candidate). This finds the closest entry point into the graph at layer l, quickly.

**Phase 2 — build connections (layers l down to 0):**
At each of these layers, run a **beam search** with ef = efConstruction candidates: maintain a min-heap of the efConstruction closest nodes seen so far and a visited set. Expand each candidate's neighbors, updating the heap if a closer node is found.

From the efConstruction candidates collected at each layer, select M (or Mmax0 at layer 0) neighbors using the **select-neighbors heuristic** and add bidirectional edges.

**Pruning:** if adding the new edges pushes any existing node above its Mmax limit, run select-neighbors on that node's edge list to prune back to Mmax.

### Select-neighbors heuristic (Algorithm 4)

The key to graph quality. Naïvely taking the M closest candidates often selects many vectors from the same cluster, creating a locally-dense but globally-disconnected graph.

The heuristic maintains the graph's navigability by enforcing a **relative neighborhood** condition:

```python
def select_neighbors(q, candidates, M):
    selected = []
    for e in sorted(candidates, key=lambda x: dist(q, x)):
        # Add e only if no already-selected neighbor r is "between" q and e
        if all(dist(q, e) < dist(r, e) for r in selected):
            selected.append(e)
        if len(selected) == M:
            break
    return selected
```

The condition `dist(q, e) < dist(r, e)` for all selected r means: "e is closer to q than it is to any already-chosen neighbor." If this fails, there exists some r that is *between* q and e in the metric space — the edge q→e would be redundant because you can reach e via r more efficiently.

Geometrically: the condition rejects a candidate if it "hides behind" an already-selected neighbor from q's perspective. The result is that the M chosen neighbors fan out in maximally different directions from q, ensuring the graph remains reachable from any approach angle.

### Search algorithm

```
1. entry_point = global_entry_point (maintained by the index)
2. for layer = L_top down to 1:
       entry_point = greedy_search(q, entry_point, ef=1, layer)
       # returns the single closest node at this layer
3. at layer 0:
       candidates = beam_search(q, entry_point, ef=efSearch, layer=0)
       # returns efSearch closest nodes found
4. return top-k from candidates
```

The greedy descent in step 2 is the "express lane" phase — fast but imprecise. The beam search at layer 0 in step 3 is the fine-grained precision phase. efSearch is the primary recall-vs-latency knob at query time.

### Key parameters

| Parameter | Controls | Typical values |
|-----------|----------|---------------|
| M | Max edges per node (higher → better recall, more memory) | 4–64 |
| Mmax0 | Max edges at layer 0 (usually set to 2M) | 8–128 |
| efConstruction | Beam width at build time (higher → better graph, slower build) | 100–500 |
| efSearch | Beam width at query time (higher → better recall, slower query) | 10–500 |

Memory overhead is approximately `n × M × 4 bytes × average_layers`. With M = 16 and float32 vectors of dimension d = 768 (BERT-size), the graph connectivity adds ~128 bytes per vector on top of the 3072 bytes for the vector itself — a ~4% overhead.

## Where it breaks

**Build cost is sequential and slow.** Each insertion depends on the current graph state, making construction hard to parallelize. Building an index over 100M vectors takes hours on a single machine with standard settings.

**Deletes and updates degrade the graph.** HNSW has no mechanism to cleanly remove a node's edges. Qdrant and other implementations handle this with soft-delete (mark the node as deleted, skip during search) and periodic re-indexing. High delete rates eventually degrade recall.

**Memory scales with M.** At M = 64 for very high recall, the graph overhead doubles. For billion-scale indexes this is often prohibitive, driving interest in disk-resident alternatives like DiskANN.

**Clustered distributions confuse the traversal.** If most queries land in one dense cluster, the greedy descent at upper layers may route everything to the same entry point into that cluster, missing candidates from nearby clusters. Tuning efSearch upward helps but at latency cost.

**Predicate-filtered search is not native.** ANN queries with metadata filters ("find nearest neighbors where price < 100") require post-filtering (which wastes work on non-qualifying candidates) or pre-filtering with a modified graph (which requires separate HNSW indexes per filter, combinatorially expensive). ACORN (Weaviate, 2024) introduces an in-graph filtering extension but adds complexity.

**efSearch/recall is a cliff, not a ramp.** There is often a threshold below which reducing efSearch causes a sharp drop in recall. This makes it hard to tune conservatively — operators need to profile on representative workloads.

## Why it works

The deeper principle unifies two classic results:

**Kleinberg's routing theorem (2000):** In a graph where nodes have long-range connections distributed as a power law over distance, a greedy routing algorithm finds any target in O(log² n) hops with high probability. This is why the internet, social networks, and neural routing all exhibit "small world" behavior: a small fraction of long-range links makes the whole structure navigable.

**Skip list hierarchy (Pugh, 1990):** A sorted linked list with multiple express levels, where level k+1 has each element with probability 1/M of level k, supports O(log n) search by descending from coarse to fine. The key insight: you trade the O(n) work of scanning all elements for O(log n) work by maintaining O(n log n) total edges across all levels.

**HNSW = skip lists generalized to metric space.** A skip list's total order (e.g., integers) is replaced by "proximity in embedding space." The express-level structure (exponentially sparser as you go up) provides the Kleinberg-style long-range connections that make the graph navigable. The greedy descent is exactly how skip list search works — start at the top, move toward the target, drop a level when stuck.

The single-sentence deeper principle: **HNSW achieves O(M log n) nearest-neighbor search by applying skip list multi-level shortcutting to proximity graphs in arbitrary metric space — the same structural insight that makes skip lists O(log n) on a sorted line also makes vector search O(M log n) in R^d.**

The select-neighbors heuristic is a second, independent insight: it computes a **greedy relative neighborhood graph** (RNG) approximation. An RNG edge between u and v exists only if no other node w satisfies dist(u, w) < dist(u, v) AND dist(v, w) < dist(u, v). RNGs are known to have excellent navigability properties (they form connected graphs for most point sets). The heuristic approximates this with a one-pass greedy algorithm, giving the navigability benefits at construction cost O(M × efConstruction) per node rather than O(n²) for the exact RNG.

## Going deeper

1. **Original paper** — Malkov & Yashunin, "Efficient and Robust Approximate Nearest Neighbor Search Using Hierarchical Navigable Small World Graphs," IEEE TPAMI 2018 (arXiv:1603.09320). Algorithms 1–4 in the paper are the canonical reference for insertion, search, and the heuristic described above.

2. **DiskANN / Vamana** — Jayaram et al., "DiskANN: Fast Accurate Billion-point Nearest Neighbor Search on a Single Node," NeurIPS 2019. Builds a single flat Vamana graph that fits on disk, avoiding HNSW's per-layer memory cost. A useful contrast for understanding the tradeoffs of hierarchical vs. flat ANN graphs.

3. **ACORN** — "Filters Matter: Incorporating Filters in HNSW for High-Accuracy Filtered ANN Search," Weaviate 2024 blog + paper. Extends HNSW to handle predicate-filtered search without materializing separate indexes per filter value — directly relevant to production RAG systems that combine semantic search with metadata constraints.
