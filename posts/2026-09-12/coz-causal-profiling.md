---
title: "Coz: Finding Code that Counts with Causal Profiling"
source: https://dl.acm.org/doi/10.1145/2815400.2815409
author: Charlie Curtsinger, Emery D. Berger
company: University of Massachusetts Amherst
date_posted: 2015-10-04
date_digested: 2026-09-12
---

# Coz: Finding Code that Counts with Causal Profiling

## What's new to learn

- **Virtual speedup**: Speeding up one thread's code by δ% is formally equivalent to slowing all other threads by δ% each time that code runs — a duality that lets you run controlled "what-if" experiments on a live program without implementing any optimization.
- **Progress points**: A user-defined annotation marking "one unit of work is complete," giving the profiler a concrete throughput or latency signal to measure against.
- **Causal profiling**: A methodology that converts profiling from correlation ("where does time go?") to causation ("where would optimization have impact?") by running randomized virtual speedup experiments at runtime.

## Prerequisites

- How sampling profilers work (hardware perf counters, statistical sampling of the PC register — gprof, perf, pprof all use this)
- Amdahl's Law: in a program where fraction *f* is parallelizable, max speedup is 1/(1−f)
- Basics of concurrent programming: threads, critical sections, contention
- Non-obvious: why Amdahl's Law misapplies in concurrent programs — what the "serialized fraction" actually measures

## The core idea

Conventional profilers answer: *where does CPU time go?* You look at the hottest functions and optimize them. This works fine for single-threaded programs. In concurrent programs, it routinely gives wrong answers.

The failure mode: thread A spends 1% of CPU time acquiring a lock. Thread B spends 40% of time in a compute-heavy function. The profiler screams "optimize compute_heavy." But if thread B is blocked most of the time waiting for thread A to release a lock — if the lock acquisition is on the *critical path* — optimizing `compute_heavy` does nothing for throughput. Every unit of work A produces is instantly consumed by B. The real bottleneck is how fast A can produce work, and that 1% CPU line is the one that controls it.

Coz makes a deceptively simple observation:

> **Speeding up component X by δ% produces the same effect on overall throughput as slowing everything else by δ% each time X runs.**

To see why: if X runs for time *t_X* per iteration and the rest of the program takes *t_rest*, the total cycle time is *t_X + t_rest*. Speeding X up by 20% makes it *0.8 · t_X + t_rest*. Alternatively, if you slow everything else by 0.25 · t_X (= 20% of the original *t_X*) each iteration, the total becomes *t_X + t_rest + 0.25 · t_X = 1.25 · t_X + t_rest*, which shrinks the ratio of X's contribution by exactly the same amount. Globally, the ratio of X-time to non-X-time shifts by the same factor in both cases.

Coz implements this mechanically: to run the experiment "what if function X were 20% faster?", it pauses all threads *other than the one executing X* for 20% of X's measured execution time, every time X runs. No code changes. No actual optimization. Just a pause injection that creates the same measurable effect as the optimization would have had.

## Mechanics

**The experiment loop**

1. Use hardware performance counters (SIGPROF-based, ~1000 Hz) to sample the program counter across all threads, tagging each sample to a source line via DWARF debug info.

2. Pick an "experiment": a random source line *L* and a random virtual speedup *s* ∈ {0%, 5%, 10%, …, 100%}. *s* = 0% is chosen with 50% probability (it serves as the baseline). The other 50% is divided evenly over the 20 non-zero levels.

3. Run the experiment for a fixed wall-clock window (~1 second). During this window, each time a sample lands on a PC in the region of *L*'s function, inject a `nanosleep(s × δ_T)` into every other running thread, where *δ_T* is the measured execution time since the last sample in that region. Excess sleep time (POSIX only guarantees *minimum* delay) is tracked and subtracted from future injected delays so they remain calibrated.

4. Count how many times progress points were hit during the window. Compute the throughput improvement: *Δ = (progress_rate_s − progress_rate_baseline) / progress_rate_baseline*.

5. After enough experiments per line, aggregate: for each source line, you get a curve mapping "virtual speedup s%" → "expected throughput improvement Δ%." That curve *is* the profile.

**Progress points**

Users instrument their code with two macros:
```c
COZ_PROGRESS          // throughput: each hit = one unit of work done
COZ_BEGIN("name")     // latency: start of one latency interval
COZ_END("name")       // latency: end of one latency interval
```

For a web server, a progress point at request completion is natural. For a batch job, it might be "one row written." The profiler measures how fast progress points accumulate over each experiment window. Without progress points Coz falls back to reporting CPU time — exactly what conventional profilers do — gaining nothing.

**Output format**

