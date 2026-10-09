---
title: "EEVDF Scheduler: The Design Behind Linux 6.6's CPU Scheduler"
source: https://kernel.org/doc/html/latest/scheduler/sched-eevdf.html
author: Peter Zijlstra (Intel); based on Stoica et al. (IEEE RTAS 1995)
company: Linux kernel / Intel
date_posted: 2023-10-01 (Linux 6.6 release)
date_digested: 2026-10-09
---

# EEVDF Scheduler: The Design Behind Linux 6.6's CPU Scheduler

## What's new to learn

1. **Eligibility and deadline are orthogonal fairness dimensions**: CFS (2007–2023) controlled fairness through a single number — virtual runtime. EEVDF splits this into two: *eligibility* (is this task allowed to run yet, based on how much it has received relative to its share?) and *virtual deadline* (among eligible tasks, who is most urgent?). Separating the two is what makes latency hints composable with fairness guarantees — something CFS could only achieve through ad hoc heuristics.

2. **EEVDF is EDF applied to virtual time**: Earliest Deadline First (EDF) is the classic real-time algorithm: always run the task whose deadline is nearest. EDF is optimal for meeting hard deadlines but not for fair sharing — a low-priority task could monopolize a CPU if given an imminent deadline. EEVDF's resolution: run EDF not in wall time but in *virtual time* (a clock that advances inversely with the number of runnable tasks). In virtual time, deadlines encode both urgency and weight, so EDF-in-virtual-time *is* fair sharing, and the deadline gives you an independent latency knob for free.

3. **Lag as a universal "credit account"**: Each task's lag = (virtual time owed) − (virtual time consumed). Positive lag means the task is owed CPU time; negative lag means it has overrun its share. Eligibility is just the sign of lag. This "account" pattern — accrue credit while idle, spend it while running, block when overdrawn — appears throughout systems (token buckets, leaky buckets, WFQ in network schedulers) and is the right mental model for any proportional-share resource system.

---

## Prerequisites

- **CFS virtual runtime**: Each task accumulates `vruntime` at a rate of 1/weight ns of CPU per ns of wall time. CFS always picks the task with the minimum vruntime. Heavier tasks advance more slowly, so they get picked more often.
- **Preemption and time slices**: The scheduler picks a task, runs it for a slice (the "targeted latency" divided by the number of runnable tasks), then re-evaluates.
- **Red-black tree**: CFS kept tasks ordered by vruntime in an O(log n) tree; picking the minimum is O(1) (leftmost node). EEVDF keeps a similar tree, now ordered by deadline.
- Helpful: weighted fair queueing (WFQ) from networking — EEVDF's virtual time is the same construction, developed in the networking community (Demers, Keshav, Shenker 1989) about six years before Stoica's CPU adaptation.

---

## The core idea

Here is the compact restatement: **EEVDF = EDF (Earliest Deadline First) applied to virtual time.**

EDF is well-understood: to minimize missed deadlines, always run the task whose deadline is closest. It's optimal for hard-real-time workloads. But plain EDF on a general-purpose scheduler breaks fairness — a task assigned an urgent deadline runs to the exclusion of others regardless of weight.

Virtual time fixes this. Instead of tracking wall-clock deadlines, EEVDF tracks *virtual* deadlines in a CPU-share-weighted clock. The virtual clock advances at a rate that accounts for how many tasks are sharing the CPU and with what weights. A task with weight 2× runs at half the rate in virtual time, so its deadlines come up twice as often — giving it twice the CPU without ever needing to override fairness.

The scheduler's algorithm collapses to two lines:

```
eligible_set = {task : task.vruntime ≤ rq.min_vruntime}
next = min(eligible_set, key=lambda t: t.deadline)
```

That's the whole picker. Two data structures, one gate.

---

## Mechanics

### Virtual time and min_vruntime

Every task has a `vruntime`: the total virtual CPU time it has consumed. The runqueue maintains `min_vruntime`: a monotonically increasing value tracking the smallest `vruntime` among all runnable tasks — the "present" of the virtual clock.

