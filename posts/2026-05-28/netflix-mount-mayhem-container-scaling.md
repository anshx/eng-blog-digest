---
title: "Mount Mayhem at Netflix: Scaling Containers on Modern CPUs"
source: https://netflixtechblog.medium.com/mount-mayhem-at-netflix-scaling-containers-on-modern-cpus-f3b09b68beac
author: Harshad Sane, Andrew Halaney
company: Netflix
date_posted: 2026-03-01
date_digested: 2026-05-28
---

# Mount Mayhem at Netflix: Scaling Containers on Modern CPUs

## What's new to learn

- **VFS seqlock as a global write barrier**: The Linux kernel protects the mount namespace with a single sequence lock (`mount_lock`). Every `mount()` and `umount()` syscall must acquire the exclusive write side of this lock, serializing all concurrent mount operations across the entire system regardless of how many CPUs are available.
- **NUMA-amplified lock contention**: On multi-socket (NUMA) servers, a contested lock cache line must be transferred between NUMA nodes on every acquisition. Moderate lock contention—harmless on a single socket—becomes a catastrophic bottleneck on dual-socket machines because the interconnect latency (100–200 ns) is paid on every lock handoff, and Intel TMA shows 95.5% of pipeline slots stalling.
- **O(n) → O(1) mount reduction via `fsconfig()` + `lowerdir+`**: The Linux 5.2 mount API lets OverlayFS accept lower-directory layers as already-resolved file descriptors rather than as path strings. Passing an FD bypasses path resolution entirely, eliminating the per-layer bind-mount lifecycle and collapsing mount syscalls from O(image_layers) to O(1) per container.

## Prerequisites

- **OverlayFS / union filesystems**: Docker/OCI containers present a merged filesystem by stacking read-only image layers beneath a writable upper layer. OverlayFS implements this in the Linux kernel; each image layer becomes a `lowerdir`.
- **Bind mounts and mount namespaces**: A bind mount re-exposes a directory tree at a different path within the VFS. User-namespaced containers use _idmapped mounts_ (bind mounts with a UID/GID translation table) so that root inside the container maps to an unprivileged host UID.
- **NUMA (Non-Uniform Memory Access)**: Multi-socket servers give each CPU socket its own local memory and cache hierarchy. Accessing memory or cache lines owned by another socket costs 2–5× more latency than a local access.
- **seqlocks**: A Linux primitive that lets readers proceed without blocking (retrying if a write occurred mid-read) but requires writers to take an exclusive lock. `mount_lock` in the kernel is a seqlock; `mount()` and `umount()` are writers.

## The core idea

Netflix modernised its container runtime but discovered that starting hundreds of containers in parallel caused performance to collapse—not in Kubernetes scheduling, not in containerd, but deep in the Linux kernel's Virtual Filesystem (VFS).

The culprit: every container startup performs O(n) bind mounts (one per image layer) just to hand those paths to OverlayFS, then immediately unmounts them. All those `mount()` and `umount()` calls are writers on the global `mount_lock` seqlock. With 64+ cores racing to call `mount()` simultaneously—easily 20,000+ syscalls per burst—every CPU serialises on that one lock.

On a dual-socket server the situation is dramatically worse. Each lock acquisition causes the lock's cache line to migrate across the NUMA interconnect from the last owner's socket to the new acquirer's socket. At high concurrency this reduces to continuous cache-line ping-pong at interconnect latency, and the kernel ends up spending almost all its time in the `pause` spin loop inside `path_init()` rather than doing real work.

The fix is architectural: the new Linux mount API (`fsconfig()` + `lowerdir+`) allows OverlayFS to accept lower directories as already-open file descriptors instead of path strings. File descriptors are resolved references; no path walk is needed, so `mount_lock` is never taken. One OverlayFS mount replaces O(n) bind mounts, and the lock falls off the flamegraph entirely.

## Mechanics

### How container rootfs creation worked (the slow path)

When containerd creates a container it needs a merged filesystem view of all the image layers. The old procedure, in pseudocode:

