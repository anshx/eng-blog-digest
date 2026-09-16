---
title: "The Zen of Erlang"
source: https://ferd.ca/the-zen-of-erlang.html
author: Fred Hebert
company: N/A (independent; author of "Learn You Some Erlang for Great Good!")
date_posted: 2016-02
date_digested: 2026-09-16
---

# The Zen of Erlang

## What's new to learn

- **"Let it crash"**: A fault-tolerance philosophy that inverts defensive programming — instead of handling every possible error at every call site, you allow processes to crash cleanly and rely on a supervisor to restart them to a known-good initial state. This is only safe because Erlang processes share no heap memory, so a crashed process cannot corrupt its neighbors.
- **Supervision trees**: The organizational pattern that makes "let it crash" work at scale. Every running process is either a *Worker* (leaf node; does actual work; may crash) or a *Supervisor* (internal node; only starts, monitors, and restarts children; should never crash). The tree creates nested fault-isolation boundaries with tunable blast radius.
- **Hierarchy as fault isolation**: The structural principle that volatile, frequently-changed code belongs at the leaves and stable, critical code belongs near the root — so a crash always stays contained within its subtree unless its supervisor's restart budget is exhausted.

## Prerequisites

- Basic concurrent programming: processes, threads, message passing
- The difference between a *crash* (unhandled failure, process dies) and an *error* (handled at the call site, execution continues) — in Erlang these are treated as structurally different categories
- Comfort with pattern matching; Erlang code examples use it heavily
- What a process mailbox is (each Erlang process has one; messages arrive there asynchronously)

## The core idea

Traditional defensive programming says: *anticipate every failure mode and handle it at the call site*. The Erlang community's observation, developed through decades of telecom systems at Ericsson, is that this strategy fails for two reasons: (1) you cannot anticipate every failure mode, and (2) attempting to handle errors you don't understand produces code that silently continues in an incorrect state, which is often worse than crashing.

The Erlang alternative: *define what correct state looks like (in `init/1`), then only handle errors you truly understand. For everything else, crash fast and clearly. Let a supervisor restore you to correct state by restarting.*

This is only safe because Erlang processes have private heaps — a crashed process's heap is reclaimed by the garbage collector without affecting any other process. There is no shared mutable state to corrupt. The crash is a clean signal: "I reached a state I don't know how to handle."

A supervision tree takes this further. Every supervisor in the tree defines a fault-isolation boundary: if the workers in its subtree crash faster than the configured restart budget allows, the supervisor itself exits, propagating the failure up to *its* supervisor. Failures bubble up the hierarchy until they either hit a supervisor that can absorb them or they escalate to the top-level application restart.

The practical implication for structuring code: stable, battle-tested code (session managers, auth services, connection pools) lives near the root where it is rarely restarted and can afford to be conservative. Volatile, experimental, or feature code lives at the leaves where frequent restarts are acceptable. This structure is enforced at process-organization time, not at compile time or code-review time.

## Mechanics

**Process links and monitors**

Supervisors detect child failure using process *links*. A link between two processes is bidirectional: if either exits abnormally, the other receives an `EXIT` signal and also dies — unless it has called `process_flag(trap_exit, true)`. Supervisors do exactly this: they trap exit signals, receive them as `{EXIT, Pid, Reason}` messages, and decide what to restart. Monitors (`erlang:monitor/2`) are the one-directional, non-fatal variant: the monitoring process gets a `DOWN` message but does not die.

**Supervisor strategies**

A supervisor is configured with one of four restart strategies, chosen at startup to match the dependency structure of its children:

- **`one_for_one`**: Only the crashed child is restarted. Use when workers are fully independent — a failed cache worker does not affect an HTTP handler worker.
- **`one_for_all`**: When any child crashes, *all* children are shut down and restarted. Use when workers share state or have coordinated startup invariants — if one is broken, the group state is suspect.
- **`rest_for_one`**: The crashed child and all children started *after* it are restarted, in the original startup order. Use when startup has ordering dependencies — child C depends on child B which depends on child A; crashing B invalidates C but not A.
- **`simple_one_for_one`** (sometimes called `one_for_one` with dynamic children): A pool of identical child processes, all started from the same spec. Used for worker pools where children are added and removed dynamically at runtime.

**Restart intensity and period**

Each supervisor has two numeric parameters: `MaxRestarts` and `MaxSeconds`. If more than `MaxRestarts` crashes occur within any `MaxSeconds` window, the supervisor concludes it cannot recover its children and exits itself, propagating the failure upward. This is the circuit-breaker mechanism built into the hierarchy: runaway restart loops can't go on forever.

**The OTP GenServer protocol**

In practice, Erlang workers are written as `gen_server` behaviours — a standardized process lifecycle that the OTP framework knows how to supervise, upgrade, and instrument. A GenServer implements six callbacks:

```erlang
init(Args) -> {ok, State} | {stop, Reason}
handle_call(Request, From, State) -> {reply, Reply, NewState}   %% synchronous
handle_cast(Request, State)       -> {noreply, NewState}        %% asynchronous
handle_info(Info, State)          -> {noreply, NewState}        %% catch-all
terminate(Reason, State)          -> ok                         %% cleanup
code_change(OldVsn, State, Extra) -> {ok, NewState}             %% hot upgrade
```

The `init/1` function defines the "correct initial state" that a supervisor's restart will return to. This is the point of the entire pattern: `init/1` should succeed unconditionally, or fail for a known reason. It should never succeed with a partially-initialized state.

**Supervisor child specifications**

