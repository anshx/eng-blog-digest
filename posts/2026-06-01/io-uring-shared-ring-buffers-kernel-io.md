---
title: "Efficient IO with io_uring"
source: https://kernel.dk/io_uring.pdf
author: Jens Axboe
company: Meta (Facebook)
date_posted: 2019-03-01
date_digested: 2026-06-01
---

# Efficient IO with io_uring

## What's new to learn

- **Shared-memory ring buffer as user-kernel IPC**: Two lockless circular buffers (Submission Queue and Completion Queue), each owned by exactly one producer and one consumer, replace per-I/O system calls with ordinary memory writes — cutting the fixed per-operation overhead from ~100 ns/syscall to ~1 ns/write.
- **SQPOLL — inverting the polling model**: A dedicated kernel thread continuously polls the Submission Queue so user space can submit I/O by writing to memory alone, with zero syscalls even at the submission side.
- **Registered resources**: Pre-pinning user buffers and pre-populating the kernel file-descriptor table amortize expensive per-operation kernel work (page-table lock, fdtable walk) to zero for the steady-state case.

## Prerequisites

- Linux system call mechanics: what a syscall is, why crossing the user-kernel boundary costs time (mode switch, TLB flush on older CPUs, security mitigations like KPTI)
- Ring buffers / circular queues: head and tail indices, wrap-around via bitmask
- `mmap(2)`: mapping a file or anonymous memory into a process's address space
- `readv`/`writev`: vectored I/O, the scatter-gather pattern
- Basic concurrency: acquire/release memory ordering — the minimal guarantee needed for a single-producer / single-consumer queue

## The core idea

For a modern NVMe SSD capable of 1 million 4 KiB random reads per second, each read takes about 100 µs of storage time. A Linux system call — just the mode switch overhead, before any I/O happens — takes roughly 100 ns. That sounds fine: 0.1% tax per operation.

The problem is that at 1 M IOPS you are making 1 million syscalls per second, burning an entire CPU core on nothing but kernel entry/exit — before the kernel has even touched the storage. And NVMe SSDs keep getting faster.

The older async I/O APIs made this worse in different ways:

- **`epoll` + `read`**: two syscalls per operation (one to learn the fd is readable, one to read it), plus a context-switch if the thread blocks.
- **Linux AIO (`libaio`)**: fully async, but silently falls back to synchronous for buffered I/O, and tops out around 600 K IOPS before the submission syscall overhead saturates a core.
- **`io_submit` (POSIX AIO)**: each submission is a separate syscall; the design scales poorly.

`io_uring` solves this with a single insight: **treat the kernel as a co-processor you communicate with via shared memory, not via remote procedure calls.**

Two circular ring buffers are mapped into both user space and kernel space via `mmap`. The user writes I/O requests into the Submission Queue (SQ); the kernel reads them, executes them asynchronously, and writes results into the Completion Queue (CQ). A single `io_uring_enter()` syscall can submit hundreds of SQEs at once, amortizing its fixed cost across the entire batch. With the `SQPOLL` flag a kernel thread polls the SQ continuously, eliminating even that one syscall.

## Mechanics

### Setup

```c
// Allocate a ring pair with 256 entries each
int ring_fd = io_uring_setup(256, &params);

// Map both rings into user space
void *sq_ptr = mmap(0, sq_size, PROT_READ|PROT_WRITE,
                    MAP_SHARED|MAP_POPULATE, ring_fd, IORING_OFF_SQ_RING);
void *cq_ptr = mmap(0, cq_size, PROT_READ|PROT_WRITE,
                    MAP_SHARED|MAP_POPULATE, ring_fd, IORING_OFF_CQ_RING);
// SQEs themselves are in a third mapping
void *sqes  = mmap(0, sqe_size, PROT_READ|PROT_WRITE,
                    MAP_SHARED|MAP_POPULATE, ring_fd, IORING_OFF_SQES);
```

After this, user space and the kernel share the exact same pages of physical memory. No copying; no kernel call needed to check for new entries.

### The SQE and CQE data structures

```c
// Submission Queue Entry — one per I/O operation
struct io_uring_sqe {
    __u8   opcode;    // IORING_OP_READV, IORING_OP_WRITEV, IORING_OP_ACCEPT, ...
    __u8   flags;     // IOSQE_IO_LINK, IOSQE_IO_DRAIN, IOSQE_FIXED_FILE, ...
    __s32  fd;        // target file descriptor (or registered-file index)
    __u64  off;       // file offset
    __u64  addr;      // buffer address (or pointer to iovec array)
    __u32  len;       // buffer length (or iovec count)
    __u64  user_data; // opaque tag echoed unchanged in the CQE
};

// Completion Queue Entry — written by the kernel when I/O finishes
struct io_uring_cqe {
    __u64  user_data; // same value set in the SQE — correlates completion to request
    __s32  res;       // return value: bytes transferred, or -errno on error
    __u32  flags;
};
```

