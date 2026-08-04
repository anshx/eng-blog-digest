---
title: "The Snowflake Elastic Data Warehouse"
source: https://dl.acm.org/doi/10.1145/2882903.2903741
author: Benoit Dageville, Thierry Cruanes, Marcin Zukowski, Vojislav Ercegovac, Can Gankidi, Jinfeng Li, Tim Sellam, Martin Seibold
company: Snowflake
date_posted: 2016-06-01
date_digested: 2026-08-04
---

# The Snowflake Elastic Data Warehouse

## What's new to learn

- **Shared-data architecture**: Multiple compute clusters can read the same immutable cloud storage objects concurrently with zero coordination — compute and storage are fully independent services, not co-located tiers that must scale together.
- **Format-level MVCC via immutability**: ACID transactions on a data warehouse fall out naturally when every write creates new immutable files and the metadata layer atomically swaps which files belong to the current table version — making time travel and zero-copy cloning structural consequences of the design, not added features.
- **Micro-partition pruning as a lightweight distributed index**: Storing per-column min/max and distinct-count statistics in each partition's header — indexed separately in a metadata service — lets the query planner skip S3 reads entirely for partitions that can't satisfy a filter, without the overhead of a traditional secondary index.

## Prerequisites

- **Columnar storage basics**: Know why storing columns contiguously (vs rows) enables efficient compression and reduces I/O for analytical projections.
- **MPP query execution**: Know the scatter-gather pattern where a coordinator shards work across N nodes and each node independently processes its fragment.
- **Object storage semantics**: S3 (and equivalents) offers strong per-object consistency — a PUT is atomic and immediately visible — but there are no partial updates and no in-place overwrites.
- **MVCC fundamentals**: Multi-version concurrency control gives readers a snapshot of the database without blocking writers by keeping multiple versions of rows. The Snapshot Isolation vs Serializability post in this archive covers the MVCC model; Snowflake applies it at the file level rather than the row level.
- **Write-ahead logging / ARIES** (helpful contrast): Traditional databases write a WAL before modifying pages. Snowflake's approach replaces the entire page-mutation model, not just the WAL.

## The core idea

Traditional shared-nothing data warehouses — Teradata, Redshift classic, MPP Greenplum — co-locate compute and storage. Each machine in the cluster owns both disk and CPU. Scaling means adding machines to both dimensions simultaneously. This couples two concerns that have completely different growth curves: you can't add compute capacity for a spike in query load without also adding storage capacity you don't need, and vice versa.

Snowflake's answer is to decompose the system into three fully independent layers:

1. **Cloud Storage** — immutable columnar files stored in S3 (or GCS / Azure Blob), organized as micro-partitions.
2. **Virtual Warehouses** — pure compute: ephemeral MPP clusters of VMs that read files from cloud storage and process queries. Zero durable local state (only a warm-up SSD cache).
3. **Cloud Services** — the long-lived, multi-tenant brain: query parser, optimizer, transaction manager, metadata store, access control, and billing.

The layers communicate only through S3 object reads/writes and a metadata API. This sounds like obvious cloud hygiene in 2026, but at the time of publication (SIGMOD 2016) it was a genuine departure — most cloud databases were either lifted-and-shifted shared-nothing clusters or HTAP systems trying to serve both OLTP and OLAP from one engine. Snowflake bet everything on "separate the things that need to scale differently."

## Mechanics

### Micro-partitions: the storage atom

Every table is horizontally partitioned into **micro-partitions**: S3 objects containing 50–500 MB of uncompressed data (~16 MB compressed). Within each file, data is organized in PAX (Partition Attributes Across) format: a header block followed by one columnar stripe per column, each compressed independently using encoding chosen per column (dictionary for low-cardinality, delta for timestamps, zstd / Snappy for generic).

Micro-partitions are written once and never modified. An UPDATE or DELETE does not touch an existing file. Instead, it reads the affected micro-partition(s), produces new micro-partition(s) with the changes applied, and writes the new files to S3. The old files persist until the retention window expires.

The file header stores per-column metadata extracted at write time:
- **min and max value** of each column
- **distinct value count estimate** per column
- **total row count** and **compressed byte count**

This per-partition header is extracted at ingest time and stored separately in the Cloud Services metadata store, where the query planner can read it without fetching the file itself. The result is a lightweight, always-current secondary index across the entire table, built for free by the write path.

### Virtual Warehouses: pure-compute elasticity

