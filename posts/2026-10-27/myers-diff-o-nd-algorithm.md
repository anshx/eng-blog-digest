---
title: "An O(ND) Difference Algorithm and Its Variations"
source: https://link.springer.com/article/10.1007/BF01840446
author: Eugene W. Myers
company: University of Arizona
date_posted: 1986
date_digested: 2026-10-27
---

# An O(ND) Difference Algorithm and Its Variations

## What's new to learn

- **Edit graph formalism**: Any sequence-comparison problem (diff, LCS, edit distance, sequence alignment) is shortest-path search on a grid DAG—a reframing that collapses a family of dynamic programming problems into one graph problem.
- **k-diagonal / furthest-reaching parameterization**: The key to sub-quadratic diffing is the observation that a D-path (a path with exactly D edit operations) can only end on O(D) diagonals, so you need O(D) state rather than O(MN)—a collapsed state space that makes the algorithm run in O(ND) time instead of O(MN).
- **Middle-snake divide and conquer**: The linear-space variant that git actually uses shows how bidirectional BFS from both ends of the edit graph can find the "middle snake" of an optimal path, enabling a space-optimal O(ND) divide-and-conquer without extra time cost.

## Prerequisites

- What a diff is: given texts A (M lines) and B (N lines), produce a minimum edit script (sequence of insertions and deletions) that transforms A into B.
- Longest Common Subsequence (LCS)—the dual: LCS length = (M + N − D) / 2, so maximizing common subsequence length is the same problem as minimizing edit distance.
- BFS on an unweighted graph: visit all nodes at distance d before any node at distance d+1.
- Why dynamic programming applies here: optimal sub-path property holds on a DAG.

## The core idea

The central insight is to recast "compute a minimal diff" as "find the shortest path through a grid."

Build an (M+1) × (N+1) grid of positions. Position (x, y) means "consumed x elements of A and y elements of B." Three kinds of moves:
- **Horizontal** (x, y) → (x+1, y): delete A[x] (costs 1 edit)
- **Vertical** (x, y) → (x, y+1): insert B[y] (costs 1 edit)
- **Diagonal** (x, y) → (x+1, y+1): only when A[x] == B[y] (costs 0—a free match)

The minimum edit script is the minimum-cost path from (0, 0) to (M, N). Diagonal edges are free, so the algorithm always follows them greedily ("snake" extensions). Total edit distance D = number of non-diagonal edges on the shortest path.

This reframing gives us BFS as a mechanism: explore all 0-cost paths first (snakes), then find all paths with 1 edit, then 2, and so on. The question is how to do that BFS without visiting all O(MN) cells.

Myers's answer is a collapsed state space. Define the **diagonal number** of a position as k = x − y. Every position (x, y) on diagonal k is uniquely identified by just x (since y = x − k). A path with D edits can only end on diagonals where k and D have the same parity (each horizontal step increases k by 1, each vertical decreases it by 1, so after D steps the parity of k equals the parity of D). That means only D+1 distinct diagonals are reachable after exactly D edits, regardless of M and N. So instead of tracking O(MN) states, the algorithm tracks one integer per diagonal—the **furthest-reaching x** on that diagonal for each value of D.

## Mechanics

**Algorithm 1 (O(ND) time and space)**

Maintain an array `V[−(N+M) .. (N+M)]` where `V[k]` is the furthest x reached on diagonal k.

```
V[1] = 0
for d = 0, 1, 2, ..., D_max:
    for k = −d to +d (step 2):
        # Choose which direction to take into diagonal k:
        if k == −d or (k != d and V[k−1] < V[k+1]):
            x = V[k+1]          # came from diagonal k+1 via a vertical (insert)
        else:
            x = V[k−1] + 1      # came from diagonal k−1 via a horizontal (delete)
        y = x − k
        # Extend diagonally as far as possible (the "snake"):
        while x < M and y < N and A[x] == B[y]:
            x += 1; y += 1
        V[k] = x
        if x == M and y == N:
            # Found! Minimum edit distance is d, recover the path.
            return d
```

The key invariants:
- `V[k]` is always the maximum x reachable on diagonal k using exactly d edits so far.
- The "snake" while-loop extends greedily at zero cost—it's always safe to match as many characters as possible.
- At each step d, we use only `V` values from step d−1 to compute step d. (Store all versions of V if you need to recover the edit script, not just the distance.)

**Path recovery** requires storing `V` for every value of d (so O(ND) space). To produce the actual edit script, trace backward from (M, N): find which diagonal you came from at each step.

**The linear-space variant (Algorithm 2—what git uses)**

The quadratic space of storing all V arrays for traceback is often prohibitive. Myers's linear-space variant uses bidirectional BFS and divide-and-conquer:

1. Run the forward BFS from (0, 0) and the backward BFS from (M, N) simultaneously for d/2 steps.
2. When the two search frontiers overlap, you've found the **"middle snake"**—the maximal diagonal run (sequence of free matches) that falls in the middle of the optimal path.
3. The middle snake splits the original problem into two independent subproblems: diff A[0..x_mid] vs B[0..y_mid] and diff A[x_mid..M] vs B[y_mid..N].
4. Recurse on each half.

