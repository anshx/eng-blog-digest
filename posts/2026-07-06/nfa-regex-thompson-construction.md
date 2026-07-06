---
title: "Regular Expression Matching Can Be Simple And Fast"
source: https://swtch.com/~rsc/regexp/regexp1.html
author: Russ Cox
company: Personal (Cox was at Google)
date_posted: 2007-01-01
date_digested: 2026-07-06
---

# Regular Expression Matching Can Be Simple And Fast

## What's new to learn

- **Thompson NFA simulation**: A regex can be compiled to an NFA and matched in O(n × m) time — where n is the pattern length and m the input length — by tracking the *set* of all active states at once, rather than committing to a single path.
- **Backtracking's exponential trap**: Most popular engines (Perl, Python, Java, Ruby) use depth-first backtracking, which can take O(2^n) time on inputs that look completely ordinary until you hit n ≈ 25.
- **Lazy DFA = NFA simulation**: The set of NFA states you track during simulation *is* a DFA state (subset construction); NFA simulation and lazy DFA construction are the same algorithm in two guises.

## Prerequisites

- What a finite state machine is (states, transitions, accept states)
- Basic regex syntax: literals, `.`, `*`, `?`, `|`, `()`
- O-notation; no formal automata theory required

## The core idea

Most people think of regex matching as: try the pattern from the start of the input; if it fails at some choice point, backtrack and try the other branch. This is correct but can fail dramatically.

Ken Thompson's 1968 insight: you do not need to commit to one path. At every position in the input you are potentially in *multiple* states of the automaton simultaneously. Instead of tracking "which state am I in now?", track "which states *could* I be in now?". For each input character you compute:

```
next_set = ε_closure({ t | s ∈ current_set, s →(char) t })
```

After reading all m characters, check whether the accept state is in the set. The set has at most n members (n = number of NFA states ≈ pattern length), so each step does O(n) work and the whole match costs O(n × m). The branching structure of the pattern never causes re-work — all branches are explored together, in parallel, with a set.

## Mechanics

### Step 1 — Thompson's construction (regex → NFA)

Build an NFA fragment for each primitive, then compose them:

| Regex | Fragment |
|-------|----------|
| literal `a` | single state with a labeled transition on `a` |
| `e1 e2` | connect e1's out-edge to e2's start with an ε-transition |
| `e1\|e2` | new start → ε → e1-start and e2-start; both out-edges → ε → new accept |
| `e*` | new start → ε → e-start or accept; e-out → ε → e-start or accept |
| `e?` | new start → ε → e-start or accept; e-out → ε → accept |

The result has at most 2n NFA states and 3n transitions, all of which are either single-character transitions or ε-transitions (no labels).

### Step 2 — ε-closure

`ε_closure(S)` = all states reachable from S via any number of ε-transitions. This is a simple BFS/DFS over the ε-edges and runs in O(n).

### Step 3 — Simulation loop

```python
current = ε_closure({start})
for char in input_string:
    nexts = set()
    for state in current:
        if state has transition on char to t:
            nexts.add(t)
    current = ε_closure(nexts)
return accept_state in current
```

**Time**: O(m × n). **Space**: O(n) for the two state sets. No recursion, no stack depth limit, no backtracking.

### The benchmark that makes this concrete

Pattern: `(a?){n}(a){n}` — n repetitions of `a?` followed by n repetitions of `a`  
Input: `a{n}` — n copies of `a`

The only way to match is for all `a?` patterns to skip (match zero characters), then all `a` patterns to each consume one `a`. A backtracking engine tries every other assignment before landing here, exploring up to 2^n combinations.