```
for each layer in image (n layers):
    bind_mount(layer_path, scratch_dir/layer_N, flags=IDMAP, uidmap=...)
    # O(n) mount() syscalls, each takes mount_lock write side

mount("overlay", rootfs, options=f"lowerdir={scratch_dir/layer_0}:{scratch_dir/layer_1}:...,upperdir=...")
# One more mount() syscall

for each layer in image:
    umount(scratch_dir/layer_N)
    # O(n) umount() syscalls, each takes mount_lock write side
```

Total kernel lock acquisitions per container: **2n + 1** writes on `mount_lock`.

A typical container image has 10–20 layers. With 100 containers starting simultaneously that is 2,000–4,000 exclusive write-lock acquisitions all racing on a single seqlock.

### The seqlock bottleneck in the kernel

The Linux kernel's path resolution entry point `path_init()` checks the mount namespace under the read side of `mount_lock`. A read-side seqlock check looks like:

```c
do {
    seq = read_seqbegin(&mount_lock);
    ... resolve path component ...
} while (read_seqretry(&mount_lock, seq));
```

If a writer (`mount()` or `umount()`) holds the write lock when a reader calls `read_seqbegin()`, the reader's loop body runs but `read_seqretry()` returns true, and the reader retries. With many concurrent writers, readers spin continuously, burning CPU in a tight retry loop. This is what Netflix observed: CPUs spending nearly all their time executing `pause` inside `path_init()`.

Intel's Topdown Microarchitecture Analysis (TMA) made the root cause unambiguous:
- **95.5% of pipeline slots stalled** on memory-bound operations
- **57% of slots** attributed to false sharing—multiple cores contending on the same cache line

### NUMA makes it exponentially worse

On a single-socket machine, all cores share the last-level cache (LLC). When Core A releases `mount_lock`, Core B (on the same socket) can acquire it from the shared LLC with a few nanoseconds of coherence latency. Contention is costly but bounded.

On a dual-socket NUMA machine (e.g., AWS r5.metal with two Intel Xeon sockets connected by UPI), the picture changes:

1. Core A (socket 0) holds `mount_lock`; the cache line lives in socket 0's LLC.
2. Core B (socket 1) tries to acquire the lock. It must send a request over the UPI interconnect to socket 0, invalidate socket 0's copy, and transfer the cache line to socket 1's LLC.
3. Core A (socket 0) tries to re-acquire immediately after releasing. The cache line is now in socket 1's LLC—another cross-socket transfer.

Each of these transfers takes ~100–200 ns. At 20,000 lock operations per second with 64+ cores all competing, the throughput of useful work approaches zero. Modern single-socket instances (AWS m7i.metal or m7a.24xlarge) do not have this problem: all cores share one LLC, so lock hand-offs are 5–10× cheaper.

This explains a counterintuitive observation Netflix made: **upgrading to newer, larger single-socket CPUs outperformed the older dual-socket servers** for container startup workloads, even though the dual-socket machines had more total cores and memory bandwidth.

### The fix: O(n) → O(1) via `fsconfig()` + `lowerdir+`

Linux 5.2 introduced a new, composable mount API that separates filesystem configuration from mounting:

```
# New API: configure OverlayFS with FDs directly, no bind mounts needed
fd0 = open(layer_0_dir, O_RDONLY | O_PATH)
fd1 = open(layer_1_dir, O_RDONLY | O_PATH)
...
fs_fd = fsopen("overlay", FSOPEN_CLOEXEC)
fsconfig(fs_fd, FSCONFIG_SET_FD, "lowerdir+", fd0, 0)  # supply layer as FD, not path
fsconfig(fs_fd, FSCONFIG_SET_FD, "lowerdir+", fd1, 0)
...
fsconfig(fs_fd, FSCONFIG_CMD_CREATE, NULL, NULL, 0)
mnt_fd = fsmount(fs_fd, FSMOUNT_CLOEXEC, ...)
move_mount(mnt_fd, "", AT_FDCWD, rootfs, MOVE_MOUNT_F_EMPTY_PATH)
```

An `open(O_PATH)` syscall resolves the path once to a vnode reference (a file descriptor) without taking `mount_lock`. All subsequent operations use that already-resolved reference. OverlayFS receives the layers as FDs and never performs path resolution itself.

