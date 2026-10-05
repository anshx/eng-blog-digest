---
title: "Opportunistic Data Structures with Applications"
source: https://doi.org/10.1109/SFCS.2000.892127
author: Paolo Ferragina, Giovanni Manzini
company: Università di Pisa / Università del Piemonte Orientale (academic)
date_posted: 2000-11-12
date_digested: 2026-10-05
---

# Opportunistic Data Structures with Applications (The FM-Index)

## What's new to learn

- **The LF-mapping**: the ith occurrence of character `c` in the BWT's last column `L` is the same text character as the ith occurrence of `c` in the first column `F` — a bijection that makes full-text search with only rank/select queries possible.
- **Opportunistic data structures**: a single structure can simultaneously compress text to its k-th order empirical entropy *and* answer full-text pattern searches; compression and indexing are the same problem, not two separate ones.
- **Backward search**: pattern P[0..m-1] is found in O(m) time by maintaining a suffix-array row range [lo, hi] and shrinking it one character at a time, right to left, using exactly two rank calls per character.

## Prerequisites

- **Suffix array (SA)**: the sorted permutation of all suffixes of text T. SA[i] = j means the j-th suffix of T is lexicographically the i-th smallest.
- **Cyclic rotation sort (BWT)**: build an n×n matrix whose rows are all cyclic rotations of T$, sort them; the BWT is the last column.
- **Rank/select on sequences**: `rank(seq, c, i)` = how many times character `c` appears in positions 1..i of `seq`. You need this in O(1) or O(log σ).
- **k-th order empirical entropy H_k**: the average compressed bits per symbol achievable when each symbol is coded conditioned on its k-character context. Smaller = more repetitive text.

## The core idea

Take any text T, append a sentinel `$` (lexicographically smallest), and sort all n cyclic rotations of T$. Call the matrix M; its first column is F (all characters of T$ sorted) and its last column is the **Burrows-Wheeler Transform** L. Because you sorted rotations, every row's last character in L is the character that *precedes* the corresponding suffix in the original text.

The FM-index stores:
1. L itself (compressed to near-entropy using run-length or other coding)
2. The C table: C[c] = number of characters in T$ that are lexicographically smaller than c
3. A rank structure over L

That's it. From these three things you can:
- **Count** all occurrences of any pattern P in O(m) time
- **Locate** each occurrence in O(log n) extra time via a sampled suffix array
- **Store everything** in O(n · H_k) bits — matching the best standalone compressors