Coz produces a profile CSV of (source file, line, virtual speedup %, measured throughput delta %). Plotting speedup% on the x-axis against throughput improvement% on the y-axis gives a "causal profile" for each line. A steep positive slope at low speedup means: even a small optimization here has large impact. A flat line means: optimizing this line does nothing for throughput regardless of how much you optimize it. A *downward* slope means: this line is a pacemaker whose speed is currently *matched* to consumers — speeding it up unbalances the pipeline.

## Where it breaks

**Only reveals the current bottleneck.** Once you fix the bottleneck Coz identified and re-run, the next bottleneck will be somewhere else entirely. Coz surfaces one layer of the critical path at a time. Each round of optimization requires a new profile.

**Requires manual progress point instrumentation.** The profiler cannot know what "work is done" means without programmer annotation. If you pick the wrong progress point — one that doesn't represent the latency the user cares about — the profile guides you toward optimizations irrelevant to user experience.

**Sleep resolution adds noise.** `nanosleep` guarantees a *minimum* delay, not an exact one. The excess-tracking compensation helps, but short experiments or programs with very fine-grained work units accumulate noise. Statistical confidence requires long enough runs.

**Short-lived programs.** Statistical convergence needs many experiment windows. Programs running for less than ~10 seconds may not generate enough data to reach significance at low speedup percentages.

**User-space only.** Coz intercepts threads via `LD_PRELOAD` and signals. It cannot profile GPU kernels, system call bodies, or programs bottlenecked on blocking I/O — though I/O wait shows up indirectly (the progress rate during I/O-dominated windows is low regardless of what you virtually speed up).

**The pacemaker anomaly.** A downward-sloping profile for a line means that line is *already* the rate-limiter in a producer-consumer chain, and speeding it up would flood a downstream consumer. The correct fix is to look at why the consumer is slow, not to make the producer faster. The profile correctly identifies the situation but requires human interpretation to act on it.

## Why it works

The deeper principle is **causal inference via controlled experiments** — the same logic as a randomized controlled trial in medicine or an A/B test in product engineering, applied to a running program.

To measure the causal effect of "X is faster" on "throughput improves," you need to compare the program in two worlds: one where X has its current speed, one where X is faster. You can't run the same program twice in identical conditions. Coz achieves the same thing by exploiting the virtual speedup duality: you don't need X to actually run faster — you just need the *ratio* of time-in-X to time-not-in-X to equal what it would be if X were faster. Inflating time-not-in-X by the right factor achieves this exactly.

This reveals the actual error in naively applying Amdahl's Law to concurrent programs. Amdahl says: "the speedup from optimizing fraction *f* of runtime is bounded by 1/(1−f)." The implicit assumption is that *f* is measured in wall-clock time. But in a concurrent program, the right quantity is *f*'s fraction of the **critical path** duration, not of total CPU time across all threads. A function consuming 0.15% of aggregate CPU time can be 100% of the critical path if it runs on the thread that gates all other progress.

This is precisely the SQLite result from the paper. Coz identified three initialization functions that, in aggregate, consumed **0.15% of total CPU time** across the profiling run — completely invisible in any sampling profiler. Yet when Coz virtually sped them up by 100%, throughput improved by **25.6%**. The functions populated function pointer tables at startup; other threads blocked waiting for this table to be ready, making those functions the critical path out of initialization. The actual optimization — replacing function pointer indirection with compile-time constants — took one afternoon and confirmed the 25% gain. A Memcached contention bug found the same way yielded a 9.4% improvement. In both cases, the hot functions identified by conventional profilers were not on the critical path.

Connections to other CS ideas:
- **Critical path method (CPM/PERT)** in project management: in any DAG of tasks, the critical path — not the most time-consuming individual task — determines project duration. Coz is CPM for programs.
- **Shadow prices in linear programming**: the Lagrange multiplier on a constraint tells you how much the objective would improve if you relaxed that constraint by one unit. Coz's slope at zero speedup is exactly the shadow price of that source line's speed.
- **Influence functions in ML**: which training data points, if removed, would most change the model? The causal framing — "what would happen to the output if I perturbed this input?" — is identical.

## Going deeper

1. **Original SOSP 2015 paper** ([ACM DL](https://dl.acm.org/doi/10.1145/2815400.2815409)) — full evaluation with SQLite, Memcached, pbzip2, and HHVM; Section 4 has the clearest formalization of the virtual speedup duality.
2. **Coz on GitHub** ([plasma-umass/coz](https://github.com/plasma-umass/coz)) — working Linux implementation; read `profiler.cpp` for the pause injection logic and how excess sleep is compensated.
3. **"Performance Matters"** by Emery Berger, Strange Loop 2019 (on YouTube) — 40-minute talk that walks through Coz with live demos and explains why conventional profilers mislead; accessible to engineers who haven't read the paper.
