---
title: "ARC: A Self-Tuning, Low Overhead Replacement Cache"
source: https://static.usenix.org/event/fast03/tech/full_papers/megiddo/megiddo.pdf
author: Nimrod Megiddo and Dharmendra S. Modha
company: IBM Almaden Research Center
date_posted: 2003-03-31
date_digested: 2026-10-18
---

# ARC: A Self-Tuning, Low Overhead Replacement Cache

## What's new to learn

1. **Ghost lists turn eviction history into free feedback**: ARC maintains two "ghost" lists of recently evicted keys (no values, just identifiers). A cache miss that hits a ghost entry tells you *which* policy you should have been running — giving you a no-cost oracle for adaptive policy selection.
2. **ARC is online learning applied to cache eviction**: the adaptive parameter `p` that balances recency vs. frequency is updated by exactly the same "increase when you were wrong in one direction, decrease when wrong in the other" rule as a multiplicative-weights / no-regret learning algorithm — without anyone calling it that in the paper.
3. **Self-tuning from a single fixed-point rule**: ARC needs zero tuning parameters and converges on the optimal LRU/LFU balance for any workload because a missed page in ghost-B1 is evidence that "T1 was too small" and a missed page in ghost-B2 is evidence that "T2 was too small" — so the algorithm adjusts the boundary each time it gets evidence, and the boundary oscillates around whatever split the workload actually rewards.

## Prerequisites

- How a basic LRU cache works: a hash map for O(1) lookup + a doubly-linked list to maintain recency order; evict from the tail on overflow
- Why LRU fails for frequency-biased workloads: a sequential scan displaces frequently-accessed pages ("scan pollution"); a hot page evicted once is just as likely to be evicted next time under pure LRU
- The LFU idea: keep a frequency counter per page, evict the page with the lowest count — great for stable-frequency workloads, terrible when access patterns shift because stale high counts survive forever
- Intuition that neither LRU nor LFU dominates: LRU wins for recency-driven workloads (websearch result pages, user session data), LFU wins for frequency-driven workloads (OS page cache, database buffer pool on hot tables)

## The core idea

LRU and LFU are not competing algorithms — they are two ends of a policy spectrum. Any fixed point on that spectrum is wrong for at least some workload phase. What you actually want is a mechanism that *observes which end of the spectrum was penalizing you* and slides the boundary toward the better end.

ARC's key insight: the information you need is already present in your eviction history. When you evict a page, you know whether it had been accessed once (recency candidate) or more than once (frequency candidate). If that page shows up again *after* being evicted, you have direct evidence that your eviction was premature — and specifically which policy was responsible.

ARC captures this evidence by retaining evicted page *identifiers* (not values) in two ghost lists, at zero marginal memory cost for values. A ghost-list hit is a loss signal; ARC immediately adjusts its internal balance parameter `p` in the direction that would have prevented that specific miss.

The result: ARC automatically adapts to any workload, including ones that change over time, without the operator specifying anything.

## Mechanics

ARC maintains four lists inside a single logical cache of capacity `c`:

```
T1  — LRU list of pages seen exactly once since last eviction from B1 or B2
T2  — LRU list of pages seen at least twice (or promoted from T1)
B1  — Ghost list of keys most recently evicted from T1   (no values stored)
B2  — Ghost list of keys most recently evicted from T2   (no values stored)
```

And one scalar parameter `p` (the "target" for |T1|), which starts at 0 and is constrained to `[0, c]`.

### On a cache request for page `x`:

**Case 1 — Hit in T1 or T2**: move `x` to the MRU position of T2. Done.

**Case 2 — Miss, but `x` is in B1** (ghost hit on a recently-evicted recency page):
```
p ← min(p + max(1, |B2|/|B1|), c)   // B1 miss → T1 was too small, increase p
```
Fetch the value, insert `x` into MRU of T2, remove from B1. Evict if needed (see below).

