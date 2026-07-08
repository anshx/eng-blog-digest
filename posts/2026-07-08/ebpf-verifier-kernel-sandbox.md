---
title: "What is eBPF? An Introduction and Deep Dive into the eBPF Technology"
source: https://ebpf.io/what-is-ebpf/
author: eBPF Foundation (Liz Rice, Daniel Borkmann, Alexei Starovoitov, and contributors)
company: Linux Foundation / eBPF Foundation
date_posted: 2021-05-01 (continuously updated through 2024)
date_digested: 2026-07-08
---

# What is eBPF? An Introduction and Deep Dive into the eBPF Technology

## What's new to learn

1. **eBPF verifier as bounded abstract interpretation**: Before any eBPF program runs, the kernel's verifier symbolically executes every possible path through the bytecode — tracking the type and value range of every register at every instruction — to statically prove the program cannot corrupt memory, crash the kernel, or loop forever.

2. **Safety-by-proof versus safety-by-isolation**: eBPF demonstrates that extending the kernel safely does not require process isolation (the mechanism that protects userspace from kernel crashes). Instead, static analysis can provide equivalent safety guarantees at load time with zero runtime overhead — the same tradeoff as compiled type systems versus runtime type checks.

3. **eBPF maps as a typed kernel–userspace channel**: Programs exchange state with userspace via eBPF maps — kernel-managed key-value stores with typed access. This lets telemetry data flow from kernel space to a monitoring agent without a single extra syscall per event, just a shared memory pointer both sides can read.

## Prerequisites

- **Kernel vs userspace**: Why the MMU enforces a hard boundary between kernel and userspace — and why crossing it (via a syscall) costs hundreds of nanoseconds.
- **Kernel modules**: What a `.ko` kernel module is, how it loads, and why it can crash the kernel (runs in ring 0 with no safety net).
- **Bytecode virtual machines**: The idea that a higher-level program can be compiled to an abstract instruction set (like JVM bytecode or WebAssembly) and then either interpreted or JIT-compiled to native code.
- **Abstract interpretation (helpful but not required)**: The technique of over-approximating program state — treating a register as holding "any value in [0, 255]" rather than one specific integer — to reason about all possible runs in one symbolic pass.

## The core idea

The original Berkeley Packet Filter (BPF, 1992) was a tiny in-kernel VM for filtering network packets. You'd write a small bytecode program like "drop this packet if destination port != 80" and the kernel would run it per-packet, avoiding expensive copies to userspace. Extended BPF (eBPF, introduced in Linux 3.18, production-ready in 4.4) kept that fundamental insight — sandboxed kernel programs that execute fast — but generalized it far beyond packet filtering.

The eBPF architecture rests on three pillars:

1. **The verifier**: A static analyzer that reads your bytecode and, before a single instruction executes, proves the program cannot crash the kernel, loop forever, or access out-of-bounds memory. If the verifier can't prove safety, the program is rejected. Programs that pass run at native speed — no further safety checks at runtime.

2. **JIT compilation**: Verified bytecode is compiled directly to native machine instructions (x86-64, ARM64, s390x, etc.), so eBPF programs execute at the same speed as compiled C code loaded as a kernel module — without the crash risk.

3. **Maps**: Typed key-value stores managed by the kernel. Both the eBPF program running in kernel space and a monitoring agent running in userspace can read and write the same map through a file descriptor, enabling zero-copy data transfer between kernel and user without extra syscalls on the hot path.

The mental model: **eBPF is to the Linux kernel what JavaScript is to a browser**. A browser lets untrusted code from any website run inside its process — but the JS engine's sandbox (type checks, memory bounds) ensures that code cannot read your passwords or crash the browser. eBPF does the same for the kernel: untrusted programs run inside kernel space but the verifier's static proof ensures they cannot corrupt kernel memory or crash the system.

## Mechanics

### The instruction set

eBPF programs are compiled from a subset of C using Clang/LLVM with target `bpf`. The result is a bytecode of 64-bit instructions across a simple 64-bit RISC ISA: 11 registers (`r0`–`r10`, where `r10` is the frame pointer), arithmetic, memory load/store, conditional jump, and a `call` instruction restricted to a whitelist of allowed kernel helpers.

Programs are loaded via the `bpf()` syscall with the `BPF_PROG_LOAD` command, which submits bytecode to the verifier.

### Verifier pass 1: Control Flow Graph validation

The verifier first builds the Control Flow Graph (CFG) of the bytecode:
- **Reachability check**: every instruction must be reachable from the entry point; dead code is rejected.
- **Termination check**: the CFG must be a Directed Acyclic Graph (DAG). Loops require a back-edge, which is allowed only if the verifier can prove the loop is bounded — it simulates up to `max_loop_iters` (typically 8 million) iterations. If the counter's range can't be bounded statically, the loop is rejected.
- **Valid jump targets**: every conditional jump must land on a valid instruction within the program.

### Verifier pass 2: Symbolic execution

This is the heart of the verifier. It walks every execution path (depth-first) while maintaining a state for every register at every instruction. Each register's state is one of:

