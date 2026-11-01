---
title: "QuickCheck: A Lightweight Tool for Random Testing of Haskell Programs"
source: https://dl.acm.org/doi/10.1145/351240.351266
author: Koen Claessen and John Hughes
company: Chalmers University of Technology
date_posted: 2000
date_digested: 2026-11-01
---

# QuickCheck: A Lightweight Tool for Random Testing of Haskell Programs

## What's new to learn

- **Property specifications as test oracles**: instead of writing individual input→expected-output test cases, you write *algebraic invariants* that must hold for all inputs — and let the framework find counterexamples automatically.
- **The `Gen` monad / generators**: a composable, type-directed vocabulary for producing arbitrary random values, including deeply nested structures, where each generator can be derived from smaller ones.
- **Automatic shrinking**: when a counterexample is found, the framework minimizes it by systematically trying smaller inputs until it reaches the simplest case that still fails — converting a 500-element list into a 2-element one.

## Prerequisites

No Haskell knowledge required — the concepts translate directly to Python (`hypothesis`), Scala (`scalacheck`), JavaScript (`fast-check`), Erlang (`PropEr`), Rust (`proptest`), and others. Helpful background: unit testing basics, the idea of a "test oracle" (something you can compare output against). The deeper insight (why shrinking is hard without types) benefits from familiarity with algebraic data types, but the mechanism is understandable without it.

## The core idea

Every unit test you write is secretly asserting a *specific instance* of a more general claim. When you write `assert sort([3,1,2]) == [1,2,3]`, you're really asserting that `sort` is correct — but you only checked one input. QuickCheck inverts this: you state the general claim directly as a *property*, and the library picks hundreds of inputs to test it against.

For sorting, the properties are:

- **Length preservation**: `len(sort(xs)) == len(xs)` for all lists `xs`
- **Ordering**: `sort(xs)[i] <= sort(xs)[i+1]` for all valid `i`
- **Permutation**: `Counter(sort(xs)) == Counter(xs)` (same elements, possibly different order)

No single property is sufficient, but together they nail the semantics precisely. This is **specification-by-invariant** rather than specification-by-example.

The framework then:
1. Generates random inputs of the right type.
2. Checks whether the property holds.
3. If it fails, *shrinks* the input to the smallest case that still fails.
4. Reports that minimal counterexample to the developer.

The shift in mindset: you move from "does this output match what I typed in?" to "what is *always true* about this function?" — which forces you to think clearly about the contract.

## Mechanics

### The `Arbitrary` typeclass (generation)

QuickCheck uses a typeclass `Arbitrary a` to generate random values of type `a`:

```haskell
class Arbitrary a where
  arbitrary :: Gen a
  shrink    :: a -> [a]   -- smaller values to try
```

`Gen` is a monad over a pseudorandom seed + a size parameter. The framework grows the size parameter over successive test runs, so early tests are small (fast to evaluate, easy to understand failures) and later tests are large (more coverage).

Generators compose: if you can generate an `Int` and a `Bool`, you can generate `(Int, Bool)` for free. This compositionality is what makes it practical — you define generators for leaves of your data type and derive the rest.

```haskell
-- Generate a tree of integers
instance Arbitrary a => Arbitrary (Tree a) where
  arbitrary = sized $ \n ->
    if n == 0
      then Leaf <$> arbitrary
      else frequency
        [ (1, Leaf <$> arbitrary)
        , (3, Node <$> resize (n `div` 2) arbitrary
                   <*> resize (n `div` 2) arbitrary) ]
```

The `sized`/`resize` combinators prevent infinite recursion and tune the depth distribution.

### Running properties

A property is just a function `(input) -> Bool` (or the richer `Property` type). The framework calls `quickCheck`:

```haskell
prop_reverse_involution :: [Int] -> Bool
prop_reverse_involution xs = reverse (reverse xs) == xs

main = quickCheck prop_reverse_involution
-- +++ OK, passed 100 tests.
```

If a test fails:

```
*** Failed! Falsifiable (after 23 tests and 3 shrinks):
[0, -1]
```

### Shrinking: the key insight

When a counterexample is found, naïve property-based testing stops. QuickCheck does more: it calls `shrink` on the failing input to produce a list of "smaller" candidate inputs, and tries each. If any still fails, it recursively shrinks *that* input. The result is the *local minimum* — the smallest input the shrinking procedure can reach.

Concrete example: suppose a graph algorithm fails on a random graph with 200 nodes and 1,500 edges. Shrinking:
- Removes one edge at a time; if the property still fails, keeps the removal.
- Removes one node (and its edges) at a time.
- Eventually produces: a 3-node graph with 2 specific edges.

That's the counterexample the developer sees — immediately debuggable.

**Why shrinking needs to be type-aware**: a string `"abc"` can be shrunk by dropping characters, by reducing character values, by swapping letters. A list of structs can be shrunk by removing elements or shrinking each element. Each type has a different notion of "smaller", and `shrink :: a -> [a]` encodes exactly that for each type. Generic shrinking (without knowing the type) can do *something* but produces much larger minimal counterexamples.

### Conditional properties