**Case 3 — Miss, but `x` is in B2** (ghost hit on a recently-evicted frequency page):
```
p ← max(p - max(1, |B1|/|B2|), 0)   // B2 miss → T2 was too small, decrease p
```
Fetch the value, insert `x` into MRU of T2, remove from B2. Evict if needed (see below).

**Case 4 — Full miss** (not in any list):
Fetch the value, insert `x` into MRU of T1. Evict if needed.

### Eviction rule (called whenever |T1| + |T2| = c before an insert):

```
if |T1| > p:
    evict LRU of T1; append its key to MRU of B1
else:
    evict LRU of T2; append its key to MRU of B2
```

The ghost lists grow unboundedly in theory; in practice they are bounded at `c` (combined), and the oldest ghost entries are simply discarded when the lists overflow.

### What `p` controls

`p` is the *target* for how many of the `c` slots should hold T1 pages. When `p = 0`, ARC acts like pure LFU (T1 is always the eviction candidate). When `p = c`, it acts like pure LRU. Most workloads drive `p` to a steady state somewhere in the middle that neither a pure LRU nor pure LFU configuration could achieve.

### A worked trace

Suppose `c = 4` and we access pages: A B C D A B A B C D E

```
Step  Access  T1        T2    B1   B2   p  Action
 1      A     [A]       []    []   []   0  cold miss → T1
 2      B     [B A]     []    []   []   0  cold miss → T1
 3      C     [C B A]   []    []   []   0  cold miss → T1
 4      D     [D C B A] []    []   []   0  cold miss → T1 (full, |T1|=c)
 5      A     [D C B]   [A]   []   []   0  T1 hit → promote to T2
 6      B     [D C]     [B A] []   []   0  T1 hit → promote to T2
 7      A     [D C]     [A B] []   []   0  T2 hit → MRU of T2
 8      B     [D C]     [B A] []   []   0  T2 hit → MRU of T2
 9      C     [D C]     [B A] []   []   0  T2 hit → MRU of T2 (was LRU of T1)
        wait—C is in T1, so it gets promoted to T2:
        T1=[D], T2=[C B A]
10      D     [D]→[]    [D C B A] []  [] 0  T1 hit → promote: T1=[], T2=[D C B A]
11      E     cold miss; |T1|+|T2|=4=c; |T1|=0 ≤ p=0, evict LRU of T2 (A), A→B2
        T1=[E], T2=[D C B], B2=[A]  p stays 0
```

Now if `A` is requested again (step 12): it's in B2! `p` decrements (was 0, stays 0 since it's clamped), but the ghost hit tells ARC that frequency matters for this workload. If we kept seeing B2 ghost hits, `p` would eventually rise.

## Where it breaks

**1. Scan pollution is only partially mitigated.** A sequential scan of `s >> c` pages will still evict T1 down to `p` slots before ARC reacts, because the ghost lists need to fill before the feedback loop kicks in. The paper acknowledges this; Megiddo & Modha later published CART (2004) which adds scan-resistance via a CLOCK-like second chance on T1.

**2. Ghost list size is a tuning question the paper hand-waves.** Bounding the ghost lists at `c` is a reasonable default, but workloads with very long inter-reference intervals (hours-apart reuse of the same hot object) may need larger ghost lists to detect the pattern. The paper's experiments use unbounded ghost lists; real implementations quietly bound them.

**3. IBM patents blocked Linux adoption for nearly two decades.** US patents 6,996,676 and 7,028,107 covered ARC until ~2023. ZFS (OpenSolaris, then FreeBSD/illumos) implemented ARC because Sun/IBM had a patent cross-license. Linux used CLOCK-Pro (an approximation) in the kernel page cache. This is a systems-design lesson in its own right: an algorithm's patent status is an architectural constraint.

**4. ARC is not scan-resistant in the OLAP sense.** A database doing a full table scan that is larger than T2 + T2's ghost list will erode T2 without B2 growing fast enough to compensate — the feedback signal lags the damage. W-TinyLFU (used in Caffeine, Guava, Redis) solves this with a frequency sketch that never admits single-access items to the main cache at all.

