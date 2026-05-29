---
title: "Dremel: Interactive Analysis of Web-Scale Datasets"
source: https://research.google.com/pubs/archive/36632.pdf
author: Sergey Melnik, Andrey Gubarev, Jing Jing Long, Geoffrey Romer, Shiva Shivakumar, Matt Tolton, Theo Vassilakis
company: Google
date_posted: 2010-09-01
date_digested: 2026-05-29
---

# Dremel: Interactive Analysis of Web-Scale Datasets

## What's new to learn

1. **Record shredding** — the algorithm that decomposes a nested record (protobuf/JSON) into flat per-column streams, storing each leaf value alongside two small integers that encode exactly where in the tree that value lives.
2. **Repetition and definition levels** — the two integers per stored value: *repetition level* says "which ancestor just restarted," and *definition level* says "how deep the tree was actually populated." Together they allow lossless reconstruction of any nested record from individual columns with zero cross-column reads.
3. **Multi-level serving tree** — a physical query execution model where leaf servers scan column shards in parallel, intermediate servers aggregate partial results upward through a tree, and the root emits a final answer — a map-reduce topology applied to column-local computation.

## Prerequisites

- **Row vs columnar storage**: why scanning one column for a billion rows is faster than scanning all columns (see the MonetDB/X100 entry in this archive).
- **Protocol Buffers (protobuf) or Thrift schemas**: nested records with `optional`, `required`, and `repeated` fields. If you know what `repeated group Name { repeated group Language { ... } }` looks like, you're set.
- **Depth-first tree traversal**: Dremel's shredding algorithm is a DFS walk, and the levels record coordinates within that walk.

## The core idea

Columnar formats give massive scan speedups for queries that touch a small subset of fields. But real-world data is rarely flat: a web document has repeated URLs, repeated anchor texts, repeated languages, each possibly absent. If you naively store nested records row by row you throw away column locality. If you just store leaf values without structure you lose the ability to reconstruct records.

Dremel's answer: **store the leaf value plus two small integers**. Those integers — the repetition level `r` and definition level `d` — encode, for each stored value, its complete structural address in the tree. They require only `⌈log₂(max_r + 1)⌉` and `⌈log₂(max_d + 1)⌉` bits per value, which for most real schemas is 2–4 bits each.

The payoff: to answer `SELECT AVG(Name.Language.Code) ... `, you read one column. Zero other columns are touched. The structural integers tell you unambiguously which records had a `Name.Language.Code` and how many times each repeated.

## Mechanics

### Schema and field paths

```protobuf
message Document {
  required int64 DocId;
  optional group Links {
    repeated int64 Backward;
    repeated int64 Forward;
  }
  repeated group Name {
    repeated group Language {
      required string Code;   // path: Name.Language.Code
      optional string Country;
    }
    optional string Url;
  }
}
```

Each leaf column is identified by its *path*, e.g., `Name.Language.Code`. Two parameters determine how many bits the structural integers need:

- **max repetition level** for a path = number of `repeated` fields in that path. For `Name.Language.Code` that's 2 (`Name` and `Language`).
- **max definition level** for a path = number of `optional` or `repeated` fields in that path. For `Name.Language.Code` that's also 2 (`Name` and `Language`; `Code` itself is `required` so it doesn't add a level).

### Two example records

```
r1: DocId=10, Links={Forward:[20,40,60]},
    Name=[{Language:[{Code:'en-us',Country:'us'},{Code:'en'}], Url:'http://A'},
          {Url:'http://B'},
          {Language:[{Code:'en-gb',Country:'gb'}]}]

r2: DocId=20, Links={Backward:[10,30], Forward:[80]}
    // no Name at all
```

### Shredding `Name.Language.Code`

Walk each record depth-first. At every leaf position emit `(value, r, d)`:

| Position in walk                | value   | r | d | Meaning                                       |
|---------------------------------|---------|---|---|-----------------------------------------------|
| r1 first Code (en-us)           | `en-us` | 0 | 2 | New record, Name present, Language present    |
| r1 second Code in first Name (en) | `en`  | 2 | 2 | Language repeated inside same Name            |
| r1 second Name — no Language    | NULL    | 1 | 1 | Name repeated; Language absent (d stops at 1)|
| r1 third Name, first Code (en-gb)| `en-gb`| 1 | 2 | Name repeated; Language present              |
| r2 — no Name at all             | NULL    | 0 | 0 | New record; Name absent (d=0)                 |

Stored column: values `[en-us, en, NULL, en-gb, NULL]`, R `[0,2,1,1,0]`, D `[2,2,1,2,0]`.

**Reading r**: `r=0` means "start of a new top-level record." `r=k` means "the k-th field in the path (counting from 1) just started a new repetition — everything above level k is still shared with the previous entry."

**Reading d**: `d=max_d` means the leaf was present. `d < max_d` means the tree was cut off: the field at depth `d+1` was absent (optional and not set, or repeated and empty). NULL is emitted for the leaf, and `d` tells you which ancestor caused the absence.

### Shredding algorithm (pseudocode)

```python
def shred(record, column_path, r_level=0):
    for each field F in column_path:
        if F is repeated:
            first = True
            for each occurrence of F in record:
                new_r = r_level if first else depth_of(F)
                first = False
                shred(occurrence, column_path_below_F, r_level=new_r)
            if no occurrences:
                emit(NULL, r=r_level, d=depth_of_last_defined_ancestor(record))
                return
        elif F is optional and F not in record:
            emit(NULL, r=r_level, d=depth_of_last_defined_ancestor(record))
            return
        else:
            record = record[F]
    emit(record, r=r_level, d=max_d)
```

