---
title: "JavaScript engine fundamentals: Shapes and Inline Caches"
source: https://mathiasbynens.be/notes/shapes-ics
author: Mathias Bynens and Benedikt Meurer
company: Google (V8 team)
date_posted: 2018-07-10
date_digested: 2026-11-13
---

# JavaScript engine fundamentals: Shapes and Inline Caches

## What's new to learn

1. **Shapes (Hidden Classes)**: Every object in a JavaScript engine has a hidden structural descriptor called a *shape* that records which properties exist, in what order, and at what memory offsets. Objects with identical property layouts share the same shape object, turning millions of homogeneous objects into a single shared structural description.

2. **Inline Caches (ICs)**: At each property-access bytecode instruction, the engine stores a small cache that maps (shape → property offset). When the cache hits, the access goes directly to the offset — no hash lookup, no name lookup, no inheritance walk. The cache lives *at the instruction*, not in a central table.

3. **IC state machine**: An IC transitions from *uninitialized* (no info) → *monomorphic* (one shape, O(1) direct access) → *polymorphic* (2–4 shapes, linear scan) → *megamorphic* (5+ shapes, no specialization). Monomorphic is the target; megamorphic means no optimization.

## Prerequisites

- JavaScript objects and prototype chains (you need to know what `obj.x` does conceptually)
- What bytecode interpretation means (the engine executes a stream of typed instructions, not raw JS source)
- Hash tables: why dictionary lookups have overhead even when O(1) amortized

## The core idea

In a dynamically-typed language, every property read on `obj.x` is *potentially* a full dictionary lookup: hash the string `"x"`, find the bucket, check for collisions, follow the chain, check visibility. That's not particularly fast when done billions of times per second.

JavaScript engines solve this by separating *structural metadata* from *per-object values*. When you write `{x: 1, y: 2}`, the engine does not store `{name: "x", offset: 0, name: "y", offset: 4}` inside every individual object. It creates a **shape** (V8 calls them *Maps*; SpiderMonkey calls them *Shapes*; the academic literature calls them *Hidden Classes*) that holds the structural description once, and stores only the values `[1, 2]` in the object itself, alongside a pointer to the shared shape.

Shapes make property *lookup* cheap: `obj.x` becomes "look up the shape to find that `x` is at offset 0, then read `obj[0]`" — two pointer dereferences instead of a hash computation.

**Inline caches** eliminate even the shape lookup. Each bytecode instruction that accesses `obj.x` caches `(shape, 0)` the first time it runs. On subsequent executions, the instruction checks "is the shape of `obj` still the cached shape?" (one comparison) and, if yes, reads offset 0 directly. No lookup of any kind — just a branch-predicted comparison and an indexed load.

The combined result: property access in hot JIT'd code runs at essentially the same speed as a struct field access in C.

## Mechanics

### Shape construction and transition chains

Shapes are built incrementally as properties are added.

```js
const obj = {};
obj.x = 1;
obj.y = 2;
```

The engine creates:

```
shape₀ (empty)
  │ transition: add x → offset 0
shape₁ { x: 0 }
  │ transition: add y → offset 4
shape₂ { x: 0, y: 4 }
```

The object itself holds `[1, 2]` and a pointer to `shape₂`. The chain from `shape₀` to `shape₂` is a *transition tree*: each node is immutable; adding a property creates a child node.

### Shape sharing

Objects created identically share the same shape:

```js
const a = { x: 1 };
const b = { x: 2 };
// a and b both point to shape₁ — one shape for two objects
```

In a typical application, millions of object instances can share a handful of shapes. The shape is a singleton per structural layout, which is the key space savings.

### Order matters: shapes are path-dependent

```js
const a = { x: 1, y: 2 };  // shape: x→0, y→4
const b = { y: 2, x: 1 };  // shape: y→0, x→4
// a and b have DIFFERENT shapes
```

The shape records the *order* in which properties were added, because that order determines their memory offsets. Reversing the order produces a different transition path and therefore a different shape. This is the source of most real-world shape diversity.

### Inline cache state machine

Consider this function called with many different argument shapes:

```js
function getX(obj) {
  return obj.x;
}
```

| Calls so far | IC state    | What happens on next call |
|---|---|---|
| 0 | Uninitialized | Look up shape, find offset, store (shape, 0) in IC |
| 1 | Monomorphic   | Check shape == cached shape (1 compare), read offset 0 |
| 2–4 shapes | Polymorphic   | Scan 2–4 (shape, offset) pairs; use first match |
| 5+ shapes | Megamorphic   | No cache; full property lookup every time |

Each bytecode instruction for `obj.x` has its *own* IC. So `getX` called with one shape is monomorphic even if other functions seeing many shapes are megamorphic — the IC is per-instruction, not per-property-name.

### Prototype property access