`user_data` is the correlation key: you set it to a pointer, an index, or any cookie, and get it back verbatim when the operation completes. No kernel-managed callback table; no allocation.

### Ring synchronization (lock-free, by ownership)

Each ring has a `head` and a `tail`. Ownership is asymmetric:

| Ring | Producer (writes to tail) | Consumer (reads from head) |
|------|--------------------------|---------------------------|
| SQ   | User space               | Kernel                    |
| CQ   | Kernel                   | User space                |

Because each pointer is written by exactly one side, no locking is needed. The only requirement is correct memory ordering:

```c
// User space: publish a new SQE
sq_ring->array[tail & mask] = sqe_index;
atomic_store_explicit(&sq_ring->tail, tail + 1, memory_order_release);

// Kernel: read it
uint32_t tail = atomic_load_explicit(&sq_ring->tail, memory_order_acquire);
```

`store-release` + `load-acquire` is the minimal fence pair that guarantees the kernel sees the SQE contents before it sees the updated tail. No full barrier, no lock, no cache line bouncing.

### Batching — the first level of optimization

```c
// Queue 64 reads into the SQ without any syscall
for (int i = 0; i < 64; i++) {
    struct io_uring_sqe *sqe = io_uring_get_sqe(&ring);
    io_uring_prep_readv(sqe, fd, &iov[i], 1, offsets[i]);
    sqe->user_data = (uint64_t)i;
}
// One syscall submits all 64 and waits for at least 1 completion
io_uring_submit_and_wait(&ring, 1);
```

One `io_uring_enter()` call submits N SQEs. The fixed ~100 ns mode-switch cost is now 100 ns / 64 ≈ 1.6 ns per operation — a 60× reduction in submission overhead at N = 64.

### SQPOLL — zero-syscall submission

```c
struct io_uring_params params = {
    .flags = IORING_SETUP_SQPOLL,
    .sq_thread_idle = 2000,  // kernel thread sleeps after 2 s of inactivity
};
int ring_fd = io_uring_setup(256, &params);
```

With `SQPOLL`, the kernel spawns a dedicated kernel thread whose only job is to spin on `sq_ring->tail`. User space writes SQEs and atomically updates the tail — no `io_uring_enter()` call at all. From the application's perspective, submitting I/O is now just writing to memory.

After `sq_thread_idle` milliseconds of inactivity the kernel thread sleeps; the next submission must call `io_uring_enter()` to wake it. At sustained high IOPS this never happens.

### Registered buffers and files

