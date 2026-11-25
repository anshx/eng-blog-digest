---
title: "Understanding the FFT Algorithm"
source: https://jakevdp.github.io/blog/2013/08/28/understanding-the-fft/
author: Jake VanderPlas
company: UW eScience Institute
date_posted: 2013-08-28
date_digested: 2026-11-25
---

# Understanding the FFT Algorithm

## What's new to learn

1. **Discrete Fourier Transform (DFT)**: evaluating a degree-n polynomial at all n-th roots of unity in O(n²) — the mathematical operation underlying spectrum analysis, convolution, and polynomial multiplication.
2. **Butterfly decomposition**: the even/odd index split that lets n-point DFT evaluation recurse into two independent (n/2)-point DFTs, cutting complexity to O(n log n) by exploiting that squaring the n-th roots of unity yields the (n/2)-th roots.
3. **Convolution theorem**: multiplication in the frequency domain equals convolution in the time domain — the bridge that turns O(n²) sliding-window sums into three O(n log n) FFT transforms, applied everywhere from audio filtering to CNN layers to polynomial hashing.

## Prerequisites

- **Complex numbers as rotations**: Euler's formula e^(iθ) = cos θ + i sin θ. The n-th roots of unity are e^(2πik/n) for k=0…n−1 — n equally-spaced points on the unit circle.
- **Polynomial evaluation**: plugging a value x into a polynomial P(x) = a₀ + a₁x + … + aₙ₋₁xⁿ⁻¹.
- **Divide-and-conquer recursion**: merge sort is sufficient background — the FFT's recurrence is T(n) = 2T(n/2) + Θ(n).

## The core idea

The DFT of a signal (a₀, a₁, …, aₙ₋₁) computes:

```
X[k] = Σ_{j=0}^{n-1}  a_j · ω^(jk)   where ω = e^(2πi/n)
```

This is exactly evaluating the polynomial `P(x) = a₀ + a₁x + … + aₙ₋₁xⁿ⁻¹` at each of the n points x = ω⁰, ω¹, …, ωⁿ⁻¹.

**Naive:** n multiplications per evaluation × n points = O(n²). For n = 10⁶ samples, that is a trillion operations.

**The FFT insight:** Split P into even-indexed and odd-indexed terms:

```
P_even(y) = a₀ + a₂y + a₄y² + …
P_odd(y)  = a₁ + a₃y + a₅y² + …
P(x)      = P_even(x²) + x · P_odd(x²)
```

Now evaluate P at all n of the n-th roots of unity. After substituting x = ω^k:

```
P(ω^k)       = P_even(ω^(2k))  +  ω^k · P_odd(ω^(2k))
P(ω^(k+n/2)) = P_even(ω^(2k))  −  ω^k · P_odd(ω^(2k))
```

The second line is free: ω^(k+n/2) = −ω^k, because rotating by half a full circle negates. And ω^(2k) cycles through all (n/2)-th roots of unity as k goes from 0 to n/2−1 — exactly the input to a half-size DFT.

**Punchline:** evaluating at all n n-th roots reduces to evaluating P_even and P_odd each at the (n/2)-th roots, plus n combine operations. Two subproblems of half the size, O(n) combine → T(n) = 2T(n/2) + Θ(n) → Θ(n log n).

## Mechanics

**The butterfly** is the O(1) combine step named for the shape of its dataflow diagram. For each k from 0 to n/2 − 1:

```python
t          = exp(-2πi·k/n) * odd[k]   # one complex multiply (the "twiddle factor")
out[k]     = even[k] + t               # one complex add
out[k+n/2] = even[k] - t               # one complex subtract (free!)
```

Each level of the recursion touches every element once, performing n/2 butterflies. With log₂(n) levels: n log₂(n)/2 complex multiplications total.

**Iterative Cooley-Tukey (the production algorithm):**
The recursion unrolls into a fixed sequence of log₂(n) passes. Before the passes, the input is reordered by **bit-reversal permutation**: element at index j moves to index bit_reverse(j). This maps the access pattern of the recursive algorithm to a sequential scan at each level.

```
Input:     [a0, a1, a2, a3, a4, a5, a6, a7]  (n=8)
After bit-reverse: [a0, a4, a2, a6, a1, a5, a3, a7]
Pass 1 (butterfly width 2): 4 butterflies
Pass 2 (butterfly width 4): 4 butterflies
Pass 3 (butterfly width 8): 4 butterflies
Output: X[0]...X[7]
```

The twiddle factors e^(−2πi·k/n) for each pass are precomputed. Total arithmetic: n/2 · log₂(n) complex multiplications, n · log₂(n) complex additions.

**Numerical stability:** Accumulated floating-point error is O(ε · log₂ n), not O(ε · n), because errors at each butterfly are uncorrelated and partially cancel in the sum. The FFT is numerically gentle.

**The inverse FFT:** IDFT uses the same butterfly network with conjugated twiddle factors, divided by n at the end:

```
x[k] = (1/n) · Σ X[j] · ω^(−jk)
```

IDFT = FFT with ω replaced by ω⁻¹, then divide by n. Many implementations use the same code path with a flag.

## Where it breaks

**Power-of-two constraint.** Radix-2 Cooley-Tukey requires n to be a power of 2. For arbitrary n, one must either zero-pad to the next power of 2 (wastes memory and time) or use mixed-radix FFT (prime factors produce sub-problems of non-uniform sizes, more complex twiddle management). Rader's algorithm handles prime-length DFTs in O(n log n) via a convolution trick.

**Cache behavior.** The last few passes of a large FFT stride over the entire array with large offsets, missing cache. The "four-step FFT" (split into √n × √n matrix, transpose in tiles, then apply 1D FFTs per row/column) keeps each stage's working set in cache; cache-oblivious FFT achieves optimal I/O complexity without tuning — but both complicate the code.

**NTT constraints.** For polynomial multiplication over finite fields (as in lattice cryptography), the Number Theoretic Transform (NTT) requires n | (q−1) where q is the field modulus, not just n being a power of 2. This tightly constrains the ring parameters in post-quantum schemes like ML-KEM, which uses q=3329 and n=256 (3329 − 1 = 3328 = 256 × 13, so 256 | 3328 ✓).

**Non-stationary signals.** The DFT treats the entire signal as one spectral snapshot. If frequencies change over time (music, speech), the Short-Time Fourier Transform (STFT) windowed over overlapping blocks, or the wavelet transform, is more appropriate. The FFT itself is then applied per window.

## Why it works

**Algebraic structure of the evaluation domain.** The n-th roots of unity form the cyclic group Cₙ under multiplication. The squaring map σ: Cₙ → Cₙ/₂ is a 2-to-1 group homomorphism. This means every polynomial-evaluation task over Cₙ decomposes into two evaluation tasks over Cₙ/₂, automatically, recursively, to depth log₂(n). The butterfly is just the O(1) *combine* step of this group-structure-driven divide-and-conquer.

This is why the FFT only exists because the evaluation domain has this algebraic property. Random sets of n points do not. Roots of unity do.

**The convolution theorem as the payoff.** Convolution is the "sliding dot product":

```
(f * g)[k] = Σ_j f[j] · g[k−j]
```

It arises in signal filtering, cross-correlation, polynomial multiplication, and image blurring. Done directly it costs O(n²). The convolution theorem:

```
DFT(f * g) = DFT(f) · DFT(g)   (pointwise product)
```

So `f * g = IDFT( DFT(f) · DFT(g) )`. Cost: 3 FFTs at O(n log n) each. The core operation of a CNN layer (a 2D convolution) would be intractable for large kernels without this — practical implementations use FFT-based convolution when kernels exceed roughly 11×11.

**X is just an instance of Y.** The FFT is divide-and-conquer on polynomial evaluation at a structured point set:
- **Reed-Solomon erasure coding** (archive 2026-06-07): k data shards define a degree-(k−1) polynomial; parity shards are evaluations at additional points. The evaluation domain (a subfield of GF(2^8)) lacks the cyclic squaring structure, so there is no FFT speedup — the encoding matrix is computed directly. FFT would apply if GF(2^8) had suitable structure (it does for special subfields, used in Reed-Solomon variants over GF(2^m)).
- **ML-KEM NTT** (archive 2026-09-18): ML-KEM multiplies polynomials in Z₃₃₂₉[X]/(X^256+1). Its NTT is literally the FFT butterfly with ω = a primitive 256-th root of unity in Z₃₃₂₉. The only difference: complex multiplication is replaced by modular multiplication.
- **Karatsuba and Strassen**: both exploit algebraic decomposition to reduce the number of costly multiplications; FFT gets T(n) = 2T(n/2) + O(n) because the butterfly combine costs exactly O(n).
- **Cache-oblivious algorithms** (archive 2026-08-22): a cache-oblivious FFT recursively halves until tiles fit in cache, achieving optimal I/O without knowing cache size — the same van-Emde-Boas-layout idea applied to the butterfly call tree.

## Going deeper

1. **Cooley & Tukey (1965)** — "An Algorithm for the Machine Calculation of Complex Fourier Series," *Mathematics of Computation* 19(90): 297–301. Two pages, dense, worth reading to see how much the authors already understood about the structure they'd discovered.
2. **CLRS Chapter 30** — "Introduction to Algorithms," 3rd ed., §30.2–30.3: the clearest proof-first treatment, including the DFT matrix interpretation (DFT = Vandermonde matrix evaluated at roots of unity) and why IDFT is conjugate-and-divide.
3. **NIST FIPS 203, §4.3** — the ML-KEM specification shows the NTT butterfly in modular arithmetic, mapping directly to the FFT algorithm with ω replaced by 17 (a primitive 256-th root of unity mod 3329), the bridge between this post's math and a shipping post-quantum standard.
