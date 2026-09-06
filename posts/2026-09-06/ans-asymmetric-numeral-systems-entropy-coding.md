---
title: "Finite State Entropy: A New Breed of Entropy Coder"
source: http://fastcompression.blogspot.com/2013/12/finite-state-entropy-new-breed-of.html
author: Yann Collet
company: (independent; later Meta/Facebook)
date_posted: 2013-12-12
date_digested: 2026-09-06
tags: [compression, entropy-coding, algorithms, data-structures]
---

# Finite State Entropy: A New Breed of Entropy Coder

## What's new to learn

- **ANS (Asymmetric Numeral Systems)**: entropy coding where the compressed stream is a single growing integer "state" rather than a shrinking interval. Encoding a symbol with probability p advances the state by ≈ log₂(1/p) bits; decoding reverses the step. One number, not two bounds.
- **tANS / FSE (Finite State Entropy)**: the table-driven ANS variant that precomputes every (state, symbol) → (next_state, bits_emitted) pair into a small lookup table, reducing encode/decode to two table lookups per symbol with no arithmetic.
- **The compression-speed trilemma resolved**: Huffman rounds probabilities to powers of 1/2 (fast, suboptimal); arithmetic coding is Shannon-optimal but slow (interval arithmetic). ANS achieves Shannon-optimal compression at Huffman speed by replacing the interval with a bijective integer state.

## Prerequisites

- **Shannon entropy**: H(X) = -Σ p_s log₂ p_s bits/symbol is the theoretical minimum cost. Know that this is the goal; you cannot compress below it.
- **Huffman coding**: each symbol maps to a codeword whose length is ⌈-log₂ p_s⌉ — the rounding to integer lengths causes the gap from optimal.
- **Arithmetic coding**: the encoder maintains an interval [low, high) ⊂ [0,1); each symbol narrows it by factor p_s. The final interval is the encoding. Near-optimal, but requires division on each symbol.
- **Finite automata**: helpful for intuition, but not required — the DFA view will become obvious after seeing the state table.

## The core idea

Arithmetic coding maintains *two* numbers (low, high) and shrinks the interval multiplicatively. The bottleneck is that two-number bookkeeping and the divisions needed to scale both ends.

ANS keeps only *one* integer — call it the state `x`. Encoding a symbol s with frequency `fs` (out of total `M`) transforms the state as:

```
x_new = (x / fs) * M + cumulative_freq_before_s + (x % fs)
```

This looks like base-change arithmetic: we're treating x as a number written in a "variable base" where the digit corresponding to symbol s contributes `fs` values out of `M`. Each step grows x by a factor of M/fs ≈ 1/p_s, which is exactly the information content of that symbol.

Decoding is the exact inverse: given x, recover s (from `x % M`), then undo the step to get x_prev. Because each step is a bijection on integers, the decoder perfectly reconstructs the original sequence — in reverse order (LIFO: last encoded, first decoded).

The key payoff: when you quantize probabilities to `fs/M` and precompute a table mapping every pair `(state, symbol)` to the next state plus how many bits to flush/absorb, each encode/decode step costs exactly two table lookups and one bit-I/O operation. No multiplication, no division, no interval bookkeeping.

## Mechanics

### rANS: the arithmetic variant

The state `x` lives in a range `[L, b·L)` where `b` is typically 2⁸ (one byte of precision) and `L` is chosen so the state fits in 64 bits.

**Encoding** symbol s (with quantized frequency `fs`, cumulative `cs`, total `M = 2^k`):
1. **Renormalize out**: while `x ≥ fs << (64 - k)`, emit the low byte of x and right-shift x by 8.
2. **Step**: `x = (x / fs) * M + cs + (x % fs)`

**Decoding** (processes symbols in reverse order):
1. **Symbol**: `s` = lookup table at `x & (M-1)` → gives `(s, fs, cs)`
2. **Undo step**: `x = fs * (x >> k) + (x & (M-1)) - cs`
3. **Renormalize in**: while `x < L`, left-shift x by 8 and read one byte.

The decode is the algebraic inverse of encode. The bit-level renormalization is what keeps x in a bounded range between steps, ensuring the 64-bit state never overflows.

### tANS / FSE: the table variant

Instead of computing the step formula on every symbol, precompute everything:

```
decode_table[x]:    (symbol, next_x, bits_to_read)
encode_table[s][x]: (next_x, bits_to_write)
```

Both tables fit in ~4 KB (a typical FSE block uses M=4096 states). The encode path becomes:

```python
def encode(sym, x, out_bits):
    (next_x, bits_out) = encode_table[sym][x & (M-1)]
    out_bits.push(x >> log2_M, bits_out)  # flush top bits
    return next_x
```

Two table lookups, one bit-shift, one bit-push. The decode is symmetric.

### Table construction

The tANS table is built so that symbol s occupies exactly `fs` entries out of M in each period. Entries for the same symbol are spread "evenly" across the range using a quasi-random permutation (in FSE, a step size derived from a hash). This ensures the automaton's period is M and that no long runs of any one symbol create degenerate states.

The construction algorithm:
1. Allocate `fs` slots for each symbol s (ensuring Σ fs = M).
2. Assign each slot a "spread" position in the decode table using the pattern: `pos = (pos + step) % M`, where `step` is a "medium" step (M/2 + M/8 + 3 in FSE) to achieve good spread.
3. Each slot gets a `next_x` value that encodes "what state to transition to after emitting the bits needed to renormalize."

