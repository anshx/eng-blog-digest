---
title: "Bigtable: A Distributed Storage System for Structured Data"
source: https://research.google.com/archive/bigtable-osdi06.pdf
author: Fay Chang, Jeffrey Dean, Sanjay Ghemawat, Wilson C. Hsieh, Deborah A. Wallach, Mike Burrows, Tushar Chandra, Andrew Fikes, Robert E. Gruber
company: Google
date_posted: 2006-11-01
date_digested: 2026-07-23
---

# Bigtable: A Distributed Storage System for Structured Data

## What's new to learn

- **Column qualifier as a first-class key**: Column qualifier names are dynamic and set at write time, not at schema creation time. This means the column *name* carries semantic payload—a many-to-many relationship can be encoded entirely in the qualifier without touching the value. Most engineers only think of column names as schema metadata; Bigtable makes them data.
- **Locality groups — explicit physical layout control**: You choose which column families are stored in the same SSTable files, independently of the logical schema. This decouples "how you access data logically" from "how it sits on disk," enabling both point lookups and batch column scans on the same table without a schema change.
- **Tablet = unit of distribution, not shard**: A Bigtable table is split into contiguous row ranges called *tablets* (~200 MB each). The cluster has thousands of tablets assigned across tablet servers. Load balancing moves tablets, not data files—this is the same distinction as a virtual-node consistent hash ring where vnodes are the unit of placement, not physical servers.

## Prerequisites

- SSTable internals: sorted, immutable key-value files with a trailing block index (LevelDB readers are fine).
- LSM tree write path: writes go to an in-memory memtable, which is periodically flushed to disk as an immutable SSTable, with background compaction merging SSTables.
- GFS or any distributed blob store: Bigtable stores its SSTable files on GFS; the specifics matter less than understanding that the storage layer is remote and replicated independently.
- A lock service (Chubby/ZooKeeper): a highly-available key-value store that supports advisory locks and change notifications, typically backed by Paxos.

## The core idea

A Bigtable table is a **sparse, distributed, persistent multidimensional sorted map**. The map is indexed by a three-part key:

```
(row_key: bytes, column: "family:qualifier", timestamp: int64) → value: bytes
```

Each part of the key serves a different purpose:

- **Row key**: the primary sort key and unit of horizontal distribution. Reads and writes to a single row key are atomic, even across multiple columns. Tablets are contiguous row-key ranges.
- **Column**: written as `family:qualifier`. The *column family* is declared at table creation and controls compression, access control, and in-memory caching. The *qualifier* is freeform—any byte string, set at write time, not at table creation. You can have billions of distinct qualifiers in one column family.
- **Timestamp**: 64-bit integer (usually microseconds since epoch). Multiple versions of the same (row, column) pair can coexist, ordered by timestamp. Garbage-collection policies (keep last N versions, or delete versions older than T) are set per column family.

The most non-obvious piece is the qualifier. In a relational table, column names are fixed schema elements. In Bigtable, the qualifier is free data. Google's canonical example: representing inbound web links. To record that `google.com` links to `example.com` with anchor text "Google":

```
row key:   "com.example"          (reversed-URL for lexicographic locality)
column:    "anchor:com.google"    (qualifier = the linking page)
value:     "Google"               (anchor text)
timestamp: <write time>
```

To find all pages linking to `example.com`, scan the row for all qualifiers in the `anchor:` family. The qualifier *is* the linking-page identity—not a value, but the key of the relationship. A row with 10,000 inbound links has 10,000 distinct qualifiers in one row, all efficiently range-scannable, all compressed together because they belong to the same column family.

## Mechanics

### Write path

1. Client sends a write mutation to the tablet server that owns the row's tablet.
2. The tablet server validates the request (ACL check via Chubby), writes to a shared write-ahead log on GFS, and inserts the mutation into the in-memory **memtable** (a sorted buffer, equivalent to a MemTable in LevelDB).
3. The write is acknowledged to the client.

When the memtable reaches a size threshold, the tablet server performs a **minor compaction**: freeze the current memtable, create a new one, and flush the frozen memtable to GFS as a new immutable SSTable. This is the same LSM write path covered in the algorithms-behind-storage-systems deep dive.

### Read path

A read at timestamp T sees all mutations with timestamps ≤ T, merged across the active memtable and all SSTables for that tablet. The tablet server merges them lazily using a k-way merge (same as LSM compaction). For a given row+column, the server scans in newest-first order until it finds the data.

To avoid reading every SSTable, each SSTable can carry a **Bloom filter** for its row+column pairs. The tablet server checks the filter before opening an SSTable block—if the filter says "absent," skip the file. With ~10 bits per entry, false-positive rate is ~1%, meaning typically only one SSTable (the one that actually contains the key) is read.

### Compaction

Three compaction levels:
- **Minor**: memtable → new SSTable (reduces RAM usage).
- **Merging**: reads a few SSTables + the current memtable, writes one merged SSTable (reduces file count, limiting the k in k-way merge reads).
- **Major**: merges all SSTables for a tablet into one, physically deleting tombstones and expired versions. Only major compaction actually reclaims disk space.

### Tablet location

Finding which tablet server owns a given row requires a three-level lookup:

1. A single Chubby file stores the location of the **root tablet** (a special row in the `METADATA` table).
2. The root tablet stores locations of all other `METADATA` tablets.
3. `METADATA` tablets store locations of all user tablets.

