---
title: "Parsing Gigabytes of JSON per Second"
source: https://arxiv.org/abs/1902.08318
author: Geoff Langdale, Daniel Lemire
company: MILA / University of Quebec at Montreal
date_posted: 2019-02-22
date_digested: 2026-10-10
---

# Parsing Gigabytes of JSON per Second

## What's new to learn

**1. SIMD as a bit-parallel automaton.**
Instead of running a character-by-character state machine that processes one byte per clock, you can run the automaton for 32 or 64 characters *simultaneously* by treating a CPU register as a 256-bit or 512-bit array of 1-bit state variables. The SIMD instruction set becomes a way to advance the automaton state across an entire cache line at once.

**2. Carry-less multiplication (CLMUL) as prefix XOR.**
The PCLMULQDQ instruction multiplies two 64-bit polynomials over GF(2) — the integers mod 2 with no carries. Multiplying a bitmap by the all-ones constant is the same as computing the prefix XOR (exclusive-or prefix sum) of that bitmap. This turns the problem "is this byte inside a quoted string?" — which naively requires scanning backward to count preceding quotes — into a single hardware multiply.

**3. Two-stage parsing: structural indexing before interpretation.**
Decouple "where is the structure?" (Stage 1, done entirely in SIMD) from "what does the structure mean?" (Stage 2, scalar code navigating a pre-built index). This separation allows the SIMD-hostile parts of parsing — integer arithmetic, stack management for nesting depth, escape handling — to work on a dense pre-indexed representation rather than a raw character stream.

## Prerequisites

- Basic JSON format: objects (`{}`), arrays (`[]`), strings, numbers, booleans.
- SIMD concepts: a SIMD register holds N lane values; operations apply to all lanes in one instruction. AVX2 has 256-bit registers (32 bytes per op).
- What a CPU cache line is (64 bytes), and why sequential reads are fast.
- That `&`, `|`, `^`, `~` operate bitwise on integers (treating each bit as an independent 1-bit value).

Non-obvious prerequisites:
- **PSHUFB (shuffle bytes)**: a SIMD instruction that permutes bytes according to an index vector. Used here as a SIMD lookup table.
- **PCLMULQDQ (carry-less multiply)**: multiplies two 64-bit integers as polynomials over GF(2). Least common knowledge — most engineers never use this.
- **PEXT / pdep**: bit-manipulation instructions that extract bits at specified mask positions into a dense integer.

## The core idea

A standards-compliant JSON parser normally operates as a recursive-descent or state-machine parser: read one byte, advance state, emit a token. The limiting factor is **branch misprediction and per-byte overhead** — on modern CPUs, a branch mispredict costs 10-20 cycles, and JSON has a branch at every character boundary.

simdjson flips this by observing that JSON's *structure* can be extracted without caring about *values*. A `{`, `}`, `[`, `]`, `:`, or `,` is structurally meaningful only if it is **not inside a quoted string**. Everything else is a value (number, string content, boolean, null) whose exact bytes only matter when you actually materialize the value.

So the algorithm runs in two stages:

**Stage 1 (SIMD)**: process the input 64 bytes at a time. Produce two outputs:
- A **structural bitmap** — one bit per input byte, set for each ` { } [ ] : , ` that is *outside* a string.
- A **string bitmap** — one bit per input byte, set for bytes inside a quoted string (used to mask the structural bitmap).

**Stage 2 (scalar)**: walk the structural bitmap left-to-right using a simple loop that extracts the next set bit with `_blsi_u64` / `tzcnt`. Each set bit is a structural character position; process it with a tiny state machine (push/pop for nesting, emit to a "tape" data structure).

The tape is the output: a flat array of tagged 64-bit words, where each word encodes `(type, offset_in_input)`. Objects and arrays also contain *forward skip* offsets so you can jump past them in O(1), making both sequential traversal and key lookup fast.

## Mechanics

### Stage 1, Step A: classify every byte with SIMD lookup tables

The PSHUFB instruction permutes 16 bytes by index. You can use it as a 16-entry lookup table applied to all 32 (or 64) lanes simultaneously.

For each byte `b`, split it into `lo = b & 0x0F` and `hi = b >> 4`. Look up `lo` in a 16-entry "lo table" and `hi` in a 16-entry "hi table". AND the two results. This classifies each byte into categories (structural, whitespace, other) in one pair of PSHUFB instructions plus one VPAND — covering 32 bytes in ~3 instructions.

