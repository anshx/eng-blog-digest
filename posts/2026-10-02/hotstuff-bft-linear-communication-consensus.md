---
title: "HotStuff: BFT Consensus with Linearity and Responsiveness"
source: https://arxiv.org/abs/1803.05069
author: Maofan Yin, Dahlia Malkhi, Michael K. Reiter, Guy Golan Gueta, Ittai Abraham
company: Cornell / VMware Research / UNC Chapel Hill
date_posted: 2019-07-01
date_digested: 2026-10-02
---

# HotStuff: BFT Consensus with Linearity and Responsiveness

## What's new to learn

1. **Threshold signatures as a communication-complexity reducer.** A threshold (k-of-n) BLS signature lets a leader aggregate 2f+1 individual votes into a single constant-size Quorum Certificate (QC), cutting one broadcast from O(N²) all-to-all messages to O(N) fan-in + O(N) fan-out — without changing the fault-tolerance bound.

2. **Pipelined QC chaining.** Basic HotStuff needs three sequential protocol phases; Chained HotStuff pipelines them so each new proposal's "Phase 1" simultaneously serves as "Phase 2" for the previous proposal and "Phase 3" for the one before, amortising three round trips into one per committed block after warmup.

3. **Responsiveness vs. synchrony in view change.** A protocol is *responsive* if it advances at actual network speed once 2f+1 replicas are ready — rather than waiting for worst-case timeouts. HotStuff is the first partially-synchronous BFT protocol that achieves both linear message complexity *and* responsiveness together; PBFT sacrificed one for the other.

## Prerequisites

- **PBFT's three phases** — pre-prepare, prepare, commit — and why 2f+1 out of 3f+1 replicas forms a quorum (covered in the PBFT archive post).
- **Cryptographic signatures** (sign, verify) at an abstract level. BLS aggregate signatures are used but you only need to know: multiple partial signatures can be combined into one that verifies against all signers simultaneously.
- **Partial synchrony** — the assumption that after some Global Stabilisation Time (GST), messages are eventually delivered within a known bound δ. This is the same assumption Raft and PBFT use; pure asynchrony is impossible for consensus (FLP).

## The core idea

**What PBFT costs and why.** In PBFT's prepare phase every replica, having received the leader's proposal, broadcasts its vote to *every other* replica — N replicas × N destinations = O(N²) messages. The same happens in the commit phase. With 200 replicas that is ~80,000 messages per decision. View change (replacing a faulty leader) is even worse: O(N³).

**HotStuff's pivot: vote to the leader, not to everyone.** Each replica sends its signed vote *only to the current leader.* The leader waits for 2f+1 votes, combines them into a threshold signature — a single constant-size Quorum Certificate — and broadcasts the QC to all replicas. Now every phase costs O(N) messages instead of O(N²). The QC is both proof of a quorum and the vote itself, so no replica needs to relay anything.

**Why three phases instead of PBFT's two?** PBFT's two-phase commit is safe but its view-change is either O(N³) (classic PBFT) or loses responsiveness. HotStuff adds one phase — prepare → pre-commit → commit — to achieve both safety and O(N) view change. The intuition: the extra phase creates a "lock" on a value before commit, giving the new leader safe ground to build on without needing to read every replica's log.

**Chained pipelining.** In Basic HotStuff, committing one block takes three explicit phases (three leader–replica round trips). Chained HotStuff overlaps them: the QC a leader generates for block b+1's prepare phase *is* the pre-commit QC for block b *and* the commit QC for block b-1. A replica needs to observe three consecutive QCs on a chain of consecutive proposals before it treats the oldest as committed. After the first three-block warmup, one new block commits per round trip.

## Mechanics

### Node roles and data structures

Every replica holds:
- **`lockedQC`**: the highest prepare-QC it has voted on — it will not vote for any block that conflicts with this.
- **`highQC`**: the highest QC it has seen from any phase — used to seed new leader proposals.

A **Quorum Certificate (QC)** is a tuple `(block_hash, view_number, type, threshold_sig)` where `threshold_sig` is a BLS aggregate of 2f+1 partial signatures. Any 2f+1 partial signatures from distinct replicas can be combined; any 2f+1 replicas can verify it. QCs are constant-size regardless of N.

### Basic HotStuff protocol (single block b, view v)

```
Leader (view v):
  1. [Prepare]   Broadcast PROPOSE(b, highQC)
  2.             Collect 2f+1 PREPARE votes → form prepareQC(b)
  3. [Pre-Commit] Broadcast prepareQC(b)
  4.             Collect 2f+1 PRE-COMMIT votes → form precommitQC(b)
  5. [Commit]    Broadcast precommitQC(b)
  6.             Collect 2f+1 COMMIT votes → form commitQC(b)
  7.             Broadcast commitQC(b) → replicas apply b

Replica r:
  Prepare:    if b extends lockedQC.block AND b.view = v, vote PREPARE
              update highQC if prepareQC.view > highQC.view
  Pre-Commit: if prepareQC valid, vote PRE-COMMIT
              update lockedQC ← prepareQC
  Commit:     if precommitQC valid, vote COMMIT → apply b
```

Message count per phase: N votes (replica→leader) + 1 broadcast (leader→N replicas) = O(N).

### Chained HotStuff

Each proposal extends the previous chain and carries the QC from the last proposal:

```
Block b at height h:
  parent:    hash of block at h-1
  justify:   QC from the leader's most recent phase
```

A replica's commit rule is: when it sees three consecutive QCs in the chain — `b1 ← b2 ← b3` where each block's `justify` points to the previous block — it considers `b1` committed. The three-QC suffix simultaneously acts as prepare/pre-commit/commit for successive blocks.

