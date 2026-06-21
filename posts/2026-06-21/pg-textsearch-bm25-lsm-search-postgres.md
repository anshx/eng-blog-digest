---
title: "pg_textsearch 1.0: How We Built a BM25 Search Engine on Postgres Pages"
source: https://www.tigerdata.com/blog/pg-textsearch-bm25-full-text-search-postgres
author: Tiger Data Engineering (Todd "TJ" Green et al.)
company: Tiger Data (formerly Timescale)
date_posted: 2026-03-31
date_digested: 2026-06-21
---

# pg_textsearch 1.0: How We Built a BM25 Search Engine on Postgres Pages

## What's new to learn

1. **BM25 ranking** — The industry-standard full-text scoring formula, a refinement of TF-IDF that adds a *saturation* term (a document stops gaining score after it repeats a query term many times) and a *length normalization* factor (a word appearing in a short document is more telling than the same word buried in a long one). These two corrections make BM25 the formula behind Elasticsearch, Solr, Lucene, and now Postgres full-text search.

2. **Block-Max WAND** — A top-k retrieval algorithm that pre-computes the maximum BM25 score achievable by any document in each 128-doc posting block, then uses those upper bounds as a skip condition. If the sum of per-block ceilings across all query terms cannot beat the current k-th best score, the entire block is skipped — turning an O(N) scan into a fraction of the work.

3. **Inverted index as an LSM tree** — A search index built on immutable segments using the same write-optimized append → flush → compact cycle as RocksDB/LevelDB. New documents go to a memtable; the memtable flushes to immutable disk segments; a background worker merges segments level by level. Every Lucene-family search engine (including this Postgres extension) is, under the hood, an LSM tree keyed on search terms.

## Prerequisites

- **Inverted index basics**: the core data structure of search — a mapping from each unique term to a sorted list of document IDs (the *posting list*) that contain it.
- **TF-IDF intuition**: term frequency (TF) rewards documents that mention a word many times; inverse document frequency (IDF) penalizes words common across the whole corpus. Their product approximates relevance.
- **LSM tree write path**: in-memory buffer (memtable) → immutable on-disk levels that merge over time. Covered in the [algorithms-behind-modern-storage-systems](../2026-06-10/algorithms-behind-modern-storage-systems.md) entry.
- **Delta encoding for sorted integers**: if you store the *differences* between consecutive sorted values instead of absolute values, the integers become small and compress tightly — same principle as Gorilla's delta-of-delta for timestamps.

## The core idea

Postgres's native full-text search (`tsvector` / `ts_rank`) scores results with a simpler formula that lacks saturation and length normalization. That omission means a 10,000-word legal brief mentioning "contract" 50 times will outscore a crisp 200-word summary mentioning it once — even though the summary's single mention is far more diagnostic. Elasticsearch and Solr avoid this with BM25; pg_textsearch brings BM25 to Postgres by implementing a complete search engine as a Postgres access method, inside standard Postgres pages, with no external libraries and full WAL/replication compatibility.

The result is a layered system: a memtable receives writes (appending to pages in the index relation itself), periodically flushes to immutable segments, and a background worker compacts small segments into larger ones. Query time uses Block-Max WAND to skip irrelevant posting blocks, so top-k results arrive without scoring every document.

## Mechanics

### The inverted index on disk

Each segment is a self-contained, immutable file containing two parts:

**Term dictionary**: a sorted array of (offset → string) pairs stored in a string pool, binary-searchable in O(log *n*). The DSA (Dynamic Shared Area) hash table used during indexing is serialized into this sorted array on segment write.

**Posting blocks**: for each term, the posting list is divided into fixed blocks of up to 128 documents. Every block stores:
- **Delta-encoded doc IDs** — the differences between consecutive sorted document IDs. If a term appears in docs 100, 103, 115, the stored values are 100, 3, 12 — small integers that bitpack tightly.
- **Packed term frequencies** — the count of how many times the term appears in each document, stored in the minimum number of bits needed for the block's maximum value.
- **Quantized fieldnorms** — a one-byte approximation of each document's total token length, used at query time for length normalization.

A separate **skip index** stores one entry per posting block with the maximum BM25 score that any document in that block can contribute to any single-term query. These upper bounds are the fuel for Block-Max WAND.

### BM25 formula

For a query *q* and document *d*, the BM25 score is:

```
score(q, d) = Σ_{t ∈ q}  IDF(t) ×  f(t,d) × (k1 + 1)
                                    ─────────────────────────────────
                                    f(t,d) + k1 × (1 − b + b × |d|/avgdl)
```

Where:
- `f(t, d)` = raw term frequency of *t* in *d*
- `|d|` = document length in tokens; `avgdl` = corpus-wide mean
- `k1 ≈ 1.2`: controls TF saturation — higher values let frequency matter more
- `b ≈ 0.75`: controls length normalization — 0 = off, 1 = full normalization
- `IDF(t) = log((N − df + 0.5) / (df + 0.5) + 1)` where `N` = total docs, `df` = docs containing *t*

The key shape of the denominator: as `f(t,d)` grows, the fraction asymptotically approaches `(k1+1)/k1` — a ceiling. Repeated occurrences have diminishing returns. The `b` term divides by a blend of actual and average length so a short document and a long document with the same raw frequency get comparable scores.

### Write path (LSM-like)

1. **Memtable (L0)**: new documents are tokenized and reduced to a `bm25vector` (term-ID + TF pairs). The record `(ctid, doc_length, bm25vector)` is appended to the memtable's tail page, which is a chain of standard Postgres pages starting from the index metapage. All mutations use `GenericXLog` so standbys can replay without loading the extension.

