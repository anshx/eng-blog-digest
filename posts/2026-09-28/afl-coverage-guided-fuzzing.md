---
title: "AFL: American Fuzzy Lop — Technical Whitepaper"
source: https://lcamtuf.coredump.cx/afl/technical_details.txt
author: Michał Zalewski
company: Google
date_posted: 2014-01-01
date_digested: 2026-09-28
---

# AFL: American Fuzzy Lop — Technical Whitepaper

## What's new to learn

1. **Edge coverage via XOR instrumentation**: At every branch point, AFL records not _which_ basic block was entered but _which transition_ was taken — the XOR of the current and previous basic-block IDs. That single operation encodes directed flow, distinguishing A→B from B→A, in one L2-cache-resident 64 KB bitmap.

2. **Hit-count bucketing**: Instead of storing exact loop-iteration counts, AFL maps raw hit counts to eight exponential buckets (`1, 2, 3, 4–7, 8–15, 16–31, 32–127, 128+`). Only a _bucket boundary crossing_ registers as new coverage — compressing out loop-count noise while preserving the semantically meaningful signal of "this loop ran once vs. many times."

3. **Corpus-guided genetic search**: AFL's seed corpus is an antichain in the coverage lattice — no retained seed's coverage is a subset of any other's. Adding new coverage is the fitness signal that admits an input into this corpus; the corpus then becomes the breeding stock for the next generation of mutations.

## Prerequisites

- **What fuzzing is**: automatically generating inputs to crash or hang a program.
- **Basic blocks and branches**: a basic block is a straight-line sequence of instructions; a branch is a conditional jump between blocks.
- **Unix `fork()` and shared memory** (`mmap`, `shmget`): AFL uses both to avoid per-test process-startup cost.
- **Coverage criteria**: the distinction between line coverage, branch coverage, and edge (pair-of-blocks) coverage.

Non-obvious prerequisite: understanding _why_ branch coverage under-counts is key. A→B→C and A→D→C both visit B and D, but only edge coverage records that A→B happened (vs. A→D) in the same trace.

## The core idea

The bug-finding problem is exponential: the space of possible inputs grows with length. AFL escapes this by reducing it to a coverage-maximizing search.

The fuzzer instruments the target binary once. During each execution, the target writes into a shared memory bitmap — one byte per edge — tracking which (source, destination) basic-block pairs were taken. After execution the fuzzer reads this bitmap and asks: _did this input take any edge, or hit a hit-count bucket, that no prior input did?_ If yes, the input is "interesting" and enters the seed corpus. If no, it's discarded.

This transforms fuzzing into evolutionary search: the corpus is a population, coverage novelty is fitness, and mutation is the variation operator. AFL never needs to know anything about the format of the input — structure emerges naturally as coverage-increasing mutations are retained.

**The key bet**: code regions that haven't been exercised are more likely to harbour bugs than regions that have. New edge coverage is, on average, correlated with new bug-discovery potential. This bet has been validated empirically: AFL found hundreds of CVEs in widely-fuzzed projects.

## Mechanics

### Compile-time instrumentation

AFL's compiler wrapper (`afl-gcc`/`afl-clang`) inserts the following at every branch:

```c
cur_location = <compile-time random constant>;   // unique per branch
shared_mem[cur_location ^ prev_location]++;
prev_location = cur_location >> 1;
```

**XOR encodes the edge.** If basic block A has ID `0x3F2` and basic block B has ID `0x810`, then the edge A→B writes into slot `0x3F2 ^ 0x810 = 0xBE2`. The reverse edge B→A writes into `0x810 ^ (0x3F2 >> 1)` = a different slot, so directionality is preserved without storing two values.

**Right-shift breaks symmetry.** Without the `>> 1` on `prev_location`, edge A→B and edge B→A would map to the same XOR value `A^B`. Shifting by 1 makes `A ^ (B >> 1) ≠ B ^ (A >> 1)` for almost all pairs.

**64 KB bitmap lives in L2 cache.** Sixty-five thousand and five hundred and thirty-six (65,536) one-byte slots = 64 KB, well within the typical 256 KB L2 cache. The increment is a single atomic store — nanosecond-scale overhead per branch. At 1,000 branches the collision probability is ~0.75%; at 10,000 branches it rises to ~14%, an acceptable false-positive rate for a heuristic search.

### Normalising the bitmap

After each execution, the raw bitmap is _normalised_: each byte is bucketed:

| Raw count | Bucket bit |
|-----------|------------|
| 0         | (none)     |
| 1         | bit 0      |
| 2         | bit 1      |
| 3         | bit 2      |
| 4–7       | bit 3      |
| 8–15      | bit 4      |
| 16–31     | bit 5      |
| 32–127    | bit 6      |
| 128+      | bit 7      |

AFL then XORs the normalised bitmap against a global `virgin_bits` bitmask (edges/buckets never seen) and ANDs with `virgin_bits`. Any set bit in the result = new coverage = interesting input.

**Why buckets?** A tight loop that runs 10 vs. 12 times doesn't change the program's logical behaviour; discarding that variation prevents trivial loop-count differences from flooding the corpus. But a loop that runs 1 vs. 4 times crosses a bucket boundary — a semantically meaningful difference AFL captures.

### Fork server

Starting a fresh process for every test input is expensive: the dynamic linker, all `__attribute__((constructor))` functions, and C runtime setup must run every time. AFL solves this with a **fork server**.

During AFL's first execution of the target, the instrumentation establishes a pipe-based handshake with the fuzzer. On receiving a "go" signal, the target process forks. The child runs the test input and exits. The parent immediately awaits the next "go" signal and forks again. The parent is kept permanently past all initialisation — fork is cheap (a few microseconds for copy-on-write page table clone), so only the test-specific code actually runs in the child.

This is the same "move initialisation out of the hot loop" pattern as:
- Database connection pools: open N connections once, reuse them.
- Thread pools: spawn threads once, dispatch work to them.
- Copy-on-paste JIT stencils: compile once, instantiate many times.

### Queue management and corpus culling

Every interesting input is written to the `queue/` directory. The corpus grows monotonically — AFL never deletes seeds once added. But the queue can grow large, and not every seed is equally valuable. Periodically, AFL runs **corpus culling**: a greedy set-cover approximation.

For each unique edge ever seen, find the shortest (fewest bytes) queue entry that covers it. The set of entries selected this way is the "favored" set. In the fuzzing loop:
- Favored entries: always fuzz.
- Non-favored entries: skip with 95% probability.

This ensures the fuzzer concentrates mutation budget on the smallest corpus that covers all discovered edges.

### Mutation pipeline

For each seed, AFL runs two phases:

**Deterministic phase** (exhaustive, reproducible):
1. **Bit flips**: flip 1, 2, 4, 8 consecutive bits at every position.
2. **Byte arithmetic**: add and subtract 1–35 to/from every byte, treating it as a `uint8_t` or `int8_t`.
3. **Interesting values**: substitute each position with values known to trigger edge cases: `0`, `1`, `127`, `128`, `255`, `-1`, `INT_MIN`, `INT_MAX`, etc. — both 8-bit and 16/32-bit variants.

**Havoc phase** (random, non-reproducible):
Repeatedly apply a random stack of operations: bit flips, byte substitutions, block insertions, block deletions, block duplication, and splicing (merging this seed with a randomly chosen other seed at a random split point). The number of iterations scales with the seed's `perf_score`.

**Power schedule** (how long to spend per seed):
`perf_score` is higher for seeds that:
- Are shorter (byte count).
- Were discovered earlier (lower queue position).
- Execute faster (lower execution time).
- Have not been generated by splicing.

AFL++ exposes this as named schedules (`explore`, `fast`, `coe`, `quad`, `exploit`, `rare`), each with a different formula for allocating time across the corpus.

### Crash deduplication

AFL records a crash as unique if the set of edges taken _just before the crash_ has not been seen before. It stores the crashing input in `crashes/`, and the AFL `afl-tmin` tool minimises it to the smallest input that still crashes. `afl-cov` maps the corpus to source lines. Together they form a lightweight triage pipeline without needing a symbolic debugger.

## Where it breaks

**Highly structured input formats.** For SQL, XML, or TLS, most random bit-flips produce inputs rejected by a shallow parser long before reaching the interesting logic. AFL effectively can't "see past" the parser. Fixes: grammar-aware fuzzers (Peach, libprotobuf-mutator), or structure-aware mutation plugins in AFL++.

**Comparison-heavy gates.** A 32-bit magic number check (`if (hdr.magic != 0xDEADBEEF)`) blocks 4 billion out of 4 billion random values. AFL's byte arithmetic won't reach the magic value without luck. Fix: the REDQUEEN/CMPLOG technique in AFL++ tracks comparison operands at runtime and replaces seed bytes that appear in failing comparisons with the expected value.

**Stateful protocols.** A TCP handshake requires SYN→SYN-ACK→ACK in the right order. AFL has no concept of protocol state. Fix: AFL-Net and other stateful fuzzers model the message sequence as the unit of mutation.

