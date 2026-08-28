---
title: "Practical Byzantine Fault Tolerance"
source: https://www.usenix.org/conference/osdi-99/practical-byzantine-fault-tolerance
author: Miguel Castro, Barbara Liskov
company: MIT
date_posted: 1999-02-01
date_digested: 2026-08-28
---

# Practical Byzantine Fault Tolerance

## What's new to learn

1. **The 3f+1 bound** — Tolerating f Byzantine failures requires n ≥ 3f+1 replicas (vs. 2f+1 for crash faults), because a Byzantine node can simultaneously behave as a crashed node for some replicas and as an honest node for others; you must account for both failure modes at once.

2. **The three-phase commit** — PBFT's pre-prepare/prepare/commit protocol adds one phase beyond 2PC: the prepare phase detects a Byzantine primary that sends different proposals to different replicas, and the commit phase provides the view-change safety guarantee that crash-fault tolerant systems get for free from their unique, honest leader.

3. **Safety vs. liveness decomposition** — PBFT guarantees safety (no two honest replicas execute different operations at the same sequence number) unconditionally even in a fully asynchronous network, and grants liveness only under partial synchrony — a clean separation that predates but foreshadows the FLP-aware design of Raft.

## Prerequisites

- State machine replication: applying the same deterministic operations in the same order to n copies of a state machine produces n identical states.
- Crash fault tolerance and Raft/Paxos: the 2f+1 quorum model and 2-phase commit (know these first — PBFT is most legible as "Raft extended for Byzantine faults").
- FLP impossibility: why any deterministic consensus algorithm in a purely asynchronous network must sacrifice either safety or liveness; PBFT picks safety.
- Digital signatures / message authentication codes: PBFT authenticates nearly every message; understand that a valid signature proves the signer sent that exact message, while a MAC proves it to one verifier.

## The core idea

Byzantine fault tolerance asks: what if a replica doesn't just crash, but actively lies — sending different, plausible-looking messages to different peers?

PBFT's answer is a state-machine replication protocol that:

1. Runs with n = 3f+1 replicas and a rotating primary, tolerating up to f Byzantine replicas among them.
2. Runs three all-to-all communication phases per request so that each phase provides a distinct epistemic guarantee the previous phase cannot.
3. Rotates to a new primary when replicas time out waiting, with cryptographic proof preventing the new primary from silently dropping values that might have been committed in the previous view.

The key intuition: a Byzantine node can dress up as a crashed node (ignore your messages) *and* as an honest node (send you a reply that looks correct but disagrees with what it told others) simultaneously. That dual nature is why you need 3f+1 instead of 2f+1, and why you need one more communication round than crash fault tolerance needs.

## Mechanics

### Why 3f+1 replicas

In a crash-fault-tolerant system with n = 2f+1 replicas:

- You can't wait for all n responses (f might be crashed), so you wait for a quorum of n - f = f+1.
- Any two quorums of f+1 must overlap by at least 1 node. That node answered, so it isn't crashed — it's honest. Correctness follows.

In a Byzantine system:

- You still can't wait for all n responses — up to f might not respond (crash-mimicking).
- Among the n-f responses you do receive, up to f could be liars (honest-mimicking).
- You need any two quorums to overlap in at least **f+1** nodes, so that even if f of them are Byzantine liars, at least 1 in the overlap is guaranteed honest.
- Minimum overlap of two quorums of size q from n total is 2q - n. Setting 2q - n ≥ f+1 and q = n - f gives:
  ```
  2(n - f) - n ≥ f+1
  n - 2f ≥ f+1
  n ≥ 3f+1
  ```

That's it. The 3f+1 bound is the smallest n where any two "I got enough responses" sets are guaranteed to share an honest witness.

Compare: Reed-Solomon erasure coding needs k+m shards to recover k data shards from m erasures. Reed-Solomon with Byzantine corruption needs k+2m shards — the same factor-of-2 overhead, for the same reason: you must account for nodes that both lie *and* appear absent.

### The normal-case protocol

Replicas maintain a sequence number counter and a view number. The primary for view v is replica v mod n.

**Phase 0 — Request**: The client sends ⟨REQUEST, o, t, c⟩_c (operation, timestamp, client id) to the primary. The timestamp t is used for exactly-once semantics.

**Phase 1 — Pre-prepare**: The primary assigns sequence number n to the request and broadcasts:
```
⟨PRE-PREPARE, v, n, d⟩_p  plus  message m  where d = digest(m)
```
A replica accepts this if: it's in view v, hasn't seen a different digest for (v, n), and sequence number n is within the water-mark window. The pre-prepare commits the primary to *one* assignment of sequence number n.

