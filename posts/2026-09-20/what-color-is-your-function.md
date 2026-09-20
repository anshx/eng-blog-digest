---
title: "What Color is Your Function?"
source: https://journal.stuffwithstuff.com/2015/02/01/what-color-is-your-function/
author: Bob Nystrom
company: Google (Dart team)
date_posted: 2015-02-01
date_digested: 2026-09-20
---

# What Color is Your Function?

## What's new to learn

1. **Function coloring**: every time a language adds async/await, it implicitly splits all functions into two mutually incompatible categories — "sync" and "async" — and the distinction propagates transitively through any call graph, forcing callers to adopt the same color as their callees.

2. **Color-free concurrency**: Go, Erlang, and threads sidestep coloring entirely by making all functions potentially blocking at the runtime level — no annotation, no propagation, every function is interchangeable.

3. **Algebraic effects as the generalization**: function coloring is a special case of effect propagation; every "viral" annotation in a type system (IO, async, checked exceptions, capabilities, purity) is the same phenomenon under different names.

## Prerequisites

- How JavaScript Promises / Python `asyncio` work at a basic level (event loop, coroutine, `await`)
- What a function call stack looks like (caller, callee)
- Familiarity with at least one language that has async/await (JS, Python, C#, or Rust)

## The core idea

Imagine a made-up language where every function must be one of two colors: **blue** (ordinary, synchronous) or **red** (asynchronous). The color rules are:

1. A red function can call any function — red or blue.
2. A blue function can *only* call blue functions.
3. To call a red function, you must be red too, and you must use a special `await` keyword.

Now imagine you write a simple utility:

```
blue function readConfig() { ... }
blue function processData() {
    config = readConfig()   // fine
}
```

Later someone deep in your call graph decides `readConfig` needs to do network I/O (inherently async). It becomes red:

```
red function readConfig() async { ... }
```

Suddenly `processData` breaks — it's blue but is calling red. To fix it, `processData` must become red. Now *its* callers break. And their callers. The redness floods upward through the entire call graph, leaf to root, until the color boundary hits an `await`-able entry point.

This isn't hypothetical — it is *exactly* what happens in JavaScript, Python (`async def`), C# (`async Task`), Rust (`async fn`), and Swift (`async`). The moment you add async I/O to one function, your diff touches dozens of files just to shuffle `async` and `await` keywords to every caller along the chain.

The post names this the **function coloring problem**: the language has inadvertently created two worlds that cannot freely interoperate, and every function at every layer of your stack must declare allegiance to one or the other.

## Mechanics

**The asymmetry that causes the spread**

In all real async languages the coloring rules are:
- `async` can call `async` or sync — fine.
- sync **cannot** call `async` without becoming async itself (or using a blocking executor that defeats the purpose).

This is not a quirk — it is structural. An async function returns a *future*/*promise*/*coroutine* rather than a value. A sync caller expecting a value receives a wrapped object instead; to unwrap it it must suspend itself, which makes it async too.

**What the split costs in practice**

Libraries end up shipping two versions of everything. Python's ecosystem has `requests` (sync) and `httpx` (async), `sqlite3` (sync) and `aiosqlite` (async), `redis` (sync) and `aioredis` (async). Rust traits that were sync cannot suddenly become async without a proc-macro workaround (`#[async_trait]`). Interface boundaries — abstract classes, traits, protocols — were designed before the split and have no "color-polymorphic" slot. The moment you need both colors, your abstraction leaks.

**The hidden performance trap**

Async runtimes are fast for I/O-bound work but add overhead for CPU-bound work. When callers are async they *expect* all callees to be non-blocking; if a red function does CPU-heavy work it starves the event loop. So in practice three colors emerge: sync (CPU work, no I/O), async (I/O-bound, non-blocking), and "blocking async" (sync work called from an async context, must be offloaded to a thread pool). The two-color system always grows into at least three.

**How the coloring shows up in other languages**

| Annotation | "Color" | Propagation rule |
|---|---|---|
| `async`/`await` (JS, Python, C#, Rust) | async | callers must be async |
| `throws` / `throws IOException` (Java, Swift) | checked exception | callers must declare or catch |
| `IO` monad return type (Haskell) | effectful | callers must be in `IO` |
| `unsafe` (Rust, C#) | unsafe | must be in an `unsafe` block |
| `@MainActor` / `@Sendable` (Swift) | actor-isolated | cross-context requires `await` |

Each row is the same problem: a property of one function forces a visible annotation on every function above it in the call chain.

## Where it breaks

**Color doesn't compose at the boundary**

The interface / trait problem is real and has no fully clean solution. In Rust, `async fn` in a trait wasn't stable until Rust 1.75 (2023), and even then, certain object-safety rules complicate it. In C# any `interface` method you want to be async must be declared `Task`-returning from day one; retrofitting it is a breaking API change.

**Infectious but not transparent**

The coloring is visible to all intermediate callers, not just the ones that care. A middleware layer that only passes data through still has to declare its color just to route a call. Go's `context.Context` is sometimes cited as an example of a similar viral annotation (cancellation propagating through call chains) even though Go has no async coloring — demonstrating that *any* transitive concern can become a coloring problem given the wrong API design.

**Sync wrappers are lies**

It is *possible* to call async from sync by spinning a new runtime (`tokio::runtime::Runtime::block_on`, `asyncio.get_event_loop().run_until_complete`). This compiles and runs — but blocks a thread, can deadlock if called from inside an async runtime, and breaks the whole point of cooperative multitasking. "Just add a sync wrapper" is a trap that production systems fall into repeatedly.

**The three-color world**

CPU-bound functions in an async system must be spawned onto a blocking thread pool. This creates a third implicit color (blocking-async). Libraries that are async-safe for I/O but not for CPU work have to document which color they are. The model intended to simplify I/O ends up with more colors than the synchronous world it was meant to replace.

## Why it works

**Function coloring = algebraic effects with two effects**

An *algebraic effect* is a principled mechanism for describing what a computation *might do* beyond just return a value. Effect systems — present in Koka, OCaml 5, Eff, and being designed for Scala 3 — let functions declare effects in their type signature and let callers handle them polymorphically. `async`/`await` is this, but bolted on after the fact, without a unified effect-polymorphic abstraction layer.

In Koka you can write:
```koka
fun read-config() : <io> Config { ... }
```
A caller that doesn't handle `io` must itself declare `<io>` in its effect row — same viral propagation. But the key difference is *effect polymorphism*: you can write functions that are generic over their effects, composing with both sync and async callers automatically, and the type system proves the composition is safe.

**Go's answer: collapse the distinction**

Go makes *all* functions potentially blocking by running every goroutine on an M:N scheduler. When a goroutine blocks (on a channel, syscall, or `time.Sleep`), the runtime parks it and runs another goroutine on the same OS thread. To the programmer, there are no colors: every function looks synchronous, no `await` anywhere, yet the runtime achieves the same event-loop efficiency as async/await systems.

The cost: goroutines are runtime objects with their own small stack; you cannot statically check that a function is non-blocking. But the benefit is huge: a function signature is a function signature. You never have to split your library into sync and async halves.

Erlang processes and Haskell's green threads achieve the same thing: every function is "color-free" because the runtime handles scheduling transparency.

**The deeper principle: type annotations that prevent composition are effects**

Whenever a language adds a property to a function's type that (a) changes how it must be called and (b) forces callers to also carry the property, that property is an algebraic effect — whether the language calls it that or not. The coloring problem is what happens when you add an effect *without* an effect-polymorphic type system to handle it cleanly. The fix is either to:

1. Make the effect invisible at the type level (Go, Erlang — runtime handles it)
2. Expose the effect explicitly with polymorphism (Koka, OCaml 5 effects)
3. Provide a combinator that absorbs the effect at a boundary (monadic `IO`, `async` blocks in Rust that return `impl Future`)

The reason `async`/`await` in mainstream languages feels painful is that it chose option 3 — expose the effect — but without option 2 (polymorphism). Every function that touches async work must be explicitly marked, with no way for an abstraction to be neutral.

This connects directly to why checked exceptions in Java died: they are the same pattern (a viral type annotation for the `throw` effect without exception polymorphism), and the Java ecosystem collectively decided the annotation cost outweighed the safety benefit.

## Going deeper

1. **Algebraic effects and handlers in Koka** (Leijen, 2017) — the research language that makes effects first-class and polymorphic, showing what async/await looks like with a clean foundation: https://koka-lang.github.io/koka/doc/book.html

2. **"Notes on structured concurrency, or: go statement considered harmful"** (Nathaniel J. Smith, 2018) — a companion piece: structured concurrency is the answer to *task lifetime* problems; function coloring is the answer to *effect propagation* problems; both stem from the same original `go`-statement design. Already in this archive: 2026-09-11.

3. **OCaml 5 effects tutorial** — OCaml 5 (2022) added typed effects as a first-class language feature, making this a working production implementation, not just theory: https://v2.ocaml.org/api/Domain.html and the Jane Street blog on how they use them.