### Assembly algorithm

Going from columns back to records uses a **finite state machine** per column. Each state corresponds to a "next field to fill in" within the schema path. Transitions are driven by the `r` value of the next entry: `r=k` means "backtrack to level k and resume from there."

The FSM for `Name.Language.Code` has states like `{new_record, new_name, new_language, inside_language}`, and `r` values trigger the right state transition at record-reconstruction time. This is the non-trivial part — the original paper specifies a compiler that generates the FSM from the schema.

### Serving tree

Dremel distributes execution across a tree of servers:

```
          Root (issue query, combine final answer)
         /       \
   Intermediate  Intermediate   (partial aggregation)
   /   \           /    \
Leaf Leaf       Leaf   Leaf     (column shard scanners)
```

Each leaf server holds a horizontal shard of the column files. It evaluates filter predicates and emits partial aggregates. Intermediates combine partials. The root returns results. Because each leaf reads only the columns touched by the query, leaf servers are embarrassingly parallel with no cross-leaf communication.

Dremel's language is SQL-like but with a `WITHIN RECORD` modifier that scopes aggregations to a nested repeated group rather than to the entire table — `COUNT(Name.Language.Code) WITHIN Name` counts Languages per Name instance, not across the whole document.

## Where it breaks

**Schema evolution is fragile at depth changes.** Adding or removing a nesting level in a path changes `max_d` and `max_r` for every downstream field. Stored columns from before the change have different-semantic `d` and `r` values than columns written after. Parquet deals with this by embedding the schema inside the file footer, so you always re-derive levels from the in-file schema — but joining old and new files requires careful schema negotiation.

**Flat schemas don't benefit.** If all your data is flat (`required int64 value;`), both R and D are always 0 and always the maximum at the same time — they add no information, just two wasted bytes per value (though encoders elide them in practice when max levels are 0).

**The assembly FSM is hard to get right.** Parquet's first Java implementation had correctness bugs in edge cases (missing intermediate nodes, back-to-back repeated groups with nulls). The paper's FSM construction algorithm is correct but complex; most descriptions simplify it to the point of omitting the tricky cases.

**Write amplification on updates.** Like all columnar formats, Dremel/Parquet is append-optimised. In-place updates require rewriting entire column groups (or row groups). BigQuery hides this behind its storage service; Parquet-on-HDFS systems (Hive, Spark) typically write new files and compact in the background.

**`WITHIN RECORD` semantics are unfamiliar.** Querying nested repeated data requires thinking in terms of path-scoped aggregations, which is foreign to flat-table SQL intuition. Engineers who are comfortable with `GROUP BY` frequently misread `WITHIN RECORD` semantics, silently producing wrong answers.

## Why it works

The deeper principle: **a tree's complete structure can be encoded in O(depth) bits per leaf**, not in O(nodes) bits.

A nested record is a tree. A depth-first walk of that tree visits every leaf exactly once. After the walk, you can reconstruct the tree if you can answer two questions for each visited leaf:

1. "How far back up did I have to climb before descending to this leaf?" → repetition level
2. "How far down did I descend before the path terminated?" → definition level

These are relative, schema-keyed coordinates, not absolute node IDs. The maximum value of each is bounded by the number of *repeated* or *optional* fields in the schema path — which is a schema property, fixed at write time. For real-world schemas this is almost always ≤ 5.

This is the same insight that makes other compact tree encodings work:

- **Balanced parentheses**: encode a tree as `(` when you enter a node, `)` when you leave. Two symbols per node encode structure without pointers. Dremel uses two integers instead of symbols, but the idea is identical.
- **Euler tour / DFS linearisation**: any tree algorithm that only needs parent-child relationships can work on the DFS sequence plus an ancestor stack — the ancestor stack is exactly what `r` and `d` implicitly encode.
- **Transformer positional encodings**: sinusoidal or RoPE encodings allow a model to recover relative position from two small per-token vectors; Dremel's levels allow a reader to recover relative tree position from two small per-value integers.

The common principle: **hierarchical structure is equivalent to positional metadata relative to a depth-bounded walk**. Once you see this, the 2-integer overhead of Dremel's encoding stops looking like a clever trick and starts looking like the minimum sufficient information.

The serving tree structure is less original — it's a standard map-reduce tree — but Dremel's contribution there is keeping all partial computation in the columnar domain. Intermediates never reconstruct full records. They operate purely on the encoded column streams, which means intermediate memory is proportional to the size of the partial aggregate, not the size of the record.

## Going deeper

1. **Apache Parquet format specification** (`parquet.apache.org/docs/file-format/`) — the production implementation of Dremel's encoding, extended with row groups, column chunks, and multiple physical encodings (RLE, dictionary, delta). Reading the spec after Dremel makes every design choice legible.
2. **"Dremel made simple with Parquet"** — Julien Le Dem's 2013 Twitter engineering post that first explained Dremel's levels to a broad audience, with hand-traced examples. The best companion to the original paper. (blog.twitter.com/engineering/en_us/a/2013/dremel-made-simple-with-parquet)
3. **"Inside Capacitor, BigQuery's next-generation columnar storage format"** (Google Cloud Blog, 2016) — how Google evolved Dremel's on-disk format to add query-aware row reordering (an NP-complete optimisation approximated with greedy column-correlation heuristics) and background re-encoding, showing how the fundamental R/D encoding holds up as the rest of the stack is rebuilt around it.
