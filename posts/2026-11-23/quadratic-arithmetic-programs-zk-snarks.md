---
title: "Quadratic Arithmetic Programs: from Zero to Hero"
source: https://medium.com/VitalikButerin/quadratic-arithmetic-programs-from-zero-to-hero-f6d558cea649
author: Vitalik Buterin
company: Ethereum Foundation
date_posted: 2016-12-11
date_digested: 2026-11-23
---

# Quadratic Arithmetic Programs: from Zero to Hero

## What's new to learn

1. **R1CS (Rank-1 Constraint System)**: A representation of any computation as a list of "gate constraints" of the form `(a·s) * (b·s) = (c·s)`, where `s` is the full witness vector — all intermediate wire values in a circuit — and `a`, `b`, `c` are selector vectors that pick out left-input, right-input, and output for each gate.

2. **QAP (Quadratic Arithmetic Program)**: The step that turns an R1CS into polynomial arithmetic via Lagrange interpolation. Each column of the R1CS selector matrices becomes a polynomial. Satisfying all constraints simultaneously becomes equivalent to a single polynomial divisibility condition: `A(x) * B(x) - C(x) = H(x) * Z(x)`.

3. **Schwartz-Zippel as the universal verifier**: Two distinct polynomials of degree ≤ d over a field F_p can agree on at most d out of p points. So checking the QAP equation at a single random τ is overwhelmingly likely to catch any cheating prover — the same lemma that makes Reed-Solomon codes correct, Rabin-Karp fast, and Freivalds' algorithm work.

## Prerequisites

- Finite field arithmetic (arithmetic mod a prime p; every nonzero element has a multiplicative inverse)
- Lagrange interpolation: given n points (x_i, y_i) with distinct x_i, there is a unique polynomial of degree ≤ n-1 through them
- Arithmetic circuits: directed acyclic graphs with field-element wire values and addition/multiplication gates
- Polynomial division: `P = H * Z + R`; we care about the case `R = 0` (Z divides P)

You do not need to know elliptic curves or pairings to follow this post — those are needed to make τ cryptographically secret, which is the topic of the follow-up post "Zk-SNARKs: Under the Hood."

## The core idea

Suppose you want to convince a verifier you computed `f(x) = y` correctly without revealing x. The route ZK-SNARKs take:

> **Computation → Arithmetic Circuit → R1CS → QAP → Check one point τ**

**The pivot insight**: satisfying a list of n gate constraints simultaneously is equivalent to a single polynomial being divisible by another. The verifier, instead of re-executing n gates, checks one field-element equation. By Schwartz-Zippel, this is indistinguishable from checking all n — the prover cannot fake divisibility at a random point without actually knowing a valid witness.

The process:

1. **Flatten** the computation into an arithmetic circuit and then into a list of R1CS constraints, one per multiplication gate.

2. **Encode** each column of the constraint matrices as a polynomial using Lagrange interpolation, one evaluation point per constraint.

3. **Compose** those polynomials with the witness to get A(x), B(x), C(x). Valid witness ⟺ `A(x)*B(x) - C(x)` is divisible by `Z(x) = ∏_i (x - r_i)`.

4. **Verify** at a random τ: `A(τ)*B(τ) - C(τ) = H(τ)*Z(τ)`.

The verifier does O(1) field multiplications. The prover does O(n) work. That asymmetry — and the near-zero false-positive probability — is the whole SNARK.

## Mechanics

### Step 1: Flattening to R1CS

Take the small computation `y = x³ + x + 5`. Introduce intermediate variables:

```
sym_1  = x * x          # x²
sym_2  = sym_1 * x      # x³
sym_3  = sym_2 + x      # x³ + x
y      = sym_3 + 5      # x³ + x + 5
```

There are two multiplication gates (the others are free additions). The witness vector packs all wire values including a constant-1 slot:

```
s = [1, x, y, sym_1, sym_2, sym_3]
  = [1, 3, 35, 9, 27, 30]      (for input x = 3)
```

Each multiplication gate contributes one R1CS constraint: left-input selector `a`, right-input selector `b`, output selector `c` — all length-6 row vectors. For gate 1 (`sym_1 = x * x`):

```
a = [0, 1, 0, 0, 0, 0]   # picks x
b = [0, 1, 0, 0, 0, 0]   # picks x
c = [0, 0, 0, 1, 0, 0]   # picks sym_1
```

The constraint `(a·s) * (b·s) = (c·s)` reads: `x * x = sym_1`. ✓

For gate 2 (`sym_2 = sym_1 * x`):