A Virtual Warehouse (VW) is an independent MPP cluster — anywhere from 1 to 128+ VMs — that executes queries. The user picks a "T-shirt size" (XS to 6XL), which determines the number of nodes. Multiple VWs can run against the same cloud storage data simultaneously with zero coordination, because the data they read is immutable.

**Within a VW**, files are assigned to worker nodes via **consistent hashing over file names**. Each node maintains a local SSD cache (LRU eviction) populated lazily by actual reads. Consistent hashing means:

- The same file reliably maps to the same node across queries, maximizing cache reuse.
- When the VW is resized, only the fraction of files whose hash slot changes need to be re-fetched from S3; unaffected assignments hit the existing cache.
- If a node fails mid-query, the query coordinator reassigns its hash slots to surviving nodes, which fetch the needed files from S3. No data is lost — S3 is the source of truth.

**Between VWs**, there is no shared local state. Two different VWs running against the same table each have their own independent cache. Their reads to S3 are independently consistent (S3 provides strong read-after-write consistency per object). The isolation is absolute: a compute-heavy batch job on VW-A cannot starve a latency-sensitive dashboard on VW-B.

Spilling follows a tiered fallback: main memory → local SSD → remote S3. Intermediate results too large for memory are automatically spilled, making Snowflake resilient to working-set pressure without query failures.

### Cloud Services: the transaction and metadata brain

Cloud Services is the one layer that never pauses. It is a multi-tenant, globally redundant service shared across all Snowflake accounts. It handles:

**Query lifecycle**: Parse → semantic analysis → logical optimization → physical plan generation. During planning, Cloud Services reads the partition metadata catalog to determine which micro-partitions can be pruned.

**Partition pruning**: For a query `WHERE order_date BETWEEN '2025-01-01' AND '2025-01-31'`, Cloud Services fetches the stored min/max for the `order_date` column in every micro-partition of the table. Partitions whose entire range lies outside the predicate are dropped from the plan. The VW never fetches or decompresses those files. For well-ordered data this can eliminate 90%+ of I/O.

**Transaction management — MVCC at the file level**: Snowflake provides Snapshot Isolation. The transaction model is:

1. Every transaction gets a monotonically increasing **transaction timestamp** at start.
2. A "table version" is the ordered set of micro-partition file references valid at a given timestamp.
3. Reads use the table version as of the transaction's start timestamp.
4. Writes produce new micro-partition files and, on commit, atomically update the metadata catalog: the table's current version now points to `[surviving_old_partitions ∪ new_partitions]`.
5. Write-write conflicts are detected by checking whether any file a writer intended to replace has already been replaced by a concurrent committed writer. On conflict, one writer is rolled back.

Because the catalog update is atomic and micro-partitions are immutable, readers and writers never block each other. A long-running analytic query reading the table at timestamp T sees a perfectly consistent snapshot regardless of concurrent writes that have committed after T.

### Time travel and zero-copy cloning

**Time travel** requires no special machinery beyond what MVCC already preserves. Since the metadata catalog records every file-pointer swap with timestamps, querying `SELECT * FROM t AT (TIMESTAMP => '2026-01-01 00:00:00')` resolves to "which set of micro-partitions was current at that timestamp?" and the VW reads those files. The physical S3 objects are identical to what they were then — they're immutable — so the query is correct by construction. Retention is configurable up to 90 days for Enterprise accounts.

**Zero-copy cloning** (`CREATE TABLE clone CLONE source`) copies only the metadata pointer tree. The clone immediately shares all existing micro-partitions with the source. No S3 data is duplicated. Cloning a 100 TB table and cloning a 100 GB table take approximately the same wall time (a few seconds) because both are purely metadata operations. Subsequent writes to either the clone or the source create new micro-partitions independently; shared micro-partitions are kept in S3 until both tables release them.

### Semi-structured VARIANT type

The VARIANT column stores JSON, XML, Avro, or Parquet inline with relational columns. At write time, Snowflake runs a schema-inference pass over the rows in each micro-partition. JSON key paths that appear in more than approximately 50% of rows in the partition are promoted to **virtual columns** — stored columnar alongside the structured columns in the same PAX stripes, with the same min/max indexing. Less-common paths remain in a flexible row-oriented binary blob.

The result: `SELECT data:order_id FROM events WHERE data:order_id > 1000` scans only the extracted `order_id` virtual column — it never decompresses the full JSON blobs. Schema migrations are free for new fields: they simply appear in future partitions and are absent in the metadata of old ones.

