---
title: "The Google File System"
source: https://research.google.com/archive/gfs-sosp2003.pdf
author: Sanjay Ghemawat, Howard Gobioff, Shun-Tak Leung
company: Google
date_posted: 2003-10-19
date_digested: 2026-07-25
---

# The Google File System

## What's new to learn

1. **Lease-based primary selection**: Instead of coordinating every write, GFS grants a time-bounded authority token (60-second lease) to one replica—the "primary"—which then orders all mutations unilaterally for that window. Coordination happens once per lease, not once per write.

2. **Atomic record append**: A filesystem primitive where the offset is chosen by the system, not the client. Multiple concurrent writers can append to the same file without any client-side synchronization; each successful append occupies a unique, non-overlapping region.

3. **Defined vs consistent**: A two-level consistency vocabulary that separates "all replicas have the same bytes" (consistent) from "clients see exactly what they wrote" (defined). This lets GFS tolerate replica divergence on failed mutations while still providing semantically useful guarantees to applications.

## Prerequisites

- What replicated state machines do (Raft/Paxos ensure all replicas apply the same sequence of operations)
- What POSIX file semantics require (`write(fd, buf, n)` at an arbitrary offset, concurrent writers to the same file conflict)
- What write amplification means in storage (a single logical write causes multiple physical writes during compaction or replication)
- Basic understanding of how TCP pipelining works (receiver can forward as it receives, not only after it has the full payload)

## The core idea

GFS's most important decision was rejecting POSIX. The POSIX `write()` syscall lets any client overwrite any byte at any offset, which means a distributed filesystem must solve the general concurrent-write conflict problem: what happens when two writers hit the same position at the same time? In a distributed setting, this is expensive—you need to detect the conflict, sequence the writes, and make all replicas agree.

GFS avoided this entirely by observing that Google's actual workloads don't need it. MapReduce outputs, web crawl data, BigTable tablet files, log streams—all are append-heavy, rarely overwritten. So GFS replaced arbitrary overwrite with a single key primitive: **record append** (atomic, concurrent, offset chosen by the filesystem). With this interface:

- Two concurrent appends can never conflict, because the filesystem assigns them different offsets.
- The general write-conflict problem is gone.
- The consistency model can be dramatically simpler.

Everything else in GFS—the large chunk size, the lease protocol, the data/control flow separation, the weak consistency for random writes—flows from this foundational workload assumption.

## Mechanics

### Architecture: master and chunkservers

```
              Client
               │ 1. "Where is chunk X for /file at offset Y?"
               ↓
           [Master]         ← all namespace metadata in RAM; never in I/O path
          /   │   \
         CS₁ CS₂ CS₃       ← chunkservers: store 64 MB chunks as Linux files
```

The master is purely a metadata server. After a client resolves a chunk's location (cached for 60 seconds), it reads and writes directly to chunkservers without touching the master again. This is the same control/data plane separation that Pingora and Netflix's Lightbulb use: the control plane establishes where to go; the data plane handles the actual bytes.

### Large chunk size: 64 MB

A conventional filesystem uses 4–8 KB blocks. GFS uses 64 MB chunks. Why?

- **Metadata fits in RAM.** A 100 TB filesystem has 100 TB / 64 MB ≈ 1.6 million chunks. At ~64 bytes of metadata per chunk (handle, version, replica locations), that's ~100 MB—fits on one master in RAM.
- **Fewer master roundtrips.** A client sequentially reading a 1 GB file touches only 16 chunks. At 4 KB blocks, the same read would need 262,144 block lookups.
- **Amortized TCP connections.** Chunkservers keep connections open; one connection handles an entire 64 MB chunk, spreading the handshake cost across a huge transfer.

Downside: a 1 KB file still occupies a full 64 MB chunk (internal fragmentation), and popular small files create single-chunkserver hot spots with no remedy.

### The write protocol: data flow decoupled from control flow

To write to chunk X at position P:

1. **Client → Master**: "Who holds the lease for chunk X?"
2. **Master → Client**: Primary chunkserver CS₁ + replica locations CS₂, CS₃ (granting a 60-second lease to CS₁ if none exists).
3. **Client → CS₁ → CS₂ → CS₃**: Push data bytes **in a pipeline**, not a broadcast. CS₁ starts forwarding to CS₂ as soon as it receives the first bytes; CS₂ does the same to CS₃. Each server forwards along the nearest-in-topology successor. This exploits TCP pipelining: total latency is proportional to total bytes / bandwidth, not bandwidth × number of replicas.
4. **Client → CS₁ (Primary)**: "WRITE at offset P" (control message, no data).
5. **CS₁ → CS₂, CS₃**: "WRITE at offset P, serial #42" (assigns serial number; all replicas apply mutations in the same sequence).
6. **CS₂, CS₃ → CS₁**: ACK.
7. **CS₁ → Client**: ACK (success) or error.

**The separation**: data bytes travel along a topology-aware chunkserver chain that maximizes network bandwidth usage. The write *order* (serial number) is controlled separately by the primary. These two concerns are completely decoupled. Compare: in Kafka's ISR replication, followers pull data independently while the leader controls which offsets are committed—same pattern.

### The lease: amortizing coordination across writes

The lease is a time-bounded authority token. For 60 seconds, CS₁ is authoritative for ordering mutations to chunk X—no master confirmation needed per write. This turns O(writes) coordination events into O(1)-per-lease.

If CS₁ fails mid-lease, the master waits for the lease to expire (at most 60 seconds) before granting a new lease to another replica. This ensures at most one primary is active at any time without distributed consensus—the master is the sole lease authority.

The pattern generalizes: Raft leader election is an unbounded lease refreshed by heartbeats. Spanner's TrueTime commit-wait is paying off a lease on time itself. ZooKeeper session timeouts are leases on client state. In all cases, a time-bounded authority token replaces per-operation coordination.

### Atomic record append: the killer primitive

Client → Primary: `APPEND data D`

Primary:
1. Computes `offset = current_end_of_chunk`.
2. If `offset + |D| > 64 MB`: pads the chunk to 64 MB, returns "retry in next chunk" to client.
3. Else: issues `WRITE(D, offset)` to all replicas with serial number.
4. All replicas ACK → Primary sends `offset` back to client: "your data is at byte offset `offset`."

Multiple clients calling `APPEND` concurrently get serialized at the primary. Each gets a distinct, non-overlapping region. No client-side locking needed.

**Failure case**: if the primary ACKs some replicas but not all before failing, the client retries. The retry writes the same bytes at a *new* (higher) offset. Result: the bytes appear twice on replicas that had already written them, at different offsets. GFS's guarantee is **at-least-once**: successful appends are in a defined region, but failed-then-retried appends may create "padding" at earlier offsets on some replicas. Applications must tolerate duplicates—typically by embedding a sequence number in each record.

### The consistency model in full

| Mutation type | All replicas agree? | Matches what writer intended? |
|---|---|---|
| Serial write (one client) | Yes (consistent) | Yes (defined) |
| Concurrent random writes | Yes (consistent) | No—interleaved fragments (consistent but undefined) |
| Record append (success) | No—some replicas have padding at failed offsets | Yes for defined regions |
| Failed mutation | No (inconsistent) | No |

GFS carefully distinguishes **defined** (clients see what the writer wrote) from **consistent** (all replicas agree). POSIX requires both for every write. GFS only guarantees both for serial writes and successful record appends—and trades the rest for simplicity and performance.

### Copy-on-write snapshot

```
snapshot("/data/crawl")
```

Master:
1. Revokes all outstanding leases on affected chunks (waits for any in-flight writes to drain).
2. Logs the snapshot op to its operation log (on disk, replicated).
3. Copies the namespace entry in RAM—just the chunk handles, not the bytes. A reference counter on each chunk handle is incremented. O(namespace size), not O(data size).

First write to a snapshotted chunk:
1. Master sees the chunk's refcount > 1.
2. Asks a chunkserver to copy the bytes locally (same-machine memcpy if possible—no network transfer).
3. Grants a new lease on the new chunk to proceed with the write.

