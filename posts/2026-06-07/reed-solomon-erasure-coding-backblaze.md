---
title: "Erasure Coding: Backblaze Open Sources Reed-Solomon Code"
source: https://www.backblaze.com/blog/reed-solomon/
author: Brian Beach
company: Backblaze
date_posted: 2015-07-01
date_digested: 2026-06-07
---

# Erasure Coding: Backblaze Open Sources Reed-Solomon Code

## What's new to learn

- **Erasure coding** — A family of redundancy schemes where k data shards are expanded to k+m shards such that *any k of them* reconstruct the original data; it decouples "failures tolerated" from "storage overhead" in a way full replication cannot.
- **Reed-Solomon codes** — The dominant erasure code: k data words define a unique polynomial of degree k-1 over a finite field; m parity shards are extra polynomial evaluations at distinct points; any k of the k+m evaluations recover the polynomial exactly.
- **Galois Field arithmetic GF(2^8)** — A 256-element field where addition is XOR and multiplication is a table-driven exponentiation; this makes RS arithmetic byte-aligned, exact (no floating-point error), and fast enough to run in software at multi-GB/s on commodity CPUs.

## Prerequisites

- Linear algebra basics: matrix multiplication and what "matrix inverse" means (need not know the algorithm, just the concept)
- Polynomial fundamentals: what coefficients and evaluation points are
- Why floating-point arithmetic is approximate (motivation for finite fields)
- Familiarity with RAID and the idea of parity disks

## The core idea

The standard approach to storage durability is replication: store 3 copies, survive any 2 failures, pay 200% storage overhead. Erasure coding breaks the tight coupling between the number of failures you can survive and the overhead you pay.

Backblaze stores user data in "Vaults" using a 17+3 scheme: every file is split into 17 equal-sized *data shards* and 3 *parity shards* are computed from them, for 20 total. Any 17 of those 20 shards reconstruct the file. This scheme survives any 3 simultaneous drive failures — but the storage overhead is only 3/17 ≈ 18%, versus 200% for three-way replication.

The mechanism is polynomial interpolation in disguise:

1. **Treat your 17 data shards as the coefficients** of a polynomial P(x) of degree 16 over GF(2^8).
2. **Evaluate P at 20 distinct points** to produce 20 shards (the first 17 evaluations are the data itself, the next 3 are parity).
3. **Recovery**: given any 17 of those evaluations, Lagrange interpolation uniquely recovers P — and therefore all 17 original data shards.

In practice this is written as a matrix multiplication: a 20×17 *generator matrix* G (a Cauchy matrix over GF(2^8)) times the 17-row data matrix D gives the codeword. The first 17 rows of G form the identity, so data shards pass through unchanged. The last 3 rows produce the parity. Decoding is: extract the 17 rows of G corresponding to whichever shards survived, invert the resulting 17×17 submatrix, and multiply by the surviving shard data.

## Mechanics

### Galois Field GF(2^8)

A Galois field of size 2^8 has 256 elements — exactly the 8-bit byte range 0–255.

- **Addition**: XOR. `a ⊕ b`. Exact, branchless, and its own inverse (`a ⊕ a = 0`).
- **Multiplication**: Uses a *generator* g such that every nonzero element equals some power of g. To multiply a × b: look up `log(a)` and `log(b)`, add the exponents mod 255, then look up the antilog. Two table reads and an addition — no floating-point, no overflow.
- **Division**: `a / b = g^(log(a) − log(b) mod 255)`. Same two-table trick.
- **Primitive polynomial**: x⁸ + x⁴ + x³ + x² + 1 is commonly used to define the field, setting the wraparound rule for the generator.

The field has a crucial property: *every square submatrix of a Cauchy matrix over any field is invertible*. This is why Cauchy matrices are used as generator matrices — it guarantees that any k-of-n shards are always sufficient, no matter which k survive.

### Encoding (step by step)

