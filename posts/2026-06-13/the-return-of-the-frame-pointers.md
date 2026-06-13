---
title: The Return of the Frame Pointers
source: https://www.brendangregg.com/blog/2024-03-17/the-return-of-the-frame-pointers.html
author: Brendan Gregg
company: Independent (formerly Netflix, now Intel)
date_posted: 2024-03-17
date_digested: 2026-06-13
---

# The Return of the Frame Pointers

## What's new to learn

1. **The four stack-unwinding methods** — Frame pointer (fp), DWARF CFI, ORC, and LBR: four distinct mechanisms that let a profiler reconstruct a call stack, each with completely different costs, reliability, and applicability. Most engineers know only one or two exist.

2. **How a 2004 compiler default silently broke profiling for 20 years** — GCC made `-fomit-frame-pointer` the default at optimization levels ≥ O1 to free the base pointer register (EBP on i386). That rationale evaporated when x86-64 doubled the register count, but the default persisted — and quietly made all frame-pointer-based profiling produce truncated stacks for two decades.

3. **The observability/performance duality in compiler state** — Every optimization that removes redundant runtime state also removes the history that external tools need to understand a running program. Frame pointers are the canonical example: architecturally redundant, observability-critical.

## Prerequisites

- The call stack and stack frames: what "return address" and "caller's base pointer" mean at the machine level
- What `perf` and flame graphs are at a conceptual level (visualizing time spent in call trees)
- x86-64 register names (RSP, RBP) and the distinction from 32-bit i386 (ESP, EBP); not required to follow the argument but needed to appreciate the historical rationale

## The core idea

A frame pointer is a saved register value that every function (by convention) writes at the top of its stack frame: it stores the frame pointer of the *caller*. The result is a singly-linked list embedded in the stack — each node is the saved base pointer (RBP on x86-64), and dereferencing it gives you the next node. The return address sits immediately above each saved RBP. Walking the chain is O(depth) and requires nothing more than a few pointer dereferences at the time of sampling.

When GCC and Clang produce optimized code, they typically skip writing this chain. RBP is freed to use as a general-purpose register, saving register spills in tight loops. The CPU executes correctly — it doesn't use the frame pointer chain. But a profiler that samples the program at an interrupt, reads RSP, and tries to walk the chain finds a broken link at the first function that omitted its frame pointer entry. The result: a flame graph that shows only the innermost few frames, cutting off everything above a shared library boundary. For 20 years on Linux, this was the silent default for any binary compiled with `-O2` or higher.

The fix, finally adopted by Fedora 38 (April 2023) and Ubuntu 23.04, was to recompile the entire package repository with `-fno-omit-frame-pointer`. On x86-64, the cost is measurable but small (typically 0–3%) because the architecture has 16 general-purpose registers — RBP's absence as a GPR is rarely the binding constraint. On the old 32-bit i386 with only 8 registers, losing EBP was genuinely costly, which is why the default was set the way it was in 2004.

## Mechanics

### Four ways to unwind a stack

**1. Frame pointer (FP)** — The conventional approach when `-fno-omit-frame-pointer` is in effect.

```
saved_rbp:   [prev_frame_rbp]   <- RBP points here
return_addr: [return address]   <- RBP + 8
local vars:  ...
```

At each frame, the profiler reads:
- `return_addr = *(rbp + 8)`
- `rbp = *rbp`

Repeat until `rbp == 0`. This is done entirely in the profiler's signal/interrupt handler — no kernel or debug-info parsing required. Works for any native compiled code that respects the convention. Fails for JIT-compiled code (JVM, V8, .NET CLR) that does not maintain the chain.

**2. DWARF CFI (Call Frame Information)**

The compiler emits a `.eh_frame` section (or `.debug_frame`) that contains a table: "at program counter PC, the canonical frame address is RSP + N, and the return address is at CFA − 8." This describes every function prologue variant, including non-standard frames from hand-written assembly.