This is classic copy-on-write: defer byte copying until the first mutation after a snapshot. Same mechanism as Unix `fork()`, ZFS clones, and Postgres page-level COW on checkpoint.

## Where it breaks

**Single-master scalability.** The master serves all namespace metadata from RAM. At petabyte scale with billions of small files, the master becomes the bottleneck—both memory (metadata doesn't fit) and CPU (too many namespace operations per second). Google eventually replaced GFS with Colossus, which uses Bigtable to shard namespace metadata across multiple servers.

**Small-file inefficiency.** A 1 KB file occupies a full 64 MB chunk (internal fragmentation factor 65,536×). GFS works beautifully for large sequential files and fails for small-file workloads (billions of 1 KB keys, for instance—which is exactly what Bigtable is designed to handle layered on top of GFS).

**Hot spots on popular files.** A 10 MB file used by thousands of simultaneous MapReduce tasks maps to one chunk → one chunkserver → serial I/O. GFS acknowledges this as a known limitation and suggests increasing replication for hot files, but provides no automated solution.

**At-least-once, not exactly-once.** Record append retries create duplicate records. Every consumer must deduplicate via an application-level sequence number—pushing complexity down to every writer and reader. Google's production code universally includes this, but it's a non-trivial contract.

**Weak consistency for concurrent random writes.** If two clients write concurrently to overlapping byte ranges, the result on any given replica depends on which write arrived first. Replicas may disagree. GFS does not define what happens and offers no mechanism to prevent it. Applications that need strong consistency on random writes (e.g., traditional databases) cannot use GFS as-is.

## Why it works

The root principle: **interface contracts determine implementation complexity.**

When POSIX says "write at any offset," the implementation must solve the general concurrent-write conflict problem—a hard distributed coordination problem. When GFS says "you can append; we choose the offset," the concurrent-write problem disappears entirely. The client loses a tiny bit of control (it can't pick the offset), but the system designer gains enormous freedom.

This same reduction appears everywhere:

- **Kafka** is fast because appending to an immutable segment file is trivially concurrent and replicable. Consumers track their own offsets—the broker never needs to resolve write conflicts.
- **LSM trees** convert random writes to sequential by only appending to a memtable and immutable SSTables. Compaction sorts retroactively. No in-place mutation = no concurrent write conflict = no locking on the write path.
- **CRDTs** are coordination-free by restricting operations to commutative, associative primitives (grow-only sets, max-register). The restriction on the *interface* eliminates the need for coordination in the *implementation*.
- **Parquet / ORC / Dremel** columnar formats are write-once and then read-many. No mutation support = no consistency model needed on the storage side = readers are trivially consistent.
- **Event sourcing** scales because the event log is append-only. Projections are computed forward from the log. No competing writers on the log = no conflicts.

GFS also teaches the **lease amortization pattern**: coordination is expensive, but it doesn't have to happen per-operation. A time-bounded authority token converts O(n) coordination events into O(1) per lease window, at the cost of a bounded recovery delay when the authority fails. This trade-off (latency in the rare failure case vs. throughput in the common success case) shows up identically in Raft leader heartbeats, Spanner commit-wait, ZooKeeper sessions, and database lock timeouts.

## Going deeper

1. **HDFS Architecture** (https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-hdfs/HdfsDesign.html) — GFS's open-source clone built for Hadoop. NameNode = master; DataNodes = chunkservers. The architecture is nearly identical; comparing the two shows which design decisions were fundamental and which were implementation choices.

2. **"Ceph: A Scalable, High-Performance Distributed File System"** (OSDI 2006) — removes the single-master bottleneck by replacing the master's chunk-location map with CRUSH, a deterministic function that any client can compute locally. No lookup needed; clients derive placement directly from the cluster map.

3. **"Finding a needle in Haystack: Facebook's photo storage"** (OSDI 2010) — a different answer to the same small-file problem. Where GFS abandoned small files to Bigtable, Haystack packs millions of small photos into a single large logical file with an in-memory index, showing how the large-file primitive can still serve small-file workloads with one more layer of indirection.