The recursion tree has O(log min(M, N)) depth, each level uses O(M + N) space, but the space is reused across levels (not stored simultaneously). Total space: O(M + N). Total time: O(ND) (same as Algorithm 1, since each level of recursion does O(ND') work for its subproblem D').

**Parity constraint** (often surprising): The algorithm only examines diagonals where k ≡ d (mod 2). This halves the work at each step. It follows from: a d-path starts at (0,0) (k=0, even) and each edit changes k by ±1, so after d edits, k has the same parity as d. This cuts the constant factor in half.

**Concrete example**:

Compare A = `["a", "b", "c", "d"]` vs B = `["a", "c", "b", "d"]`:
- D=0: (0,0)→diagonal 0, snake extends to (1,1) ("a"="a"). V[0]=1.
- D=1: from diagonal 0 move right to k=1, x=V[0]+1=2, y=1, snake: A[2]="c", B[1]="c"→match, extend to (3,2). V[1]=3. OR move down to k=-1, x=V[0]=1, y=2, A[1]="b"≠B[2]="b"→match! extend to (2,3). V[-1]=2.
- D=2: from k=1, move right to k=2: x=4, y=2, snake: A[4]→out of bounds, stop at (4,2). Check x==M? 4==4, y==N? 2≠4. Not done. Move left to k=0: take max of V[k-1]=2 and V[k+1]+1=3, so x=3, y=3, snake: A[3]="d"=B[3]="d"→(4,4). x==M and y==N! Done. D=2.

The minimum edit script has 2 operations: delete "b" from position 2, insert "b" at position 3 (swap "b" and "c").

**Scale numbers**: On typical source code (mostly similar), Myers runs significantly faster than O(MN) because D is small. Git uses Myers by default and falls back to a histogram/patience diff for semantically clearer results in certain patterns.

## Where it breaks

**Worst case is quadratic**: If A and B share no characters, D = M + N, and the algorithm runs in O((M+N)²) time. In practice this is rare for text files, but it's why git has a `--diff-algorithm=histogram` option for large or pathological diffs.

**Equal edit costs only**: Myers assumes every insertion or deletion costs exactly 1. For weighted edit distance (e.g., transposition costs 0.5, some characters cost more to substitute), you need the full O(MN) Needleman-Wunsch dynamic programming. Myers is not a drop-in for those cases.

**No semantic awareness**: Myers finds a *syntactically* minimal diff, not a semantically natural one. Moving a function to a different file, renaming a variable, or reordering struct fields will show as a large deletion + large insertion rather than a rename. Patience diff (also in git) reduces this by anchoring on unique lines, but still can't understand the programmer's intent.

**Space is still O(ND) for traceback without the linear variant**: The simple algorithm needs to save all V arrays to produce the actual edit script. For very large files with many differences, this can dominate memory. The linear-space variant avoids this.

**LCS variant isn't always Myers diff**: Some tools use "patience LCS" or "histogram LCS" to prefer diffs that humans find more readable. Git ships all three (`myers`, `patience`, `histogram`) precisely because no algorithm is universally best for human-readable output.

## Why it works

The deeper principle is: **optimal subproblems over DAGs admit BFS, and BFS over structured graphs admits state-space collapse.**

The edit graph is a DAG with a specific structure—diagonal edges are free and only go forward. This means:
1. The optimal path has the substructure needed for DP/BFS: a prefix of an optimal path is itself optimal.
2. The parity constraint means only half the diagonals are reachable at each distance d.
3. The greedy snake extension is provably safe: since diagonal moves are free, taking them eagerly can only improve or maintain optimality, never worsen it.

The k-diagonal parameterization is the key insight that separates Myers from naïve BFS. Instead of tracking "where exactly have I been?" as an O(MN) bitmap, Myers tracks "on each diagonal, how far right have I gotten?" as a length-O(D) array. This is possible only because the cost structure (0 for diagonal, 1 for horizontal/vertical) makes the optimal path's diagonal structure predictable.

This "find the right state space for your graph's structure" principle is the same insight in:
- **Dijkstra vs BFS**: BFS is Dijkstra for unit-weight graphs—when costs are uniform, distance alone identifies the state.
- **Bellman-Ford vs DAG DP**: On a DAG, processing in topological order eliminates the need for relaxation loops.
- **HNSW in vector search**: The multi-level shortcut graph reduces O(N) scan to O(log N) by collapsing the search state space.
- **Bioinformatics alignment (Smith-Waterman / Needleman-Wunsch)**: Use the same grid DP structure, extending Myers to affine gap costs and local alignment.

The middle-snake linear-space refinement is an instance of the **bidirectional BFS / meet-in-the-middle** pattern, where searching from both ends cuts the exponential (or in this case quadratic) frontier in half. The same idea underlies bidirectional Dijkstra in routing, meet-in-the-middle attacks in cryptanalysis, and the two-pointer technique in sorted-array problems.

The fundamental transfer: **anytime you have a sequence-to-sequence transformation problem with uniform insertion/deletion costs, map it to a shortest-path problem on an edit grid and ask: what structure does the optimal path have?** That structure is the key to an efficient algorithm.

## Going deeper

1. **James Coglan's three-part series** (*The Myers diff algorithm*, blog.jcoglan.com, 2017): the clearest walkthrough of the algorithm with code and worked examples, also covers the linear-space variant. Written as a companion to his *Building Git* book.

2. **Needleman-Wunsch and Smith-Waterman** (the canonical bioinformatics aligners): both use the same edit graph with affine gap costs and local vs global alignment variants—reading them after Myers shows exactly what the unit-cost assumption buys and what it costs.

3. **Git source code**: `xdiff/xdiffi.c` in the git repository implements a Myers-derived algorithm with the "too expensive" heuristic (falls back to greedy when D grows large). Reading the C implementation against the paper is a good exercise in connecting theory to production code.
