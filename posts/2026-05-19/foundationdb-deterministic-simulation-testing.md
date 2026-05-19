---
title: "Simulation and Testing — FoundationDB's Deterministic Simulation Framework"
source: https://apple.github.io/foundationdb/testing.html
author: Will Wilson and the FoundationDB team
company: Apple (FoundationDB)
date_posted: 2021-06-20
date_digested: 2026-05-19
tags: [distributed-systems, testing, simulation, foundationdb]
---

# Simulation and Testing — FoundationDB's Deterministic Simulation Framework

## What's new to learn

1. **Deterministic simulation testing (DST)**: A technique for running an entire multi-node distributed cluster inside a single OS thread, with every source of non-determinism (network, disk, time, thread scheduling, randomness) replaced by a seeded PRNG-driven simulator. Same seed → same execution → every bug is perfectly reproducible.

2. **BUGGIFY**: A macro embedded directly in production code (not test scaffolding) that, during simulation, fires dangerous internal code paths with a low probability. It's a chaos-engineering tool that lives in the algorithm, not in the test harness, letting the simulator explore rare internal behaviors no external fault injector could reach.

3. **Test oracle via invariant rings**: Rather than asserting specific outputs, FDB's simulation checks a continuously-maintained data invariant (key-value pairs arranged in a ring whose integrity must survive any sequence of faults). The oracle is a correctness proof, not an expected-output check.

## Prerequisites

- **Cooperative multitasking / event loops**: Understanding why single-threaded, yield-based concurrency enables deterministic scheduling is essential. If you know Node.js's event loop or Python's `asyncio`, you have the mental model.
- **Seeded pseudo-random number generators**: Why a PRNG with a fixed seed produces the same sequence every time, and why that makes non-determinism "injectable."
- **Basic distributed systems failure modes**: Network partitions, crash-recovery, split-brain — the failure scenarios that DST injects.

## The core idea

The hardest bugs in distributed systems aren't logical errors — they're timing bugs: a network partition that lasts exactly 300 ms while a leader election is mid-flight, a disk flush that stalls for 10 seconds during a commit. These bugs can't be reproduced from a log. They can't be written as regression tests. They manifest once in ten billion requests and disappear before anyone can attach a debugger.

FoundationDB's answer was radical: **build the testing framework before the database.**

The team spent roughly two years building a deterministic simulation environment before writing a single line of database code. The insight: if you replace every source of non-determinism with a seeded PRNG-controlled substitute, you can replay any failure exactly by re-running with the same seed. A bug that takes ten years of production traffic to surface can be found in an afternoon of simulation.

To get there, you need to abstract away the physical world. The FoundationDB team built **Flow**, an actor-based extension to C++ that makes all concurrency cooperative (no OS threads, no OS scheduler) and routes every I/O call through pluggable interfaces. In simulation mode, those interfaces are backed by in-memory, deterministic fakes. In production mode, they call the real OS. The database code is identical in both modes.

## Mechanics

### Flow: The Actor Model That Enables Determinism

Flow extends C++11 with two constructs:
- `ACTOR`: marks a function as a cooperative coroutine
- `wait(future<T>)`: suspends the actor until a value is ready, yielding control to other actors

All concurrency in FDB is cooperative. There are no OS threads in the server process — one thread runs the Flow scheduler, which picks the next runnable actor from a queue. Because actors only switch at explicit `wait()` points, the scheduler decides exactly when each logical process runs.

This matters for simulation because: **no OS thread scheduler → no OS non-determinism**. The Flow scheduler itself becomes the source of all concurrency, and it can be made deterministic.

### The Simulation Environment

When FDB runs in simulation mode, the following substitutions happen:

| Real mode | Simulation mode |
|---|---|
| TCP socket send/recv | In-memory message queue with PRNG-drawn latencies |
| `pread`/`pwrite` to disk | In-memory key-value store with PRNG-drawn delays |
| `clock_gettime()` | Simulated clock advanced by the scheduler |
| `random()` / `rand()` | `deterministicRandom()->randomInt()` |
| N OS processes across machines | N logical "processes" as actors in ONE OS thread |