Result: the entire container rootfs is created with **O(1) `mount_lock` acquisitions**—exactly one, for the final OverlayFS mount—regardless of how many layers the image has. In Netflix's flamegraphs, mount-related operations went from dominating the profile to being invisible without explicit highlighting.

### Hardware-level mitigation

Independently of the software fix, Netflix found that routing container-heavy workloads to single-socket instance types (AWS m7i.metal, m7a.24xlarge) dramatically improved throughput. This is a practical lesson: when a workload's critical path hits a global lock, the NUMA topology of the hardware determines how badly that lock scales. Workload-to-hardware placement is a valid mitigation when a software fix requires an OS update.

## Where it breaks

- **Kernel version dependency**: `fsconfig()` with `lowerdir+` requires Linux 5.2+, and containerd needs a corresponding update. Older kernels must stay on the O(n) path.
- **Runtime OverlayFS overhead is separate**: The fix eliminates _startup_ contention. OverlayFS path lookups at container runtime still traverse all n lower directories; deeply-layered images remain slower for filesystem operations like `stat()` on files that need to be checked through many layers.
- **Not a cure for all global kernel locks**: `mount_lock` is one of many global kernel locks. Similar patterns exist in the dcache (directory entry cache), inode tables, and the network socket layer. The fix does not help workloads whose bottleneck is a different global lock.
- **Single-socket is not always the right hardware**: Workloads that are memory-bandwidth-bound (not lock-bound) often benefit from multi-socket NUMA. The hardware fix is specific to high-concurrency mount-heavy workloads.

## Why it works

The deeper principle is: **a global lock + high concurrency + NUMA topology produces super-linear performance degradation**.

A standard seqlock is designed for a world where either (a) write frequency is low (readers dominate) or (b) all cores share a cache. The Linux mount namespace lock was designed in the single-socket era. On dual-socket NUMA machines, the lock behaves as a distributed object: every write acquisition requires a remote cache-line transfer, which is orders of magnitude more expensive than a local L3 hit.

The pattern is universal in kernel engineering:
- **Fork scalability**: Linux's page table lock, copied during `fork()`, serialises mass forking on NUMA machines. Solutions involve copy-on-write range locking.
- **SLAB allocator**: The early SLAB allocator had per-object-class global locks; SLUB replaced them with per-CPU partial lists to avoid cross-CPU contention.
- **Network RX path**: The `sk_receive_queue.lock` in the socket receive path is a classic contention point under high packet rates, solved by SO_REUSEPORT spreading connections across CPUs.

In each case, the architecture pattern is the same: _shared mutable state under a global lock, exercised at high frequency_. The standard remedies are also the same: (1) eliminate the need for the lock by using already-resolved references (FDs instead of paths), (2) partition the state per-CPU to make lock domains local, or (3) route workloads to hardware where the "global" lock is actually local.

The O(n) → O(1) fix is a special case of a broader principle: **prefer handles over names**. A file descriptor is an already-resolved, stable reference. A pathname requires re-resolution through a shared namespace on every use. Any time you can pass a pre-resolved handle instead of a name, you avoid re-entering the namespace and taking its lock. This principle appears in: passing open FDs across exec/fork instead of re-opening by path; using inode numbers in database WAL instead of filenames; passing object references in RPC instead of identifiers that require a lookup.

## Going deeper

1. **Linux VFS internals — lwn.net's "A new API for mounting filesystems" (2018–2019 series)**: David Howells' lwn.net articles documenting the design and rationale behind `fsopen()`, `fsconfig()`, `fsmount()`, and `move_mount()`. Essential for understanding why the new API eliminates path resolution.
2. **"Linux NUMA Basics" — kernel.org/doc**: The official kernel documentation on NUMA scheduling, memory placement, and the numactl tooling. Explains why cache-line transfer costs differ between intra- and inter-socket accesses.
3. **"Seqlocks in the Linux kernel" — lwn.net (2003, Paul McKenney)**: The original lwn.net article introducing seqlocks, explaining their read-retry semantics and the constraints they place on write-side users. Timeless for understanding why write-heavy seqlock workloads degrade under contention.
