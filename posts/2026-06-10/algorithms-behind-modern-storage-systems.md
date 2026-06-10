---
title: "Algorithms Behind Modern Storage Systems"
source: https://queue.acm.org/detail.cfm?id=3220266
author: Alex Petrov
company: ACM Queue / Communications of the ACM
date_posted: 2018-07-25
date_digested: 2026-06-10
---

# Algorithms Behind Modern Storage Systems

## What's new to learn

1. **The RUM conjecture**: Any data access method is subject to a three-way tradeoff between Read amplification, Update (write) amplification, and Memory (space) amplification — lowering any two of these costs provably raises the third. This gives you a language for comparing every storage engine ever built.

2. **LSM tree write-path mechanics**: Buffering writes in a sorted in-memory structure (MemTable), flushing as immutable sorted files (SSTables), then merging those files in the background (compaction) converts every random write into a sequential one — deferring all sort-maintenance cost to background I/O.

3. **Leveled vs. tiered compaction as a spectrum, not a binary choice**: The two canonical compaction strategies make opposite tradeoffs on write amplification vs. space amplification, and real systems (RocksDB, Cassandra, ScyllaDB) sit at different points on this spectrum — understanding why requires the RUM conjecture as a frame.

## Prerequisites

- **B-tree internals**: Know that a B-tree keeps all data sorted on disk in pages, that insertions cause page splits, and that a lookup traverses O(log_B N) pages (B = branching factor ≈ hundreds).
- **Sequential vs. random I/O costs**: On spinning disks, sequential throughput is 100–1000× faster than random; on NVMe SSDs, the gap is smaller but still meaningful for large writes. An LSM tree's write-path leverage depends entirely on this gap.
- **What a "sorted run" is**: A file where keys appear in ascending order, no duplicates within the file, so a binary search or merge over it is O(log n) or O(n) respectively.

## The core idea

A B-tree pays the sort-maintenance cost on every write: to insert a key, the engine finds the right leaf page, writes the key in sorted order, and splits the page if it's full — potentially writing multiple pages, all at scattered positions on disk. At high write rates this creates a random-write bottleneck.

An LSM tree defers that cost. Writes land sequentially in a small in-memory buffer (the MemTable). When the buffer fills, the engine snapshots it to disk as a new, already-sorted, immutable file — a sequential write. A background compaction thread periodically reads several such files, merges them (like merge sort's merge step), and writes a single larger sorted file, also sequentially. The key data is never modified in place.

The payoff: every write to an LSM tree is sequential. The penalty: reads may now need to check several levels of files to find a key, and background compaction consumes I/O that competes with foreground reads.

## Mechanics

### Write path

```
client write
    │
    ▼
MemTable ──── WAL ──── (crash safety)
    │
    │  (MemTable threshold reached)
    ▼
L0 SSTable (immutable, sorted)
    │
    │  (L0 file count threshold reached)
    ▼
Compaction → L1 SSTable
    │
    ▼
  ...  → Ln SSTable (largest level)
```

1. **MemTable**: An in-memory sorted structure (usually a skip list or red-black tree). Writes go here first, making insertions O(log n) in RAM.
2. **WAL (Write-Ahead Log)**: Every write is also appended to a sequential log on disk. If the process crashes before the MemTable is flushed, the WAL lets the engine replay those writes on restart.
3. **SSTable flush**: When the MemTable hits its size limit, it is "frozen" and flushed to disk as an immutable SSTable. A new, empty MemTable takes its place. The flush is sequential — one large write.
4. **Compaction**: The background thread picks SSTables from adjacent levels, reads them into memory, performs a k-way merge, and writes the result as a new SSTable in the next level. Old SSTables are then deleted.

### Leveled compaction (LevelDB/RocksDB default)

- L0 is the exception: multiple SSTables may overlap in key range (they were flushed independently).
- L1 through Ln are each a *single sorted run*: no two SSTables in the same level share key ranges.
- Compaction trigger: Ln's total size exceeds its budget (typically 10× Ln-1's budget).
- Compaction action: pick one SSTable from Ln, merge it with the (potentially many) overlapping SSTables from Ln+1, write a new set of Ln+1 SSTables.

