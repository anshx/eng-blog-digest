---
title: "The Error Model"
source: https://joeduffyblog.com/2016/02/07/the-error-model/
author: Joe Duffy
company: Microsoft (Midori Research OS)
date_posted: 2016-02-07
date_digested: 2026-10-17
---

# The Error Model

## What's new to learn

1. **Abandonment vs. recovery are categorically different**: "errors" conflates two things that need different language mechanisms — expected failures (network timeout, disk full) that you can meaningfully handle, and programming bugs (null dereference, broken invariant) that cannot be safely handled because the program's state is already corrupt.
2. **Bugs should abandon, not throw**: a process in a state where its own preconditions were violated cannot trust any of its subsequent code; "catching" a bug is theater that propagates corruption rather than containing it.
3. **Recoverable errors belong in the type system**: making expected failures part of the function's return type (rather than an out-of-band exception path) lets the compiler enforce that callers handle them, without the verbosity tax of Java's checked exceptions.

## Prerequisites

- Familiarity with how exceptions work in Java or C++ (stack unwinding, try/catch)
- Basic understanding of C-style error codes (returning -1 or errno)
- A sense of why unchecked exceptions are hard to reason about in large codebases
- Helps to have seen Rust's `Result<T, E>` and `panic!` side by side, even if just superficially

## The core idea

Every error-handling mechanism in production use conflates two things that are actually opposites:

**Expected failures** — conditions the caller anticipated might occur and wrote code to handle: a file not found, a network timeout, a malformed input. These are not bugs. They are normal program states. The right response is to propagate information about them to a caller that can do something useful.

**Programming bugs** — conditions that should never occur if the code is correct: a null dereference, an array out-of-bounds access, a violated precondition, integer overflow in an arithmetic operation that was supposed to be bounded. These mean the code is wrong. No caller anticipated them because the code was not supposed to produce them. The right response is to *stop*.

Midori's insight (developed over years of building a capability-based OS in a safe language derived from C#) is that mixing these into one "error handling" system produces a mechanism that is bad at both: you can neither silently ignore errors (types force you to handle them) nor safely catch bugs (the state is already corrupt). The fix is to give each case a separate mechanism and never conflate them.

- **Recoverable errors** → typed return values, forced handling, composable with `?`-style propagation
- **Bugs** → **abandonment**: the process terminates immediately, no user code runs during teardown, a supervisor may restart it

## Mechanics

### Recoverable errors as types

In Midori's language, a function that might fail returned a discriminated union:

```
int Divide(int numerator, int denominator)
    throws DivideByZeroException;
```

But unlike Java's checked exceptions, Midori's version was a first-class value in the return type, not a parallel control-flow channel. The caller was forced to match on success or failure before accessing the result:

```
var result = Divide(10, 2);
if (result.IsError) { ... }
else { use(result.Value); }
```

The language added syntactic sugar equivalent to Rust's `?` operator so that propagating errors up required only one character, not four lines of if-else. The critical property: the compiler rejected any code that could silently discard an error. There was no "catch-and-ignore" escape hatch for expected failures.

### Abandonment for bugs

Contracts — preconditions, postconditions, loop invariants — were first-class citizens:

```
int Divide(int numerator, int denominator)
    requires denominator != 0
{
    return numerator / denominator;
}
```

A precondition violation is **not** a DivideByZeroException. It is a bug in the *caller*, which passed an argument that it promised it would not. The right response is not to propagate an exception to whatever called Divide — that caller also has no idea what to do, because it also violated a contract. The right response is to terminate the process immediately, write a crash dump, and let the supervisor restart into a clean state.

Integer overflow, array out-of-bounds, and null dereference all triggered the same mechanism: abandonment. No finalizers ran. No catch blocks ran. The process just stopped.

The isolation boundary was the *process*, not the thread. Each process had its own address space and could fail without corrupting its neighbors. This is exactly the Erlang model: each actor (process) can crash; the supervisor tree above it decides whether and how to restart.

### What the split looks like in practice

Midori's team measured a 1:10 ratio in their codebase: roughly 1 recoverable error for every 10 bug-triggered abandonments. This ratio flips common intuition: most "error handling" code is actually handling programming mistakes, not the expected environmental failures it appears to be handling.

Classic patterns that collapse under Midori's taxonomy:

- **`errno` in C**: a global mutable side-channel that conflates bug (EINVAL, bad argument) with environment (ENOENT, file not found). Callers can check it or not; most don't.
- **Java's checked exceptions**: got the "force handling" right, but modeled everything as recoverable, inflated method signatures, and training showed that `catch (Exception e) {}` — the empty-catch anti-pattern — was the dominant real-world response.
- **Go's `if err != nil`**: structurally like Midori's typed returns, but `panic()` is underspecified (both bugs and "I give up" use the same mechanism), and the language provides no syntactic sugar for propagation, making the boilerplate the dominant visual element of Go error-handling code.
- **Rust**: closest to Midori. `Result<T, E>` for recoverable errors with `?` for propagation; `panic!` (which unwinds or aborts depending on configuration) for bugs. Rust adds `unwrap()` as an escape hatch that triggers panic, deliberately making "I'll handle this later" code visible and greppable.