**Coverage ≠ bugs.** 100% edge coverage does not imply bug-freedom. AFL finds crash-inducing inputs; logic errors that produce wrong-but-non-crashing output are invisible. Address Sanitizer (ASAN) and Memory Sanitizer (MSan) are typically run alongside AFL to catch more classes of bugs.

**Bitmap collisions at scale.** At 50,000+ branches, the 64 KB bitmap starts experiencing ~50% collision rates. Two distinct edges may hash to the same slot, causing AFL to believe it has covered an edge it hasn't, or vice versa. AFL++ addresses this with larger bitmaps and better hash functions.

## Why it works

### The fundamental structure: evolutionary search on a coverage lattice

AFL maintains a corpus that is an **antichain in the coverage lattice**: no seed's edge-coverage set is a strict subset of any other favored seed's coverage set. Each time a new input is added, it provides at least one edge or bucket that no existing seed provides. The corpus is thus a minimal spanning set for the discovered coverage space.

Mutations walk the corpus through the neighbourhood of known-good inputs. Coverage novelty as fitness is a surprisingly good proxy: empirically, code that hasn't been reached is more likely to have latent bugs (it received fewer reviews, less testing). AFL is a greedy hill-climber on this proxy — which is why it far outperforms random fuzzing even when the fitness function (coverage) is imperfect.

### XOR is the canonical "combine two signals" primitive

The choice of XOR to encode directed edges (`cur ^ (prev >> 1)`) is part of a broader pattern throughout systems:
- **Zobrist hashing** (chess engines): XOR game-state features into a single hash for transposition tables.
- **Rabin fingerprinting**: XOR over polynomial multiplication for rolling checksums.
- **RAID-5 parity**: XOR of N data blocks produces a parity block; any one block recoverable.
- **Cuckoo filter self-relocation** (in this archive): `fp ^ hash(bucket_index)` moves fingerprints without knowing the original key.

In all cases, XOR's self-inverse property (`A ^ B ^ B = A`) makes it easy to combine and separate values. Here it embeds two basic-block IDs into one table index, preserving the directionality of flow.

### Bucketing is lossy compression that preserves decision boundaries

The 8-bucket scheme is structurally identical to:
- **IEEE 754 floating point**: exponent encodes magnitude, mantissa encodes precision — you get the same number of significant bits regardless of scale.
- **µ-law audio encoding** (and AWQ weight quantization in this archive): non-linear quantisation allocates more resolution where signal changes fastest (near zero for audio; near important weights for LLMs).
- **HyperLogLog register max** (in this archive): track only the maximum leading-zero run, not the exact count — a sufficient statistic for cardinality estimation.

The principle: represent enough information to make the right decision (is this semantically different from what we've seen?) while discarding information that doesn't affect the decision (exact loop counts within a bucket).

### The fork server is the universal "move setup out of the loop" pattern

AFL's fork-server optimisation is a specific instance of the principle that motivates:
- **Connection pools**: database handshake once, share connection many times.
- **Copy-and-Patch JIT** (in this archive): compile each opcode stencil once, stamp it into machine code many times.
- **Maglev's lookup table** (in this archive): pre-compute the hash-ring assignment, do O(1) lookup per packet.
- **Warp specialisation** (in this archive): specialise producer/consumer warps at kernel launch, amortise the specialisation over millions of GPU cycles.

The invariant: when initialisation cost >> per-test cost, batch the initialisation.

## Going deeper

1. **AFL++ documentation and power schedules** — https://github.com/AFLplusplus/AFLplusplus: the actively-maintained fork that adds CMPLOG (comparison-operand feedback to defeat magic-byte guards), multiple power schedule formulas, and persistent-mode fuzzing (calling the fuzz target function in a loop in the same process, avoiding even fork overhead).

2. **"Fuzzing with AFL is an Art"** (Brendan Dolan-Gavitt, 2016) — https://moyix.blogspot.com/2016/07/fuzzing-with-afl-is-an-art.html: empirical lessons on corpus selection, dictionaries, deferred fork points, and parallelisation from real-world fuzzing campaigns; covers the gap between AFL's theoretical guarantees and its practical behaviour.

3. **OSS-Fuzz** — https://google.github.io/oss-fuzz/: Google's continuous fuzzing infrastructure, which runs AFL-style fuzzers against hundreds of open-source libraries 24/7 and has found 10,000+ bugs. Explains how to integrate a project, how harnesses are written, and how coverage-guided fuzzing scales to a fleet.