The key invariant: the encoder for symbol s, when given state x, emits `bits_out = floor(log2(x/fs))` bits (the high bits of x scaled by 1/fs) and stores the remainder as the new state.

### Why the state is a DFA

Reading the decode_table as a state-transition function: decode_table maps every state x to (next symbol output, next state). With bit I/O making the state space finite (M states × 2⁸ pending bits), this is exactly a DFA. The "compression" happens because high-probability symbols spend more time in states where only a few bits are emitted; low-probability symbols in states where more bits come out. The average bits emitted converges to H(X).

### Zstandard's usage

Zstandard applies two separate passes over a block:

1. **LZ77**: identifies back-references (offset, match-length, literal-length) to remove redundancy. Output is a stream of match records + raw literal bytes.
2. **Entropy coding** (the FSE/ANS step): the literal bytes are Huffman-coded; the match record fields (offset codes, match-length codes, literal-length codes) are FSE-coded with separate tables. Multiple interleaved FSE streams run simultaneously to amortize the LIFO overhead.

The reason literals use Huffman (not FSE) is that Huffman trees compress better for very short sequences (< ~200 bytes) where the FSE table overhead would dominate.

## Where it breaks

**LIFO order**: ANS decodes in reverse of encoding order. Zstandard handles this by coding blocks of fixed size (128 KB) and reversing the bitstream; single-symbol streaming codecs cannot use ANS directly without a buffer.

**Quantization error**: probabilities rounded to `fs/M` introduce a rate penalty of at most log₂(M) - log₂(fs) - (-log₂ p_s) bits. For M=4096 this is bounded but nonzero. For nearly-deterministic symbols (p > 99%), Huffman may actually win slightly.

**Adaptive coding is harder**: Arithmetic coding naturally supports "update the model after each symbol" (adaptive compression). ANS needs to rebuild its table when the model changes, making it better suited for "one table per block" (semi-static) coding than true symbol-by-symbol adaptation.

**No inherent parallelism within a stream**: the state x for symbol n depends on the state after symbol n-1. To use SIMD, Zstandard interleaves several independent ANS streams (typically 2 or 6), each encoding every k-th symbol. This recovers parallelism at the cost of a few extra state bytes of overhead.

**Error recovery is difficult**: a single-bit flip corrupts the state and all subsequent symbols in the block (no resync points within a stream), unlike some other codecs that allow partial decoding.

## Why it works

The deep principle: **entropy coding = finding a bijection from symbol sequences to bit strings where each sequence of length n using H(X)·n bits maps to a unique bitstring of that length, and the bijection is computable in O(1) per symbol.**

Arithmetic coding builds this bijection via interval shrinking. The interval [low, high) after encoding n symbols has width ∏ p_sᵢ. To represent it in bits you need -log₂(∏ p_sᵢ) = -Σ log₂ p_sᵢ ≈ n·H(X) bits. Correct, near-optimal — but you need two numbers and divisions.

ANS makes the insight: **replace the interval with its numerator alone**. Think of [low, high) as the rational `low / M` with denominator M growing at each step. ANS just keeps the numerator, applies the same multiplicative growth, and handles the denominator implicitly by tracking when the numerator overflows or underflows a power of 2 (emitting/absorbing bits at those moments).

Put differently: arithmetic coding is addition in a *non-integer* positional number system (the denominator shifts continuously). ANS is addition in an *integer* positional number system where the carry rule is "emit bits when you overflow." The state is the integer mantissa; bits are emitted/absorbed to keep it normalized.

The "so X is just Y" connection: **ANS is to arithmetic coding what fixed-point arithmetic is to floating-point arithmetic.** Both achieve the same range of values; fixed-point just eliminates the explicit exponent, instead emitting bits whenever the value overflows the representable range. This one simplification — from two-variable floating-point to one-variable fixed-point — halves the state and makes table precomputation practical.

Broader connections in the archive:
- **Thompson NFA** (2026-07-06): NFA reachability-set tracking is also a bijection between input symbol sequences and states; ANS is the same pattern for the information-theoretic rather than the language-recognition problem.
- **Copy-and-Patch JIT** (2026-08-02): FSE tables are precomputed offline and filled at runtime with problem-specific values — the same offline/online phase-separation behind JIT stencils.
- **Parallel Prefix Scan** (2026-07-15): the interleaved ANS streams that Zstandard uses to enable SIMD decoding are an instance of parallelizing an inherently sequential recurrence by running multiple independent sequences — the same split-and-merge trick behind efficient parallel scans.

## Going deeper

1. **Jarek Duda's arxiv paper** (2013, revised 2014): the original mathematical treatment of all ANS variants. Dense but complete — https://arxiv.org/abs/1311.2540

2. **Fabian "ryg" Giesen's rANS notes** (2014): the clearest engineering walkthrough of rANS, with worked bit-level examples and the renormalization invariant spelled out precisely — https://fgiesen.wordpress.com/2014/02/02/rans-notes/

3. **Zstandard source code (fse.c / huf.c)**: the production FSE implementation is ≈1000 lines of annotated C with the table construction and encode/decode loops — https://github.com/facebook/zstd/tree/dev/lib/common — reading these after the two references above makes the theory concrete.
