---
title: "An Introduction to Bε-trees and Write-Optimization"
source: https://www.usenix.org/system/files/login/articles/login_oct15_05_bender.pdf
author: Michael A. Bender, Martin Farach-Colton, William Jannen
company: Stony Brook University / TokuDB
date_posted: 2015-10-01
date_digested: 2026-09-09
---

# An Introduction to Bε-trees and Write-Optimization

## What's new to learn

- **Bε-tree**: A B-tree variant that buffers writes as messages in internal nodes, flushing them to children in batches — achieving O(log_B N / B^(1−ε)) amortized write I/Os, roughly 32× fewer than a standard B-tree under typical parameters.
- **The buffer-cascade pattern**: Any data structure that defers work from expensive individual operations into amortized batches (B-trees, LSM trees, Bε-trees) is an instance of the same "cascade of buffers" discipline — Bε-trees make the parameter governing this tradeoff explicit and tunable.
- **LSM trees as degenerate Bε-trees**: Viewed through the Bε lens, an LSM tree's compaction cascade is a special case where ε → 0 and each "buffer" is an entire sorted run on disk — the unifying abstraction that collapses two seemingly unrelated data structures into one parameterized family.

## Prerequisites

- B-trees and their I/O complexity (O(log_B N) point lookups, O(log_B N) writes — from the RUM conjecture post if unfamiliar)
- LSM trees at a structural level (memtable → L0 → L1 → … merge cascade)
- The I/O model: disk bandwidth >> disk seek cost, so sequential block reads are cheap and random seeks are expensive (B is the block size in units of keys)
- Big-O in terms of B and N (N = number of keys, B = keys per disk block, M = memory size in keys)

## The core idea

Traditional B-trees pay O(log_B N) I/Os to insert a single key because they route each insert all the way from root to leaf, touching O(log_B N) nodes. That's unavoidable if you must immediately place the key in its final sorted position.

The Bε-tree's insight: **don't**.

