---
title: "Spectre Attacks: Exploiting Speculative Execution"
source: https://spectreattack.com/spectre.pdf
author: Paul Kocher, Jann Horn, Anders Fogh, Daniel Genkin, Daniel Gruss, Werner Haas, Mike Hamburg, Moritz Lipp, Stefan Mangard, Thomas Prescher, Michael Schwarz, Yuval Yarom
company: Google Project Zero + Graz University of Technology + multiple universities
date_posted: 2019-05-20
date_digested: 2026-10-01
---

# Spectre Attacks: Exploiting Speculative Execution

## What's new to learn

**1. Speculative execution** — Modern CPUs execute instructions *before* knowing whether they will be needed, speculating that a branch will go a certain way, then discarding work if the guess was wrong. This is the CPU equivalent of optimistic concurrency control.

**2. The architectural/microarchitectural split** — A CPU's "architectural state" (registers, memory, program counter) is what the program model says happened. Its "microarchitectural state" (caches, branch predictor tables, TLB, store buffer) is the internal hardware substrate used to implement the architecture. Branch misprediction rollback restores architectural state — but does *not* erase microarchitectural side effects.

**3. Cache timing as an information channel (Flush+Reload)** — Cache hits take ~50 CPU cycles; main-memory accesses take ~200–300. An attacker who controls what lines are cold can measure which of 256 probe addresses is warm after a victim runs, learning one byte of secret per round. This is the covert channel Spectre rides.

## Prerequisites

- **Branch prediction basics** — CPUs maintain a Pattern History Table (PHT) for conditional branches and a Branch Target Buffer (BTB) for indirect branches. These are indexed by branch address (and sometimes recent history), and they can be trained by repeated executions.
- **Cache coherency** — CLFLUSH is an x86 instruction that evicts a cache line by virtual address. After a flush, the next access to that line pays full DRAM latency.
- **Virtual address spaces** — In the Linux kernel model, kernel memory is mapped into every process's address space (KPTI later removed this). This means a user-space process could, if it could read kernel memory, access kernel data without a privilege switch.
- **Out-of-order and speculative pipelines** — CPUs buffer inflight instructions in a Reorder Buffer (ROB) and commit them in order; speculative instructions may execute out of order but commit only after their branch resolves correctly.

## The core idea

Suppose a victim function looks like this in C:

```c
if (x < array1_size) {
    y = array2[array1[x] * 512];
}
```

In normal execution `x` is always a valid index, so the branch is always taken. After thousands of calls, the branch predictor is confident: "this branch is always taken."

Now an attacker passes `x = attacker_x`, a carefully crafted value far outside `array1_size` — pointing, say, at a password in kernel memory. The CPU starts executing speculatively: the branch predictor predicts "taken," so it begins loading `array1[attacker_x]` (a secret byte) and using it as an index into `array2`. One of 256 × 512-byte-spaced lines of `array2` loads into the L1 cache.

Then the branch resolves: `x >= array1_size`. The CPU discards the architectural effects — `y` is never written from the attacker's perspective. But the cache line stays warm.

The attacker now times all 256 probes into `array2`. One of them is fast. That index *is* the secret byte. Repeat 8 times; exfiltrate a full byte per round.

The attack does not require a bug in the victim's code. It exploits a **fundamental property of speculative execution**: the speculative window has read access to the victim's entire address space, and any speculative memory load has a permanent microarchitectural trace.

## Mechanics

### Step 1: Train the branch predictor

Call `victim_function(benign_x)` several hundred times with valid in-bounds values. The PHT entry for the bounds-check branch fills with "taken" predictions. The branch predictor now *believes* the check will pass.

### Step 2: Flush the timing array

```c
for (int i = 0; i < 256; i++)
    _mm_clflush(&array2[i * 512]);  // evict all 256 probe lines
```

All probe lines are now cold (DRAM-resident).

### Step 3: Trigger speculative access

Call `victim_function(attacker_x)` where `attacker_x` computes as `&secret - array1_base`. The CPU speculatively executes the body:

1. Speculatively loads `array1[attacker_x]` — this is `secret` (one byte, 0–255).
2. Speculatively loads `array2[secret * 512]` — warms one of the 256 probe lines.
3. Branch resolves as misprediction; architectural state rolls back.
4. The cache line for `array2[secret * 512]` stays warm.

### Step 4: Flush+Reload — reconstruct the secret byte

```c
for (int i = 0; i < 256; i++) {
    int t0 = rdtsc();
    volatile uint8_t v = array2[i * 512];
    int elapsed = rdtsc() - t0;
    if (elapsed < CACHE_HIT_THRESHOLD)
        scores[i]++;
}
```

