---
title: "Notes on structured concurrency, or: Go statement considered harmful"
source: https://vorpus.org/blog/notes-on-structured-concurrency-or-go-statement-considered-harmful/
author: Nathaniel J. Smith
company: Independent (creator of Python Trio)
date_posted: 2018-04-25
date_digested: 2026-09-11
---

# Notes on structured concurrency, or: Go statement considered harmful

## What's new to learn

- **The nursery / task-group primitive**: A scoped container for concurrent tasks that guarantees all spawned child tasks complete (or are cancelled) before the container exits — giving concurrent code the same "well-defined lifetime" guarantee that a function call gives sequential code.
- **The control-flow isomorphism**: `go`/`Thread.start()`/`asyncio.ensure_future()` are to concurrent programs what `goto` is to sequential programs — they create flow that "jumps" without any guarantee of returning to the originator's scope.
- **Task trees as call stacks**: Structured concurrency provides a concurrent program with an analogue of the sequential call stack — a tree of task lifetimes that mirrors the lexical nesting of scopes — making cancellation, error propagation, and debugging tractable.

## Prerequisites

- Basic familiarity with async/await syntax in any language (Python, JavaScript, Rust, etc.)
- Understanding of threads or goroutines as a programming model
- Knowing what a coroutine or an event loop is at a conceptual level
- Dijkstra's 1968 letter "Go To Statement Considered Harmful" is not required but the parallel is tight enough to be worth reading first

## The core idea

In 1968, Edsger Dijkstra wrote that the `goto` statement was harmful because it made program control flow impossible to reason about locally: a `goto` lets you jump to *anywhere*, breaking the guarantee that "calling a subroutine returns to the call site." Structured programming replaced `goto` with `if`, `while`, `for`, and function calls — all of which preserve that guarantee. The result is code you can read top-to-bottom, compose, and reason about.

Smith's 2018 argument: we made the exact same mistake in concurrent programming.

When you write:

```python
asyncio.ensure_future(do_work())      # Python pre-3.11
go fetchUser(ctx)                      # Go
Thread(target=task).start()            # Python threads
executor.submit(task)                  # Java
```

you have launched a task with *no defined relationship to the calling scope*. The caller may return; the task may still be running. The task may raise an exception; the caller will never see it. The task may hold a resource; the caller cannot know when to release it. This is `goto` for concurrency.

The fix is the **nursery** (called a *task group* in most languages that adopted the idea):

```python
async with trio.open_nursery() as nursery:
    nursery.start_soon(fetch_user, user_id)
    nursery.start_soon(fetch_prefs, user_id)
# Control reaches here only after BOTH tasks are done.
# Any exception from either task is re-raised here.
```

The nursery scope is a contract: every task you start inside it is guaranteed to complete before the scope exits. The nursery is a structured control-flow primitive — it has one entry and one exit, and control always returns to the exit.

## Mechanics

### What a nursery provides

**1. Bounded lifetime.** No task started in a nursery outlives the nursery. If the nursery body finishes but child tasks are still running, the nursery waits. If a child task is still running when an error occurs, the nursery cancels it and then re-raises the exception.

**2. Automatic error propagation.** If any child task raises an uncaught exception, the nursery cancels all remaining sibling tasks and propagates the exception outward to the nursery's parent scope. No manual error channels or callbacks needed.

**3. Transitive cancellation.** If the nursery's outer scope is cancelled (e.g., because *its* parent is shutting down), the cancellation propagates inward to all child tasks. Cancellation flows down the task tree automatically.

**4. Back-pressure / flow control.** Because nurseries are synchronous scopes, you cannot accidentally spawn unbounded goroutines — the nursery blocks the parent until all children finish.

### The task tree as a call stack

In sequential programs, the call stack shows you what functions are currently executing and how they're nested. Structured concurrency gives concurrent programs an analogous structure: a **task tree**.

```
main()
├── nursery scope A
│   ├── fetch_user() [running]
│   └── fetch_prefs() [running]
└── (waiting for A to exit)
```

This tree is always well-defined because no task can outlive its parent nursery. When an exception occurs, the tree tells you exactly which tasks are still live and which parent scope will receive the error. This is as useful as a stack trace is in sequential debugging.

### Code structure with vs. without nurseries

**Without nurseries (Go-style):**
```go
func handler(ctx context.Context) {
    go fetchUser(ctx, userID)     // spawned, forgotten
    go fetchPrefs(ctx, userID)    // spawned, forgotten
    // How do we wait? How do we catch their errors?
    // Answer: explicit WaitGroup + error channel boilerplate
}
```