**Write amplification**: Each byte written at L0 is merged once into L1, once into L2, ..., once into Lmax. With a size ratio R=10 and L levels, worst-case write amplification ≈ R × L (e.g., 10 × 6 = 60×). In practice, "some-to-some" compaction (RocksDB's implementation, not the original "all-to-all" LSM paper) brings this closer to R × L / 2.

**Space amplification**: Because adjacent levels are non-overlapping sorted runs, only Ln and Ln+1 data exists simultaneously during compaction. Space amplification ≈ 1 + 1/R ≈ 1.1× (10% overhead). This is the lowest of any compaction strategy.

**Read amplification**: A point lookup must check L0 (potentially all L0 SSTables due to overlap), then one SSTable per level L1..Lmax. With Bloom filters (see below), almost all unnecessary SSTable probes are skipped; real read amplification is O(L0_files + L) ≈ 5–10 I/Os in typical deployments.

### Tiered (universal) compaction (Cassandra STCS, BigTable)

- Each level holds T sorted runs, and compaction merges all T runs into one run at the next level.
- **Write amplification**: O(L) — each byte is merged only once per level, regardless of size ratio. Substantially lower than leveled.
- **Space amplification**: O(T) — at the moment of compaction, T runs plus their merged result coexist. Space spikes can be large.
- **Read amplification**: O(T × L) — within each level, a lookup might need to probe all T runs.

This is the right choice for write-heavy workloads that can tolerate slower reads and more storage headroom.

### Bloom filters: asymmetric read optimization

Each SSTable carries a Bloom filter — a compact bit array that answers "is key K definitely absent from this SSTable?" in O(1) with a tunable false-positive rate (typically 1%, achieved with ~10 bits/key using two independent hash functions).

For a non-existent key, a Bloom filter eliminates ~99% of SSTable probes at each level. Without Bloom filters, a negative point lookup (key not in the database) would incur O(L) disk reads; with them it drops to O(L × 0.01) ≈ nearly zero. For the most common cold-read pattern (cache miss, item doesn't exist), Bloom filters change LSM read performance from "awful" to "acceptable."

Space cost: 10 bits/key × 10⁹ keys ≈ 1.25 GB per level. For a 6-level tree the Bloom filters for all levels below L_max are small; only L_max's filter is large, and it dominates.

### Deletes via tombstones

SSTables are immutable, so deletes cannot remove data in-place. Instead, the engine writes a *tombstone* — a special entry for the key with a "deleted" flag. Reads that encounter a tombstone before finding an older version return "not found." During compaction, tombstones suppress older versions of the same key. A tombstone is only physically purged when compaction can prove that no reader holds a snapshot older than the tombstone's timestamp.

This has a subtle implication: tombstone accumulation can slow reads and compaction significantly in delete-heavy workloads — an issue called "tombstone storms" in Cassandra deployments.

## Where it breaks

**Write stalls**: Compaction is a background I/O consumer. If the write rate outpaces compaction throughput, L0 fills up and the engine pauses writes entirely ("write stall") to give compaction time to catch up. In systems with bursty write traffic, stall duration can spike to seconds.

**Range scan penalty**: B-trees shine at range queries because all keys in a range appear in a contiguous sorted run at the leaf level — a pure sequential scan. In an LSM tree, a range query must merge results from all levels simultaneously; with L levels each contributing a sorted cursor, a range scan is O(k log L) per key returned (k = number of keys returned, log L for the heap merge). For scan-heavy workloads, B-trees often win.

**Bloom filter RAM cost**: At 10 bits/key, a 1-billion-key table needs ~1.25 GB of Bloom filter data just for the largest level. Embedding all level filters in RAM is feasible for medium-scale systems but becomes a significant fraction of available memory at very large scale.

**Write amplification headroom**: Leveled compaction's 60× theoretical write amplification is real on SSDs. NVMe SSDs are rated for 1–3 device write/day (DWPD); 60× application write amplification × 3× SSD internal write amplification = 180× physical wear. Systems like RocksDB invest significant engineering in reducing this (dynamic level sizing, lazy compaction scheduling).

**Compaction correctness with snapshots**: A snapshot — a read at a past point in time — prevents tombstone purging and old-version deletion until the snapshot is released. Long-lived snapshots in replication or backups can cause SSTable accumulation and unbounded space growth.

## Why it works

The deeper principle is that **LSM trees are merge sort applied lazily and incrementally to immutable sorted files**.

Merge sort's defining property is that it needs no random access: given two sorted arrays, you can produce one sorted output in O(n) time with a single sequential pass over each. LSM trees exploit this property relentlessly:

- A flush is writing a new sorted array (O(n) sequential).
- A compaction is merging k sorted arrays (O(n log k) sequential).
- A read on a single level is binary search in a sorted array (O(log n) with I/O).

The alternative — B-trees — maintains a *single* sorted structure and updates it in-place, paying random I/O on every write. LSM trees trade that random write cost for *read amplification* (many sorted runs) and *compaction I/O* (periodic merges). This is not a free lunch; it is a deliberate shift from random writes to sequential I/O, which is exactly the right trade on hardware where sequential bandwidth >> random IOPS.

The RUM conjecture formalizes why you can't escape the tradeoff:

- **Read-optimal** (B-tree): low read amplification, but writes must maintain sorted order in-place → high write amplification, reserved page space → high memory overhead.
- **Write-optimal** (log-append): write everything sequentially to one file → zero write amplification, but reads must scan the whole file → unbounded read amplification.
- **Space-optimal** (heap file): store records where they fit → no wasted space, but neither reads nor writes can use sorted-order shortcuts → high amplification for both.

Every real storage engine is a point in this 3D space. Leveled compaction slides toward the read-optimal corner (low space amplification, moderate write amplification). Tiered compaction slides toward the write-optimal corner. FIFO compaction (drop oldest files) approaches the write-optimal corner for time-windowed data. B-trees occupy the read-optimal corner. No engine reaches the origin — and the RUM conjecture says none ever will.

This same "you can't minimize all three" structure appears elsewhere in CS:
- **CAP theorem**: Consistency, Availability, Partition tolerance — pick two.
- **The Tail at Scale**: Reducing P99 latency requires redundancy (space overhead) or hedging (write overhead).
- **Compression tradeoffs**: compression ratio, encode time, decode time — can't optimize all three simultaneously.

The RUM framing is more actionable than CAP because it quantifies the tradeoffs with concrete amplification numbers, letting you reason about hardware cost directly.

## Going deeper

1. **"Constructing and Analyzing the LSM Compaction Design Space"**, Sarkar et al., VLDB 2021 (https://vldb.org/pvldb/vol14/p2216-sarkar.pdf) — a formal taxonomy of all compaction strategies (Classic Leveled, Tiered, Leveled-N, Tiered+Leveled, FIFO) with analytical models for write amplification, read amplification, and space amplification under each. The paper the RocksDB team references when evaluating new compaction designs.

2. **RocksDB Compaction Wiki** (https://github.com/facebook/rocksdb/wiki/Compaction) — the production engineering side: how L0's tiered behavior interacts with leveled L1+, what `max_bytes_for_level_multiplier` does, when to use Universal (tiered) vs. Level (tiered+leveled) compaction, and how to diagnose write stalls.

3. **"The Log-Structured Merge-Tree"**, O'Neil et al., *Acta Informatica* 1996 (https://www.cs.umb.edu/~poneil/lsmtree.pdf) — the original paper. The two-component C0/C1 model is simpler than modern multi-level variants, and reading it reveals how far LevelDB and RocksDB departed from the original design while keeping its core write-optimization insight intact.