2. **Segment flush**: when the memtable exceeds a configured size threshold, it is flushed as a new immutable segment on disk.

3. **Compaction**: a background worker monitors per-level segment counts. When a level accumulates 8 or more segments, it merges them into a single segment in the next level — the same exponential-growth compaction strategy as LevelDB. The merged segment is written atomically; displaced pages are moved to a tombstone chain and freed only after `hot_standby_feedback` confirms no standby is reading them.

### Query path: Block-Max WAND

For a multi-term query with *n* terms:

1. Open the posting list for each query term and initialize a min-heap of (next-doc-ID, term-index) pairs.
2. **WAND pivot selection**: find the smallest doc-ID threshold *D* such that the sum of maximum per-block scores (from the skip index) for all terms at or beyond *D* exceeds the k-th best score found so far.
3. If such a *D* exists: advance all posting list cursors past the block boundary — the whole block is skipped.
4. If no safe skip exists: fully score the next candidate document and update the top-k heap.
5. Repeat until all posting lists are exhausted.

Because blocks are 128 documents wide and skip index entries are small, cache pressure during block skipping is minimal. In practice, on MS-MARCO passage ranking datasets, Block-Max WAND reduces the number of documents fully scored to roughly 25% of the total, producing a **~4× throughput gain** over naive BM25 that scores every candidate.

### Compression numbers

Delta encoding + bitpacking reduces index size by **41%** on typical corpora, with a **10–20% query latency improvement** for shorter queries (fewer bytes to read per block).
Parallel index builds (each worker owns a disjoint set of heap pages) reduce build time by **4×** on multi-core hardware.

## Where it breaks

**Semantic gap**: BM25 is a lexical match — it cannot connect "automobile" to "car" without stemmer preprocessing, and it has no notion of document meaning. Hybrid search (BM25 + pgvector approximate nearest-neighbor) bridges this, but the two scores need careful normalization before combining (Reciprocal Rank Fusion or score normalization).

**Freshness latency**: documents land in the memtable and are invisible to GIN-style index scans until the next flush. The system is not truly MVCC-integrated with the heap — a freshly inserted row visible to a `SELECT` may not yet appear in a full-text search.

**Parameter sensitivity**: `k1 = 1.2` and `b = 0.75` are well-calibrated for news articles of moderate length. Code documentation (very short, structured) and legal filings (very long, repetitive) may both need different parameters.

**Write amplification**: segment compaction rewrites data repeatedly as it moves through levels — the same RUM-conjecture write overhead as any LSM system. Heavy mixed read/write workloads can generate compaction I/O spikes.

**Standby requirement**: `hot_standby_feedback = on` must be set on standbys. Without it, the background compaction worker cannot safely determine when displaced tombstone pages are safe to reclaim, because standbys might still be scanning them.

## Why it works

All three major optimizations — delta encoding, Block-Max WAND, and the LSM write path — exploit the same structural invariant: **posting lists are sorted arrays**.

**Delta encoding** works because consecutive sorted integers have small differences. For a term appearing in 1% of a million-document corpus, the average gap between consecutive doc IDs is 100. That fits in 7 bits, not 20. The compression ratio improves the more common the term — the most frequently accessed posting lists are also the most compressed.

**Block-Max WAND** works because BM25 scores are *decomposable*: the score contribution of term *t* to document *d* depends only on *f(t,d)*, `|d|`, and corpus-level statistics. The per-block maximum is therefore a tightly computable upper bound. Skipping a block is safe whenever no document in the block can move any candidate's total score above the current threshold. This is the same pruning principle as **branch-and-bound** in discrete optimization: avoid exploring subtrees whose upper bound is below the best known solution.

**The LSM write path** works because inverted index mutations are pure appends: a new document adds new entries to posting lists; nothing is updated in place. This matches LSM's fundamental assumption — writes are sequential, immutable, and batch-merged — allowing the memtable to absorb write bursts at memory bandwidth while compaction amortizes the cost of sorted merge across background time.

The deepest insight: **an inverted index segment is an LSM SSTable where the key is a search term and the value is a compressed sorted-integer array**. Every concept from LSM maps directly onto Lucene, Elasticsearch, Solr, and pg_textsearch:

| LSM concept | Search engine equivalent |
|-------------|------------------------|
| Memtable | In-memory inverted index buffer |
| SSTable | Immutable segment with term dict + posting lists |
| Level compaction | Segment merge (fewer segments → faster multi-segment OR/AND) |
| Tombstone | Deleted document marker in posting list |
| Bloom filter per SSTable | Per-segment term membership filter |

Once you see this mapping, you can reason about search engine performance with the exact same tools you use for storage engine performance.

## Going deeper

1. **"Introduction to Information Retrieval"** — Manning, Raghavan & Schütze (Cambridge, 2008). Free online at nlp.stanford.edu/IR-book. Chapters 1–2 cover inverted index construction from scratch; Chapter 6 covers TF-IDF; Chapter 11 covers probabilistic models including the BM25 derivation.

2. **"Efficient Query Evaluation using a Two-Level Retrieval Process"** — Broder, Carmel, Herscovici, Soffer & Zien (CIKM 2003). The original WAND paper. Block-Max WAND (Ding & Suel, SIGIR 2011) is the direct descendant and the algorithm used by Lucene and pg_textsearch.

3. **`BlockMaxWANDScorer.java` in Apache Lucene source** — the production implementation used by Elasticsearch. The comments walk through each pruning decision exactly, making it the clearest executable specification of how the algorithm behaves in practice.