As a task runs, its `vruntime` increases at rate 1/weight (heavier tasks advance more slowly). `min_vruntime` advances as the currently-running task's vruntime pulls the minimum upward.

### Lag and eligibility

A task's **lag** = `min_vruntime − vruntime`.

- **Positive lag**: the task is behind the virtual clock — it has received *less* CPU than its weight entitles it to. It is **eligible**.
- **Negative lag**: the task is ahead — it has received *more* than its share. It is **ineligible** and must wait for `min_vruntime` to catch up before it can run again.
- **Zero lag**: exactly caught up; the boundary of eligibility.

The eligibility check in the kernel is approximately `se->vruntime ≤ rq->min_vruntime`.

### Virtual deadline assignment

When the scheduler picks task *i* to run with slice size *r*:

```
vdeadline = max(vruntime_i, min_vruntime) + r / weight_i
```

The `max(...)` floors the start to "now" in virtual time. A task with positive lag gets `vdeadline = min_vruntime + r/w` — no bonus time for having waited, just the standard slice allocation. A task with negative lag is ineligible, so this path never executes for it.

The deadline is stored in `se->deadline` and used as the sort key for the runqueue tree.

### Why the eligibility gate matters

Consider a task that slept for a long time. It wakes with a large positive lag — its vruntime is far below `min_vruntime`. Without the eligibility gate, it would wake with a very small deadline, immediately dominate the eligible set, and run continuously until it "caught up" its lag — starving every other task.

With the gate, waking with large positive lag is fine: the task is eligible, gets a normal-sized deadline, and runs its one slice. Then its vruntime advances, and the cycle continues. It doesn't get to gorge.

CFS handled this with a `sleeper_fairness` toggle: clamping the vruntime of a waking task to `min_vruntime − latency_target/2`. This was heuristic, configurable, and interacted badly with other tuning knobs. EEVDF's eligibility gate is structural — it falls out of the lag math automatically.

### Walk-through: three equal-weight tasks

Tasks A, B, C; weight=1 each; slice=4ms; start with all vruntimes at 0.

| Step | A.vruntime | B.vruntime | C.vruntime | min_vruntime | Eligible | Deadlines | Pick |
|------|-----------|-----------|-----------|-------------|----------|-----------|------|
| 0 | 0 | 0 | 0 | 0 | A,B,C | all=4 | A (tie→A) |
| 1 (A ran) | 4 | 0 | 0 | 0 | B,C | B=4, C=4 | B |
| 2 (B ran) | 4 | 4 | 0 | 0 | C only | C=4 | C |
| 3 (C ran) | 4 | 4 | 4 | 4 | A,B,C | all=8 | A (tie) |

Perfect round-robin emerges from the math. No explicit queue needed.

Now introduce a latency-sensitive task D with weight=1 but slice=1ms:

- D's deadline after each pick = `min_vruntime + 1/1 = min_vruntime + 1`
- A/B/C deadlines = `min_vruntime + 4`
- Among eligible tasks, D's deadline is always smallest → D is scheduled first each round

D gets 1/(1+4+4+4) ≈ 7.7% of CPU (roughly), but is scheduled first in every round of 13ms — it experiences 1ms latency even though it has the same weight as A, B, C. This is the `latency_nice` mechanism.

### Latency-nice: the new knob

CFS offered only `nice` (weight). EEVDF adds `latency_nice` (exposed via `sched_setattr(2)`), which sets the task's base slice size:

- `latency_nice < 0` → smaller slice → smaller `r/w` → lower deadline → scheduled sooner per eligible round
- `latency_nice > 0` → larger slice → higher deadline → scheduled later but preempted less

This decouples two independently meaningful questions: *How much CPU should this task get?* (weight / nice) and *How quickly should it get its next turn?* (slice / latency_nice). CFS conflated both into one number.

---

## Where it breaks

### Empty eligible set