### View change

If a replica does not hear from the leader within timeout T, it broadcasts a `VIEW-CHANGE(v+1, highQC)` to the next leader. The new leader waits for 2f+1 such messages, takes the `highQC` with the highest view number among them, and uses that as the seed for its first proposal. This is O(N) messages, not O(N³) like PBFT's view change.

### Threshold BLS signatures (the aggregation primitive)

BLS signatures on an elliptic curve (e.g., BLS12-381) support linear combination: if f: G → H is the pairing and you have partial signatures s_i = sk_i · H(m), you can compute the aggregate Σ s_i in O(N) field operations. The threshold (2f+1-of-3f+1) variant uses Shamir secret sharing: the global secret key is split into shares such that any 2f+1 shares reconstruct it (Lagrange interpolation over a prime field). In practice, only the aggregate signature is transmitted — never the key itself.

## Where it breaks

**One more network round trip per block.** Basic HotStuff takes three phases vs. PBFT's two. In the common case (no faults), PBFT commits faster on a per-block basis. Chained HotStuff recovers pipelining throughput but latency per individual block still needs three QC-forming rounds.

**Leader is a single point of bottleneck and trust.** Every vote flows through the leader. A Byzantine leader cannot break safety (it cannot forge threshold signatures it doesn't have), but it can *stall* the protocol by refusing to broadcast QCs, forcing view changes. View change adds latency.

**BLS signature setup requires a trusted dealer or DKG.** The threshold key shares must be generated by a Distributed Key Generation (DKG) ceremony or trusted setup. This is non-trivial and introduces bootstrapping complexity that PBFT (plain ECDSA) avoids.

**Does not tolerate adaptive adversaries well.** HotStuff (like PBFT) assumes the adversary corrupts a fixed set of at most f replicas. An adaptive adversary that can corrupt replicas after seeing votes can compromise threshold signatures by collecting enough partial signatures.

**Partial synchrony assumption remains.** Progress requires that eventually the network behaves synchronously (GST). Under a sustained network partition, HotStuff stalls exactly like Raft and PBFT.

**3f+1 is still the minimum.** HotStuff does not reduce the replica count requirement. It is still Byzantine: you need n ≥ 3f+1 for safety and liveness. Tolerating f = 1 fault needs at least 4 replicas; f = 10 needs 31.

## Why it works

**All-to-all broadcasting is the O(N²) culprit, not the quorum size.** The fundamental requirement for BFT safety is that any two quorums of size 2f+1 intersect in at least one honest node. That intersection bound is what forces n ≥ 3f+1 — it does not force all-to-all message exchange. PBFT's design choice of having replicas verify each other's votes directly is what causes O(N²); HotStuff separates the *aggregation step* (the leader) from the *verification step* (the threshold signature) to achieve the same quorum intersection proof in O(N) messages.

This is the same pattern as:

| System | Problem | O(N²) naive approach | HotStuff-style O(N) |
|---|---|---|---|
| MapReduce | Aggregate results | Every worker sends to every worker | Workers → driver who returns to all |
| Ring AllReduce | Sum gradients | All-pairs exchange | Pipeline through ring segments |
| Merkle tree | Prove N items are consistent | Check all N×N pairs | Hash up a tree, broadcast root |
| HotStuff | Certify N votes | Replicas send to all replicas | Replicas → leader → threshold sig → broadcast |

The threshold signature is the *algebraically correct* representation of "2f+1 parties agreed": it is a mathematical certificate whose verification cost is O(1) regardless of N. Everything else in the system is about *routing* that certificate through the right nodes.

**Chaining is pipelining.** The three-QC commit rule in Chained HotStuff is exactly the same insight as a CPU instruction pipeline: divide the operation into stages (prepare, pre-commit, commit) and overlap them across consecutive operations. The observation that "QC for block b+1's prepare can serve as block b's pre-commit" works because the safety property of each stage is *monotonic*: once a replica has locked on a QC, it will never vote for a conflicting value, regardless of when the next stage completes.

**Responsiveness from the structure, not from timing.** A responsive protocol advances as soon as 2f+1 messages arrive, not after a timer expires. HotStuff achieves this because the leader drives all three phases sequentially without waiting for timeouts between them. PBFT's view change uses timeouts that must dominate worst-case network delay, making it non-responsive. HotStuff's view change only needs a `highQC` (already available from the prepare phase), so the new leader can start immediately after seeing 2f+1 view-change messages.

## Going deeper

1. **DiemBFT v4 / Jolteon (2021)** — `arxiv.org/abs/2106.10362` — The production refinement of HotStuff deployed in the Diem blockchain. Introduces the two-chain commit rule (down from three) using an optimistic linear-time fast path that reverts to the three-phase path under faults. A clean engineering-facing description of threshold signature ceremonies and leader rotation.

2. **Fast-HotStuff / PBFT revisited (Tendermint)** — `arxiv.org/abs/2209.00955` — Shows that Tendermint (the Cosmos/Ethereum BFT) and HotStuff are instances of a single abstraction parameterized by the lock/unlock predicate. Understanding both together gives a full map of the "single-leader, partially-synchronous BFT" design space.

3. **SBFT: a Scalable and Decentralized Trust Infrastructure for Blockchains** — `research.vmware.com` — The VMware Research group's follow-up that adds optimistic fast-path commit (2 phases if no faulty replicas) and shows how threshold signatures scale to 10,000 replicas, directly pointing to the BLS12-381 library integration used by Ethereum's Beacon Chain.
