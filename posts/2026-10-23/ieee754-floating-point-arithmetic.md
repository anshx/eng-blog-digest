---
title: "What Every Computer Scientist Should Know About Floating-Point Arithmetic"
source: https://docs.oracle.com/cd/E19422-01/819-3693/ncg_goldberg.html
author: David Goldberg
company: Xerox PARC / ACM Computing Surveys
date_posted: 1991-03-01
date_digested: 2026-10-23
---

# What Every Computer Scientist Should Know About Floating-Point Arithmetic

## What's new to learn

1. **Machine epsilon and the relative error model**: Every IEEE 754 operation is computed as if with infinite precision and then rounded to the nearest representable float. This gives a uniform relative error bound of ε_machine (≈ 2⁻²³ ≈ 1.19 × 10⁻⁷ for float32) per operation — turning floating-point error analysis from ad-hoc guesswork into a theorem.

2. **Catastrophic cancellation**: Subtracting two nearly-equal floats does not introduce new rounding error. Instead it *reveals* the error that was already hidden in each operand, amplifying their relative errors by a factor of |a| / |a - b|. This is qualitatively different from normal rounding, and the only fix is algebraic reformulation.

3. **IEEE 754's faithfully-rounded specification**: Before 1985, every CPU vendor chose its own FP behavior, making portable numerical software impossible. IEEE 754 mandated that +, -, *, /, and √ each return the closest representable float to the exact mathematical result (round-to-nearest-even by default). This one guarantee is what lets numerical analysts write portable error bounds.

## Prerequisites

- Binary positional notation (how a sign-magnitude integer works)
- What "relative error" means (|computed - exact| / |exact|)
- Basic familiarity with computer arithmetic operations

Non-obvious prerequisite: the difference between *rounding error* (introduced by representation) and *cancellation* (pre-existing error revealed by subtraction). Most engineers conflate them; the paper sharply distinguishes them.

## The core idea

A floating-point number represents the value ±1.f × 2^e, where f is the binary fraction (the *significand*) and e is an integer exponent. float32 gives 24 bits of significand (including the implicit leading 1), float64 gives 53 bits. The *machine epsilon* ε_machine is the gap between 1.0 and the next representable float above it.

IEEE 754's key contract: for every basic operation ⊕ ∈ {+, -, ×, /, √},

```
fl(a ⊕ b) = (a ⊕ b)(1 + δ),    |δ| ≤ ε_machine
```

That is, the computed result is the exact mathematical result, multiplied by a factor within 1 ± ε of 1. No rounding error can exceed ε_machine in relative terms.

This makes error analysis composable. After a chain of n operations, the accumulated relative error is at most *nε* (first-order) — a tractable bound even for complex computations. The dangerous case is when the "exact" intermediate result is much smaller than the operands themselves, because the relative error of that small result can be orders of magnitude larger.

## Mechanics

### Representation

float32 layout — 32 bits total:

```
[s][eeeeeeee][fffffffffffffffffffffff]
 1     8              23
```

- **s**: sign (0 = positive)
- **e**: biased exponent (stored = actual + 127; e=0 and e=255 are special)
- **f**: fractional significand bits (the leading 1 is implicit, giving 24 bits total)

Value = (-1)^s × 1.f × 2^(e − 127)  for 0 < e < 255.

float64 uses 11 exponent bits (bias = 1023) and 52 fractional bits (53 bits total significand).

| Type | Significand bits | ε_machine | Range |
|------|-----------------|-----------|-------|
| float16 | 11 | 2⁻¹⁰ ≈ 9.8×10⁻⁴ | ±65,504 |
| bfloat16 | 8 | 2⁻⁷ ≈ 7.8×10⁻³ | ±3.4×10³⁸ |
| float32 | 24 | 2⁻²³ ≈ 1.2×10⁻⁷ | ±3.4×10³⁸ |
| float64 | 53 | 2⁻⁵² ≈ 2.2×10⁻¹⁶ | ±1.8×10³⁰⁸ |

bfloat16 is float32 truncated to 8 mantissa bits. It sacrifices 75% of float32's precision but keeps the same 8-exponent-bit range — exactly what ML training needs.

### Rounding

IEEE 754 mandates four rounding modes: round-to-nearest-even (the default, ties round to even last bit), round-toward-zero (truncation), round-toward-+∞, and round-toward-−∞. The last two are used in interval arithmetic, where you deliberately track an upper and lower bound.