For example, the lo-nibble of `{` (0x7B) is `B` and its hi-nibble is `7`. The tables are crafted so that `lo_table[0xB] & hi_table[0x7]` has a bit set in the "structural character" mask position, while `lo_table[0xA] & hi_table[0x6]` (for `:`) sets a different bit, and so on.

This replaces per-byte `switch` / `if`-chains with ~3 vector instructions that process 32 bytes at once, producing a 32-bit bitmap of structural candidate positions.

### Stage 1, Step B: detect string boundaries with carry-less multiplication

The structural bitmap found in Step A is wrong wherever a structural character appears inside a string. We need to build a **string mask** — a bitmap where bit `i` is 1 iff byte `i` of the input is inside a quoted string (including the opening quote, excluding the closing one is a choice of convention).

Detecting string interiors requires knowing whether the number of preceding unescaped `"` characters is odd (inside) or even (outside). That is a prefix parity problem.

Define a 64-bit integer `Q` where bit `i` = 1 iff byte `i` is an unescaped `"`. The string interior mask for those 64 bytes is the **prefix XOR** of `Q`: bit `i` of the result = XOR of bits 0..i of Q.

Prefix XOR of `x` = `x ^ (x << 1) ^ (x << 2) ^ ...` — but computing this iteratively takes O(64) serial steps.

The CLMUL insight: multiplying `Q` by the all-ones constant `0xFFFF...FFFF` over GF(2) *is exactly* the prefix XOR. Over GF(2), "multiply by all-ones" is a polynomial shift-and-XOR that accumulates every bit from position 0 to i at position i. A single PCLMULQDQ instruction does this in ~3 cycles regardless of the number of bits set.

