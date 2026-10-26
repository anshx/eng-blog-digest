---
title: "Damas-Hindley-Milner inference two ways"
source: https://bernsteinbear.com/blog/type-inference/
author: Max Bernstein, River Dillon Keefer
company: Personal blog (Scrapscript project)
date_posted: 2024-10-15
date_digested: 2026-10-26
---

# Damas-Hindley-Milner inference two ways

## What's new to learn

1. **Unification (Robinson's algorithm)**: given two type expressions t₁ and t₂, find the most general substitution θ such that θ(t₁) = θ(t₂). Every type variable is an unknown; every type expression is a term; solving the system is unification — the same algorithm Prolog uses for logic programming.
2. **Let-polymorphism (generalization)**: quantifying free type variables at `let`-bindings is what makes `id : ∀α. α → α` work for both `id 1` and `id "hello"` in the same program. It is the mechanism that separates ML-family type systems from simple monomorphic inference.
3. **Algorithm W vs Algorithm J**: two equivalent formulations. W threads a global substitution through the AST as it recurses; J first collects all type-equality constraints as a list, then solves them in one pass with union-find. Same expressiveness, different architecture — and J's separation is the reason modern compilers produce better error messages.

## Prerequisites

- Familiarity with generic / parametric types in any language (TypeScript `<T>`, Rust `fn id<T>(x: T) -> T`, Haskell `id :: a -> a`)
- Intuition for what "this function works on any type" means at the use site
- No requirement for category theory, formal logic, or previous study of type theory

## The core idea

Every modern statically-typed language — TypeScript, Rust, Swift, Kotlin, Scala, OCaml, Haskell — infers types without requiring you to write them down. How? The key insight from Damas & Milner (1982) is that **type inference is a two-phase constraint-solving problem**:

1. **Constraint generation**: walk the AST and assign a fresh type variable (α, β, γ, …) to every subexpression whose type is unknown. For each language construct, emit one or more type equations that must hold. You don't solve anything yet; you just accumulate equations.

2. **Constraint solving (unification)**: take the pile of equations and find a substitution — a mapping from type variables to type expressions — that satisfies all of them simultaneously. If such a substitution exists, every variable is resolved and inference succeeds. If not, the program has a type error.

A worked example with `let f = λx. x + 1`:

```
AST:   let f = (λx. (x + 1))

Assign type variables:
  x  : α
  1  : Int              (literal)
  +  : Int → Int → Int  (built-in)
  (x + 1): β
  λx. (x + 1): γ

Emit constraints:
  x has type α, and x is an argument to +, so:   α = Int
  result of + is Int, so:                          β = Int
  λx.(x+1) is a function from α to β:             γ = α → β

Unify:
  α = Int                → substitute α ↦ Int
  γ = α → β = Int → Int

Conclusion: f : Int → Int
```

The insight: we never "guessed" any types. We assigned unknowns and let equations do the work.

## Mechanics

### The type language

```
τ ::= Int | Bool | Str   -- base types
    | α                  -- type variable
    | τ → τ              -- function type
    | List τ             -- type constructor

σ ::= τ                  -- monotype (no quantifiers)
    | ∀α. σ             -- type scheme (polymorphic)
```

A *context* Γ maps variable names to type schemes: `Γ = {id: ∀α. α → α, succ: Int → Int, …}`.

### Robinson unification

The unification algorithm takes two types and either returns a *substitution* (a mapping `{α ↦ τ, β ↦ τ', …}`) or raises a type error:

```
unify(t₁, t₂):
  if t₁ == t₂:
      return {}                     # trivially equal, empty substitution

  if t₁ is type variable α:
      occurs_check(α, t₂)           # prevent infinite types like α = List α
      return {α ↦ t₂}

  if t₂ is type variable β:
      occurs_check(β, t₁)
      return {β ↦ t₁}

  if t₁ = C(a₁,…,aₙ) and t₂ = C(b₁,…,bₙ) (same constructor, same arity):
      s = {}
      for i in 1..n:
          s = compose(s, unify(apply(s, aᵢ), apply(s, bᵢ)))
      return s

  else:
      raise TypeError("cannot unify {} with {}".format(t₁, t₂))
```

The **occurs check** (`α ∉ free_vars(τ)`) prevents the unifier from producing `α = List α`, which would mean an infinite type. Most production implementations skip it for performance (OCaml, Standard ML), but it catches real bugs in practice.

Unification is **O(n α(n))** with union-find (nearly linear, where α is the inverse Ackermann function). With a naive substitution representation it is O(n²), and with term DAGs it is O(n log n).

### Algorithm W

W traverses the AST and returns both a type and a substitution. It *threads the substitution* — each step applies previous results before emitting new equations:

```python
def infer_W(ctx, expr):
    match expr:
        case Var(name):
            scheme = ctx[name]
            return (instantiate(scheme), {})   # fresh vars for each ∀-bound var

        case Lam(param, body):
            alpha = fresh()                    # type of param
            ctx2 = ctx | {param: alpha}
            (tau, s1) = infer_W(ctx2, body)
            return (apply(s1, alpha) → tau, s1)

        case App(f, arg):
            (tau_f, s1) = infer_W(ctx, f)
            (tau_a, s2) = infer_W(apply(s1, ctx), arg)
            beta = fresh()                     # return type
            s3 = unify(apply(s2, tau_f), tau_a → beta)
            return (apply(s3, beta), compose(s3, s2, s1))

        case Let(name, rhs, body):
            (tau_rhs, s1) = infer_W(ctx, rhs)
            scheme = generalize(apply(s1, ctx), apply(s1, tau_rhs))
            ctx2 = apply(s1, ctx) | {name: scheme}
            (tau_body, s2) = infer_W(ctx2, body)
            return (tau_body, compose(s2, s1))
```

**Instantiation** replaces every ∀-bound variable with a distinct fresh variable: `instantiate(∀α. α → α)` might return `β → β` for fresh β.

**Generalization** quantifies over type variables that are free in the inferred type but not free in the current context (otherwise we'd overgeneralize across scope boundaries):

```python
def generalize(ctx, tau):
    free_in_tau = free_vars(tau)
    free_in_ctx = union(free_vars(s) for s in ctx.values())
    to_quantify = free_in_tau - free_in_ctx
    return ForAll(to_quantify, tau)
```

This is why `let id = λx. x` gives `id : ∀α. α → α` but a function parameter `f` with unknown type stays as a monotype — it's free in the surrounding context.

### Algorithm J (constraint-based)

J separates constraint *collection* from constraint *solving*. Instead of threading substitutions, it simply accumulates equations in a mutable list:

```python
equations = []

def infer_J(ctx, expr):
    match expr:
        case Var(name):
            return instantiate(ctx[name])

        case Lam(param, body):
            alpha = fresh()
            ctx2 = ctx | {param: alpha}
            tau_body = infer_J(ctx2, body)
            return alpha → tau_body

        case App(f, arg):
            tau_f = infer_J(ctx, f)
            tau_a = infer_J(ctx, arg)
            beta = fresh()
            equations.append((tau_f, tau_a → beta))   # defer unification
            return beta

        case Let(name, rhs, body):
            tau_rhs = infer_J(ctx, rhs)
            # Must solve equations for rhs before generalizing
            solve(equations)
            scheme = generalize(apply_current_subst(ctx),
                                apply_current_subst(tau_rhs))
            ctx2 = ctx | {name: scheme}
            return infer_J(ctx2, body)

# After traversal:
def solve(equations):
    for (t1, t2) in equations:
        s = unify(apply_current_subst(t1), apply_current_subst(t2))
        update_global_substitution(s)
```

The key difference: J uses a **union-find** data structure instead of explicit substitution composition. Each type variable is a node; unification is `union(α, τ)`. Finding the current type of any variable is `find(α)`, which traverses the union-find path. This makes J faster in practice (union-find is nearly O(1) per operation vs. O(n) for applying a substitution chain in W).

### Algorithm M (top-down)

The post also mentions Algorithm M (Lee & Yi, 1998), which propagates type constraints top-down rather than bottom-up. Where W infers the type of a subexpression and then checks it against context, M passes an *expected type* downward and fails earlier. M detects type errors at the outermost failing expression rather than at the innermost — which is why error messages in some languages feel "further from the real mistake" than in others.

## Where it breaks

**1. Subtyping**: HM requires structural equality, not subtype compatibility. A function expecting `{x: Int}` cannot accept `{x: Int, y: Bool}` in pure HM — there is no record width subtyping. TypeScript adds a separate constraint-solving layer (with `extends`, conditional types, template literal types) built on top of HM-style inference, which is why TypeScript errors can feel completely different from OCaml's.

**2. Rank-2 and higher polymorphism**: HM cannot infer types where a `∀` appears inside a function argument, e.g. `(∀α. α → α) → Int`. Consider `applyToInt :: (∀α. α → α) → Int; applyToInt id = id 42`. The `id` in `applyToInt id` must itself be polymorphic — a "rank-2" type. GHC's `RankNTypes` extension handles this but requires explicit annotations at the higher-rank position.

**3. The value restriction**: in a language with mutable references, generalization can go wrong. `let r = ref []` should not have type `∀α. ref (list α)`, because after `r := [1]` the same reference cannot safely hold `"hello"`. Standard ML restricts generalization to *syntactic values* (expressions that cannot cause side effects). Rust sidesteps this entirely: lifetimes and ownership ensure mutable references are never polymorphic in the same way.

**4. Exponential worst-case**: certain deeply nested let-expressions have exponential inference complexity (the famous "let-in" blowup). In practice, this is never triggered in real programs and is EXPTIME only for contrived adversarial inputs.

**5. Module-level polymorphism vs. expression-level**: HM operates at the expression level. Haskell adds type classes, which require solving a separate *dictionary-passing* constraint system on top. Rust adds traits with the same semantics but a different solver architecture. Both require explicit `impl` declarations that HM's pure inference cannot derive.

## Why it works

Type inference in HM works because the type language is a *term algebra*: every type is a finite tree, type variables are leaves, and constructors (→, List, etc.) are internal nodes. The type constraint system is first-order equational logic over this tree algebra, and Robinson's algorithm is a complete, terminating decision procedure for it.

The **deepest principle**: type inference is the same problem as unification in logic programming. Prolog's resolution algorithm and Algorithm W both reduce to Robinson unification over first-order terms. Every Prolog query is a type inference problem in disguise, and every type error is an unsatisfiable system of first-order equations. The only reason HM terminates in polynomial time while Prolog can loop is that HM's types are *simple* terms (no function symbols that could recurse indefinitely), and the occurs check prevents self-reference.

Connections to the archive:

- **CDCL SAT (Oct 22)**: CDCL solves propositional constraints; HM solves first-order term constraints. HM is lucky — term unification is polynomial, while propositional satisfiability is NP-complete. The difference is structural: term unification is a single-pass unification over trees, while SAT requires backtracking over Boolean combinations.
- **Cascades query optimizer (Sep 5)**: the optimizer's `Memo` deduplicates equivalent plan fragments. W's context deduplicates polymorphic type schemes. Both memoize to avoid redundant recomputation of equivalent structures.
- **Differential dataflow / incremental view maintenance (Jul 3)**: Datalog's bottom-up fixed-point evaluation is equivalent to constraint propagation in type inference. Algorithm J's constraint collection phase is producing a Datalog-style set of facts; the solve phase is a fixed-point evaluation over those facts.
- **Union-find**: the data structure powering Algorithm J's constraint solver appears across the archive — in consistent hashing (ring traversal), in Dynamo's anti-entropy (connected components), and in Kruskal's MST. Its O(α(n)) amortized complexity makes it uniquely suited as the substrate for type inference.

## Going deeper

1. **Damas & Milner, "Principal type schemes for functional programs" (POPL 1982)** — the original proof that Algorithm W is sound (infers only valid types), complete (infers a type whenever one exists), and produces the *principal* type (most general: every other valid type is an instance of it).
2. **Pierce, "Types and Programming Languages" (TAPL), Ch. 22–23** — the definitive textbook treatment of type reconstruction, constraint-based typing, and the formal proof of HM correctness.
3. **The Rust Reference, "Type Inference"** — Rust extends HM with trait bounds (which become additional constraints `T: Trait`) and lifetime constraints (which are a second unification problem over *regions*). This page plus Niko Matsakis's blog (nikomatsakis.github.io) shows how HM scales to a systems language.