Each child in a supervisor is defined by a `ChildSpec` map that describes how to start it, whether it should be restarted on crash (`permanent`, `transient`, or `temporary`), the shutdown timeout, the child type (worker vs supervisor), and the modules it uses (for hot-code loading). A `permanent` child is always restarted. A `transient` child is only restarted if it crashed abnormally (not on a clean exit). A `temporary` child is never restarted.

**Full example topology**

A typical OTP application tree for a web service might look like:

```
ApplicationSupervisor   (one_for_one)
├── ConnectionPoolSup   (one_for_all)  — pool workers share the socket
│   ├── Worker1
│   ├── Worker2
│   └── Worker3
├── RequestHandlerSup   (simple_one_for_one) — dynamic pool
│   ├── Handler<pid1>
│   └── Handler<pid2>
└── CacheServer         (permanent GenServer)
```

If a `Worker2` crashes, `ConnectionPoolSup`'s `one_for_all` policy restarts all three workers to ensure the shared socket state is consistent. If a `Handler` crashes while serving a request, only that handler is restarted (or not, if `transient`); the pool supervisor absorbs it silently. If `CacheServer` crashes faster than its parent's intensity/period allows, `ApplicationSupervisor` itself exits.

## Where it breaks

**External state that outlives the process.** A GenServer that opens files, holds database connections, or acquires OS-level locks may crash without releasing them. `terminate/2` is called for clean shutdowns but *not always* for abnormal exits in all OTP versions. The fix is to use OTP's link-based cleanup (`trap_exit + terminate`) carefully, or to manage resources through a separate supervised pool process.

**Restart loops burning the intensity/period budget.** If `init/1` itself fails repeatedly (e.g., a downstream service is down at startup), the supervisor exhausts its budget and crashes upward. This is the correct behavior — it surfaces that something is fundamentally broken — but it can cascade to an application shutdown when the issue is transient. Common mitigation: use `{ok, State}` with a retry timer in `init/1`, deferring the real connection until the first `handle_info`.

**Shared in-process tables.** ETS (Erlang Term Storage) tables survive the death of their owning process... unless the owner crashes, in which case the table is deleted unless ownership is transferred via `heir`. This is an easy footgun: a supervisor restarts a worker, but the ETS table it was using is gone.

**"Let it crash" is wrong for user-visible errors.** If an HTTP request arrives with a malformed JSON body, crashing the handler and restarting to initial state is wrong — the request is lost and the client gets no useful error response. "Let it crash" applies to *unexpected internal errors*; input validation and user-facing errors still require explicit, local handling.

**Not a substitute for correctness.** A process that enters an infinite loop or consumes unbounded memory doesn't crash — it just degrades. Supervision trees handle crashes, not all failure modes. Watchdog-style monitoring (e.g., erlang:monitor_node, or explicit timer-based health checks) is still needed for liveness.

## Why it works

The deepest principle: **a supervision tree is a fault isolation lattice, and "let it crash" is just choosing restart-to-initial-state as the default recovery action for unknown failures.**

This mirrors several patterns that appear elsewhere in systems:

- **OS kernel / user-space isolation**: The kernel runs processes in isolated address spaces so a crashing user process doesn't corrupt kernel state. The kernel recycles the process's resources and reports the exit to waiting parents — exactly what Erlang's supervisor does for its children.
- **Kubernetes pod restarts**: A kubelet is a supervisor using `one_for_one` semantics. `restartPolicy: OnFailure` is `transient`; `restartPolicy: Always` is `permanent`; `CrashLoopBackOff` is intensity/period exhaustion forcing human intervention.
- **Docker `--restart=on-failure`**: One-for-one restart with optional max-retries budget — the same intensity/period model.
- **React Error Boundaries**: A `componentDidCatch` + fallback render is a `one_for_one` supervisor over a React subtree. The boundary decides whether to restart (re-render) or escalate (throw to a parent boundary).
- **Microservice circuit breakers**: Hystrix/Resilience4j define a service-level supervisor boundary: N failures in T seconds opens the circuit (supervisor exits), requests fail fast, periodic half-open probes attempt `init/1`-equivalent recovery.

The "X is just Y" insight: **"let it crash" is the application-level manifestation of the same principle behind every OS process model, every container runtime, and every circuit breaker: define an isolation boundary, detect failure at that boundary, and recover to a known-correct initial state rather than continuing in an unknown-incorrect one.** The only thing Erlang adds is making this hierarchy *first-class* in the language runtime rather than bolted on at the infrastructure layer.

There is a second, subtler insight: the topology of the supervision tree *encodes your assumptions about which failures are related*. `one_for_all` is a statement that "these workers share state and a failure in one invalidates the others." `rest_for_one` is a statement that "these workers have ordered startup dependencies." The strategy choice is executable documentation of your system's fault-coupling structure.

## Going deeper

1. **"Learn You Some Erlang for Great Good!"** by Fred Hebert — free online at learnyousomeerlang.com. Chapters 16–19 cover OTP GenServer, supervisors, and applications in rigorous depth with examples.
2. **"Making Reliable Distributed Systems in the Presence of Software Errors"** by Joe Armstrong (2003 PhD thesis) — the original formalization of the Erlang concurrency and fault-tolerance model; Armstrong was one of Erlang's creators at Ericsson. The first 50 pages alone are worth reading.
3. **Elixir / Phoenix** source code — the modern OTP ecosystem. Phoenix's channel supervisor and presence system are clean, real-world examples of `one_for_one` and `simple_one_for_one` at production scale.