```
1. Split the file into 17 equal-sized byte arrays: D[0] … D[16].
   (Pad the last chunk with zeros if the file size is not divisible by 17.)

2. Build the 20×17 Cauchy generator matrix G over GF(2^8):
   - Rows 0–16: identity matrix (top block).
   - Rows 17–19: Cauchy rows — G[i][j] = 1 / (x_i ⊕ y_j)
     where x_i, y_j are distinct nonzero field elements.

3. Compute the 20-row codeword matrix:
   for each byte position b:
     codeword[row][b] = ⊕_{col=0}^{16}  G[row][col] * D[col][b]   (GF multiply/add)

4. Rows 0–16 of codeword are the 17 data shards (identical to D).
   Rows 17–19 are the 3 parity shards.
```

The inner loop — one row of G times all 17 data columns for each byte position — is where the CPU time goes. Because GF(2^8) multiplication is a table lookup followed by XOR, it is trivially SIMD-vectorizable (AVX2 can do 32 GF multiplies in one instruction using shuffle-based table lookup).

### Decoding (k shards survive)

```
1. Let S = {s0, s1, …, s16} be the indices of the 17 surviving shards (in any order).

2. Extract rows S from G → call this the 17×17 decode matrix M.

3. Invert M over GF(2^8) (Gaussian elimination; swap rows, scale by GF inverse,
   subtract multiples — all operations in GF(2^8)).

4. Recovered data: for each byte position b,
   D_recovered[row][b] = ⊕_{col=0}^{16}  M_inv[row][col] * surviving[col][b]

5. (Optional) Re-encode the lost parity shards.
```

The inversion takes O(k³) GF operations, but k is fixed (17 for Backblaze) so this is a constant-work one-time setup per recovery event.

### Backblaze's library structure

The open-sourced Java library (`JavaReedSolomon`) has three classes:

| Class | Role |
|-------|------|
| `Galois` | Static log/antilog tables for GF(2^8); `multiply(a,b)` is two lookups + index addition |
| `Matrix` | Gaussian elimination over GF(2^8); `invert()`, `multiply()` |
| `ReedSolomon` | Encode/decode using a Cauchy generator matrix; the main API |

Calling `encode(dataShards, parityShards, shardSize)` fills in the parity byte arrays in-place. Calling `decodeMissing(shards, shardPresent, shardSize)` fills in any missing shards (data or parity) from the k surviving ones.

### Backblaze Vault numbers

| Metric | Value |
|--------|-------|
| Data shards (k) | 17 |
| Parity shards (m) | 3 |
| Total shards (n = k+m) | 20 |
| Failures tolerated | 3 drives |
| Storage overhead | 17.6% |
| 3-way replication overhead | 200% |

Each shard lives on a separate hard drive in the Vault, spread across different enclosures and data centres. A read goes to all 17 data drives in parallel; a recovery reads any available 17 of the 20.

## Where it breaks

**Repair bandwidth is expensive.**
Fixing one lost shard requires downloading all k surviving data shards to re-encode. With 3-way replication, repairing 1 lost replica costs 1× the object size downloaded (copy from one survivor). With 17+3 RS, it costs 17× the shard size ≈ the full object size — roughly the same total bytes, but read from 17 different drives, so repair is latency-bound and coordination-heavy. *Locally Repairable Codes (LRCs)* (used by Azure, Facebook) add extra "local" parities so you only need to read a handful of shards to repair one failure, at the cost of a slightly larger n.

**Reads are fan-out or latency-sensitive.**
A read requires k = 17 parallel shard reads. If one drive is slow, you wait for it (or incur extra redundant reads). Three-way replication can answer from the fastest of 3 replicas. EC is best suited for *warm/cold* data where latency is acceptable.

**Small objects are wasteful.**
Splitting a 1 KB file into 17 shards makes no sense. EC shines for large, sequentially-written blobs. Hot-path object stores (like S3 Standard) use replication for small or hot objects and switch to EC for large cold objects.

**Partial writes need care.**
Updating a byte in one data shard requires recomputing all 3 parity shards (they encode the full k data shards). In practice, EC systems are written in whole-object or whole-stripe units, never in-place.