Observed timings (Cox's original benchmarks, circa 2007):

| n  | Perl (backtracking) | grep / awk (NFA/DFA) |
|----|---------------------|----------------------|
| 23 | ~12 s               | <1 ms                |
| 25 | ~40 s               | <1 ms                |
| 29 | >60 s               | <1 ms                |

Perl's curve is exponential; grep/awk's curve is flat. Same pattern. Same input. A factor of >10^8 difference at n=29.

### Why backtracking hits 2^n here

For input `a^n` and pattern `(a?){n}(a){n}`, the backtracker must decide for each `a?`: "should I consume this character or skip?" It tries consume first; when the `(a){n}` suffix eventually fails, it backtracks and tries skip for the last `a?`; when that fails, it backtracks further; and so on, until it has explored every 2^n subset of `a?` decisions. Only the all-skip assignment succeeds.

The NFA simulation never backtracks. At each input character it holds the set of ALL states any assignment could have produced at that point. The information about all 2^n paths is compressed into a single set of ≤n states.

## Where it breaks

**Backreferences make the problem NP-complete.** The pattern `(a+)\1` — "some a's then the *same* a's again" — cannot be compiled to a finite automaton because the accept condition depends on what was captured. Perl/Python/Java support backreferences and therefore *cannot* use Thompson NFA for those patterns. This is not a design flaw in those engines; it is a fundamental limit. The Thompson approach applies only to "pure" regular languages (no backrefs, no lookaround).

**Submatch extraction is harder to implement.** Reporting *which* positions `(...)` groups matched requires augmenting each NFA state with capture metadata; the state set grows proportionally. Cox's subsequent articles (regexp2–regexp4) cover this.

**DFA state explosion.** The related approach — precomputing the full DFA by running subset construction up front — can produce exponentially many DFA states for certain patterns (e.g., `(a|aa)*b`). Lazy DFA construction (build DFA states on demand) is the practical solution used by grep; it keeps memory bounded while still delivering O(m) per-character cost after the first run.

**Look-around assertions.** `(?=...)` and `(?<=...)` require matching in multiple directions and don't fit cleanly into the forward-scanning NFA model.

## Why it works

The deepest principle here is **BFS vs DFS for existential path queries** — and it applies far beyond regex.

A backtracking regex engine asks: "does *this specific* assignment of choices lead to a match?" and re-runs the machine from scratch for each. It is DFS with branching factor b at depth d: O(b^d) worst case.

NFA simulation asks: "does *any* assignment lead to a match?" and computes the set of all reachable states simultaneously. It is BFS on the NFA graph: O(|states| × |input|) worst case.

The same choice separates exponential algorithms from polynomial ones across CS:

- **Dynamic programming vs naïve recursion**: DP computes a table that represents ALL subproblem outcomes at once (reachable cost set) instead of recomputing each path.
- **Dijkstra's algorithm vs DFS shortest-path**: Dijkstra maintains a frontier of all tentatively-reached nodes; blind DFS would re-visit exponentially many paths.
- **Model checking with BDDs**: symbolic model checkers represent entire sets of reachable states as Boolean decision diagrams, avoiding explicit enumeration.
- **Set-based type inference** (Hindley-Milner unification): instead of trying assignments one by one, unification propagates constraints across all unknowns simultaneously.

Even more precisely: the set of NFA states reachable after reading prefix p is the DFA state that subset construction would produce for p. NFA simulation IS lazy DFA construction — you build only the DFA states you actually visit during the match. Thompson NFA and "on-the-fly DFA" are two names for the same algorithm.

The takeaway: whenever you find yourself writing a loop that tries one option, backtracks, and retries, ask whether a set-based formulation exists. If the state space is finite and bounded (as an NFA's is), you can often turn an exponential search into a polynomial sweep.

## Going deeper

1. **The full series** — Russ Cox, swtch.com/~rsc/regexp/: regexp2 covers submatch tracking (tagged NFA states), regexp3 covers running the engine in parallel with string search, regexp4 covers the lazy DFA with caching and the trigram-index trick grep uses for whole-file search.

2. **RE2** (github.com/google/re2) — the production library Cox built from these ideas, now powering Go's `regexp` package and Google RE2. The source is ~5 kloc of annotated C++ that is unusually readable as algorithm code.

3. **"Parsing with Derivatives"** (Owens, Reppy, Turon, 2009; JFP 2011) — an alternative to Thompson construction that re-derives the classic NFA simulation from the Brzozowski derivative of a regex, connecting automata theory to functional-programming concepts and showing how the "set of states" insight reappears in a purely algebraic form.