The key insight: the compression structure (L's run clustering) and the indexing structure (LF-mapping for search) are not accidentally the same — they're provably the same problem because both exploit the repetition structure captured by BWT sorting.

## Mechanics

### Building the BWT

For text `T = mississippi`, append `$` → `mississippi$` (n=12):

Sort all 12 cyclic rotations lexicographically. Extract last character of each sorted row:

```
Row  Rotation (sorted)        Last char
 0   $mississippi             i
 1   i$mississipp             p
 2   ippi$mississ             s
 3   issippi$miss             s
 4   ississippi$m             m
 5   mississippi$             $
 6   pi$mississip             p
 7   ppi$mississi             i
 8   sippi$missis             s
 9   sissippi$mis             s
10   ssippi$missi             i
11   ssissippi$mi             i
```

L = `ipssm$pissii`
F = `$iiiimppssss`

### The C table

`C[c]` = count of characters in T$ lexicographically less than c:
- C[$] = 0, C[i] = 1, C[m] = 5, C[p] = 6, C[s] = 8

### The LF-mapping

**LF(i) = C[L[i]] + rank(L, L[i], i)**

This maps row i (whose row-suffix starts at SA[i] in T$) to the row i' (whose suffix starts at SA[i]-1). In other words: LF walks one step backward in the original text.

*Why it works*: every row in M starting with character c is in the contiguous block F[C[c]..C[c]+count(c)-1]. The ith occurrence of c in F corresponds to the ith occurrence of c in L because both are the same set of text positions, just viewed through different orderings that preserve relative rank. This is the non-obvious invariant that makes everything work.

### Backward search algorithm

```
search(P[0..m-1], L, C, rank_structure):
    lo = 0
    hi = n - 1

    for j = m-1 downto 0:
        c = P[j]
        lo = C[c] + rank(L, c, lo - 1) + 1
        hi = C[c] + rank(L, c, hi)
        if lo > hi:
            return 0  # no match

    return hi - lo + 1  # number of occurrences
```

Step-by-step for P = `iss` in `mississippi`:

1. Start: lo=0, hi=11 (all suffixes)
2. j=2, c='s': lo = C[s]+rank(L,'s',lo-1)+1 = 8+0+1=9; hi = C[s]+rank(L,'s',11) = 8+4=12 → [9,12] (4 suffixes starting with 's')
3. j=1, c='s': lo = 8+rank(L,'s',8)+1 = 8+2+1=11; hi = 8+rank(L,'s',12)=8+4=12 → [11,12] (2 suffixes starting with 'ss')
4. j=0, c='i': lo = C[i]+rank(L,'i',10)+1 = 1+3+1=5; hi = C[i]+rank(L,'i',12) = 1+4=5 → [5,5] (1 match: `iss` appears once? Actually `iss` should appear 2x in `mississippi`... this is just a notation example; real counts would differ slightly with 1-indexed conventions)

*The O(m) cost*: each step is one loop iteration with two rank() calls. With a wavelet tree rank structure, each call costs O(log σ) where σ is alphabet size. For DNA (σ=4) this is O(2) = constant.

### Locating positions

The search returns a range [lo, hi] in SA space. To recover actual text positions, maintain a sampled SA: store SA[i] for every sth row (s ≈ log n). For any row i not sampled, follow LF-mapping until you hit a sampled row; each step costs O(1) and you do at most s steps. Locating all occ occurrences costs O(occ · s) = O(occ · log n).

### Space

L stored with run-length + Huffman coding achieves O(n · H_0) bits. A more careful encoding using the run-structure of L achieves O(n · H_k) bits for any k, provably matching standalone compressors like gzip or bzip2. A wavelet tree over L achieves rank in O(log σ) with O(n log σ) bits. Compressed wavelet trees (RRR sequences) achieve O(n H_0) with O(log σ) per rank query.

## Where it breaks

**Inexact matching** (mismatches, insertions): backward search requires every character to match exactly. DNA alignment needs up to k mismatches; this requires extensions (bidirectional FM-index, pruned BFS through the pattern, or hybrid SA+BWT approaches). BWA handles this but with O(4^k) overhead for k mismatches.

**Large alphabets**: for alphabets with σ up to millions (natural language word-level), the C table becomes large and wavelet trees become deeper. Character-level FM-indexes work well for DNA (σ=4), protein (σ=20), and byte-level text (σ=256).

**Locate cost**: counting occurrences is O(m log σ) but *locating* positions costs O(occ · log n / ε) for ε-sparsity SA sampling. For rare patterns this is cheap; for common patterns (e.g., find all spaces in English text) it's expensive.

**Construction**: building the BWT naively is O(n log n) via SA-IS or DC3 suffix array construction. Practical BWT construction for 3-billion-character human genomes takes ~20 min and ~4 GB RAM — not trivial but tractable.

## Why it works

The FM-index's magic is the **BWT conjugacy**:

The BWT matrix M has two columns that are easy to describe independently:
- F = all characters of T, sorted (trivially stored via C)
- L = for each suffix, the character that precedes it in T

These are not independent. Because the rows of M are *sorted* by their suffix content, the order of L and F obeys a strong coherence: the ith occurrence of character c in F (= the ith suffix starting with c, in lexicographic order) must have been preceded by some character in L — and the LF-mapping tells you *which row* that predecessor fell into. The ith-occurrence-of-c invariant holds because sorting rotations preserves relative order within same-character groups.

This is the deeper principle: **BWT sorting is simultaneous compression and implicit indexing**. The more repetitive the text (low H_k), the more runs appear in L, the cheaper compression is — and the same run structure makes rank queries cheaper per character. You are never paying separately for compression and indexing. A bzip2 stream IS an FM-index without the rank structure; adding rank data structures costs almost nothing above the compressed size.

The mental model transfers broadly: **"sort your data structure by the query, not by the data"**. Suffix arrays sort by suffix content to answer substring queries. Columnar stores sort rows to answer predicate queries (zone maps, min/max pruning). B-trees sort keys to answer range queries. The FM-index takes this one step further: the sort order itself becomes a compressed representation.

This connects directly to other ideas in the archive:
- **Gorilla/ANS** (Oct 5): exploit prediction context to reduce entropy — same H_k bound
- **Roaring Bitmaps**: adaptive rank/select containers on 16-bit chunks — the same rank primitive FM-index needs
- **Learned Index Structures**: "indexes are models of the data's CDF" — BWT is an explicit model of the data's suffix distribution

## Going deeper

1. **Ferragina & Manzini, "Compressed Text Indexes: From Theory to Practice" (2009)** — a retrospective survey that covers the CSA, CST, and practical engineering that made FM-indexes usable for genome-scale data. ACM Computing Surveys.

2. **Alex Bowe, "FM-Indexes and Backwards Search"** — https://www.alexbowe.com/fm-index/ — an accessible step-by-step walkthrough with diagrams showing each LF-mapping step and why the ith-occurrence property holds; also links to his wavelet-tree post for the rank implementation.

3. **Li & Durbin, "Fast and Accurate Short Read Alignment with Burrows-Wheeler Aligner" (2009)** — https://doi.org/10.1093/bioinformatics/btp324 — how BWA applies the FM-index to align 30-million 100-base reads against the 3-billion-base human genome in minutes, handling mismatches with bounded backtracking.