Walking a DWARF CFI stack:
1. Binary-search `.eh_frame` for the current PC.
2. Evaluate the CFA expression (usually a simple `RSP + constant`).
3. Read the return address from the computed location.
4. Advance to the caller's PC and repeat.

DWARF CFI is more general than frame pointers — it works for functions that use unusual prologues, for exception handling (its original purpose), and for code that computes the frame address non-trivially. The cost is heavy: the `.eh_frame` section can be 10–15% of binary size, and parsing it at sample time requires locking and binary search. Production binaries are often stripped of `.debug_frame` though `.eh_frame` is usually kept for C++ exception handling. Still fails for JIT.

**3. ORC (Oops Rewind Capability)**

The Linux kernel cannot use DWARF — parsing complex structures at interrupt time is too slow, and DWARF wasn't designed for the kernel's interrupt and exception entry points. Josh Poimboeuf (Red Hat) designed ORC as a kernel-specific simplified CFI format (merged in 4.14, 2017).

ORC entries are 8 bytes each (vs ~50 bytes for DWARF entries), enabling faster lookup. Each entry describes just two things: where to find SP and where to find the return address. The kernel also emits a sorted lookup table, so finding the ORC entry for a given PC is a fast binary search without locking. Kernel stacks unwind reliably even in panic handlers and NMI contexts.

ORC handles kernel stacks only. Userspace stacks still require frame pointers or DWARF on the user side.

**4. LBR (Last Branch Record)**

Intel CPUs include a hardware ring buffer — the LBR — that records the last N branch targets (N = 16–32 depending on the microarchitecture) with zero software overhead. The profiler can read the LBR MSRs at sample time to reconstruct the last few calls.

LBR provides perfect accuracy for the frames it captures and has negligible performance overhead. Limitations: fixed depth (only the most recent N branches), Intel-only (AMD has its own variant called BRBE), and not typically available inside VMs unless the hypervisor exposes hardware PMU registers.

### What broken profiling looked like

Before these distribution changes, a typical perf flamegraph workflow:

```
perf record -F 99 -g ./my-service
perf script | flamegraph.pl > out.svg
```

would produce stacks like:

```
my-service`handle_request
  libc`malloc          <- chain breaks here; malloc omits frame pointer
  [unknown]
  [unknown]
```

The `[unknown]` entries aren't an error — perf simply couldn't find valid frame pointers above the library boundary. Hot code inside `malloc` appeared to have no callers. Aggregated profiles attributed all allocator time to `malloc` itself, hiding the actual callers.

Off-CPU profiling (measuring time spent blocked — in I/O, mutex waits, page faults) was even more broken. Off-CPU profiles require correlating a user-space blocked context with the kernel schedule point. With broken user-space stacks, the "who was waiting" half was always `[unknown]`.

### After re-enabling frame pointers

Flame graphs show complete stacks end-to-end. `perf sched` latency profiles correctly identify which user-space context was waiting for each scheduler wakeup. Tools like Polar Signals (Parca), Pixie, and Grafana Beyla — eBPF-based continuous profilers — work correctly without kernel modules or debug symbols, since they walk frame-pointer chains in the eBPF program itself.

## Where it breaks

**JIT-compiled code (JVM, V8, .NET CLR)**: The JIT compiler generates code at runtime and does not maintain the frame pointer convention. Java and JavaScript stacks still appear as `[unknown]` even on frame-pointer-enabled distributions. The solutions are JVM-specific:
- Java: `async-profiler` uses `AsyncGetCallTrace` via a JVMTI agent and a signal trick to safely capture stacks inside the JVM's own machinery.
- Node.js/V8: `--perf-prof` flag makes V8 emit a `/tmp/perf-<pid>.map` file mapping JIT code addresses to function names; `perf inject --jit` merges this with perf records.
- .NET: Similar `.map`-file tricks via `DOTNET_PerfMapEnabled`.