## Where it breaks

**Row-level DML write amplification** is the sharpest edge. An UPDATE touching 0.1% of rows in a 1 GB micro-partition still rewrites the entire 1 GB micro-partition, because immutable files cannot be partially updated. Snowflake's own engineering blog documented a MERGE operation on a standard warehouse where <1% of rows were modified yet 86% of scanned bytes (413 GB out of 483 GB) were rewritten. High-frequency small-update patterns (UPSERT streams, CDC sinks with frequent merges) are genuinely expensive.

**S3 commit latency sets the write floor**. Every committed transaction must land all new micro-partitions durably in S3 before the metadata swap. S3 PUT latency sits at roughly 50–200 ms per request. For analytical workloads with large batches this is invisible. For OLTP-style writes (dozens of single-row inserts per second per session), it is prohibitive.

**Cold VW starts eliminate cache benefit**. A VW that has been paused resumes with an empty local SSD cache. First-run queries run at S3 bandwidth speeds, which may be 5–10× slower than warm-cache queries. Latency-sensitive dashboards typically keep a minimum VW running to avoid this.

**Cloud Services is a centralized coordinator**. All metadata reads and writes flow through the Cloud Services tier. Very high concurrency (thousands of simultaneous short queries) can queue behind metadata operations. Snowflake addresses this with query queuing, multi-cluster VW configurations, and dedicated metadata caches, but the centralization is a fundamental architecture choice, not a tunable.

**No HTAP**. Snowflake is an OLAP system. There is no row-level transactional engine for online writes with sub-millisecond latency. Mixed workloads still require a separate OLTP database (Postgres, Aurora) and a CDC pipeline to feed Snowflake. The paper does not pretend otherwise — it explicitly targets "ad-hoc, complex, multi-table queries" and ETL/ELT at scale.

## Why it works

The mechanism is **immutability as a universal enabler**.

An immutable object, once written, has a fixed identity. That identity is stable across time, across readers, and across the entire storage tier. This one property collapses an enormous amount of system complexity:

- **Readers never conflict with writers** — there is no shared mutable file to lock, no read-write latch, no buffer pool page pin.
- **MVCC is just a list of file names at a timestamp** — the transaction state fits entirely in the metadata catalog; no undo log, no rollback segments in the data files.
- **Time travel is trivially correct** — historical snapshots are historical file sets, and the files themselves are unchanged.
- **Zero-copy clone is a metadata memcpy** — because both the source and clone can safely share a file pointer forever (neither can mutate the underlying object).

This is the same principle that makes git commits correct (each commit is a content-addressed immutable tree), Kafka log segments append-only and safe to replicate (segments are never overwritten), Apache Parquet files safely distributable (the file is complete and sealed at write time), and functional persistent data structures cheap to version (structural sharing is possible only when nodes are never mutated).

The second principle is **separating access patterns**: each of the three layers serves a radically different access pattern — tiny frequently-updated metadata (Cloud Services OLTP-style store), large seldom-updated columnar data (S3 object storage), and temporary ephemeral compute state (local SSD cache). By routing each concern to the medium optimized for it, Snowflake avoids the jack-of-all-trades mediocrity of monolithic storage.

The insight from the Snowflake paper that every subsequent cloud data warehouse internalized: "if your storage layer is immutable, you don't need a distributed lock manager, you don't need a buffer pool flush protocol, and your scaling knobs are compute and metadata — not storage."

## Going deeper

1. **The Amazon Aurora SIGMOD 2017 paper** — a complementary immutability insight applied to the OLTP side: only redo log records cross the network instead of full pages, and the storage tier reconstructs pages on demand. Aurora and Snowflake independently discovered that offloading state reconstruction to the storage tier unlocks independent scaling.
2. **Apache Iceberg specification** — the open-table-format that encodes the same micro-partition + metadata-tree pattern as a vendor-neutral standard. Iceberg's snapshot model (catalog → metadata file → manifest list → manifest file → data files) is a direct generalization of Snowflake's catalog model, now used by Delta Lake, Apache Hudi, and every major lakehouse.
3. **DuckLake: SQL as a Lakehouse Format** (already in this archive, 2026-07-16) — the post that comes one step further: once your metadata is just versioned file pointer trees, why not store it in a real SQL database instead of proprietary metadata services? DuckLake and the broader catalog-over-SQL movement are the direct response to a decade of pain maintaining bespoke metadata layers like the one Snowflake describes.