The entire cluster — clients, servers, coordinators — runs as a set of actors inside a single OS thread. Because everything is actors and every I/O goes through swappable interfaces, the simulation is **structurally identical to the production binary**, not a mock.

Time is compressed roughly **10:1**: the simulator advances simulated time 10 seconds for every real second, letting a nightly test run cover days of simulated cluster activity.

### `deterministicRandom()`: The Seed Binds Everything

Every random choice in FDB — network latency values, backoff delays, crash timings, even internal data-structure rebalancing — goes through `deterministicRandom()`, a seeded PRNG that replaces all calls to `random()` or `rand()`.

At test start, the framework picks a seed. That seed determines every subsequent random draw. The consequences:
- Same seed → identical execution, down to the order of individual actor wakeups
- A test failure is reported with its seed
- Any developer can reproduce the failure: `./fdbserver --seed=0xDEADBEEF`

When a bug manifests after a trillion simulated operations, the seed is the entire bug report.

### BUGGIFY: Chaos Injection in the Algorithm, Not the Harness

External fault injection (killing processes, dropping packets) can only test how the system reacts to *external* failures. It cannot reach internal code paths that are triggered by subtle combinations of timing — a flush that takes twice as long, a connection that is dropped and re-established in the middle of a transaction.

BUGGIFY solves this by embedding fault triggers in the production code itself:

```cpp
// In the production flush path:
BUGGIFY { wait(delay(10.0)); }  // Simulate an unusually slow flush

// In the commit path:
BUGGIFY { throw disk_full(); }  // Simulate disk-full during commit
```

The `BUGGIFY` macro evaluates to:
- **In simulation**: `if (deterministicRandom()->random01() < 0.01)` — fires ~1% of the time
- **Outside simulation**: `false` — compiled away to a no-op, zero runtime cost

This lets the developer annotate "this is an interesting edge case" in the algorithm, and the simulator then explores it at a controlled, reproducible rate. BUGGIFY is the bridge between external fault injection and internal heisenbug exploration.

### Fault Injection Modes

The simulation injects failures at three levels:

- **Network**: Connection failures, packet delivery delays, reordering. The simulator can drop every packet between two nodes, or introduce 500 ms of jitter across the cluster.
- **Machine**: Process crashes mid-transaction, reboots, performance degradation (a machine running at 10% speed).
- **Datacenter**: Entire datacenter going offline, forcing promotion of a replica DC.

A particularly effective pattern is **"swizzle-clogging"**:
1. Select a random subset of cluster nodes
2. Clog (stop) their network connections sequentially over several seconds
3. Unclog them in random order, one by one, until the cluster reforms

This pattern finds bugs in leader election and recovery that only manifest when nodes come back online in unusual orderings — scenarios almost impossible to reproduce with real hardware.

### Test Oracles

The simulation needs a specification of "correct." FDB uses **ring tests**: key-value pairs are arranged in a ring data structure, and transactions continually modify the ring while maintaining its integrity. The ring's invariant is checked after every fault injection. A single violation fails the test.

Additional oracles:
- **Durability**: after a simulated crash, all previously committed writes must survive
- **Read-your-writes**: a read after a committed write always observes that write

The ring oracle is clever because it's a *continuous* correctness check, not an expected-output check. It fails on any violation of a data invariant, not just the specific scenario the test author imagined.

### Scale

FDB's simulation runs tens of thousands of tests nightly. Over the product's lifetime, the team estimates they have run the equivalent of roughly **one trillion CPU-hours** of simulation — far more stress than any production environment delivers. Kyle Kingsbury (of Jepsen fame) has publicly declined to test FDB, noting that its own simulator already stress-tests it more thoroughly than Jepsen can.

## Where it breaks

**You must design for it from day one.** Retrofitting DST onto an existing codebase is extremely hard. It requires that all I/O be abstracted behind interfaces (effectively dependency-injecting all hardware). In FoundationDB's case, this required building an entirely new programming language (Flow) before any database code was written.

