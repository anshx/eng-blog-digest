---
title: "Basic Lattice Cryptography: The concepts behind Kyber (ML-KEM) and Dilithium (ML-DSA)"
source: https://eprint.iacr.org/2024/1287
author: Vadim Lyubashevsky
company: IBM Research / IACR ePrint Archive
date_posted: 2024-09-01
date_digested: 2026-09-18
---

# Basic Lattice Cryptography: The concepts behind Kyber (ML-KEM) and Dilithium (ML-DSA)

## What's new to learn

1. **The Learning With Errors (LWE) problem** — Given a matrix A and a vector b = As + e where e is tiny random noise, find the secret s. This single problem is the hard core of nearly all post-quantum cryptography.

2. **Why quantum computers can't break lattices** — Shor's algorithm works by exploiting hidden periodic subgroup structure in cyclic groups (factoring, discrete log). Lattice problems have no such group structure; their worst-case hardness is what makes them quantum-resistant.

3. **The Number Theoretic Transform (NTT)** — The FFT applied to Z_q[x]/(x^n+1), the finite-field polynomial ring at the heart of ML-KEM. It turns O(n²) polynomial multiplication into O(n log n), making the whole scheme practical.

## Prerequisites

- Modular arithmetic (Z_q = integers mod q)
- Basic linear algebra (matrix-vector multiplication)
- What a key encapsulation mechanism (KEM) is: like ECDH, two parties establish a shared secret using public messages; the shared secret then seeds symmetric encryption
- High-level intuition for why RSA/ECDH are hard (one-way functions based on factoring / discrete log) — useful contrast

## The core idea

Every public-key cryptosystem is a one-way function with a trapdoor:

- RSA: multiplying two primes is easy, factoring the product is hard. The trapdoor is the prime factors.
- ECDH: computing g^x mod p is easy, finding x from g^x is hard (discrete log). The trapdoor is x.
- **ML-KEM**: computing As + e for random A and small e is easy, finding s from (A, As + e) is hard (LWE). The trapdoor is s.

The genius is where the noise e goes. Both the encryptor and decryptor know A and the ciphertext. Only the holder of s can *cancel* the noise and recover the plaintext — because the noise terms in the final computation are all multiples of s, and knowing s lets you subtract them exactly. Without s, the noise terms are indistinguishable from random.

The "aha" moment: **encryption is just intentionally losing information to noise, and decryption is knowing the exact noise seed to recover it.** This is dual to error-correcting codes: an ECC adds controlled redundancy so you can recover bits lost to channel noise; LWE adds controlled noise so an adversary can't recover the signal without the private key.

## Mechanics

### Step 1: The LWE problem

Pick a prime q (in ML-KEM, q = 3329), a dimension n = 256, and a small-norm distribution χ (binomial distribution with parameter 2, sampled values in {-2,...,2}).

LWE asks: given random A ∈ Z_q^{m×n} and b = As + e (mod q) with secret s ∈ Z_q^n and small error e ← χ^m, find s.

The key property: without error, Gaussian elimination solves this in O(n³). With even a tiny error, no known classical or quantum algorithm does better than exponential in n.

### Step 2: Ring-LWE — compressing to polynomials

Plain LWE has O(n²) key sizes. Ring-LWE replaces integer vectors with polynomials in R_q = Z_q[x]/(x^n+1):

- Elements of R_q are polynomials of degree < n with coefficients in Z_q
- Addition is componentwise; multiplication is polynomial multiplication mod (x^n+1)
- The ring identity x^n ≡ −1 creates a "rotation with negation" structure that allows an n×n matrix to be described by a single polynomial

This shrinks key sizes from O(n²) to O(n) while keeping the LWE hardness assumption.

### Step 3: Module-LWE — the sweet spot

ML-KEM uses Module-LWE: a k×k matrix **A** of R_q elements (k=3 for ML-KEM-768). Think of it as a Lego block system:

- k=1 recovers Ring-LWE (fastest, smaller parameters)
- k→∞ approaches plain LWE (largest parameters, most conservative security)
- k=2,3,4 are the ML-KEM parameter sets

Key generation:
```
A  ← Z_q^{k×k}[x]/(x^n+1)   # public random matrix (768 bytes for k=3)
s  ← χ^k                     # small-norm secret
e  ← χ^k                     # small-norm error
t  = A·s + e                  # public key "noisy product"
pk = (A, t),  sk = s
```

### Step 4: Encapsulation

To send a random message m ∈ {0,1}^256:

```
r   ← χ^k          # fresh random
e1  ← χ^k          # fresh error
e2  ← χ            # fresh error (scalar)
u   = A^T · r + e1                      # noisy version of A^T r
v   = t^T · r + e2 + ⌊q/2⌋ · m        # message embedded at 0 or q/2
ciphertext = (u, v)
shared_secret K = Hash(m, Hash(pk, u, v))
```

Why embed at ⌊q/2⌋? Because the interval [0, q/2) decodes to bit 0 and [q/2, q) decodes to bit 1. Values near 0 or q/2 survive small perturbations without flipping.

### Step 5: Decapsulation

```
m'_raw = v - s^T · u
       = t^T·r + e2 + ⌊q/2⌋·m   −   s^T·(A^T·r + e1)
       = (As+e)^T·r + e2 + ⌊q/2⌋·m  −  s^T·A^T·r  −  s^T·e1
       = s^T·A^T·r + e^T·r + e2 + ⌊q/2⌋·m  −  s^T·A^T·r  −  s^T·e1
       = e^T·r + e2 − s^T·e1 + ⌊q/2⌋·m
         ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
         all small (product of small terms)
m' = round(m'_raw)   # recover bit by rounding each coefficient to 0 or 1
```

