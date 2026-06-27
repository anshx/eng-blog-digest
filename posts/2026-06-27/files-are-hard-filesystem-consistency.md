---
title: "Files are hard"
source: https://danluu.com/file-consistency/
author: Dan Luu
company: personal blog
date_posted: 2015-11-26
date_digested: 2026-06-27
---

# Files are hard

## What's new to learn

- **The write hierarchy is longer than you think.** Between `write()` returning and bits reaching magnetic platters, there are at least four independently-buffered layers — and each layer has its own durability guarantee (or lack of one). Most application code implicitly assumes the hierarchy is two layers: "in memory" and "on disk." It's not.

- **`fsync()` is a barrier, not a flush.** `fsync()` doesn't flush all pending writes; it enforces ordering — nothing after the `fsync()` returns is observable before everything submitted before it. The common mental model ("I called fsync so my data is definitely on disk") is almost right, but breaks on macOS and on many Linux filesystems under specific journaling modes.

- **Directory entries are separate durability units from file data.** Creating a file, writing to it, and `fsync()`-ing it leaves the *directory entry* in an undurable state. A crash can produce a file whose data is durable but whose name is gone. On several popular filesystems, this also applies to renames: the old name can disappear and the new name fail to appear after a crash.

## Prerequisites

- Familiarity with the basic POSIX file API: `open`, `write`, `read`, `fsync`, `rename`, `close`.
- A rough understanding of what the OS kernel "page cache" is — a write-back in-memory cache of disk blocks managed by the kernel.
- Basic awareness that disks (and SSDs) have their own write buffers that are separate from the kernel's.

The post assumes no prior knowledge of crash-recovery theory, journaling, or database internals — but knowing what a WAL (write-ahead log) is will make the "Going Deeper" section more accessible.

## The core idea

When application code calls `write(fd, buf, n)`, the operating system copies `buf` into the *page cache* — a pool of kernel-managed DRAM pages that serve as a write-back cache for the disk. The system call returns to the caller as soon as the page cache is updated; no disk I/O has happened. Eventually the kernel's pdflush/writeback daemon will push dirty pages to the disk, but on its own schedule — possibly seconds or even minutes later.

Above the disk sits another buffer: the disk controller (or SSD) has its own volatile DRAM write cache. When the kernel issues a write command to the device, the device acknowledges as soon as data lands in *that* buffer, not when bits are safely on the storage medium. On HDDs this distinction matters during power failures; on SSDs it matters on power loss or controller failures.

So the real hierarchy is:

```
Application buffer
      ↓
OS page cache           ← where write() lands
      ↓
Disk controller cache   ← where write-to-device lands
      ↓
Persistent storage      ← the actual medium
```

`fsync(fd)` is the POSIX call that forces data all the way down to persistent storage. It blocks until the kernel has issued writes for all dirty pages of `fd` AND the disk reports they are stable. This is expensive: a single `fsync` on a spinning disk can take 5–30 ms.

The catch: **`fsync` on Linux only drains the kernel page cache to the device; it relies on the device to report stable storage accurately.** If the device lies (and some consumer SSDs and HDDs do, to improve benchmarks), data can be lost. On macOS, `fsync` doesn't even drain to the device — it only flushes the page cache. Achieving true durability on macOS requires `fcntl(fd, F_FULLFSYNC)`.