**Silent corruption requires checksums.**
EC handles *erasure* (known-absent shards) perfectly; it handles *error* (corrupted-but-present shard) only if you can identify which shard is bad. Production systems pair EC with per-shard checksums so that a bit-rot shard is detected, treated as an erasure, and repaired.

**GF(2^8) limits shard count.**
GF(2^8) has 256 elements, so you can't have more than 255 distinct nonzero evaluation points. Systems needing very high n (hundreds of shards) need larger fields: GF(2^16) has 65536 elements, but multiplications cost more CPU time.

## Why it works

The deep principle is the **fundamental theorem of polynomial algebra**: a polynomial of degree k-1 is uniquely determined by *any* k of its values. This is Lagrange's interpolation formula (1795) — which predates computers by two centuries.

Reed-Solomon (1960) observed that if you evaluate a degree-(k-1) polynomial at n = k+m distinct points, you get n "shares" that satisfy this uniqueness property with m to spare. Any k shares reconstruct the polynomial and hence the original k data words. The "coding" part is just evaluating at more points than you need.

Why GF(2^8) instead of real numbers? A field must have exact arithmetic and a multiplicative inverse for every nonzero element. Real numbers have floating-point rounding; rational numbers have unbounded precision requirements. Finite fields (Galois fields, or GF(q)) are the only algebraic structure that is both *finite* and *a field*. GF(2^8) gives you all the algebraic guarantees of interpolation with arithmetic that fits in a byte and adds as XOR.

**The X-is-Y chain:**

- **Reed-Solomon = Shamir's Secret Sharing** for storage. Adi Shamir (1979) proposed splitting a secret into n shares using a random polynomial of degree k-1 over a prime field, so that any k shareholders can reconstruct it. Backblaze's 17+3 scheme is exactly this, with the "secret" being the file contents and GF(2^8) replacing a prime field. The techniques are mathematically identical; only the use case differs.
- **RAID-6 = Reed-Solomon with m=2**. RAID-6's P and Q parities are the two Reed-Solomon parity evaluations. Standard RAID-6 is a fixed-n, m=2 special case of the general scheme.
- **EC = information-theoretic optimality**. The Singleton bound proves that an (n, k) code can tolerate at most n-k erasures. Reed-Solomon codes *meet* this bound exactly — they are *Maximum Distance Separable (MDS)* codes. You cannot do better with any encoding scheme.
- **The Cauchy matrix = guaranteed invertibility**. Any k×k submatrix of a Cauchy matrix is invertible, which is what makes Cauchy generator matrices always decodable from any k-of-n shards. This is a theorem from linear algebra over fields, not a special property of RS codes — it's why Cauchy matrices were chosen over the simpler Vandermonde matrices (which can lose invertibility for certain subsets).

The same polynomial-over-finite-field primitive appears in: TLS's AES-GCM authentication tag (GF(2^128) arithmetic), QR codes (BCH/RS error correction), Blu-ray disc error correction, deep-space telemetry, and streaming video forward error correction. The mathematical insight from 1960 runs on your phone, your disk array, and interplanetary spacecraft.

## Going deeper

1. **Klaus Post's high-performance Go Reed-Solomon library** (https://github.com/klauspost/reedsolomon) — A production library showing how to accelerate GF(2^8) matrix multiply with SIMD/AVX2 shuffle tables; includes benchmarks showing multi-GB/s encode/decode on a single core. The README explains the algorithm alongside the optimizations.

2. **"Erasure Coding for Distributed Systems"** (transactional.blog, 2024, https://transactional.blog/blog/2024-erasure-coding) — A deep dive into implementation choices for production distributed systems: MDS vs near-MDS codes, the repair bandwidth problem, LRCs, and where specific designs like Azure's LRC and Facebook's XOR-based codes trade off differently.

3. **"A Survey of the Past, Present, and Future of Erasure Coding for Storage Systems"** (ACM Transactions on Storage, 2025, https://dl.acm.org/doi/10.1145/3708994) — A comprehensive survey covering minimum storage regenerating (MSR) codes that provably minimize repair bandwidth, locally repairable codes used in production at Azure, Facebook, and Google, and open research problems.
