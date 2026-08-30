---
title: "WiscKey: Separating Keys from Values in SSD-Conscious Storage"
source: https://www.usenix.org/conference/fast16/technical-sessions/presentation/lu
author: Lanyue Lu, Thanumalayan Sankaranarayana Pillai, Hariharan Madhyastha, Andrea C. Arpaci-Dusseau, Remzi H. Arpaci-Dusseau
company: University of Wisconsin–Madison
date_posted: 2016-02-22
date_digested: 2026-08-30
---

# WiscKey: Separating Keys from Values in SSD-Conscious Storage

## What's new to learn

- **Key-value separation**: In an LSM tree, only keys need to be sorted during compaction — values don't. Storing values in a separate append-only log reduces write amplification from O(L × F) to nearly O(1) for values, where L is the number of LSM levels and F is the fanout.
- **SSD-aware random I/O parallelism**: SSDs serve random reads at near-sequential bandwidth when requests are parallelized, so fetching scattered values concurrently from a value log can match the throughput of a single sequential scan.
- **GC as a lightweight compaction alternative**: Instead of moving all data during LSM compaction, stale values are reclaimed via a separate online garbage collector that scans only the tail of the value log — decoupling space reclamation from key ordering.

## Prerequisites

- **LSM tree basics**: How data flows from MemTable to immutable L0 SST files, through multi-level compaction to larger levels. Why compaction is necessary (reads need sorted files; stale entries need removal).
- **Write amplification**: Each byte written by the application may be physically rewritten 10–30× by the storage engine as it moves through LSM levels. For a 5-level tree with fanout 10, each byte could be rewritten at every level boundary.
- **SSD vs HDD I/O characteristics**: HDDs have a 100–1000× gap between sequential and random bandwidth. SSDs narrow that to 3–10×, and to near-1× when random reads are parallelized across multiple outstanding requests (SSDs have deep internal parallelism across NAND channels and planes).

## The core idea

In a standard LSM store like LevelDB or RocksDB, a key-value pair `(k, v)` is written into the MemTable and eventually sorted and compacted into increasingly large SST files. During compaction, **both** the key and the value are read, sorted, and rewritten. If the value is 4 KB and the key is 16 bytes, the compaction pays 250× more I/O on the value than on the key for the same organizational benefit.

WiscKey's observation: **sorting only needs the keys.** The value just needs to be retrievable; it doesn't need to live adjacent to the key in sorted order.

So WiscKey splits the write:
1. Append the value to a separate **value log (vLog)** — a flat append-only circular file. Record the resulting file offset.
2. Insert `(key → (vLog_offset, value_len))` into the LSM tree. The LSM entry is now tiny.

At compaction time, only the small `(key, offset, len)` tuples move. The actual value bytes stay in the vLog undisturbed. Write amplification for values drops from O(L × F) to O(1) — the one initial append.

The result on SSDs: **2.5–111× faster database loads** than LevelDB, and **1.6–14× faster random lookups**, depending on value size (larger values see larger gains, since compaction overhead scales with value size).

## Mechanics

### Write path

```
put(k, v):
  offset = vLog.append(v)          # sequential write to tail
  lsm.put(k, (offset, len(v)))     # small entry into LSM MemTable
```

The vLog append is sequential, matching the LSM's own write pattern. The MemTable entry is tiny — effectively just a pointer.

### Read path

```
get(k):
  (offset, size) = lsm.get(k)      # standard LSM point lookup
  return vLog.read(offset, size)   # one random read at known offset
```

The read now has two I/Os: one LSM lookup (which may touch multiple levels) and one vLog random read. On SSD, a single random read is fast enough that this is rarely a bottleneck.

### Range queries and parallel prefetch

Range scans are the main sacrifice. In a standard LSM, scanning `[key_start, key_end]` is a single sequential read over sorted SSTs — extremely cache-friendly. In WiscKey, the keys are still sorted in the LSM, but values are scattered throughout the vLog at their insertion offsets. Iterating `[k1, k2, ..., kn]` in order requires n random reads into the vLog.