Now layer in the second surprise: **directory entries.** When you create a file, you are actually making two distinct durability units: the inode (which holds the file's data and metadata) and the directory entry (which maps a name to that inode). These are separate blocks on disk. You can call `fsync(fd)` and flush the inode perfectly, but the directory entry remains in the page cache. A crash at the wrong moment removes the name; the inode becomes orphaned.

The fix is to also call `fsync(dirfd)` where `dirfd` is an open file descriptor to the parent directory. This is non-obvious because directories are not "files" in the everyday programmer's mental model, yet they are first-class durability units in the OS.

The same logic applies to `rename`. The classic pattern for atomic file updates is:

```c
fd = open("tmp.XXXX", O_WRONLY | O_CREAT);
write(fd, new_content, len);
fsync(fd);
close(fd);
rename("tmp.XXXX", "target");
```

The `rename` is atomic in the POSIX sense — readers either see the old file or the new file, never a partial file — but *the rename itself is not durable* until the parent directory is fsynced. Without `fsync(dirfd)`, a crash can leave the system with neither file: the old file's directory entry was replaced by the new one in the rename, but neither entry was flushed to disk.

Correct version:

```c
fd = open("tmp.XXXX", O_WRONLY | O_CREAT);
write(fd, new_content, len);
fsync(fd);
close(fd);
rename("tmp.XXXX", "target");
dirfd = open(".", O_RDONLY);
fsync(dirfd);
close(dirfd);
```

## Mechanics

**Ext3 journaling modes.** Ext3 (and ext4) supports three journaling modes that trade performance for durability:

- `data=writeback` — only metadata is journaled. File data and metadata can be written in any order. After a crash, a file can contain blocks from before the crash interleaved with blocks from after the crash. The journal guarantees only that metadata is consistent.
- `data=ordered` (the default in most distributions) — data blocks are written to disk *before* the metadata journal entry that references them. This prevents old data from appearing in new files. It is the safe default.
- `data=journal` — both data and metadata are journaled. Maximum safety, lowest throughput.

Ext3 in `writeback` mode is particularly dangerous because the documentation is sparse and misleading, and the mode is the default on several distributions. An `fsync` on ext3 `writeback` does not guarantee that data wrote before metadata — a subsequent crash can produce files where new metadata points to old data.

**The Pillai et al. study (2014).** The post cites a study by Pillai, Chidambaram, Bautin, and Arpaci-Dusseau, "All File Systems Are Not Created Equal: On the Complexity of Crafting Crash-Consistent Applications" (OSDI 2014). The researchers built a tool called BOB (Block Order Breaker) that replays block-level filesystem traces with permuted ordering to find crash-consistency bugs. Their findings:

| Application | Bugs found |
|---|---|
| LevelDB | Yes — multiple bugs in key paths |
| SQLite | Yes — then fixed upstream |
| HDFS | Yes — multiple bugs |
| PostgreSQL | Yes (later versions fixed) |
| GDBM | Yes |

Most bugs had the same root cause: the application called `fsync` on files but not on parent directories, or assumed that `rename` was durable without a subsequent directory `fsync`, or assumed that data was ordered before metadata on all filesystem modes.

**How databases handle this correctly:**

- *SQLite* uses a WAL (write-ahead log) and calls `fsync` at carefully orchestrated points: once on the WAL before updating the main database file, once on the main database file, once on the directory when creating the WAL. SQLite also opens files with `O_SYNC` on platforms where `fsync` is unreliable.
- *LevelDB / RocksDB* (post-fix) uses an explicit `sync` option and calls `fsync` on both the log file and the directory when creating a new SST file.
- *PostgreSQL* calls `fsync` on checkpoint and uses `sync_file_range` for staged flushing, plus a directory sync on the data directory during recovery.

**The macOS corner case.** The macOS HFS+ and APFS filesystems do not honor `F_FULLFSYNC` by default on many storage devices, and standard `fsync` is documented as a best-effort hint that may not reach the drive. Applications that need real durability on macOS must use `fcntl(fd, F_FULLFSYNC)` — which is rarely documented or used. This is why many macOS database deployments silently have lower durability than their authors believe.

## Where it breaks

**Performance cost of correctness.** An `fsync` is one of the most expensive system calls: it serializes all pending writes and waits for the device to confirm. On a 7200 RPM HDD, a single `fsync` takes ~10 ms — that's 100 durable writes/second. On a datacenter NVMe SSD without a write cache, `fsync` might take 30–150 µs. This is why most databases batch writes and sync at controlled checkpoints, not on every `write()`.

**Filesystems that lie.** Some consumer SSDs and cloud block devices acknowledge `fsync` before data has reached stable storage. This is legal on the "best-effort" interpretation of POSIX but breaks the assumptions of any crash-consistent application. The post notes this is especially common on products marketed for "high speed." There is no portable way to detect this from user space.

**Not generalizing to distributed FS.** The rules above apply to POSIX local filesystems. Distributed filesystems (HDFS, GFS, S3) have completely different durability contracts — in particular, S3 `PutObject` is durable once the 200 response is received, but all the ordering constraints above disappear because there is no shared page cache.

**Journaling mode surprises.** Many system administrators run ext3 in `data=writeback` mode for performance without realizing the ordering guarantees they're giving up. Application code that was tested on `data=ordered` can silently become unsafe on `data=writeback`.

## Why it works

The deeper principle here is that **`fsync` is just a happens-before barrier for the storage subsystem** — the single-machine analogue of what `commit()` in a distributed database or `barrier()` in an MPI program does.

In distributed systems, we distinguish between "a write was sent to the network" and "the write is durably committed to a majority of nodes." The gap between those two points is where the system can produce inconsistent state if a crash happens. The resolution is explicit coordination: a commit protocol that acknowledges only after durability is confirmed.

On a single machine, the gap is between "write() returned" and "data is stable on the medium." Every layer of the write hierarchy introduces a window where a crash produces inconsistency. `fsync` closes that window by establishing a synchronization point — everything submitted before `fsync` is guaranteed to precede everything submitted after, in the view that a crash recovery would see.

This is precisely the definition of a barrier in memory-ordering literature: it prevents reordering across the barrier point. The same concept appears as:

- `MFENCE` / `SFENCE` in x86 memory ordering (prevents CPU reorder of stores)
- `msync()` in memory-mapped file I/O
- `barrier()` in Linux kernel driver code
- `OrderedCommit` in Raft (must persist log entry before acknowledging to leader)
- Kafka's `acks=all` mode (must persist to ISR before acknowledging to producer)

The fundamental insight: **any system with multiple buffering layers between "write" and "durable" needs explicit synchronization points between layers.** Whether those layers are CPU caches, an OS page cache, a disk write buffer, a network, or a cluster of machines, the shape of the problem and the shape of the solution are identical. `fsync` is not a file system arcana — it is a barrier instruction for the storage memory hierarchy.

The second key insight is that **named resources (files, directory entries) are different durability units.** In a distributed system this maps to: "the transaction committed" vs. "the catalog entry for the new table was committed." Both must be durable before you can claim the operation is complete. This exact issue caused the PostgreSQL fsync saga (2018), where a single `fsync` error could silently mark dirty pages as clean — the same "multiple durability units, need to sync both" problem.

## Going deeper

- **Pillai et al., "All File Systems Are Not Created Equal" (OSDI 2014)** — the study that quantified crash-consistency bugs in LevelDB, SQLite, HDFS, and others. The BOB tool and methodology are described in full; excellent for anyone who wants to test their own storage code: https://www.usenix.org/conference/osdi14/technical-sessions/presentation/pillai

- **"Ensuring Data Reaches Disk" (LWN.net)** — a comprehensive review of every Linux mechanism for reaching the disk (sync, fsync, fdatasync, sync_file_range, O_SYNC, O_DSYNC) with explanations of when each is and isn't sufficient: https://lwn.net/Articles/457667/

- **PostgreSQL's documentation on "Reliability and the Write-Ahead Log"** — explains exactly what PostgreSQL does at each checkpoint and why, with specific syscall sequences and the rationale for each. Reading this alongside the Pillai paper is the fastest way to deeply understand crash-consistent application design: https://www.postgresql.org/docs/current/wal-reliability.html