| State type | What it means |
|---|---|
| `NOT_INIT` | register has never been written |
| `SCALAR_VALUE` | an integer with bounds `{umin, umax, smin, smax, var_off}` |
| `PTR_TO_CTX` | a pointer to the program's context (e.g., the `sk_buff` for network programs) |
| `PTR_TO_MAP_VALUE` | a pointer into an eBPF map value |
| `PTR_TO_STACK` | a pointer into the current stack frame |
| `PTR_TO_PACKET` | a pointer into the raw packet data |

For arithmetic on scalars the verifier propagates bounds. For example:
```c
r1 = map_lookup(map, &key);   // r1: PTR_TO_MAP_VALUE or NULL
if (r1 == NULL) goto exit;    // after this check, verifier knows r1 is non-null
r2 = *(r1 + 8);               // verifier checks: is 8 < map_value_size?
```

For pointer arithmetic, the verifier checks the **worst-case offset**:
```c
r3 = r1 + (r4 & 0xFF);  // r4 ∈ [0, 255] (verifier knows from AND with literal)
r5 = *(r3 + 0);          // is 255 < map_value_size? if yes, safe for all values of r4
```

**State merging**: when two execution paths converge (e.g., at the end of an `if/else`), the verifier merges register states by taking the union of their ranges. A key optimization is state caching — if the verifier reaches an instruction with a register state it has seen before, it can prune that path (since any further execution would be a subset of already-verified paths).

**Allowed operations**:
- eBPF programs can only read memory they were passed (context, maps, stack, packet) — never arbitrary kernel memory addresses.
- `bpf_probe_read_kernel()` is the only way to read kernel data structures, and it handles faults safely.
- No uninitialized register reads (caught by `NOT_INIT` tracking).
- No type confusions: you can't cast a map pointer to a scalar and then dereference it.

### JIT compilation

Once the verifier approves the program, the kernel's JIT compiler translates the verified bytecode to native machine code. On x86-64:
- eBPF registers `r0`–`r5` map to x86 registers `rax`, `rdi`, `rsi`, `rdx`, `rcx`, `r8`.
- The BPF stack frame maps directly to the native stack.
- Helper calls become direct `call` instructions to known kernel function addresses.
- The result is executable machine code indistinguishable in performance from a compiled kernel module.

The JIT is the reason eBPF programs used for hot-path networking (XDP, TC) can process millions of packets per second per core.

### Maps

eBPF maps are kernel-managed data structures shared between eBPF programs and userspace. They're created with `bpf(BPF_MAP_CREATE, ...)` and passed to programs as file descriptors. Common types:

| Map type | Use case |
|---|---|
| `HASH` | associative key-value store, O(1) average |
| `ARRAY` | integer-indexed fixed-size array (fast, no hash) |
| `RINGBUF` | multi-producer, single-consumer ring buffer for streaming events to userspace |
| `PERCPU_HASH` | per-CPU sharded hash — avoids contention in hot paths |
| `LRU_HASH` | eviction of least-recently-used entries automatically |
| `STACK_TRACE` | records kernel/user stack traces keyed by stack ID |

Maps can be **pinned** at `/sys/fs/bpf/` so multiple programs or userspace tools can share the same map, even across program reloads.

### Hook types

eBPF programs attach to kernel events through hooks:

| Hook | When it runs | Typical use |
|---|---|---|
| **XDP** | At the NIC driver, before kernel network stack | DDoS mitigation (drop/redirect before any allocation) |
| **TC (ingress/egress)** | In the traffic control layer | Rate limiting, packet classification |
| **kprobe / kretprobe** | Entry/exit of any kernel function (dynamic) | Performance profiling, tracing |
| **uprobe / uretprobe** | Entry/exit of any userspace function | Distributed tracing without instrumentation |
| **tracepoint** | Static points in the kernel (stable ABI) | Syscall monitoring, scheduler events |
| **perf events** | CPU counters, software events | Flame graphs, cache miss analysis |
| **LSM** | Linux Security Module decision points | Policy enforcement without a custom LSM |
| **socket / sock_ops** | Socket lifecycle events | TCP connection tracking, retransmission monitoring |

A program attached to a hook receives a context pointer (e.g., `struct xdp_md*` for XDP) and returns an action code (e.g., `XDP_DROP`, `XDP_PASS`, `XDP_TX`).

### End-to-end flow

```
Developer writes C → Clang (target=bpf) → eBPF bytecode (.o file)
                                              ↓
                                     bpf() syscall: BPF_PROG_LOAD
                                              ↓
                                     Verifier pass 1 (CFG)
                                              ↓
                                     Verifier pass 2 (symbolic exec)
                                              ↓   rejected → errno = EACCES + verifier log
                                     JIT compile → native machine code
                                              ↓
                                     Attach to hook (XDP, kprobe, etc.)
                                              ↓
                                     [hook fires] → program executes → writes to map
                                              ↓
                                     Userspace reads map via fd / ring buffer
```

## Where it breaks

**1. Verifier rejects correct programs (false rejections).** Abstract interpretation is inherently an over-approximation — the verifier can fail to prove a program safe even when it is. Complex pointer arithmetic or loops with non-obvious bounds trigger rejection. Developers must restructure code to make the proof visible to the verifier, which can be non-intuitive.