Each entry is ~1 KB. The root tablet fits ~128k pointers; each METADATA tablet also fits ~128k pointers. That's 2^17 × 2^17 = ~17 billion tablets addressable, each up to 200 MB—enough to hold exabyte-scale data. Clients aggressively cache these locations; cache misses walk the three levels.

### Locality groups

Column families are assigned to **locality groups**. Each locality group materializes as a separate set of SSTable files on GFS. By default, all column families are in one locality group, so every read sees all columns. If you assign the `anchor:` family (sparse, large) and the `language:` family (one byte per row) to different locality groups, a read that only needs language reads only the tiny `language:` SSTable, skipping the large `anchor:` SSTables entirely.

A locality group can also be marked **in-memory**: the tablet server loads its SSTables into RAM on startup and keeps them there. This is effectively a persistent in-process key-value cache for hot column families without changing the data model.

Compression is also per locality group (Bentley-McIlroy for cross-block patterns + Zippy/Snappy per block), achieving 10:1 ratios on web data because qualifiers and values within a locality group tend to share structure.

### The master

One master server (elected via Chubby) handles tablet assignment, load balancing, schema changes (add/drop column families), and GFS garbage collection of orphaned SSTables. Critically, the master is **not in the read/write path**—clients talk directly to tablet servers after the initial tablet-location lookup. The master can be a bottleneck only for schema-change operations, not for data access.

## Where it breaks

**Single-row atomicity only.** Bigtable guarantees atomic reads and writes within a single row, but not across rows. Multi-row transactions require a separate coordination layer—this limitation is why Google built Percolator (covered in this archive) as an MVCC layer on top of Bigtable, using a single "primary lock" row as the 2PC coordinator.

**Fan-out reads are expensive.** Reading one value from a million rows (e.g., "get the age column for every user") is an efficient sequential scan of the `user-info:` locality group. But reading a hundred columns each from a different locality group per row multiplies the SSTable opens. Unlike a column store, Bigtable doesn't pre-join locality groups into vectorized batches.

**Qualifier cardinality must match your access pattern.** If you encode a set via qualifiers (10k qualifiers per row for 10k-member sets), row reads are O(qualifiers) not O(1). Poorly chosen key structure turns point lookups into scans. The "row key contains the range boundaries" design imperative is harder to reason about than SQL indexes.

**Chubby is a hard dependency.** Bigtable's master election, tablet server liveness detection, and root tablet location all run through Chubby. A Chubby outage freezes schema changes and can stall tablet-server fault detection, degrading availability even when the data plane (GFS + tablet servers) is healthy.

**No secondary indexes.** In Bigtable, queries without a known row key range require a full table scan. Real applications maintain secondary-index rows manually (a row in an index table with the indexed value as key and the primary row key as the cell value), which requires application-level consistency maintenance.

## Why it works

The deeper principle: **Bigtable gives you two independent levels of specification—logical and physical—while most databases silently merge them.**

In a relational database the schema defines both what the data means and, implicitly, how it's stored (rows together, one file per table or index). Changing the physical layout requires DDL or an index rebuild. In a pure key-value store there is no logical schema at all. Bigtable threads between these: the logical model (row × column family:qualifier × timestamp) is fixed and general, but the physical layout (locality groups, in-memory flags, compression codec) is declared separately and can be changed without touching the data model or the application code.

This is the relational model's **physical data independence** made explicit. Gray and Putzolu described it in 1987 as an aspirational property; most databases achieve it imperfectly. Bigtable achieves it by giving operators a first-class "physical layout declaration" API (locality group config) that sits above the storage engine but below the application.

The second principle: **column qualifier = secondary key in a sparse triple store.** The (row, column family:qualifier, timestamp) → value map is isomorphic to an RDF triple (subject, predicate, object) plus a time axis. Encoding a graph relationship in the qualifier (like the `anchor:com.google → "Google"` example) is the same pattern as RDF's `<example.com, anchor, google.com>` triple. Bigtable just happens to be a horizontally scalable, compressible, version-controlled triple store layered on top of an LSM engine.

This is why Bigtable's data model survived well beyond the NoSQL boom: Cassandra, HBase, and Azure Table Storage are all narrow-waist adaptations of Bigtable's (partition key, clustering key, column) triple. Every "wide-column store" in production today is an instance of this same three-part key idea.

## Going deeper

1. **The Bigtable paper itself** (HTML render, easier to navigate than PDF): [https://mwhittaker.github.io/papers/html/chang2008bigtable.html](https://mwhittaker.github.io/papers/html/chang2008bigtable.html) — the original OSDI 2006 paper; worth reading Section 7 ("Performance Evaluation") to see how the tablet-server design scales under sequential vs. random reads with many vs. few tablet servers.

2. **Percolator: Large-scale Incremental Processing Using Distributed Transactions and Notifications** — already in this archive (2026-07-18). Percolator is the exact story of what happens when you need cross-row transactions on Bigtable: understanding the constraints described here (single-row atomicity, no native 2PC) makes Percolator's shadow-column 2PC design immediately obvious.

3. **Cassandra: A Decentralized Structured Storage System** (Lakshman & Malik, LADIS 2009, available at [https://www.cs.cornell.edu/projects/ladis2009/papers/lakshman-ladis2009.pdf](https://www.cs.cornell.edu/projects/ladis2009/papers/lakshman-ladis2009.pdf)) — Cassandra replaces Bigtable's Chubby dependency and GFS with consistent hashing and peer-to-peer replication (Dynamo-style), making the wide-column data model available without Google's internal infrastructure. Reading Cassandra after Bigtable shows exactly which pieces of the architecture are essential to the data model and which are deployment choices.