**Phase 2 — Prepare**: Every replica that accepted the pre-prepare broadcasts:
```
⟨PREPARE, v, n, d, i⟩_i
```
A replica is **prepared** for (v, n, d) when it has: 1 pre-prepare plus 2f matching prepares (from different replicas). A prepared certificate is the union of those 2f+1 messages.

*Why this phase exists*: A Byzantine primary could send PRE-PREPARE with digest d₁ to half the replicas and digest d₂ to the other half. The prepare phase forces replicas to broadcast what they received. No replica can collect 2f prepare messages for (v, n, d₁) *and* 2f prepare messages for (v, n, d₂) simultaneously — there aren't enough honest replicas. So after the prepare phase, at most one digest can be "prepared", and that fact is known to anyone who collected 2f+1 prepares.

In a crash-fault system, the primary's uniqueness provides this guarantee for free. Byzantine systems don't have a trusted primary, so this phase is the protocol's substitute for leader authority.

**Phase 3 — Commit**: Every prepared replica broadcasts:
```
⟨COMMIT, v, n, D(m), i⟩_i
```
A replica **commits** when it has 2f+1 matching commit messages from different replicas, then executes operation o and replies to the client.

*Why this phase exists*: Suppose a replica receives 2f+1 prepare messages and is about to execute. A view change begins before it can tell anyone. The new primary needs to know whether any replica committed in the old view. If a replica committed, 2f+1 replicas sent commit messages, meaning 2f+1 replicas hold a prepared certificate. Any subsequent quorum of 2f+1 replicas must include at least 1 that holds that certificate (by the 3f+1 quorum intersection argument). So the new primary will learn about it.

Without the commit phase, a replica might execute locally with only 2f+1 prepare messages. After a view change the new primary's quorum could include f Byzantine replicas that lie about what they prepared — not enough information to reconstruct what executed.

### Checkpoints and watermarks

The protocol would accumulate unbounded log entries without garbage collection. PBFT uses a stable checkpoint mechanism: when a replica has executed through sequence number n, it broadcasts a CHECKPOINT message. After 2f+1 matching CHECKPOINT messages, that checkpoint is stable — all log entries below n can be discarded. The water-mark window [h, h + L] bounds how far ahead a primary can assign sequence numbers, preventing a Byzantine primary from jumping arbitrarily ahead.

### View change

When a replica timer expires waiting for the primary to process a request, it broadcasts:
```
⟨VIEW-CHANGE, v+1, n, C, P, i⟩_i
```
where n is the latest stable checkpoint sequence number, C is the 2f+1 CHECKPOINT messages proving that checkpoint, and P is the set of prepared certificates (the 2f+1 pre-prepare + prepare sets) for all sequence numbers higher than n.

The new primary for view v+1 waits for 2f+1 VIEW-CHANGE messages, then broadcasts:
```
⟨NEW-VIEW, v+1, V, O⟩
```
where V is those 2f+1 VIEW-CHANGE messages and O is the set of pre-prepares for every sequence number that appeared prepared in any message in V (plus null prepares for sequence numbers not mentioned).

The new primary must re-propose any value that *might* have committed in the previous view. Formally: if any VIEW-CHANGE message contains a prepared certificate for (v, n, d), the new primary must include ⟨PRE-PREPARE, v+1, n, d⟩ in O. This is safe to re-execute because of state machine determinism — executing the same operation twice produces the same state as executing it once, once replicas track which client requests they've already replied to.

### Message complexity

| Phase          | Messages per round |
|----------------|--------------------|
| Pre-prepare    | O(n)               |
| Prepare        | O(n²)              |
| Commit         | O(n²)              |
| View change    | O(n²)              |

The O(n²) complexity in prepare and commit comes from each of n-1 replicas broadcasting to all n-1 peers. For n = 4 (f=1): 12 messages per round. For n = 100 (f=33): ~10,000 messages per round. This is why PBFT is practical only for small clusters (n ≤ 20–30 in most deployments).

## Where it breaks

**Quadratic message complexity.** Each of the three broadcast phases costs O(n²) messages. At n=100 replicas, a single consensus round generates ~30,000 messages. Real deployments rarely exceed 20 nodes.

**Leader rotation is expensive.** A Byzantine primary can stall progress (trigger many timeouts), and each view change costs O(n²) messages plus the overhead of proving what was prepared — potentially replaying O(n) pre-prepares. Compare to Raft, where a new leader is elected and starts replicating in O(1) messages per heartbeat interval.

