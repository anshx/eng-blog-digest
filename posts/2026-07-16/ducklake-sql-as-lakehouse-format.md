---
title: "DuckLake: SQL as a Lakehouse Format"
source: https://duckdb.org/2025/05/27/ducklake
author: Hannes Mühleisen, Mark Raasveldt (DuckDB team)
company: DuckDB / MotherDuck
date_posted: 2025-05-27
date_digested: 2026-07-16
---

# DuckLake: SQL as a Lakehouse Format

## What's new to learn

1. **File-based ACID is a category error** — Iceberg, Delta Lake, and Hudi all try to implement transactional metadata management by carefully writing JSON files to object storage and using atomic file-rename tricks. DuckLake's bet: this is the wrong substrate. A real RDBMS catalog gives you serializable transactions, multi-table atomicity, and indexing for free.

2. **Lakehouse metadata is just a relational schema** — Every table format (snapshots, manifests, partition mappings, statistics, schema history) maps cleanly onto SQL tables. Once you accept this, you get time-travel, zero-copy cloning, and multi-table transactions by querying with WHERE snapshot_id = …, not by reinventing MVCC in JSON.

3. **The "separation of concerns" in storage formats** — Data and metadata have different access patterns (large sequential scans vs. small random transactional reads) and different consistency requirements. Routing each to its natural substrate — object storage for data, a database for metadata — eliminates an entire class of engineering complexity.

## Prerequisites

- Basic understanding of what Apache Iceberg or Delta Lake does (they store tables as immutable Parquet files plus a transaction log)
- What "object storage" (S3, GCS) means: durable, cheap, eventually consistent, no atomic rename across prefixes
- What "ACID transactions" means: atomicity, consistency, isolation, durability
- Optional: how Iceberg's optimistic concurrency control works (compare-and-swap on a metadata pointer)

## The core idea

Every existing lakehouse table format — Iceberg, Delta Lake, Apache Hudi — shares the same architecture: data files live in Parquet on S3, and metadata (which files belong to which snapshot, schema history, partition statistics) is stored as a chain of JSON files, also on S3. Commits happen by atomically swapping a pointer to the new head JSON file. This is essentially hand-rolling an append-only log + optimistic concurrency control on top of object storage.

DuckLake's thesis: stop re-implementing the database. Use a real SQL database for metadata. Keep data files in Parquet on object storage. Wire them together.

The split is:

| Layer | What goes there | Why |
|-------|-----------------|-----|
| Object storage (S3/GCS) | Data files (Parquet) | Large sequential writes, cheap at scale, immutable blobs |
| SQL catalog DB (Postgres / DuckDB / SQLite) | Schema, snapshots, file manifests, statistics, transaction log | Small random reads/writes, transactional, indexed |

The catalog database is a standard relational schema: a `snapshots` table, a `data_files` table, a `schemas` table, a `statistics` table. A new commit is a SQL transaction that inserts rows — it's serializable by the database's own transaction manager, requiring no file renaming, no compare-and-swap, no retry loops.

## Mechanics

### What the catalog schema looks like

At the core, DuckLake maintains something conceptually like:

```sql
CREATE TABLE snapshots (
    snapshot_id  BIGINT PRIMARY KEY,
    table_id     BIGINT,
    parent_id    BIGINT,          -- for time travel chaining
    created_at   TIMESTAMPTZ,
    schema_id    BIGINT
);

CREATE TABLE data_files (
    file_id      BIGINT PRIMARY KEY,
    table_id     BIGINT,
    snapshot_id  BIGINT,
    file_path    TEXT,            -- S3 URI
    row_count    BIGINT,
    file_size    BIGINT,
    -- per-column statistics stored here or in a sibling table
    col_min      JSONB,
    col_max      JSONB
);
```

Writing a new batch of data:
1. Write Parquet files to S3 (can be done by any engine).
2. Open a SQL transaction on the catalog.
3. `INSERT INTO snapshots …` — create the new snapshot row.
4. `INSERT INTO data_files …` — register each new file.
5. Commit the SQL transaction. Done. No JSON, no file rename.

### Time travel

Query a past snapshot:

```sql
SELECT * FROM my_table AT (SNAPSHOT_ID => 42);
-- or by timestamp:
SELECT * FROM my_table AT (TIMESTAMP => '2025-05-01 12:00:00');
```

The engine looks up which `snapshot_id` was current at that timestamp from the catalog, then reads only the data files registered under that snapshot. Old files are not deleted until a VACUUM that exceeds the retention window — exactly like PostgreSQL's dead tuple visibility rules, but coarser-grained (file-level rather than row-level).

### Zero-copy cloning

Clone a table in O(1) time:

```sql
CREATE TABLE my_table_clone CLONE my_table;
```

The implementation: insert one row into `schemas` (pointing to the same schema as the original) and one row into `snapshots` (pointing to the same current snapshot as the original). No Parquet files are copied. The clone and the original share all existing data files. Diverging writes create new Parquet files under their own snapshot lineage — classic copy-on-write branching.

This is exactly how `git branch` works: a branch is just a pointer to an existing commit object. Creating a branch is free.

### Multi-table transactions

Because the catalog is a real database:

```sql
BEGIN;
INSERT INTO orders SELECT * FROM staging_orders;
UPDATE inventory SET qty = qty - staged_qty FROM staging_inventory;
COMMIT;
```