**5. The `max(1, |B2|/|B1|)` update step size is empirical.** The paper shows it works well experimentally but gives no theoretical justification for the ratio. The original no-regret framing would use a fixed step size; the ratio is a heuristic acceleration that makes p converge faster in practice.

## Why it works

The mechanism is clearest when viewed as an online learning algorithm over two experts.

Define two experts:
- **Expert R** (recency): "always evict the LRU page" — equivalent to pure LRU
- **Expert F** (frequency): "always evict the least frequently accessed page" — equivalent to pure LFU

At each time step, ARC picks a convex combination: use Expert R with probability proportional to `p/c` and Expert F with probability proportional to `(c-p)/c`. Whenever Expert R incurs a loss (a page in B1 is re-requested — it *would not* have been evicted under a more frequency-aware policy), ARC increases the weight on Expert F. Whenever Expert F incurs a loss (B2 hit — a page would not have been evicted under a more recency-aware policy), ARC increases the weight on Expert R.

This is exactly the structure of the *Weighted Majority* algorithm from Littlestone & Warmuth (1994), which achieves no-regret relative to the better fixed expert, for any sequence of inputs. ARC achieves something similar in practice — it converges to the best fixed LRU/LFU split for any stationary workload — but the paper does not cite or invoke this literature.

The ghost lists are the crucial enabler: they make the loss observable at zero value-storage cost. Without them, you would not know whether a cold miss *should* have been a hit under a different policy.

**The abstraction that generalizes**: "maintain eviction history as keys-only ghost lists; use ghost hits as a gradient signal to adapt a policy parameter" is reusable. LIRS (Low Inter-reference Recency Set, 2002) uses a similar ghost structure. CLOCK-Pro approximates it using a clock hand. The pattern is: *instrumented eviction → counterfactual feedback → online policy update*.

## Going deeper

**CART (2004)** — Megiddo & Modha's own follow-up that adds "scan-resistance" by treating T1 as a CLOCK queue instead of a strict LRU, giving pages in T1 a second chance before eviction. This prevents a long sequential scan from immediately polluting B1 and driving `p` up.

**CLOCK-Pro** — Jiang, Chen & Zhang (2005). Approximates ARC using three clock hands instead of four lists, making it O(1) with tiny constant factors. This is what ended up in the Linux kernel as the replacement for plain CLOCK-LRU in some subsystems.

**W-TinyLFU** — Ben Manes (2015). The algorithm behind the Caffeine Java cache and Redis's LFU mode. Uses a Count-Min Sketch frequency approximation as an admission filter: a page is only admitted to the main cache (an ARC-like structure) if its frequency sketch count exceeds the eviction candidate's count. This eliminates the scan-pollution failure mode entirely because single-access pages never reach the main cache. W-TinyLFU consistently outperforms ARC on Zipfian and mixed workloads in benchmarks.

**The regret-minimization view** — Blum & Burch (2000), "On-line Learning and the Metrical Task System Problem", and Cesa-Bianchi & Lugosi (2006), *Prediction, Learning, and Games*. Chapter 2 covers the Weighted Majority algorithm. ARC's update rule is a variant with a variable step size; reading this literature gives you the tools to prove convergence bounds and understand what "no regret" means precisely.

**ZFS ARC** — the ZFS implementation differs from the paper: it uses an ARC sized to *available RAM* (not a fixed `c`) and adds an L2ARC (level-2 ARC on SSD) as a victim cache for B1/B2 ghost hits. The `arc_p` kernel tunable is exactly the paper's `p`. Reading the ZFS source in `module/zfs/arc.c` (illumos or OpenZFS) is the best study of a production ARC implementation.

**Connection to TCP congestion control**: AIMD (additive increase / multiplicative decrease) in TCP is the same feedback-loop structure — increase a parameter on success, decrease on loss, with the loss signal provided by dropped ACKs rather than ghost-list hits. Both are instances of "use observable outcomes as a gradient signal to track an unknown optimum in a non-stationary environment."
