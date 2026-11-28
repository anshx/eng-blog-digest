---
title: "A new register allocator for ZJIT"
source: https://railsatscale.com/2026-05-27-a-new-register-allocator-for-zjit
author: Shopify Ruby team
company: Shopify
date_posted: 2026-05-27
date_digested: 2026-11-28
---

# A new register allocator for ZJIT

## What's new to learn

1. **Register allocation as interval scheduling**: the problem of mapping unlimited virtual registers to K physical registers is equivalent to scheduling non-overlapping intervals to K "rooms" — the same combinatorial structure as meeting-room scheduling, CPU process assignment, and interval graph coloring.
2. **Global vs local allocation**: a local allocator (ZJIT's previous design, inherited from YJIT) sees only one basic block at a time; a global allocator sees the whole function's control-flow graph (CFG), enabling it to keep values in registers across branches and loops — fewer spills, faster native code.
3. **SSA form makes the interference graph chordal**: Static Single Assignment form is not just a compiler convenience; it is a structural guarantee that changes register allocation from NP-complete (general graph k-coloring) to polynomial (chordal graph coloring via perfect elimination ordering).

## Prerequisites

- **Control-flow graphs (CFGs)**: programs as directed graphs where nodes are basic blocks (straight-line sequences with no branches inside) and edges are possible transfers of control.
- **Static Single Assignment (SSA) form**: a program representation where every variable is defined exactly once; branches are handled by φ-functions at join points that pick the value from whichever predecessor ran.
- **What a JIT compiler does**: converts bytecode (Ruby's YARV instructions, in ZJIT's case) to native machine code at runtime; compilation latency subtracts directly from application latency, so "fast enough to compile" is a first-class constraint alongside "fast native code."
- **Virtual vs physical registers**: a compiler's IR uses an unlimited pool of symbolic virtual registers; the CPU offers a fixed number of physical registers (16 on x86-64, 31 on ARM64). The allocation step bridges these two worlds.

## The core idea

A CPU does its fastest work when values sit in physical registers rather than in RAM or on the stack. The challenge: a function may reference hundreds of distinct values, but the CPU offers only a handful of registers. Two values that are both "live" (needed after the current point) at the same program point cannot share a register — that is a *conflict*. 

The abstract structure is exactly a **graph coloring problem**. Build an *interference graph*: one node per virtual register, one edge between any two that are live simultaneously. Assigning physical registers is k-coloring that graph — give each node one of K colors (physical registers) such that no two adjacent nodes share a color. Unfortunately, graph k-colorability is NP-complete in the general case.

**Linear scan** (Poletto & Sarkar, 1999) escapes the NP trap by reformulating the problem in terms of *live intervals* rather than point-in-time liveness. Each virtual register has a live interval [start, end] spanning its first definition to its last use. Two intervals conflict if and only if they overlap. Now the problem is equivalent to:

> Assign K colors to a set of intervals such that no two overlapping intervals share a color.

This is **interval graph coloring**, which is solvable in O(n log n) by a greedy sweep — the same algorithm that solves the meeting-room scheduling problem. ZJIT adopted a version of this algorithm based on Wimmer & Franz (2010), which extends linear scan to operate directly on SSA form rather than requiring SSA to be destructed first.

The payoff is compilation speed: linear scan compiles a function-level allocation in O(n log n) versus the multiple passes and large data structures of graph-coloring allocators. For a JIT that must compile before the first execution (or on the second, or the third), that matters.

## Mechanics

### 1. Build live intervals from SSA definitions and uses

For every SSA value *v*, its live interval begins at the instruction that defines it and ends at the instruction of its last use. In SSA form each value has exactly one definition, so the interval is unambiguous. When a value is live across a loop back-edge its interval is extended to cover the whole loop body, ensuring it stays allocated for all iterations.

### 2. Collect φ-function hints

φ-functions at join points ("value v₁ from block A or v₂ from block B becomes v₃") ideally want all three to land in the same physical register to avoid a move instruction. These are recorded as *hints* that the allocator will try to satisfy after the core allocation step.

### 3. Sort intervals by start position

All live intervals are sorted by their start instruction index. The sweep then processes them left to right.

### 4. Greedy allocation sweep

Maintain an *active set* of intervals currently occupying physical registers:

```
for each interval I sorted by start:
    expire intervals in active whose end < I.start  → free their registers
    if a free register exists:
        assign it to I; add I to active
    else:
        spill_candidate = interval in active with the furthest end
        if spill_candidate.end > I.end:
            steal register from spill_candidate; spill spill_candidate to stack
            assign stolen register to I
        else:
            spill I to stack
```

The spill heuristic — evict the interval that lives longest into the future — is Bélady's approximation applied locally. It is not globally optimal, but performs well in practice and runs in O(1) per step.

### 5. Assign stack slots for spilled values

Spilled intervals are assigned stack offsets. A spilled read becomes a `mov reg, [rsp + offset]` before the use; a spilled write becomes a `mov [rsp + offset], reg` after the definition. ZJIT must also account for *call-clobbered* registers: on x86-64 certain registers are trashed by the C calling convention, so values live across a call must either be in callee-saved registers or spilled before the call.

### 6. Resolve φ-functions

After allocation, each φ-function's inputs may have landed in different physical registers than its output. A *resolution pass* inserts `mov` or parallel-copy sequences at the predecessor block's exit to route the right value into the output register on each path. The classic pitfall here is the *lost copy problem* (two φ-copies that overwrite each other's inputs); the resolution algorithm performs a topological walk of the copy graph to detect and break cycles.

### 7. Coalescing (optional)

If a copy's source and destination do not interfere, they can be assigned the same register and the copy deleted. This "coalescing" pass runs after the main allocation and can significantly reduce move-instruction count.

### Key numbers from ZJIT

The switch from YJIT's local allocator to global linear scan was motivated by the prevalence of local variables in real Ruby code: the local allocator assigned a fresh register for each block-local use of a variable, forcing constant reloads. The global allocator keeps a local variable in a register across the whole function, which eliminates those reloads and also enables better use of machine registers in loops.

## Where it breaks

**Register pressure in loops is underestimated by naive intervals.** A variable used only inside a loop has an interval that spans the entire loop body, even if it is only actually needed for a few iterations. If many such variables exist simultaneously, they all appear live together and exhaust registers, causing unnecessary spills. Loop-aware extensions (such as interval splitting at the loop boundary) partially address this.

**Two-address constraints on x86.** Many x86 instructions encode "destination = source1 OP source2" where destination must be the same register as source1. Linear scan does not inherently produce this form; a *fixup pass* must insert copies where the allocator assigned destination ≠ source1. These copies reintroduce the interference they were meant to avoid.

**Spill slots are non-trivial on a managed runtime.** Because Ruby's garbage collector must be able to scan the stack for live object references, every stack slot that might hold a Ruby VALUE must be declared in a GC map. Getting that right for spilled values across calls is engineering complexity that the local allocator avoided by keeping everything in registers within a single block.

**Compilation is not the only constraint.** Linear scan prioritizes compilation speed over code quality. For hot methods running billions of times, a graph-coloring allocator's globally optimal coloring pays for itself. ZJIT's choice of linear scan makes sense for a tier-1 JIT that is later superseded by a more aggressive optimizer; it would be a poor choice for an ahead-of-time compiler.

**Linear scan is brittle under interval splitting.** The core algorithm assumes each variable has one contiguous interval. Real code often benefits from *splitting* a live interval into two pieces (a "live range splitting") to reduce spilling. Adding splitting to linear scan substantially increases implementation complexity; Wimmer & Franz's SSA-based paper is partly motivated by SSA form's ability to express splits naturally as new SSA names.

## Why it works

The deeper principle is that **SSA form converts an NP-hard problem into a polynomial one by restricting the shape of the interference graph**.

An interference graph is a *chordal graph* on SSA form. A chordal graph is one where every cycle of length ≥ 4 has a *chord* — an edge connecting two non-adjacent cycle nodes. Gavril (1974) proved that any chordal graph has a *perfect elimination ordering*: a sequence in which each node's neighbours that appear later form a clique. Coloring in that order is a greedy algorithm that never needs more than the graph's maximum clique size, which equals the chromatic number. So for chordal interference graphs, greedy coloring is globally optimal.

Why is the SSA interference graph chordal? Informally, SSA's single-definition rule means that if two variables *a* and *b* both "see" a third variable *c* (all three mutually interfere), then one of *a* or *b* must dominate the definition of the other — forming the triangle that prevents a chord-free 4-cycle.

Linear scan exploits this chordality implicitly: it processes intervals in order of a topological traversal of the SSA dominance tree, which turns out to follow a perfect elimination ordering. The O(n log n) algorithm is not an approximation; on SSA form, it computes an *exact* optimal coloring for the variables it doesn't spill.

The analogy to meeting-room scheduling is not just mnemonic — it is the same theorem. Meeting-room scheduling is an interval scheduling problem; interval graphs are chordal; chordal graphs are optimally colorable greedily. When you read "register allocation" in a compiler paper, you are reading about meeting-room scheduling on the dominance tree of a program.

This insight generalizes: **any resource assignment where "concurrent use" is the conflict relation, and use spans form a partial order consistent with a tree, reduces to greedy interval scheduling**. That covers CPU register allocation, instruction-level parallelism slot assignment, database lock escalation analysis, and lock-free data structure field reuse — all structurally the same problem.

## Going deeper

1. **Poletto & Sarkar (1999), "Linear Scan Register Allocation"** — ACM TOPLAS 21(5), pp. 895–913. The original paper; describes the O(n) algorithm and its performance on the SUIF framework. Available via ACM DL.
2. **Wimmer & Franz (2010), "Linear Scan Register Allocation on SSA Form"** — Proceedings of the 8th Annual IEEE/ACM International Symposium on Code Generation and Optimization (CGO). Extends the algorithm to SSA; this is the paper ZJIT's implementation is based on.
3. **Chaitin et al. (1982), "Register Allocation via Coloring"** — Computer Languages 6(1), pp. 47–57. The founding graph-coloring paper; reading it alongside the linear scan papers shows why the two approaches converge to the same answer on SSA form.