Both tables are updated atomically. With Iceberg or Delta Lake, multi-table atomicity requires an external transaction coordinator (Nessie, Unity Catalog), because each table's commit is a separate file CAS — they cannot be grouped into a single atomic operation on object storage.

### How query pruning works

Partition and column statistics live in indexed SQL columns. A query like:

```sql
SELECT * FROM events WHERE event_date = '2025-05-01';
```

translates to:

```sql
SELECT file_path FROM data_files
WHERE table_id = :tid
  AND snapshot_id = :current_snapshot
  AND col_min->>'event_date' <= '2025-05-01'
  AND col_max->>'event_date' >= '2025-05-01';
```

The catalog answers with a list of file paths to fetch from S3. This replaces the Iceberg pattern of reading a manifest list file → reading each manifest file → filtering by partition → listing data files — a chain of sequential S3 reads that can number in the thousands for a large table.

### Performance gap

The DuckDB team benchmarks show ~926× faster query planning for small targeted reads and ~105× faster ingestion in cases that stress metadata operations. These numbers reflect the cost structure difference: a SQL catalog answers metadata queries in microseconds with indexed lookups; an Iceberg manifest chain requires one S3 GET per manifest file, each taking 10–30 ms. At 1000 manifest files, that's 10–30 seconds just for planning.

For full-table scans that read every Parquet file anyway, the difference vanishes — the bottleneck shifts to S3 throughput.

## Where it breaks

**Catalog availability is a new dependency.** With pure Iceberg/Delta, everything is in S3 — if your Iceberg catalog service goes down, you can still read the files by parsing the JSON yourself. DuckLake requires the SQL catalog to be available for any read or write. If you're using Postgres, Postgres uptime now matters to query availability.

**The catalog can become a bottleneck at extreme write concurrency.** Object-storage-based formats scale writes by splitting the commit log across independent files (no shared lock). DuckLake inherits the serialization constraints of the underlying database. For workloads with thousands of concurrent writers, a Postgres-based catalog may need careful tuning or sharding.

**Interoperability is limited while the ecosystem is young.** Iceberg can be read by Spark, Trino, Flink, Presto, BigQuery, Redshift Spectrum, Athena, Snowflake, and dozens of others because the metadata format is documented and open. DuckLake's catalog schema is open, but engines haven't implemented a DuckLake reader yet (DuckDB 1.0, MotherDuck, and a growing list). DuckLake does support Iceberg import/export as a migration bridge, but the ecosystem lag is real.

**VACUUM complexity.** Because old files are retained for time travel, you need to run periodic VACUUM to delete expired files from S3 and prune old catalog rows. This is manageable but not free.

## Why it works

The deeper principle: **every system that reimplements ACID semantics from scratch eventually rebuilds a database.**

Iceberg's commit protocol (compare-and-swap on a catalog pointer), Delta Lake's transaction log (sequential commit files with optimistic retry), Hudi's multi-writer protocol (lock manager via ZooKeeper or a catalog service) — all of these are partial, domain-specific implementations of transactional consistency. They have the familiar failure modes: clock skew edge cases, retry storms under contention, multi-table transactions that require external coordinators.

DuckLake draws the boundary at the right abstraction level: ACID transactions are a solved problem for small structured data. Parquet is a solved problem for large columnar data. Use each for what it's actually good at.

The "X is just Y in disguise" formulation:

- A **lakehouse table** is just a relational schema of metadata rows + a set of immutable blobs.
- A **snapshot** is a SQL transaction's commit point.
- **Time travel** is MVCC snapshot reads at a historical transaction ID.
- **Zero-copy clone** is a new branch pointer in the metadata graph.
- **Data pruning** is a WHERE clause on indexed statistics columns.

Once you see it this way, the file-based designs look like accidental complexity — the artifact of implementing these semantics before you were allowed to depend on a relational database.

This is structurally the same as:
- **Event sourcing antipatterns**: teams re-implement projections, MVCC, and snapshots in Kafka because they refuse to use a database — and rediscover all the hard parts of databases.
- **Distributed lock services**: teams implement Zookeeper-based coordination that is a subset of a serializable database.
- **The "use Postgres for everything" movement**: message queues (LISTEN/NOTIFY), cron (pg_cron), search (tsvector) — Postgres already implements ACID, so it wins the "simplest thing that works" competition in many domains.

The transferable heuristic: **before building a consistency mechanism on a substrate that doesn't natively support ACID, check whether a thin layer over a substrate that does is an option.** Often it is, and the resulting system is smaller, more correct, and cheaper to operate.

## Going deeper

1. **The official DuckLake blog post** — the full announcement with architecture diagrams and benchmark details: [duckdb.org/2025/05/27/ducklake](https://duckdb.org/2025/05/27/ducklake)

2. **"Delta Lake: High-Performance ACID Table Storage over Cloud Object Stores" (VLDB 2020)** — the paper describing Delta Lake's file-based commit protocol, which is the approach DuckLake is replacing: what a compare-and-swap on a transaction log looks like in practice, and where it breaks under contention.

3. **Jack Vanlightly's analysis series "Analysing Serverless Data Systems"** — multi-part deep dives into ClickHouse Cloud, Snowflake, and Iceberg's consistency models at [jack-vanlightly.com](https://jack-vanlightly.com/analyses/2024/7/30/understanding-apache-icebergs-consistency-model-part1), grounding the abstractions in real failure scenarios and linearizability proofs.