```
a = [0, 0, 0, 1, 0, 0]   # picks sym_1
b = [0, 1, 0, 0, 0, 0]   # picks x
c = [0, 0, 0, 0, 1, 0]   # picks sym_2
```

Addition gates need a constraint of the form `(A + B) * 1 = C`, filling in `b = [1, 0, ...]` (picks the constant 1). The full R1CS is a 3-column matrix of selector vectors, one row per gate.

### Step 2: R1CS to QAP via Lagrange interpolation

Pick evaluation points `r_1, r_2, ..., r_n` — one per constraint (say `r_i = i`). For every *column* j of the left selector matrix `A`, use Lagrange interpolation to build a polynomial `A_j(x)` that passes through the points `(r_i, A[i][j])` for all i. Same for `B_j` and `C_j`.

The composed polynomials:

```
A(x) = sum_j  s[j] * A_j(x)
B(x) = sum_j  s[j] * B_j(x)
C(x) = sum_j  s[j] * C_j(x)
```

Each is at most degree n-1 in x. The key fact: **at each evaluation point r_i, A(r_i) = a_i·s, B(r_i) = b_i·s, C(r_i) = c_i·s** — exactly the left, right, and output values for gate i.

If the witness satisfies all constraints, then for every r_i:

```
A(r_i) * B(r_i) = C(r_i)
```

which means the degree-2n polynomial `A(x)*B(x) - C(x)` vanishes at every r_i. Since all roots are known:

```
Z(x) = (x - r_1)(x - r_2)···(x - r_n)   divides   A(x)*B(x) - C(x)
```

The **QAP check** is: given the witness, the prover can compute the quotient:

```
H(x) = (A(x)*B(x) - C(x)) / Z(x)
```

and this division is exact (no remainder) if and only if the witness is valid.

### Step 3: Verification at a random point τ

The verifier samples τ uniformly at random from F_p and checks:

```
A(τ) * B(τ) - C(τ)  ==  H(τ) * Z(τ)
```

The prover supplies `(A(τ), B(τ), C(τ), H(τ))`.

**Why this is sound (Schwartz-Zippel):** Suppose the prover is cheating — the witness is invalid, so `A(x)*B(x) - C(x) - H(x)*Z(x)` is a non-zero polynomial of degree ≤ 2n. By Schwartz-Zippel, the number of roots is at most 2n, so a random τ is a root with probability ≤ 2n / p. For typical p ≈ 2^254 and n ≤ 10^6, this is ≈ 10^{-69}. The verifier safely ignores the negligible false-positive probability.

### Step 4: Making τ secret (sketch)

If the prover knew τ in advance, they could fabricate a fake A(τ), B(τ), C(τ) satisfying the check without knowing a witness. The fix is a **trusted setup**: before the system runs, a trusted party (or multi-party ceremony) samples τ, computes `{g^(τ^0), g^(τ^1), ..., g^(τ^d)}` on an elliptic curve group, and publishes these "blinded powers of τ." Then it destroys τ.

The prover can evaluate any polynomial P at τ homomorphically:

```
g^P(τ) = g^(p_0 + p_1·τ + ... + p_d·τ^d)
        = ∏_i  (g^(τ^i))^p_i
```

without ever learning τ. The verifier uses an elliptic curve *bilinear pairing* `e(·, ·)` to check the multiplicative relationship `A(τ)*B(τ) = C(τ) + H(τ)*Z(τ)` on group elements:

```
e(g^A(τ), g^B(τ))  ==  e(g^C(τ), g) · e(g^H(τ), g^Z(τ))
```

This is the full SNARK (specifically, the Pinocchio / Groth16 construction). Two pairing evaluations — constant time regardless of circuit size.

## Where it breaks

**Trusted setup ("toxic waste"):** The secret τ, if retained, lets anyone forge proofs. Real deployments use Powers of Tau multi-party ceremonies where τ is secure as long as *any one participant* honest-deletes their secret. Groth16 also requires a *per-circuit* setup, so each new circuit needs a new ceremony — a significant operational burden. Universal SNARKs (PLONK, Marlin) need only one setup for all circuits but pay with slightly larger proofs.

**Prover cost:** The bottleneck is multi-scalar multiplication (MSM) over the elliptic curve group: computing `∑ a_i · G_i` for n group points and n field scalars. This is O(n / log n) with Pippenger's algorithm and dominates at O(100ms–10s) for circuits with 10^6–10^8 gates, even on specialized hardware. This is why ZK prover acceleration (GPUs, FPGAs, ASICs) is an active hardware problem.