Instead, treat an insert as a *message* and leave it near the root. An internal node at height h now carries two things: a compact routing index (like a B-tree's usual keys) and a *buffer* that accumulates pending messages. A message enters at the root's buffer with one I/O, and only descends when its buffer overflows — at which point an entire batch of messages is flushed together in a single I/O.

Think of it like a postal sorting depot. Rather than hand-delivering each letter the moment it arrives, the depot collects mail until it has a full truck-load bound for a given district, then ships the batch. Each letter still arrives at its final destination, but the cost per letter drops because you're never moving one letter at a time.

The parameter ε ∈ (0, 1] governs the split between routing and buffering within each node:
- **ε = 1**: each node's entire block is routing space, zero buffering — this is a regular B-tree.
- **ε → 0**: almost all node space is buffering, almost no routing — this approaches LSM-tree territory.
- **ε = 1/2**: a practical sweet spot where writes are ~32× cheaper than a B-tree while reads are only 2× more expensive.

## Mechanics

**Node layout** at height h in an N-key tree with block size B:

- Fanout (branching factor): B^ε children per node
- Buffer capacity: Θ(B^(1−ε)) pending messages per node
- Height of tree: log_{B^ε}(N) = log(N) / (ε · log B) levels

**Insert path**:
1. Encode the insert as an upsert message `(key, value, timestamp)`.
2. Append it to the root's buffer — O(1/B) amortized I/Os (amortized over B messages sharing one block write).
3. If the root buffer overflows, pick a child c whose keys overlap the fullest portion of the buffer. Flush all messages bound for c's subtree in a single I/O.
4. The same overflow check propagates recursively down the tree.

**Amortized insert cost**:  
Each message must cross log_{B^ε}(N) levels. At each level, it is part of a batch of Θ(B^(1−ε)) messages flushed together in one I/O. Therefore the cost per message per level is 1/B^(1−ε) I/Os, and the total is:

```
Insert I/Os = log(N) / (ε · log B)  ×  1/B^(1−ε)
            = O(log_B N / B^(1−ε))
```

For B = 4096 (a 16 KB block holding 4096 4-byte keys), ε = 1/2:
- B-tree insert: O(log_4096 N) ≈ log(N)/12 I/Os
- Bε-tree insert: O(log(N) / (0.5 × 12 × 64)) ≈ log(N)/384 I/Os — **32× cheaper**.

**Point query path**:
1. Read the root node.
2. At each level: scan the node's buffer for messages that modify the target key (must check all buffered messages for the subtree containing this key).
3. Descend to the appropriate child.
4. At the leaf, apply any buffered messages to derive the final value.

**Point query cost**:  
Must read O(log_{B^ε}(N)) nodes = O(log N / (ε · log B)) I/Os. For ε = 1/2 this is 2× the B-tree query cost — the cost of the write optimization.

**Range scans**: Unaffected asymptotically. Leaves are still stored in sorted order. Buffered messages add a constant scan overhead per flushed node, negligible for large scans.

**Deletes and updates** are also messages: a delete is a tombstone message that cancels earlier inserts when it reaches the leaf. This means Bε-trees naturally handle delete-heavy workloads with the same amortized write cost.

**Message types table**:

| Operation | Message | Leaf resolution |
|-----------|---------|-----------------|
| Insert    | `(k, v, ts)` | write k→v |
| Delete    | `(k, tombstone, ts)` | remove k |
| Update    | `(k, f, ts)` | apply f(old_v) |

The timestamp allows correct merge ordering: when a key has multiple buffered messages, the one with the highest timestamp wins at the leaf.

**Flush heuristic**: When flushing a buffer, choose the child whose pending messages are most numerous (heaviest child) to maximize I/O amortization and minimize the number of flushes needed.

## Where it breaks

**Write amplification on flush cascades**: In the worst case a single flush at the root triggers O(log_{B^ε}(N)) cascading flushes down the tree (each child's buffer happens to be nearly full). This makes the **worst-case** insert O(log^2_B N) — not the amortized cost. For workloads with tight latency SLOs on writes, this tail can matter.

**Buffer scanning on reads**: Each point query must scan all buffered messages in every node on the path. For B = 4096 and ε = 1/2 that's 64 messages to scan per node; for ε = 1/4 it's 256. This is usually fast (sequential scan of a hot cache line) but it's real overhead.

**Random-read penalty**: If the same key is updated many times between flushes, its old value at the leaf can only be read after applying all in-flight buffer messages in order. Contrast with a B-tree where the value is always at the leaf in its final form.

**Not a drop-in for B-tree workloads**: Point-update-heavy transactional workloads (OLTP with small writes to a few hot rows per transaction) don't benefit much. The amortization only helps when many distinct keys are written before a flush.

**TokuDB's practical lessons**: TokuDB was the production Bε-tree for MySQL (Percona's TokuDB plugin). Its main user complaint was **high disk space amplification** during compaction: the buffers themselves must be on disk, adding Θ(B^(1−ε)) overhead per node level. It was deprecated in MySQL 8.0 partly because RocksDB (LSM-based) achieved similar write throughput with simpler compaction and lower space overhead.

## Why it works

The deeper principle here is **amortized I/O via batch scheduling**, and it has the same shape as every other write-optimization pattern in storage:

- **WAL in PostgreSQL/MySQL**: Instead of writing every change immediately to the heap, buffer them in a sequential log and write the heap lazily (checkpointing). The WAL is a one-level Bε-tree buffer.
- **LSM trees (RocksDB, Cassandra)**: A sequence of sorted runs where merges (flushes) batch-convert L0 writes into L1, L1 into L2, etc. This is a Bε-tree where the "buffer" at each level is an entire sorted run, and the fanout per level is constant (typically 10).
- **Group commit in databases**: Defer fsyncing until a batch of transactions is ready, amortizing the fsync cost across them.

The Bε-tree makes the unifying pattern explicit: **any data structure can amortize a hard operation (random I/O write to a leaf) over a batch of pending operations (buffered messages) by adding one level of indirection**. The cost is determined by two things:
1. How large the batch is when it finally executes (batch size = B^(1−ε))
2. How deep the cascade goes (tree height = log_{B^ε}(N))

The RUM conjecture (Read / Update / Memory amplification) says you can't minimize all three simultaneously. The Bε-tree is a proof by construction that ε gives you a continuous knob to move along the RU tradeoff axis:

| ε | Reads | Writes | Structure |
|---|-------|--------|-----------|
| 1 | log_B N | log_B N | B-tree |
| 1/2 | 2·log_B N | log_B N / 32 | Bε-tree |
| → 0 | k·log_B N | log_B N / B | LSM-like |

The "oh, so X is just an instance of Y" insight: **B-trees and LSM trees are not two unrelated inventions — they are the endpoints of a one-parameter family of write-optimized trees indexed by how aggressively you buffer writes in internal nodes.** The Bε-tree is that family made explicit.

A second insight: the message-based update model (insert/delete/update as messages flowing down) is identical in spirit to MVCC version chains and Kafka consumer offsets. Deferred application of ordered operations at "commitment time" (flush) is the same pattern as:
- MVCC: version chain resolved at read time instead of write time
- Kafka: consumer resolves event ordering at its own pace
- CRDTs: operation log merged lazily into state

All of these convert a synchronous, expensive operation (immediate consistency) into an asynchronous, amortized one (eventual leaf update) with a buffer absorbing the burst.

## Going deeper

1. **BetrFS: Write-Optimization in a Kernel File System** (Jannen et al., FAST 2015)  
   Implements a Bε-tree as a Linux kernel file system, achieving 2.5× faster sequential writes, 690× faster recursive `grep`, and 1.5× faster small-file creates than ext4 — a production-quality demonstration of what Bε-tree amortization buys in practice.  
   https://www.usenix.org/conference/fast15/technical-sessions/presentation/jannen

2. **The RUM Conjecture** (Athanassoulis et al., CIDR 2016)  
   Formalizes the three-way tradeoff between Read, Update, and Memory amplification, showing Bε-trees, B-trees, and LSM trees as distinct operating points and proving no structure can minimize all three. The companion post in this archive (2026-06-10) is a prerequisite.  
   https://stratos.seas.harvard.edu/files/stratos/files/rum.pdf

3. **Fractal Tree Index (TokuDB)** blog series by John Esposito (Percona, 2013)  
   A practitioner's account of shipping Bε-trees inside MySQL, covering compaction scheduling, compression, and the real-world tradeoffs that forced design changes from theory to production.  
   https://www.percona.com/blog/tokudb-fractal-tree-index-slide-by-slide/