In pathological cases (all tasks have negative lag simultaneously — possible during CPU idle recovery or after a burst), the eligible set can be empty. Linux's fallback: pick the task with the smallest deadline among *all* tasks, ignoring eligibility. This is a bounded deviation: at most one task runs "ahead of schedule" by one slice, then normal eligibility resumes.

### Deadline ties

Equal-weight tasks assigned the same slice get identical deadlines. EEVDF breaks ties by `vruntime` (smaller = picked first), which is equivalent to CFS behavior. For the common equal-weight case this is fine; adversarially crafted workloads could exploit it, but no real-world scheduling policy prevents this without additional state.

### Weight × latency-nice interaction

A high-weight task with default latency-nice has deadline `min_vruntime + r/w`. Large weight means small `r/w` → *lower* deadline. So high-weight tasks naturally win in the eligible set even before latency-nice is applied. A low-weight task requesting low latency via `latency_nice` competes against a high-weight task's intrinsically low deadline. This is "correct" behavior — the high-weight task deserves more CPU — but can surprise operators who expect latency-nice to be absolute.

### Not a real-time scheduler

EEVDF's deadlines are *virtual*, not wall-clock. `SCHED_FIFO` and `SCHED_RR` tasks bypass EEVDF entirely and always preempt it. Latency-nice makes wakeup latency *more predictable* under normal load; under CPU overload or against real-time tasks, it provides no guarantee.

---

## Why it works

The elegance is that eligibility and deadline decompose the problem cleanly:

| Question | Mechanism |
|----------|-----------|
| "Does this task deserve to run?" | Eligibility gate (lag ≥ 0) |
| "Among deserving tasks, who goes first?" | Minimum virtual deadline |

CFS compressed both questions into `vruntime`, then added heuristics to recover the lost dimension (sleeper fairness, min_vruntime clamping, interactive penalty/bonus). Every heuristic added a new knob that interacted with every other knob. EEVDF's two-field representation has no such entanglement.

The deeper principle: **every proportional-share resource system needs two orthogonal concepts: a credit gate (eligibility) and a priority within the eligible set**. When you encode both in one number you will eventually need heuristic patches to disentangle them.

The lag = `min_vruntime − vruntime` accounting is the same "token bucket" pattern used in network rate limiting, CPU bandwidth cgroups (`CONFIG_CFS_BANDWIDTH`), and I/O schedulers. The virtual-time clock is the same construction as Weighted Fair Queueing (WFQ) in network switches. EEVDF is a point in a broader design space that extends from network packet scheduling through CPU scheduling to storage I/O — all use the same virtual-time + eligible-set structure.

---

## Going deeper

- **Original paper**: Stoica, Abdel-Wahab, Jeffay, Baruah, Gehrke, Plaxton — "A Proportional Share Resource Allocation Algorithm for Real-Time, Time-Shared Systems" (IEEE RTAS 1995). The lag bound proof (service deviation ≤ one slice) is short and worth reading.
- **Peter Zijlstra's patch series**: `[PATCH 00/15] sched: EEVDF and latency-nice and/or slice-attr` on lore.kernel.org (2023). The commit messages are unusually detailed engineering prose — rare in kernel patches — and explain every design tradeoff.
- **CFS design document**: `Documentation/scheduler/sched-design-CFS.txt` in the kernel tree. Understanding what EEVDF replaced makes the improvements concrete; CFS's virtual runtime and red-black tree structure carry over almost intact.
- **Weighted Fair Queueing** (Demers, Keshav, Shenker 1989) and **Generalized Processor Sharing** (Parekh and Gallager 1993): the networking lineage. GPS is the ideal fluid model that WFQ and EEVDF both discretely approximate; its service bound proof maps directly to EEVDF's lag bound.
- **`sched_setattr(2)` man page**: the userspace interface for latency-nice and custom slice sizes; shows how EEVDF's knobs are exposed.
- **`/proc/sched_debug`**: on a Linux 6.6+ system, each task's `vruntime`, `deadline`, and `lag` are printed here — a concrete way to watch the algorithm live.