**Leaf functions and tight loops**: Adding `push rbp; mov rbp, rsp; pop rbp` to very short functions (2–3 instructions) is not free. Cryptographic primitives, SIMD inner loops, and some allocator internals intentionally retain `-fomit-frame-pointer` even on the newer distributions. These frames still break the chain locally.

**Performance-critical code**: The 0–3% typical overhead is not uniform. Floating-point-heavy code that spills registers more aggressively can see up to ~10%. Code that already register-spills frequently sees near-zero cost. This is why the fix was made at the distribution level, not in GCC upstream defaults — distributions can measure the cost for their actual workloads.

**Inlined functions**: Frame pointers only restore the chain of *physical* stack frames. Functions that were inlined by the compiler have no stack frame at all. DWARF inlining info (`.debug_info`) is needed to reconstruct inlined frames. Production binaries are usually stripped of this, so inlined callers still appear merged into their inlinee in CPU profiles.

## Why it works

The frame pointer chain is *physically redundant state* — the CPU never reads RBP during execution to determine control flow. It exists solely so that an external observer (a debugger, profiler, crash reporter) can reconstruct the call history from a snapshot of memory at any instant.

This identifies a general principle: **profiling is the art of reading history from present state**. A sampler fires a signal or interrupt, freezes the CPU's architectural state (registers, stack), and must reconstruct the sequence of calls that led here. The frame pointer chain is the data structure encoding that history. When the compiler removes it, the history disappears from the observable state — not from the execution, but from anything trying to understand the execution after the fact.

The deeper pattern is **the observability/optimization duality**: every optimization that eliminates redundant runtime state also eliminates the artifact that lets external tools understand the system. This appears everywhere:

- **Inlining**: eliminates call overhead, also eliminates the stack frame that identified the inlinee — DWARF inlining records are the frame-pointer chain equivalent.
- **Register allocation**: eliminates spills, also means local variable values aren't at a fixed stack offset where debuggers expect them — DWARF location expressions compensate.
- **TCP segmentation offload (TSO)**: eliminates per-packet CPU overhead, also means `tcpdump` never sees the large packets the application sent — only the segmented ones the NIC emits.
- **CPU branch prediction**: eliminates control-flow stalls, also means the instruction stream the CPU actually executes differs from what a simple trace would predict.

In each case, the optimization works precisely because it removes state that the system never *needs*. But tooling, by definition, reads state. There is a fundamental tension between "minimize redundant state for performance" and "maintain redundant state for observability."

The frame pointer story also illustrates **path dependency in ecosystem defaults**: the decision to omit frame pointers was correct in 2004 for i386 (8 GPRs, meaningful cost). When x86-64 arrived with 16 GPRs, the rationale evaporated but the default did not, because no single party owned the decision holistically — GCC upstream, distro maintainers, and application developers each only saw a slice of the consequences. It took 19 years and a coalition of distribution engineers, perf tool authors, and continuous profiling vendors with broken products to create enough pressure for the default to change. This is how technical debt in defaults works: it accumulates invisibly until someone bears enough cost to quantify it and force a change.

## Going deeper

1. **Brendan Gregg's Flame Graphs** — The original paper/blog (https://www.brendangregg.com/FlameGraphs/cpuflamegraphs.html) and the `flamegraph.pl` tool explain the visualization itself and what the common distortions (merged stacks, broken chains) actually look like in practice.

2. **Josh Poimboeuf's ORC unwinder design doc** — The 2017 LWN writeup "Unwinding the stack the hard way" (lwn.net/Articles/728339/) explains exactly why DWARF is unsuitable for the kernel and how ORC makes different tradeoffs. Reading it alongside the frame pointer story makes the design space clear.

3. **async-profiler** (github.com/async-profiler/async-profiler) — The go-to solution for the remaining hard case: profiling JVM applications. Its README explains the `AsyncGetCallTrace` trick and why it's needed, which directly extends the frame pointer mental model to runtimes with their own managed stacks.