The noise error accumulates as e^T·r + e2 − s^T·e1. With χ having values in {−2,...,2} and n=256 coefficients, the noise magnitude stays well below q/4 ≈ 832 with overwhelming probability — so rounding always recovers m.

### Step 6: CCA security via the Fujisaki-Okamoto (FO) transform

Naive LWE encryption is only CPA-secure (an attacker who can ask decryption queries can extract bits). ML-KEM adds the FO transform:

1. During decapsulation, re-encrypt m' to get (u', v')
2. If (u', v') ≠ (u, v), return a pseudorandom K'' derived from a secret seed (implicit rejection)
3. Otherwise return K' = Hash(m', Hash(u, v))

This constant-time comparison defeats active attacks: a ciphertext that wasn't honestly generated will fail the re-encryption check and return garbage, so an adversary gains nothing from submitting malformed ciphertexts.

### Step 7: NTT — making polynomial multiplication fast

The bottleneck is computing A·s and A^T·r: k² polynomial multiplications mod (x^256+1) over Z_3329.

Naive polynomial multiplication is O(n²). The Number Theoretic Transform (NTT) is FFT over Z_q:

- Choose q = 3329 because 3329 = 13 × 256 + 1, so 256th roots of unity exist in Z_q (Fermat's little theorem guarantees this when 256 | q−1)
- Map polynomial coefficients to their "evaluation form" via NTT in O(n log n)
- Multiply pointwise in O(n)
- Invert via inverse-NTT in O(n log n)

The ring structure x^256 ≡ −1 enables a negacyclic NTT variant where the "butterfly" uses twiddle factors ωi that are primitive 512th roots of unity (since 512 = 2×256 and 512 | q−1).

For k=3: a 3×3 matrix-vector product is 9 polynomial multiplications, each taking ~1000 operations on 256-element vectors — a few thousand arithmetic operations total. A modern CPU can encapsulate or decapsulate a key in under 100μs.

## Where it breaks

**Decryption failures (rare but non-zero)**: The noise bound is not absolute. If all n=256 error coefficients conspire to be at their maximum simultaneously (probability ≈ 2^{-140}), the final noise exceeds q/4 and decryption fails. ML-KEM's parameters are chosen so this probability is negligible.

**Constant-time implementation is critical**: Timing variations in the comparison step of the FO transform leak bits. KyberSlash (2023) exploited a non-constant-time division in the reference implementation to extract private keys via timing side-channel — a reminder that mathematical security doesn't imply implementation security.

**Side channels in NTT**: The NTT's butterfly operations access memory non-sequentially, creating cache timing channels. Production implementations use bitreversed-order NTT to make access patterns predictable.

**Key size inflation**: ML-KEM-768 public key = 1184 bytes; ECDH-P256 = 64 bytes. 18× larger. TLS handshakes balloon. Cloudflare and Google measured ~1.8 ms added to TLS connection time on high-BDP paths due to the larger messages in flight, not computation.

**Not a drop-in for signatures**: ML-KEM is a KEM (key exchange), not a signature scheme. ML-DSA (Dilithium) covers signatures but has a different hardness proof (Module-SIS, not Module-LWE) and different trade-offs. You cannot use ML-KEM where you need non-repudiation.

## Why it works

The deepest insight is why LWE resists quantum attacks. Shor's algorithm applies to any problem reducible to "find the period of a group homomorphism":

- Factoring: order of (ℤ/Nℤ)×
- Discrete log: discrete logarithm in a cyclic group

LWE has no such group structure. Its hardness connects via a worst-to-average-case reduction (Regev, 2005) to the *Shortest Vector Problem* (SVP) on random lattices — finding the shortest nonzero vector in a lattice defined by a random integer matrix. No quantum algorithm is known that gives better than polynomial speedup on SVP in the worst case, despite 50+ years of effort.

The reduction is probabilistic: if you could solve LWE on average (as a random oracle), you could solve SVP in the worst case (the hardest possible instance). This makes LWE an unusually robust hardness assumption — breaking it in the average case (which is what an attacker sees) is provably as hard as breaking it in the worst case.

The NTT connection reveals another deep principle: **structured algebraic operations over finite fields are the universal building block for both cryptography (LWE over Z_q polynomials) and signal processing (FFT over ℂ polynomials)**. The same Cooley-Tukey butterfly works in both contexts; the field changes but the algorithm does not.

## Going deeper

1. **"A Decade of Lattice Cryptography" by Chris Peikert (2016)** — the definitive survey of lattice crypto up to Ring-LWE; covers the proof techniques and historical development that led to Kyber.

2. **"Crystals-Kyber Algorithm Specifications" (NIST FIPS 203)** — the actual standard, surprisingly readable; Appendix A gives the full algorithm and pseudocode with parameter tables. Pay attention to the compress/decompress functions, which are the implementation of "round to 0 or q/2."

3. **"The Impact of Kyber in TLS" — IETF RFC draft** (various versions, 2024–2026) — tracks how hybrid X25519+ML-KEM key exchange is rolling out in TLS 1.3, including measurement data from Cloudflare and Chrome on handshake latency cost.