WiscKey mitigates this with a **prefetch buffer**: as the iterator yields keys in sorted order, it issues `posix_fadvise(WILLNEED)` or async I/O requests for the next 64–128 values in parallel. This exploits SSD internal parallelism so that by the time the application processes value i, values i+1 through i+64 are already in the page cache. On SSD, with 32 outstanding requests, random read throughput matches sequential read throughput for values ≥ 64 KB.

For small values with dense range scans (e.g., time-series data), this parallelism does not fully compensate, and range scan performance degrades noticeably.

### Value size threshold

For very small values (below a configurable threshold, typically 64 bytes to 1 KB), the overhead of an extra vLog I/O exceeds the write-amplification savings. WiscKey inlines small values directly into the LSM tree, exactly like a standard LSM. Only values above the threshold are separated.

This means WiscKey behaves identically to LevelDB for small values and kicks in the separation for large values.

### Value log structure and garbage collection

The vLog is a circular log. The **head** pointer tracks where new values are appended; the **tail** pointer tracks the oldest data. Valid data occupies `[tail, head)`.

When a key is overwritten or deleted, its old value in the vLog becomes garbage — but it isn't removed immediately. Space reclamation happens via **online GC**:

```
gc_round():
  chunk = vLog.read_chunk_from_tail(size=4MB)
  valid_pairs = []
  for each (k, v) in chunk:
    if lsm.get(k).offset == current_offset(v):
      valid_pairs.append((k, v))   # value is still live
  for (k, v) in valid_pairs:
    new_offset = vLog.append(v)    # re-append to head
    lsm.put(k, (new_offset, len(v)))
  vLog.advance_tail(size=4MB)
```

GC reads a chunk from the tail, checks the LSM to see if each value is still current (by comparing vLog offsets), re-appends live values to the head, and advances the tail. This runs concurrently with normal reads and writes. The I/O cost of GC is proportional only to live data, not total data — cold data that's never overwritten costs nothing.

The tail-to-head re-appending means the vLog head keeps growing. If the application write rate exceeds GC throughput, the log grows without bound; WiscKey can throttle foreground writes to pace GC.

### Crash recovery

Traditional LSM trees need a WAL (write-ahead log): if the process crashes before the MemTable is flushed to L0, the WAL replays the lost writes.

WiscKey's vLog already functions as a WAL. Every `put(k, v)` appends `v` to vLog before touching the LSM MemTable. After a crash:

1. The LSM recovers normally from its own WAL (or it already has no WAL if both the vLog and a small per-key log are used).
2. Scan the vLog from the last known good head position forward.
3. For any `(k, v)` found in the vLog but not in the LSM, re-insert `(k, offset)` into the LSM.

One subtlety: the vLog head pointer is stored in the LSM as a special sentinel key. After a crash, the actual vLog head may be ahead of what the LSM knows about. WiscKey verifies the head pointer by scanning forward and validating key-offset consistency.

### No need for a separate WAL

With the vLog acting as the WAL, WiscKey can disable the LSM's own WAL entirely (LevelDB's `WriteOptions::sync = false` path). This saves one extra sequential write per `put` — a meaningful saving for write-heavy workloads.

## Where it breaks

**Range scans on cold data**: If the workload frequently scans large key ranges over recently written data (e.g., analytics on an event log), the random vLog access pattern can degrade range scan throughput by 3–5× vs a standard LSM on SSD, and by orders of magnitude on HDD. WiscKey is not suitable as-is for workloads where range scan dominates.

**GC latency spikes**: A GC round that discovers a heavily overwritten chunk must re-append many live values, stressing the write path. Applications may see latency hiccups when GC and foreground writes compete for vLog I/O. Production deployments (Titan in TiKV, BlobDB in RocksDB) add rate limiters and compaction priority hints to smooth this out.

**Small-value workloads**: For key-value pairs with values < 1 KB (e.g., metadata stores, session state), the separation overhead (extra I/O, extra complexity) exceeds the write-amplification savings. WiscKey's threshold parameter must be tuned per workload; a misconfigured threshold hurts performance in both directions.