## Where it breaks

**The bugs-vs-errors line is fuzzy in practice.** Consider parsing an integer from a string. Is `ParseInt("abc")` a bug (the caller should have validated the input) or an expected failure (the caller is processing user input that might be anything)? The answer depends on context. Midori required developers to make this decision explicitly for every API, which imposed real cognitive and API-design costs.

**Abandonment has high cost in low-isolation environments.** The model works cleanly when every process is isolated and a supervisor can restart it in milliseconds. In embedded systems, safety-critical controllers, or long-running stateful processes where restart is expensive, abandonment's "just die" policy is unacceptable. Midori assumed OS-level process isolation; the model does not translate directly to C++ code that must run inside a single process with shared mutable state.

**Finalizer/destructor guarantees weaken.** If abandonment skips cleanup code, resources may leak. Midori addressed this with a combination of ownership types (preventing most leaks statically) and process-level resource reclamation by the OS. Systems without those guarantees must rely on OS cleanup and accept that in-flight operations may be left in undefined states.

**Error type proliferation.** Making errors first-class types encourages creating many specific error types, which produces combinatorial explosion in error-handling code for functions that call many subsystems. Rust partially addresses this with trait objects (`Box<dyn Error>`) and libraries like `thiserror` and `anyhow` that provide ergonomic wrapping.

## Why it works

The deeper principle is that **the feasibility of recovery depends on whether the program's own invariants still hold**.

When a network connection times out, the program's internal state is fine — it just didn't get the bytes it wanted. Every invariant the programmer established still holds. Resuming computation after handling the timeout is safe because the program is in a known, intended state.

When a precondition is violated, the programmer's model of what state the program is in has already failed. Any catch block that runs does so in a world that violates the assumptions under which that catch block was written. Recovery code written for normal execution paths is now executing in an abnormal state it was never designed for. This is why catch-all exception handlers so frequently produce "ghost" bugs: they swallow corruption and let it propagate until a much harder-to-debug symptom surfaces later.

This maps to a type-theoretic observation: the `Result<T, E>` monad is compositional — you can chain operations, map over successes, and accumulate errors — *because the monad laws hold only when each step leaves the program in a consistent state*. A bug breaks the monad's semantic guarantee. Abandonment is the correct response to a broken monad: exit the computation entirely rather than pretend the laws still hold.

The pattern recurs everywhere:

- **Erlang supervision trees** (in this archive): the "let it crash" philosophy is exactly Midori's abandonment principle applied at the actor level. Each actor's job is not to defend against every possible corruption; its job is to do its work correctly, and if something goes wrong, to crash fast so the supervisor can restart from a known state.
- **Rust's `panic!`**: deliberately not catchable in stable Rust by default (unwinding is compile-time-optional, `catch_unwind` is opt-in and discouraged outside FFI boundaries) because Rust's authors share Midori's view that bugs should terminate, not be caught.
- **Go's `recover()`**: the exception that proves the rule — it exists but is explicitly documented as only appropriate for converting panics at package boundaries into errors for the caller, not for general error handling.
- **Database transaction abort**: a transaction that hits a constraint violation or deadlock does not partially commit; it rolls back entirely. The database's invariant (ACID) is restored by abandonment (rollback) rather than by patching the corrupt intermediate state.

The error handling design of any language is ultimately an answer to: *at what boundary does the invariant-violation get contained, and who is responsible for ensuring that boundary is clean?* Midori answered: the process boundary, enforced by the OS, with the type system eliminating accidental ignorance of expected failures everywhere below that level.

## Going deeper

1. **Joe Duffy's full Midori blog series** (joeduffyblog.com/2015/11/03/blogging-about-midori/) — twelve posts covering the entire OS design: memory safety, concurrency, the type system, and performance. The error model post assumes context from the earlier posts on the language design.
2. **"Designing Error Handling" — Herb Sutter's P0709 proposal for C++** — a modern attempt to bring Midori-style deterministic error handling to C++, separating "exceptional situations" (typed error values, zero-overhead when not triggered) from "logic errors" (aborts). Traces the same design space with a C++ implementation lens.
3. **"Error Handling in Rust"** (doc.rust-lang.org/book/ch09-00-error-handling.html) — the definitive statement of Rust's two-tier model. Reading it after "The Error Model" makes the `panic!` vs `Result<T, E>` split feel like an inevitable conclusion rather than an arbitrary API choice.