Every `readv` normally requires the kernel to:
1. Walk the fdtable to look up the `struct file *` for the fd.
2. Call `get_user_pages()` to pin every page of the user buffer in physical memory (so DMA can't land on a page that got swapped out).

Both happen for every operation. `io_uring_register` performs them once at setup time:

```c
// Pin 16 buffers once; reference them by index in SQEs
io_uring_register_buffers(ring_fd, iov_array, 16);

// Pre-populate fdtable; SQEs use IOSQE_FIXED_FILE + an index
io_uring_register_files(ring_fd, fds, n_fds);
```

With registered resources, the per-operation kernel path does an O(1) array lookup instead of a lock + walk.

### Linked SQEs — chaining without roundtrips

```c
struct io_uring_sqe *read_sqe  = io_uring_get_sqe(&ring);
struct io_uring_sqe *write_sqe = io_uring_get_sqe(&ring);

io_uring_prep_readv(read_sqe,  src_fd, &read_iov,  1, 0);
io_uring_prep_writev(write_sqe, dst_fd, &write_iov, 1, 0);

// write_sqe starts only after read_sqe completes successfully
read_sqe->flags |= IOSQE_IO_LINK;

io_uring_submit(&ring);
```

`IOSQE_IO_LINK` forms a chain: the next SQE in the chain is not submitted until the previous one completes (and not submitted at all if the previous one fails). You get sequenced operations without a roundtrip to user space between them.

### Performance numbers (from the original paper)

| Mode                          | Throughput (4 KiB random reads) |
|-------------------------------|--------------------------------|
| Linux AIO (libaio)            | 608 K IOPS                     |
| io_uring, no polling          | 1.2 M IOPS (≈ 2× libaio)       |
| io_uring, SQPOLL + IOPOLL     | 1.7 M IOPS (≈ 2.8× libaio)     |
| io_uring, no-op (theoretical) | 20 M ops/sec                   |

`IOPOLL` (a separate flag) makes the kernel poll the NVMe completion queue directly rather than waiting for a hardware interrupt — the storage analogue of SQPOLL.

## Where it breaks

**SQPOLL burns CPU at idle**: The kernel thread spins even during quiet periods. If your I/O is bursty rather than sustained, you're wasting an entire core. The `sq_thread_idle` timeout is a rough knob; there is no smooth adaptation.

**Buffered I/O and the worker pool**: For cached reads (page-cache hit), io_uring completes inline — fast. For cache misses, it dispatches to an internal thread pool. That pool has a fixed maximum size, is rate-limited by `RLIMIT_NPROC`, and can become a hidden bottleneck under mixed cached/uncached workloads. Cloudflare documented bugs related to this worker pool exhaustion in production.

**io_uring is a major security attack surface**: Because io_uring allows many kernel operations (including file opening, socket accept, and splice) via a ring buffer that any UID-mapped process can write to, it has been fertile ground for privilege escalation bugs. Android disabled io_uring for untrusted apps. Multiple CVEs were filed against io_uring between 2022 and 2025.

**epoll is still competitive for large-message streaming**: For a web server sending multi-KB HTTP responses, one syscall amortizes over thousands of bytes transferred. The crossover point where io_uring's syscall savings exceed its setup overhead depends on request rate and message size; at low rates and large messages, the difference is negligible.

**Linux-only**: BSD has `kqueue`, Windows has IOCP. Building cross-platform high-performance I/O (as TigerBeetle did) requires a portability layer that unifies three different APIs behind a single completion-event abstraction.

## Why it works

The deeper principle: **every "call" across an abstraction boundary has fixed per-call overhead; replace it with a shared data structure to eliminate that overhead.**

This pattern shows up at every level of the stack:

- **DPDK** (Data Plane Development Kit): user space reads directly from NIC ring buffers, bypassing the kernel network stack entirely. Same ring-buffer idea, for network instead of storage, one layer down.
- **GPU command buffers**: you don't issue one CUDA kernel launch at a time; you fill a command buffer and submit it as a batch. The CPU-GPU boundary is expensive to cross, so you cross it rarely.
- **SPSC ring queues** between threads: faster than a mutex-protected shared queue because there is no contention on a shared cache line — each side owns one pointer, just as in io_uring.
- **RDMA** (Remote Direct Memory Access): bypass the OS on *both* ends of a network connection, writing directly to the remote host's memory via DMA. Same principle, extended to a physical network link.

SQPOLL illustrates a second, complementary principle: **polling beats interrupt-driven notification for high-rate workloads, at the cost of a dedicated CPU core.** The same tradeoff appears in:
- Busy-wait spinlocks vs. OS futex-based mutexes (spinlock wins if the wait is short and predictable)
- DPDK polling vs. interrupt-driven NIC drivers
- Reactive (event-loop) vs. proactive (completion-queue) I/O models

The two principles together — shared-memory IPC + polling — are why io_uring's SQPOLL + IOPOLL mode nearly triples throughput over the already-fast no-polling mode. Removing the last two "calls" (submit syscall and hardware interrupt) gives the final 40% boost from 1.2 M to 1.7 M IOPS.

io_uring is also an example of **co-design across the abstraction boundary**: rather than optimizing the kernel's internals and leaving the API unchanged, Axboe changed the API contract itself. The performance gain comes not from a faster syscall, but from calling the kernel *less often* — which required redesigning the communication channel from RPC to shared memory. The same move appears in exokernel research (libOS), in DPDK's userspace network stack, and in Apple's network extensions framework. When an API boundary costs too much to cross, the answer is often to redesign who crosses it and how often.

## Going deeper

1. **Jens Axboe, "Efficient IO with io_uring" (2019)** — https://kernel.dk/io_uring.pdf — the canonical design document, describes every data structure and synchronization invariant with C pseudocode.

2. **"io_uring for High-Performance DBMSs: When and How to Use It" (2024)** — https://arxiv.org/abs/2512.04859 — a systematic empirical study across OLTP, buffer manager, and scan workloads showing where io_uring wins (storage-bound, small random I/O) and where it doesn't (network, large sequential I/O).

3. **TigerBeetle, "A Programmer-Friendly I/O Abstraction Over io_uring and kqueue" (2022)** — https://tigerbeetle.com/blog/2022-11-23-a-friendly-abstraction-over-iouring-and-kqueue/ — how TigerBeetle unified io_uring (Linux), kqueue (macOS/BSD), and IOCP (Windows) behind a single completion-event loop in Zig, and why the three APIs converge on the same logical model despite wildly different syscall interfaces.