**Increased read amplification for point lookups**: Standard LSM may serve a hot-path read from the MemTable or block cache with zero disk I/Os. WiscKey's vLog read always costs a disk access unless the value is already in the OS page cache. For read-heavy workloads with large values and a hot working set, this trade-off can be neutral; for a cold dataset, point-lookup latency increases.

**GC correctness under concurrency**: Determining whether a value is live requires checking the LSM atomically. If a concurrent write overwrites a key between the GC's LSM check and its tail-advance, the new value could temporarily be garbage-collected. WiscKey handles this with careful epoch-based validation, but the implementation is significantly more complex than standard LSM compaction.

**Space amplification during GC**: While GC is running, both the old data at the tail and the re-appended live data at the head exist simultaneously. Peak space usage can reach 2× the logical data size.

## Why it works

The deeper principle is **sort by pointer, not by payload**. 

When you need to sort large records, the naive approach moves the records themselves. The classic optimization from external sort algorithms is: create an array of `(key, pointer)` tuples, sort those (cheap, they're small), then access the original records via the pointers. WiscKey applies this exact optimization to LSM compaction:

- The "records" are values (potentially megabytes).
- The "keys" are the sort keys (bytes to tens of bytes).
- Compaction sorts `(key, vLog_pointer)` tuples, leaving the large payloads untouched.

This same principle appears everywhere in systems:

- **Database secondary index**: The B-tree index stores `(key → rowid)`, not the full row. Heap page updates don't touch the index; the index touches the heap only on lookup.
- **git object model**: Tree objects (directory listings) store `(filename → SHA-1 blob hash)`. Blobs (file contents) are never moved when you rename a file; only the tree object changes.
- **Sorted string tables with block index**: In Parquet/SST files, a small block index (sorted column-chunk min/max → byte offset) enables skipping large data blocks entirely. The index moves during compaction; the data pages don't need to be fully re-read.
- **Pointer-based indirection in column stores**: Apache Arrow separates the string dictionary (sorted unique values) from the integer encoding array. Sorting the dictionary doesn't touch the encoding array.

The underlying invariant: **re-organization cost scales with the size of the thing being organized, not the size of the logical entity**. If you can decouple ordering key from payload, compaction cost collapses from O(key\_size + value\_size) to O(key\_size) — often a 100–1000× reduction for typical value sizes.

The second key enabler is the **collapse of the sequential/random I/O gap on SSDs**. On HDDs, random I/O is ~100–1000× slower than sequential I/O, making any data structure that requires random access unacceptable for throughput-critical paths. WiscKey's vLog value reads are random; on HDD, this would be catastrophic. On NVMe SSDs, random reads are only 3–5× slower than sequential, and with request parallelism (multiple outstanding I/Os), throughput converges. WiscKey is therefore a technique that would have been impractical in 1990 but is correct in 2016 — the hardware change unlocked the abstraction.

The broader lesson: **when hardware performance ratios change by 100×, data structure design assumptions from the prior generation are candidates for inversion**. This is the same principle as the "database for SSDs" insight (questioning WAL design and page sizes), but applied to the LSM compaction mechanism specifically.

## Going deeper

1. **Titan: WiscKey for TiKV/RocksDB** — PingCAP's production implementation of key-value separation inside RocksDB, with additional features like discardable-ratio-based GC scheduling and blob file management. Source: https://docs.pingcap.com/tidb/stable/titan-overview

2. **RocksDB BlobDB** — Meta's integration of blob value separation directly into RocksDB (shipped in RocksDB 6.18+), covering the implementation trade-offs between in-lined and separated values with a configurable `min_blob_size` threshold. Source: https://rocksdb.org/blog/2021/05/26/integrated-blob-db.html

3. **Badger** — a pure-Go key-value store implementing the WiscKey design, with a clean codebase that's easier to read than C++ RocksDB. Its design document explains the GC algorithm and crash-recovery protocol concretely. Source: https://dgraph.io/blog/post/badger/