The one index with a fast read is the secret byte. Run many trials to handle noise; pick `argmax(scores)`.

### Variant 2: branch target injection

Variant 1 requires a gadget that exists in the victim's own code. Variant 2 is more powerful: the attacker *poisons the indirect branch predictor (BTB)* to redirect one of the victim's own indirect jumps to an attacker-chosen "trampoline" gadget anywhere in the victim's address space. This allows exfiltration even without a natural bounds-bypass gadget in the victim.

### Scaling to cross-process attacks

`array2` does not need to be shared with the victim — a Prime+Probe variant works where the attacker occupies all cache sets, then observes which one the victim evicted. This allows cross-process (and kernel→user) exfiltration.

## Where it breaks

**Requires measurable timing.** JavaScript mitigations removed high-resolution timers (`performance.now()` resolution lowered to 5 µs, `SharedArrayBuffer` disabled in many browsers post-2018) precisely to eliminate the timing oracle.

**Requires a gadget (Variant 1).** The victim process must contain a code sequence that loads a secret-indexed value into a register, even transiently. JIT-compiled code and kernel syscall paths are common gadget sources.

**Doesn't generalize across all architectures equally.** ARM and AMD have different branch predictor designs; some variants require specific microarchitecture features (e.g., Intel's BTB layout for Variant 2). Portable exploit code is harder to write.

**Mitigations introduce real overhead.** Retpoline (replaces indirect jumps with a "call/pause/lfence/pop" sequence that traps speculative execution in a spin) eliminated Variant 2 exploitation but costs 5–15% on indirect-call-heavy code (interpreters, VMs). IBRS/IBPB microcode updates add 10–35% overhead on syscall-heavy workloads. KPTI (kernel page-table isolation, the Meltdown fix) adds 5–30% on I/O-bound workloads because it flushes the TLB on every kernel entry.

## Why it works

The deeper principle is this: **any optimization that reuses state based on secret values creates a potential information channel between the optimizer and an observer of that state**.

Speculative execution is a form of memoization across a branch boundary: "I'll assume the branch is taken and start loading data I'll need if it is." That speculative loading populates the cache, which is shared microarchitectural state observable by any code that can measure timing.

This is identical in structure to several patterns already in this archive:

- **Postgres ghost tuples** (the atomicity post): a rolled-back transaction leaves a dead heap tuple with an aborted `xmin`. The transaction is logically invisible, but the tuple physically persists until VACUUM reclaims it — observable via table bloat.
- **The end-to-end argument**: "the CPU rolled it back" is a correct statement about architectural state. It says nothing about microarchitectural state. The end-to-end principle would say: if correctness depends on the side effects being invisible, the mechanism that erases them must be at the level where they were created — and CPU branch rollback doesn't reach into the cache.
- **Copy-on-write semantics**: two processes "share" a page as long as neither writes; the OS creates the illusion of isolation while sharing physical memory. Spectre breaks that illusion at one level lower: even read-only speculative accesses create observable shared state in the cache hierarchy.
- **MVCC in databases**: speculative (or aborted) transactions leave behind visible physical traces. The gap between the logical model ("this transaction never happened") and the physical reality ("it left writes in the buffer pool / log") is where all the interesting bugs live.

The unifying observation: **rollback in any layered system only rolls back the layer that defines the semantics. Layers below that definition are unaware of the rollback, and their state can diverge.**

In CPUs, the ISA defines the architectural layer. The cache is below the ISA's semantic level — it is not mentioned in the x86 programming model. The spec says the CPU may cache what it likes; it says nothing about when caches must be invalidated. Spectre exploits precisely that gap.

## Going deeper

1. **Meltdown** (Lipp et al., 2018): the companion attack. Where Spectre tricks the *victim* into speculating past a bounds check, Meltdown tricks the *attacker's own code* into speculatively reading kernel memory before the CPU raises the page-fault exception. The mitigations are different (KPTI vs. retpoline), but the underlying principle — microarchitectural persistence of transient execution — is the same. https://meltdownattack.com/meltdown.pdf

2. **"A Systematic Evaluation of Transient Execution Attacks and Defenses"** (Canella et al., USENIX Security 2019): a taxonomy of the full attack space, classifying variants by which microarchitectural element they exploit (cache, BTB, PHT, STL, RSB). Shows there are many more variants than the original two, and that most deployed mitigations only cover specific variants. https://arxiv.org/abs/1811.05441

3. **"KAISER: Hiding the Kernel from User Space"** (Gruss et al., 2017): the KPTI design paper, which was designed originally to prevent side-channel attacks on kernel ASLR. Explains the cost model of full page-table isolation in terms of TLB shootdowns and CR3 switches. https://gruss.cc/files/kaiser.pdf