IC entries for prototype properties carry extra validation: they cache the shape of the object *and* the shapes of all objects in the prototype chain up to the point where the property is found. Any mutation to any prototype in the chain invalidates the IC.

```js
class Foo { getX() { return this.x; } }
const a = new Foo();  // IC on getX caches (Foo.prototype shape, offset for x)
Foo.prototype.newProp = 42;  // Invalidates all ICs that cached Foo.prototype's shape
```

## Where it breaks

**Shape explosion from conditional property addition**: If a property is sometimes added and sometimes not, every execution path produces a different shape. The function that accesses that property becomes polymorphic or megamorphic.

```js
// BAD: creates two shapes — one with debug, one without
function makePoint(x, y, debugMode) {
  const p = { x, y };
  if (debugMode) p.debug = true;  // branches create a shape fork
  return p;
}
```

**Heterogeneous arrays**: An array holding objects of different shapes causes every loop body access to become megamorphic:

```js
const mixed = [{ x: 1 }, { y: 1 }, { z: 1 }];  // three shapes
mixed.forEach(obj => obj.x);  // megamorphic; falls back to hash lookup every iteration
```

**Prototype mutation at runtime**: Adding a property to a constructor's prototype after objects have been created invalidates every IC that cached the old prototype shape, requiring all affected sites to re-specialize.

**Intentionally polymorphic utility functions**: A helper like `function get(obj, key) { return obj[key]; }` is inherently megamorphic. It sees every object shape and cannot specialize. This is not a bug; it is the correct trade-off — but callers should be aware that such functions always run at generic-lookup speed.

**When the assumption fails**: Shapes and ICs only help when objects are *structurally homogeneous* at each call site. The technique delivers no benefit for truly polymorphic usage — but megamorphic ICs degrade no worse than a plain hash lookup, so they don't make things worse than a naive implementation.

## Why it works

The deeper principle is **type specialization via observational feedback**: the engine observes what *actually happens* at runtime and builds a specialized fast path for it, with the generic path as the fallback.

This is not specific to JavaScript. The same pattern appears everywhere:

- **CPU branch predictors** observe which branches are taken and cache a prediction per instruction address (the branch predictor is an IC for control flow). The Branch Prediction archive entry shows the same uninitialized→specialized→generic state machine.
- **Copy-and-Patch JIT (CPython 3.13)** pre-compiles opcode *stencils* with holes. The first execution fills the holes with observed types and patches the bytecode to call the specialized stencil. Structurally identical to IC monomorphization, but at binary code generation granularity.
- **Python 3.11+ Adaptive Interpreter (PEP 659)** rewrites bytecode instructions in-place (quickening) based on observed types: `LOAD_ATTR` becomes `LOAD_ATTR_MODULE` or `LOAD_ATTR_SLOT` depending on what the runtime sees. The same IC state machine, applied to CPython.
- **Java HotSpot virtual call devirtualization**: An IC at a virtual dispatch site becomes monomorphic once only one implementing class is observed, turning the vtable dispatch into a direct call with a shape-check guard.
- **Adaptive Query Execution (AQE)**: Spark and Trino observe actual row counts at runtime shuffle boundaries and re-optimize downstream plan fragments. Same structure: plan is specialized on observed statistics, with the original plan as a fallback.

The *"oh, X is just an instance of Y"* insight: **inline caches are Profile-Guided Optimization (PGO) applied at the per-instruction granularity, in real time**. Ahead-of-time PGO collects type profiles in a training run and bakes specializations into the binary; JIT ICs collect the same profiles at the instruction level and apply specializations in the same process, with zero training overhead. The difference is temporal scope, not mechanism.

The shape-sharing design is an application of **interning**: instead of each object carrying its own structural description, structural descriptions are deduplicated into a shared singleton (the shape). This is the same design used for string interning, Python's small-integer cache, Java's string pool, and deduplicated symbol tables in linkers.

## Going deeper

1. **"JavaScript engine fundamentals: optimizing prototypes"** (mathiasbynens.be/notes/prototypes) — the follow-up post from the same authors, extending Shapes and ICs to cover how engines cache prototype property access chains and what breaks them.

2. **"PEP 659: Specializing Adaptive Interpreter"** (peps.python.org/pep-0659/) — the design document for Python 3.11's adaptive interpreter. Describes the same IC state machine applied to CPython bytecodes, with the specific implementation detail that specialization is done by in-place bytecode rewriting (quickening) rather than a separate cache object.

3. **"V8: A Tale of Two Compilers"** by Andy Wingo (wingolog.org) — an older but historically important post describing how V8 originally combined a full-codegen baseline compiler with Crankshaft, using ICs as the bridge between them. Establishes the vocabulary of *type feedback* and *on-stack replacement* that the Shapes post builds on.