You need a `sync.WaitGroup`, an `errgroup`, or a channel to collect results. Error propagation requires manual coordination. Cancellation requires passing `context.Context` everywhere and checking `ctx.Done()` at every loop iteration. It works, but it's entirely implicit.

**With nurseries (Python Trio / asyncio.TaskGroup):**
```python
async def handler():
    async with asyncio.TaskGroup() as tg:
        user_fut = tg.create_task(fetch_user(user_id))
        prefs_fut = tg.create_task(fetch_prefs(user_id))
    # Both done here. Exceptions automatically propagated.
    user = user_fut.result()
    prefs = prefs_fut.result()
```

All coordination is structural, not manual. The reader knows immediately: when this `async with` block exits, both tasks are done.

### Language adoption

| Language | Construct | Added |
|---|---|---|
| Python | `asyncio.TaskGroup` | 3.11 (2022) |
| Python | `trio.Nursery` | 2018 (original) |
| Java | `StructuredTaskScope` | JDK 21 preview, stable JDK 24 |
| Swift | `TaskGroup`, `async let` | Swift 5.5 (2021) |
| Kotlin | `CoroutineScope` (structured by default) | 2018, influenced by this post |
| Rust | `tokio::JoinSet`, rayon `join()` | Various |
| C++ | `std::jthread` (limited) | C++20 |

Every major modern concurrency system converged on this model.

## Where it breaks

**Daemon tasks.** Some concurrent tasks genuinely have no natural parent: a background metrics collector, a TCP connection acceptor, a heartbeat. These need to outlive any particular scope. Trio handles this via `nursery.start()` (as opposed to `start_soon`) and careful design of long-lived nurseries at the top of the program. But "a task with no parent" remains a conceptual friction point — it exists outside the structured tree.

**Actor systems.** In actor models (Erlang, Akka, Elixir), actors spawn other actors and can outlive them — deliberately. The ownership is inverted: the supervisor outlives its children, but children are not confined to a lexical scope. Structured concurrency and actor models solve different problems; they are complementary but not identical.

**Fine-grained parallelism overhead.** Creating nurseries for very short-lived tasks (sub-millisecond) can add overhead. This is the same tradeoff as function call overhead — trivially fine for almost everything, a real constraint for tight numerical loops.

**Dynamic fan-out.** When you don't know at compile time how many tasks you'll spawn (e.g., one task per request that arrives), the API is slightly more verbose because you need to manage the nursery lifetime manually across the request lifecycle.

**Cancellation semantics.** When you cancel a task group, should all exceptions from cancelled tasks be surfaced, or suppressed? Python's `TaskGroup` surfaces all exceptions as an `ExceptionGroup`. This is correct but unfamiliar — new syntax (`except*`) was needed to handle it.

## Why it works

The deeper principle: **every safe control-flow primitive has exactly one entry point and exactly one exit point, and control always returns to the exit.**

| Sequential primitive | Entry | Exit | Returns to caller? |
|---|---|---|---|
| `if/while/for` | condition | end of block | Yes |
| function call | first line | `return` | Yes |
| `goto` | call site | target label | **No** |

| Concurrent primitive | Entry | Exit | Returns to spawner? |
|---|---|---|---|
| Nursery | `async with` | end of block | **Yes** (waits for children) |
| `go` statement | call site | goroutine completion | **No** |

This is the same invariant. Structured programming proved that restricting to single-entry/single-exit control flow makes programs composable, reasoning-tractable, and exception-safe. Structured concurrency applies the same restriction to the space of task lifetimes.

A function that uses a nursery internally is still a function from the caller's perspective — it starts, it runs, it returns. The caller has no obligation to know that it ran concurrent tasks internally. This is the composability property. With `go` statements, the caller cannot know whether callee has leaked tasks into the background — a caller is forced to know the concurrency model of every function it calls.

This is exactly the argument Dijkstra made about `goto`: `goto` breaks local reasoning. You cannot understand a function in isolation if it might jump to an arbitrary label outside its body. The `go` statement breaks local reasoning about task lifetimes for the same structural reason.

## Going deeper

1. **Python Trio documentation** — Smith's own library implementing the ideas; the docs are among the best explanations of why the API is designed the way it is: https://trio.readthedocs.io/en/stable/

2. **"Structured Concurrency in Java"** (JEP 453) — The Java design document for `StructuredTaskScope` shows how the JDK designers adapted Smith's ideas for a statically typed, multi-threaded (rather than async) language, including interesting decisions around virtual threads: https://openjdk.org/jeps/453

3. **Roman Elizarov, "Structured Concurrency"** (KotlinConf 2018) — The Kotlin coroutines team's account of how Smith's post influenced their CoroutineScope design; shows the design decisions in a production language runtime: https://elizarov.medium.com/structured-concurrency-722d765aa952