**2. Verifier bugs enable malicious programs.** The verifier itself has had security vulnerabilities where incorrect range tracking allowed an unprivileged user to read/write arbitrary kernel memory: CVE-2021-3490 (incorrect 32-bit range bounds), CVE-2022-23222 (broken pointer arithmetic tracking). An attacker who passes a crafted program to a buggy verifier gets kernel memory access — the opposite of the security guarantee. Verifier correctness is very hard to ensure, and researchers continue to find bugs.

**3. The 1M-instruction limit.** Verified programs can be at most 1 million instructions (this limit was raised from 4,096 in older kernels). Long-running computations — like full packet reassembly or complex cryptographic operations — can't fit. Workarounds include splitting into multiple programs linked with tail calls, or pre-processing in userspace.

**4. Restricted function calls.** eBPF programs cannot call arbitrary kernel functions; they can only call a whitelisted set of `bpf_helper_func` (e.g., `bpf_map_lookup_elem`, `bpf_ktime_get_ns`, `bpf_trace_printk`). Kfuncs (kernel functions exposed via BTF) expand this, but the list is curated and changes across kernel versions.

**5. No dynamic memory allocation.** eBPF programs can't call `kmalloc`. All data lives on the stack (limited to 512 bytes) or in pre-allocated maps. This restricts what you can express — you can't build a linked list dynamically.

**6. Privileged-only loading.** Loading most eBPF program types requires `CAP_BPF` (Linux 5.8+) or root. While some program types allow unprivileged loading (socket filter), the high-value hooks (XDP, kprobes, LSM) remain privileged.

**7. Portability overhead.** eBPF programs compiled against kernel headers for version X may not work on version Y if the kernel data structures they read have changed layout. BTF (BPF Type Format) and CO-RE (Compile Once, Run Everywhere) mitigate this by embedding type information in the kernel and doing field-offset adjustment at load time — but getting CO-RE right adds build complexity.

## Why it works

The deepest principle here is: **safety guarantees can come from the structure of code rather than from runtime enforcement**.

Historically, kernel safety relied on **isolation** — code in userspace can't crash the kernel because the MMU physically prevents it from accessing kernel addresses. Kernel modules break this guarantee: they run in ring 0 with no enforced boundaries. When a kernel module has a bug (a null pointer dereference, an out-of-bounds array access), it corrupts kernel state and the system crashes.

eBPF's approach is structurally different. It says: **prove the program is safe before running it, and then run it with no overhead**. This is the same design philosophy as:

- **Static type systems** (Rust's borrow checker): A program that compiles with Rust's borrow checker is proven to be free of data races and use-after-free bugs — not because a runtime monitor catches violations, but because the type system rules them out structurally. No runtime cost.
- **WebAssembly**: Wasm modules run in browsers without OS process boundaries because the Wasm bytecode verifier proves memory bounds at load time — same principle, different application domain.
- **Java bytecode verifier**: Java's verifier checks type safety before JIT compilation, which is why the JVM can run untrusted code without POSIX process isolation.
- **Proof-carrying code** (Necula, 1997): The formal idea that a program can carry a machine-checkable proof of its own safety — the loader verifies the proof rather than trusting the origin.

All of these are instances of **the proof-at-compile-time vs. enforcement-at-runtime tradeoff**. When you can express your safety invariant in a decidable static analysis, you get the guarantee for free at runtime. When you can't, you need runtime enforcement (with its overhead and failure modes).

The CrowdStrike incident of July 2024 is a perfect case study. A faulty content update to CrowdStrike's Falcon sensor — which uses a Windows kernel driver — caused 8.5 million machines to Blue Screen of Death (BSOD). The driver was not malicious, just buggy, but there was no mechanism to stop it from crashing the kernel. If Falcon ran on eBPF (which Microsoft is bringing to Windows), the worst case would be the verifier rejecting the faulty program at load time: the update would silently fail to load and the machine would continue running normally. The kernel cannot be crashed by a rejected eBPF program — there is simply nothing to crash.

This is the fundamental difference: a kernel module's bugs have kernel-wide blast radius; an eBPF program's worst case is a no-op.

## Going deeper

1. **"The eBPF Runtime in the Linux Kernel"** (Starovoitov et al., arXiv:2410.00026, October 2024) — a comprehensive academic survey of the full eBPF subsystem, including the verifier's abstract interpretation in detail, the JIT backends, BTF, and the CO-RE ecosystem.

2. **"Learning eBPF"** by Liz Rice (O'Reilly, 2023) — builds eBPF programs bottom-up, then walks the kernel-side implementation of the verifier and JIT. Chapter 7 ("eBPF Program and Attachment Types") and Chapter 8 ("eBPF for Networking") are particularly strong.

3. **"No More Blue Fridays"** by Brendan Gregg et al. (brendangregg.com, July 2024) — written immediately after the CrowdStrike incident, explains exactly how the eBPF safety model would have prevented the outage and compares it to the kernel module model, with a focused treatment of the verifier's guarantees.