**Liveness requires partial synchrony.** If a network adversary can delay messages indefinitely (fully asynchronous network), a Byzantine primary can keep forcing view changes and prevent any request from committing — FLP applies here too. PBFT only guarantees liveness after GST, when message delay is bounded by Δ.

**Doesn't scale the authentication.** The original paper uses public-key signatures only for view-change messages (because they must be verifiable by third parties) and uses MACs elsewhere. MACs are 100× faster but require the authenticator to share secrets with every other replica — O(n²) key pairs total.

**No Byzantine client protection.** A Byzantine client can send different requests to different replicas. PBFT detects and ignores these, but the primary must coordinate rejection explicitly, adding latency.

## Why it works

### The epistemic ladder

PBFT's three phases correspond to three levels of distributed knowledge:

| After phase    | What a replica knows                                      |
|----------------|-----------------------------------------------------------|
| Pre-prepare    | "The primary claims sequence n maps to digest d"          |
| Prepared       | "2f+1 replicas received the primary's claim for (v,n,d)" |
| Committed      | "2f+1 replicas know that 2f+1 replicas are prepared"      |

Each level gives safety for a different scenario:
- Prepared → "I can execute safely now, assuming no view change"
- Committed → "I can execute, and any future view change will preserve this"

This is the same epistemic nesting as Lamport's two-general problem: you need one round of communication per layer of certainty. Crash-fault systems get the second layer for free because an honest leader's pre-prepare is already a reliable certificate (the leader can't fork its own log). Byzantine systems must build that certificate explicitly — hence the second round.

### Byzantine faults are crash faults squared

The table below summarizes why Byzantine costs twice the redundancy:

| Property          | Crash faults (CFT)  | Byzantine faults (BFT) |
|-------------------|---------------------|------------------------|
| Replicas needed   | 2f+1                | 3f+1                   |
| Quorum size       | f+1                 | 2f+1                   |
| Phases (normal)   | 2                   | 3                       |
| Honest replicas   | n - f = f+1         | n - f = 2f+1            |
| Erasure overhead  | k+m shards          | k+2m shards            |

The factor-of-2 overhead appears everywhere because a Byzantine node occupies *two* roles simultaneously: it counts toward the "can't wait for" quota (crash role) *and* toward the "might be lying among the ones who responded" quota (liar role). You need 3f+1 replicas so that when f don't respond and f respond with lies, there's still f+1 honest responders — a quorum.

### The connection to the archive

PBFT is to Raft what Byzantine fault tolerance is to crash fault tolerance:

- **Raft** needs 2f+1 nodes, 2 phases (AppendEntries + commit advancement), and a unique honest leader to provide the prepare-phase guarantee for free.
- **PBFT** needs 3f+1 nodes, 3 phases, and an explicit prepare round to substitute for what Raft's leader uniqueness provides.

The FLP impossibility result (already in this archive) explains why both systems rely on timeouts: no deterministic protocol can guarantee consensus in a fully asynchronous network, so both use partial synchrony. PBFT just extends the synchrony assumption to cover an additional failure mode.

Flexible Paxos (also in this archive) showed that CFT quorum sizes can be tuned: Q₁ × Q₂ ≥ n+1. The analogous result for BFT (Q₁ × Q₂ ≥ n+f+1) is explored in HotStuff and later BFT variants.

Chain Replication (in this archive) achieves linearizability *without* BFT by assuming all replicas are honest — a classic safety/trust tradeoff.

## Going deeper

1. **HotStuff: BFT Consensus with Linearity and Responsiveness** (Abraham et al., 2019) — https://arxiv.org/abs/1803.05069 — The direct successor: by adding threshold signature aggregation (n votes → 1 compact certificate) and chaining votes across rounds (each vote certifies the previous block), HotStuff achieves O(n) message complexity in all phases, including view change. Used as the basis of Facebook Diem's LibraBFT and studied for Ethereum PoS finality.

2. **The Bedrock of Byzantine Fault Tolerance: A Unified View** (NSDI 2024) — https://www.usenix.org/system/files/nsdi24-amiri.pdf — Shows that PBFT, HotStuff, Tendermint, Casper, and related protocols are all instances of a single abstract framework parameterized by quorum size and phase count. A great way to see the design space.

3. **Castro & Liskov's original paper (PDF)** — https://www.usenix.org/conference/osdi-99/practical-byzantine-fault-tolerance — The full paper includes the optimizations, the proof of safety and liveness, and the NFS file system demonstration. The proof of safety (Lemma 1) is short (~10 lines) and worth reading; it crystallizes why the quorum intersection argument works.