Sometimes you only care about inputs satisfying a precondition:

```haskell
-- Division only defined for non-zero divisor
prop_div :: Int -> Int -> Property
prop_div x y = y /= 0 ==> x `div` y * y + x `mod` y == x
```

The `==>` combinator discards tests where the precondition is false. If too many are discarded (> 99%), QuickCheck reports a coverage failure — a hint that your generator is too broad relative to your precondition, and you should write a more focused generator instead.

### Model-based / stateful testing

The original paper also describes **stateful testing**: model a system's state as an abstract type (a simple reference implementation), generate random command sequences (Insert, Delete, Lookup), run them on both the real implementation and the model, and assert that observable outputs match. This is the pattern later used by:
- Jepsen (Clojure) for linearizability testing of distributed databases.
- `hypothesis.stateful` for testing Python stateful APIs.
- Erlang PropEr for testing OTP GenServers.

The model is purposely simple — a pure dict, a set, an ordered list — because it only needs to be *obviously correct*, not efficient.

## Where it breaks

**The oracle problem**: you need to be able to specify what "correct" means. For compression, you can check round-tripping (`decompress(compress(xs)) == xs`) but not output quality. For LLM outputs, there's no algebraic oracle. QuickCheck excels at functions with precise mathematical contracts and degrades toward ordinary testing as the specification becomes harder to formalize.

**Generator coverage gaps**: a naive `arbitrary :: [Int]` mostly generates medium-length lists of small integers. Important edge cases (empty list, singleton, duplicates, sorted, reverse-sorted) may occur with probability ≈ 0. Good QuickCheck practice combines random generation with *explicit* boundary cases using `Gen.oneof` or custom generators. Hypothesis (Python) addresses this with "covering" strategies and "data-driven" generation.

**Slow properties**: if checking one property invocation takes 100ms, running 10,000 tests takes 17 minutes. Properties must be cheap. This rules out end-to-end integration tests — PBT works best at the unit or small integration level.

**Shrinking can miss the global minimum**: `shrink` returns *candidate* smaller inputs; the framework picks among them greedily. It can get stuck in a local minimum that's still large. Integrated shrinking (as in Hedgehog) builds the shrink tree into the generator, guaranteeing the global minimum.

**Model drift**: in stateful testing, the model and implementation can drift as the implementation evolves. Keeping the model accurate is a maintenance burden, and a wrong model means wrong tests passing.

## Why it works

**Bugs cluster near boundaries, and random generation finds the unexpected boundary.** Human-authored test cases unconsciously avoid weird inputs. Random generation has no such bias — it finds the empty list, the all-equal list, the overflow-causing integer, the Unicode edge case. The generator's job is to sample the *full* input space, weighted toward realistic inputs.

**Shrinking converts noise into signal.** A 200-element failing input is nearly undebuggable. A 3-element failing input is immediately obvious. Shrinking is a gradient descent on input size (with "size" defined by the type's `shrink` function), and it reliably reaches a local minimum in O(log n) steps because most structural reductions that preserve the failure cut size by a constant factor.

**Algebraic properties are stable specifications.** Individual test cases encode both the *input* and the *expected output*, meaning they must be updated every time the output format changes — even if the logic is unchanged. Properties like "sort is idempotent" or "encode ∘ decode = id" are format-independent. They express *what the function does* rather than *what it returned on Tuesday*.

**The deeper principle — specification as a generative grammar**: writing `Arbitrary` instances for your domain types is isomorphic to writing a *grammar* for valid inputs. The generator defines the "language" of acceptable inputs. Property-based testing is then: sample many strings from this grammar, check that the program accepts them correctly. This is the same insight behind **fuzzing** (AFL: grammars for protocol inputs), **model checking** (TLA+: state space as a grammar), and **abstract interpretation** (program analysis over abstract domains). All are forms of "enumerate the plausible input space systematically".

The core distinction from coverage-guided fuzzing (AFL): AFL's fitness function is *structural coverage* (new code paths), which is a proxy for "interesting inputs". QuickCheck's fitness function is *property violation* (a Boolean), which is exact. AFL excels at finding crashes in programs without a spec; QuickCheck excels at finding logic errors in programs with a spec. They are complementary.

## Going deeper

1. **John Hughes, "Don't Write Tests" (2019 talk)** — Hughes demonstrates property-based testing on a model-based state machine for Erlang process communication, showing how 1,000 lines of unit tests can be replaced by 50 lines of properties that find more bugs. Available on YouTube: search "John Hughes Don't Write Tests".

2. **David MacIver, "What Every Computer Scientist Should Know About Property-Based Testing" (Hypothesis docs)** — https://hypothesis.readthedocs.io/en/latest/details.html — explains Hypothesis's integrated shrinking, the swarm testing technique, and how the "database of recent examples" achieves deterministic replay in CI.

3. **"Finding More Bugs Faster" — Testing Erlang/OTP with PropEr** — the classic Erlang experience report on finding a subtle race condition in `ets` (Erlang's in-process hash table) by using PBT with a model that a 600-test suite had never triggered. Demonstrates stateful PBT at its most powerful.