Result: a 64-bit `S` where bit `i` = 1 iff byte `i` is inside a string. The structural bitmap gets AND-ed with `~S` to remove false structural characters that were inside strings. An additional mask removes the `"` characters themselves (they're structural from the string-delimiter perspective but not part of JSON syntax at the token level).

Across a 64-byte block boundary, carry from the previous block is tracked in a single bit (whether the block ended inside a string), applied before the multiply.

### Stage 1, Step C: escape handling

Backslash-escaped characters (`\"`, `\\`, etc.) would cause quote-detection errors if not handled. The paper handles this with a separate bitmap of backslash positions, propagating "escaped" state via a similar carry trick. Characters following an odd-run of backslashes are marked escaped and excluded from the quote bitmap.

### Stage 2: tape building via bit extraction

With the structural bitmap `B` in hand, Stage 2 is:

```
while B != 0:
    pos = tzcnt(B)          // position of next set bit (= next structural char)
    B &= B - 1              // clear lowest set bit
    emit(input[pos], pos)   // push to tape based on character type
```

`tzcnt` is a single instruction (BSF/TZCNT) that extracts the bit position of the lowest set bit. The loop body is branch-free for the common case, processing roughly one structural character per 3-5 cycles.

"Emit to tape" means:
- For `{` / `[`: push current tape index onto a stack; record type and input offset.
- For `}` / `]`: pop the stack, write the forward-skip distance into the matching `{`/`[` tape entry, and write the backward-skip distance into the current entry.
- For `:`: record that the next value is a key's value.
- For `,`, whitespace: update parser state.
- For string starts (the `"` marks, tracked separately): record string start positions.
- For number starts: record number start positions; the actual number parsing happens lazily on demand.

The output tape is a flat array of 64-bit words. Each word packs a type tag and an offset into the input buffer. Objects and arrays additionally get a "skip to end" word inserted, enabling O(1) skipping over nested structures without recursion.

### Performance numbers

On a Skylake CPU (circa 2018), parsing a 100 MB JSON file:
- simdjson: ~50 ms = 2 GB/s throughput
- RapidJSON (the prior fastest parser): ~200-250 ms = 400-500 MB/s
- Speedup: ~4-6× on typical JSON; up to 10× on large, shallow files

Instruction count: simdjson uses roughly ¼ the instructions of RapidJSON on the same input. Most of the savings are in Stage 1: the SIMD path replaces per-byte branch-heavy code with a handful of vector instructions per cache line.

simdjson's throughput is within 2× of the raw `memcpy` speed for the same data, meaning it is close to the memory-bandwidth ceiling.

## Where it breaks

**1. SIMD availability.** The full-speed path requires AVX2 (Intel Haswell+, AMD Zen+). Fallbacks to SSE4.2 or NEON exist but are slower. On ARM, the CLMUL equivalent (PMULL) has different lane widths, requiring algorithm adjustments.

**2. Streaming / chunked input.** The two-stage design assumes the full input is in memory (or at least contiguous large blocks). Parsing a streaming network socket where chunks arrive at arbitrary byte boundaries requires buffering or more complex carry-state management across chunk boundaries. simdjson v2 ("On-demand" API) addresses some of this but the fundamental batch-processing model remains.

**3. Deeply nested or pathologically structured JSON.** The Stage 2 stack depth is bounded at 1024 by default. Files with 1025+ nesting levels are rejected. Real JSON rarely hits this limit, but it is an architectural constraint.

**4. Unicode validation overhead.** Standard-compliant parsers must validate UTF-8. simdjson does validate, but this adds overhead. Applications that can skip UTF-8 validation (known-good internal data) could be faster with a non-validating parser.

**5. Write-heavy workloads (serialization).** simdjson is a parsing (deserialization) library. It provides no SIMD-accelerated serialization; JSON writers are typically already fast because sequential writes are buffer-friendly.

**6. Generalizability of CLMUL.** The carry-less multiply trick is specific to problems with prefix parity structure (counting even/odd occurrences across a sliding window). It applies to string boundaries, balanced-parenthesis depth tracking, SIMD regex, and base64 decoding — but it is a specialized tool, not a general-purpose "make parsing fast" hammer.

## Why it works

The paper reveals that **JSON parsing is fundamentally a prefix-sum problem dressed in Unicode clothing**.

The "am I inside a string?" question is equivalent to "is the prefix XOR of the quote bitmap odd at this position?" — a prefix scan over a binary array. Prefix scans are one of the most fundamental parallel primitives (covered, for example, in GPU parallel programming as Blelloch's scan). On a single core, the CLMUL instruction provides this scan for a 64-position window in hardware.

This maps to a deeper principle: **any state that is a function of a prefix of the input is parallelizable across a fixed window**. The "inside string" state depends only on the count (mod 2) of preceding unescaped quotes. The "current nesting depth" depends on the count of preceding opens minus closes. Both are prefix sums. Prefix sums are associative and therefore data-parallel: Blelloch's GPU scan, Ring AllReduce, and CLMUL all exploit the same algebraic property.

The PSHUFB-as-lookup-table technique generalizes even further: any function from a byte value to a small integer can be computed in O(1) per 32 bytes by splitting the byte into nibbles and using two SIMD shuffles. This pattern is used in AES-NI key schedule operations, in Base64 decoding (the codec by Alfred Klomp), and in network packet classification. It is the SIMD version of a lookup table — "replace a branch-heavy per-element function with two indexed reads and an AND."

The broader lesson: **separate structure from content before interpreting either**. JSON's structural tokens (the 6 delimiters) are rare compared to string and number content. A 100-byte JSON string value contributes exactly two structural events (string start and string end). Extracting structure first with a branch-free SIMD pass, then interpreting structure with a branch-tolerant scalar pass over a dense index, exploits the sparsity of structure. The same insight is behind Dremel's repetition/definition levels (extract structural nesting before encoding column values), MonetDB/X100's vector-of-column-values (separate value extraction from predicate evaluation), and columnar storage formats generally.

## Going deeper

1. **simdjson paper (full text):** Geoff Langdale and Daniel Lemire, "Parsing Gigabytes of JSON per Second," *The VLDB Journal* 28(6), 2019. arXiv:1902.08318. The paper includes AVX2/SSE4.2 code snippets and detailed throughput breakdowns by file type and CPU microarchitecture.

2. **Wojciech Muła's blog on SIMD string processing:** Muła coined many of the PSHUFB lookup-table techniques used in simdjson, and his posts (e.g., on SIMD-accelerated UTF-8 validation, SIMD base64, and SSE/AVX string algorithms) are the best reference for the underlying SIMD primitive patterns. https://0x80.pl

3. **"Hyperscan: A Fast Multi-pattern Regex Matcher for Modern CPUs" (Intel Labs, 2019):** Takes the same "SIMD bit-parallel automaton" idea from string matching and applies it to full NFA/DFA execution for thousands of regex patterns simultaneously — showing that the bit-parallel automaton technique generalizes well beyond JSON structure.