**Requires the actor/cooperative model.** Code with real OS threads cannot be simulated deterministically without heroic effort (Antithesis, a commercial platform built by FDB alumni, solves this via a deterministic hypervisor, but at the cost of running everything inside a custom VM). If you use `std::thread` or Go goroutines backed by the OS scheduler, you cannot deterministically control interleaving.

**The simulator is not the real world.** OS kernel bugs, NUMA topology effects, hardware-specific interrupt timing, memory bus contention — none of these appear in simulation. FDB's simulation finds algorithmic correctness bugs, not hardware-interaction bugs.

**Test oracle quality determines test quality.** The ring invariant is carefully chosen, but it's possible to have bugs that don't violate the ring. If your correctness specification is incomplete, bugs slip through.

**BUGGIFY requires discipline.** Developers must identify and annotate interesting code paths. This requires deep domain knowledge and can be forgotten under schedule pressure. Code paths without BUGGIFY annotations are invisible to the simulator's internal chaos.

**Performance testing diverges.** Simulated timing is compressed and artificial. A system that looks performant in simulation may have latency issues in production due to real network round-trips, NUMA effects, or memory bandwidth.

## Why it works

The deeper principle is **discrete-event simulation (DES) applied to real production code**.

DES has been used in networking research, queuing theory, and operations research for decades. In a classic DES, you define a set of event types (packet arrives, process crashes, timer fires), a simulated clock, and a priority queue of pending events. You advance time by processing the next event, which may schedule future events. The simulation is deterministic by construction: same initial event list → same execution.

FoundationDB recognized that **if your production code is already structured as an event-driven actor system, it IS a discrete-event simulation** — the actors are processes, `wait()` points are event boundaries, and the scheduler is the DES engine. The only thing left to do was make all I/O inject-able so you can substitute simulated events for real ones.

Two connected insights make DST uniquely powerful:

1. **Non-determinism = external state**. Bugs in distributed systems are caused by specific, rare combinations of timing from external sources (network, disk, clock). If you control all external state through a seeded PRNG, you control all timing, and therefore all bugs become reproducible.

2. **Run real code, not mocks**. Traditional unit testing mocks away complexity to isolate the unit under test. DST inverts this: it mocks only the physical hardware (I/O interfaces) while running the exact production algorithm. This means the simulator finds bugs in real consensus logic, real replication code, and real recovery paths — not just in mock interactions.

The BUGGIFY connection: **BUGGIFY is property-based testing (à la QuickCheck) applied to internal code paths instead of API boundaries.** Instead of generating random inputs to the public API, you generate random internal behaviors at developer-annotated "interesting" points. It's the difference between fuzzing the surface and fuzzing the depth.

The commercial follow-on: Antithesis (founded by Will Wilson, FDB's original simulation architect) generalizes DST to any codebase using a deterministic hypervisor. Instead of requiring actor-based code, the hypervisor controls thread scheduling and I/O at the OS level, making any software deterministically simulatable. DST has gone from a FoundationDB-specific technique to an emerging industry practice adopted by TigerBeetle, WarpStream, Convex, and others.

## Going deeper

1. **FoundationDB SIGMOD 2021 paper** — Section 4 covers simulation in technical depth, including the test oracle design and limitations. The paper demonstrates that simulation found bugs that Jepsen could not: https://www.foundationdb.org/files/fdb-paper.pdf

2. **Will Wilson, "Testing Distributed Systems w/ Deterministic Simulation" (Strange Loop 2014)** — The original public explanation of FDB's simulation philosophy by its inventor. Still the clearest exposition of the mental model: https://www.youtube.com/watch?v=4fFDFbi3toc

3. **TigerBeetle's VOPR simulator** — A modern DST implementation in Zig that shows how to apply the technique without the Flow programming language, with a public write-up that extends the ideas with coverage-guided simulation: https://tigerbeetle.com/blog/