"Guard digits": before rounding, the hardware keeps extra internal precision bits (guard, round, sticky). These ensure that the rounded result is the *closest* representable float, not merely *a close one*.

### Catastrophic cancellation, illustrated

Compute f(x) = √(x² + 1) − 1 near x = 0.

Naive: with x = 10⁻⁴ in float64:
- x² = 10⁻⁸ (rounds fine)
- x² + 1 = 1.00000001 → but float64 can only represent 1 + 2⁻⁵² ≈ 1 + 2.2×10⁻¹⁶, so 1.00000001 is representable to full precision
- √(1.00000001) ≈ 1.000000005 — still fine
- Subtract 1: answer ≈ 5×10⁻⁹ ✓

Now with x = 10⁻⁸:
- x² = 10⁻¹⁶ (near float64's ε), so x² + 1 rounds to exactly 1.0
- √(1.0) = 1.0 exactly
- 1.0 − 1 = 0.0 — the true answer is ≈ 5×10⁻¹⁷, but we get 0!

The fix — reformulate to avoid the subtraction:

```
f(x) = (√(x² + 1) − 1) × (√(x² + 1) + 1) / (√(x² + 1) + 1)
     = x² / (√(x² + 1) + 1)
```

Now near x = 0, the denominator → 2, and the result is x²/2 with no cancellation.

The general pattern: if you find yourself computing a - b where a ≈ b, look for an algebraically equivalent form that avoids it. The quadratic formula's small root, computing `log(1 + x)` for small x via `expm1`, and `sin(x)/x` for small x via `sinc` are classic examples.

### Kahan Summation: compensated summation in O(ε) error

Naive sum of n numbers: error ≈ O(nε) in relative terms (error grows linearly with n).

Kahan's algorithm keeps a *compensation variable* c that captures the bits lost in each addition:

```python
def kahan_sum(values):
    total = 0.0
    c = 0.0          # running compensation for lost low-order bits
    for x in values:
        y = x - c    # compensate: c is the error from last step
        t = total + y
        c = (t - total) - y  # algebraic identity, but in FP this != 0
        total = t
    return total
```

Why this works: after `t = total + y`, the low-order bits of `y` are lost (the gap between `t` and `total` is large). The expression `(t - total)` recovers the high-order bits that were actually added, so `c = (t - total) - y` captures the part of `y` that was *not* added. This residual is compensated in the next iteration.

Result: Kahan summation's error is O(ε) regardless of n — essentially just one rounding error for the whole sum. SQL databases that compute SUM() in float64 still get O(nε) naive error; systems that need high-fidelity aggregations (financial, scientific) typically use Kahan or switch to exact integer arithmetic.

### Subnormals (denormals)

When the exponent bits are all 0, the float is "subnormal": value = (-1)^s × 0.f × 2⁻¹²⁶ (for float32). These fill the gap between 0 and the smallest normal float (≈ 1.2×10⁻³⁸) with evenly-spaced values, giving "gradual underflow" instead of a sudden jump to zero.

Why gradual underflow matters: without subnormals, the error in `a - b` near zero would be 0 or ∞ — catastrophic. With subnormals, `|fl(a) - fl(b)| / |a - b|` is bounded near zero.

Performance trap: on many CPUs, operations involving subnormals trigger the FPU's exception handler and run 10-100× slower. In deep learning, subnormal gradients can silently stall training. PyTorch and CUDA both set "flush-to-zero" (FTZ) and "denormals-are-zero" (DAZ) CPU flags by default — trading gradual underflow for performance.

### Special values: NaN, ±∞, −0

IEEE 754 defines three special values beyond normals and subnormals:

- **NaN (Not a Number)**: exponent all 1s, non-zero mantissa. Result of 0/0, √(−1), ∞ − ∞. Key property: *NaN ≠ NaN* in every comparison, including itself (`x == x` is false if x is NaN). This is the standard test for NaN.
- **±∞**: exponent all 1s, zero mantissa. Result of 1/0, log(0). Infinity propagates sensibly through most operations (∞ + 1 = ∞, but ∞ − ∞ = NaN).
- **−0**: the signed zero. −0 == +0 is true in all comparisons, but 1/(−0) = −∞ ≠ 1/(+0) = +∞. Mostly invisible in practice, but can surprise code that takes 1/x.

### Fused Multiply-Add (FMA)

The FMA instruction computes a×b + c with a single rounding at the end (not two):

```
fma(a, b, c) = round(a × b + c)    # one round, result of infinite-precision a×b+c
vs.
a * b + c    = round(round(a × b) + c)   # two rounds
```

FMA gives a more accurate result and is essential for computing dot products (the core of matrix multiply). Nearly all modern CPUs and GPUs have FMA instructions. Ironically, FMA can break naive error analyses that assume two separate roundings.

## Where it breaks

**Non-associativity**: `(a + b) + c ≠ a + (b + c)` in floating-point. SIMD vectorization and parallel reductions reorder summations for performance, producing non-deterministic results when the values span many orders of magnitude. This is why ML training loss can vary between CPU and GPU, between one GPU and eight, and between compilers.

**`-ffast-math` and friends**: GCC/clang's `-ffast-math` flag (and its constituent `-fassociative-math`) allows the compiler to reorder floating-point operations as if they were exact. Commonly used in HPC and ML kernels for 20-40% speedups. The resulting code is no longer IEEE 754 compliant — it can produce NaN where IEEE would not, and its error bounds are much harder to analyze.

**Compound transcendentals**: IEEE 754 mandates faithfully-rounded results only for +, -, ×, /, and √. It does *not* mandate correct rounding for sin, cos, exp, log, etc. In practice these functions are accurate to 0.5–2 ULP but are not guaranteed to be the closest representable float.

**Integer corner cases**: Integers up to 2^24 are exactly representable in float32 (24-bit significand). Beyond that, consecutive integers differ by more than 1 ULP — e.g., `(float32)(16777217) == (float32)(16777216)` is true. This bites any code using floats as integer IDs or loop counters beyond ~16M.

**Determinism in distributed ML**: Two GPUs with slightly different instruction scheduling can compute gradients in different orders, producing different parameter updates. The accumulation diverges exponentially over training steps. Building reproducible ML training requires synchronized reductions (e.g., AllReduce with a deterministic NCCL kernel) and disabled FTZ/DAZ flags for diagnostics.

## Why it works

The deeper principle: **IEEE 754 made error analysis tractable by separating representation from rounding semantics, then specifying the rounding exactly**.

Before 1985, different FP hardware on different vendors applied different rounding, maintained different precision, and had different special-value behaviors. Writing a portable numerical subroutine was impossible — algorithms valid on one machine could silently fail on another. The IEEE 754 standard (1985, revised 2008) changed this by:

1. Specifying a precise bit layout for single and double precision
2. Mandating *faithfully rounded* (i.e., correctly rounded to nearest) results for the five basic operations
3. Defining gradual underflow via subnormals, preventing abrupt loss of significance near zero
4. Specifying NaN/infinity propagation rules so IEEE arithmetic is a closed algebraic system

This is the same insight as Raft replacing "Paxos Made Simple" with a fully-specified algorithm: not just defining what the right answer is, but defining *exactly how to compute it* so that implementations can be verified and composed.

**Connection to modern ML**: Mixed-precision training (Micikevicius et al., ICLR 2018) is a direct application of this model. Weights and activations are stored in float16 or bfloat16 for throughput; a master copy of weights is kept in float32 for accumulation precision; gradient scaling multiplies the loss by a large constant to push gradients away from the subnormal range before computing them in float16. Each design choice reflects an explicit trade against the ε_machine formula:

- bfloat16 has the same dynamic range as float32 (8 exponent bits) so gradients rarely overflow
- bfloat16's 7 mantissa bits give ε_machine ≈ 7.8×10⁻³ — coarse but sufficient for noisy SGD
- Float32 accumulation gives ε_machine ≈ 1.2×10⁻⁷ for the master copy, sufficient for convergence

The key observation: SGD's noise floor (stochastic gradient variance) is much larger than float32's ε, so bfloat16's coarser precision is hidden inside the optimizer's noise. This is the same "precision commensurate with the signal" reasoning behind JPEG's quantization tables.

## Going deeper

1. **Goldberg (1991)** — the original paper. The sections on guard digits and exactly why IEEE chose round-to-nearest-even are worth re-reading closely. Available at the Oracle documentation mirror.

2. **Kahan, W. (1997). "Lecture Notes on the Status of IEEE 754."** — Kahan was the principal architect of IEEE 754. His notes on why certain decisions were made (especially signed zero and gradual underflow) are uniquely authoritative. Available at the University of California, Berkeley.

3. **Micikevicius, P. et al. (2018). "Mixed Precision Training."** ICLR. arXiv:1710.03740. The canonical reference for float16 training in deep learning — applies the IEEE 754 error model directly to decide which operations need float32 precision.