**Field arithmetic gadgets:** Everything must be expressed as additions and multiplications in F_p. Native 64-bit integers require range-check circuits (~30 constraints per 64-bit integer). SHA-256 requires ~30,000 constraints. A single Keccak-256 round requires ~150,000. Any application that needs cryptographic hashes inside a circuit pays this cost.

**R1CS doesn't scale to RAM programs naturally:** The original construction assumes a fixed-size circuit representing the whole computation. Programs with loops, variable-length inputs, or memory access (loads/stores) need a zkVM — either a full CPU emulator expressed as a circuit (RISC Zero, Jolt) or a custom VM with lookup arguments (Polygon zkEVM, Aztec's Noir). These add factors of 10–100× circuit overhead compared to hand-written circuits.

**Proof size is small but not trivial:** A Groth16 proof is 192 bytes (3 group elements), and verification is ~3 ms. STARKs avoid the trusted setup and are post-quantum safe but produce ~100 KB proofs. The SNARK/STARK tradeoff is essentially trusted-setup + small proof vs. transparent + larger proof.

## Why it works

The deeper principle is **Schwartz-Zippel as a universal proof primitive.**

The lemma states: a non-zero polynomial of degree d over F_p has at most d roots. Therefore, two distinct degree-d polynomials can agree at at most d of the p field elements. Checking one random point correctly distinguishes them with probability 1 - d/p ≈ 1.

This same lemma is the mathematical core of:

- **Reed-Solomon codes**: a codeword is the evaluation of a degree-k polynomial at n > k points; any k values reconstruct the polynomial. Two distinct codewords differ in at least n-k positions (since their difference polynomial has at most k roots). Error correction = polynomial interpolation.

- **Rabin-Karp string matching**: the string is treated as a polynomial over Z_p evaluated at a random point; two strings match iff they have the same polynomial hash value (with probability d/p of collision).

- **Freivalds' algorithm**: matrix multiplication verification `AB = C` is checked as `(AB - C)r = 0` for a random vector r; each wrong entry creates a non-zero polynomial that is detected with probability 1/2.

- **Interactive Proof systems (IP = PSPACE)**: the deep reason IP is so powerful is that "computation" can be arithmetized — expressed as polynomials — and then verified at a random point. The sum-check protocol is the core subroutine; ZK-SNARKs are non-interactive versions of the same idea.

The unifying abstraction: **polynomial arithmetic over a finite field is both universal (every computation can be arithmetized) and efficiently verifiable (Schwartz-Zippel makes one random check as good as n checks)**. The homomorphic hiding in a SNARK is the mechanism that makes the random point τ genuinely unknown to the prover, turning a probabilistic verification into a cryptographic one.

The profound connection: the same algebraic structure that corrects burst errors on a CD (Reed-Solomon: a codeword is a polynomial evaluation, and burst errors create a small-degree residual that the decoding polynomial identifies) **also proves execution of an arbitrary program without revealing its inputs** (ZK-SNARK: a valid witness makes the residual polynomial divisible by Z, and Schwartz-Zippel turns that divisibility into a one-query test). Error correction and zero-knowledge proofs are instances of the same theorem.

## Going deeper

1. **"Zk-SNARKs: Under the Hood"** by Vitalik Buterin — the third post in this series, which covers how bilinear pairings make τ cryptographically secret and how the Pinocchio protocol assembles all the pieces. Pairing-based cryptography is the one part of ZK-SNARKs that has no simpler analogy; this post is the clearest accessible treatment of it. URL: https://medium.com/@VitalikButerin/zkSNARKs-under-the-hood-b33151a013f6

2. **"Proofs, Arguments, and Zero-Knowledge"** by Justin Thaler (Georgetown) — a full free textbook at https://people.cs.georgetown.edu/jthaler/ProofsArgsAndZK.pdf. It builds from interactive proofs (IP = PSPACE) through the sum-check protocol and GKR circuit verification to practical SNARK constructions including Groth16, PLONK, and STARKs. Chapters 1–4 are standalone reading; they build the arithmetization intuition that QAPs implement in a specific way.

3. **"PLONK: Permutations over Lagrange-bases for Oecumenical Noninteractive arguments of Knowledge"** by Gabizon, Williamson, and Ciobotaru (eprint.iacr.org/2019/953) — the universal SNARK that most real zkVM systems (Aztec, Polygon zkEVM, Starknet) use as a building block. The key new idea: wiring constraints between gates are expressed as a "copy constraint" checked via a polynomial encoding of a permutation argument, eliminating the per-circuit trusted setup.
