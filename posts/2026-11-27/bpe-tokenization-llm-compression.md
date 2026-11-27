---
title: "Let's Build the GPT Tokenizer (minbpe)"
source: https://github.com/karpathy/minbpe
author: Andrej Karpathy
company: Independent
date_posted: 2024-02-21
date_digested: 2026-11-27
---

# Let's Build the GPT Tokenizer (minbpe)

## What's new to learn

1. **Byte Pair Encoding (BPE) as a compression algorithm applied to vocabulary learning.** BPE was originally a 1994 lossless compression technique; applying it to text gives LLMs a data-driven vocabulary whose boundaries reflect corpus frequency, not linguistic rules.

2. **Merge-rule priority as the encoding contract.** Training produces an ordered list of merge rules; inference must apply them in that exact order. The vocabulary is not a lookup table — it is a priority-ordered program.

3. **Token boundaries as the hidden inductive bias of every LLM.** Because tokenization is learned from the same corpus as the model, the vocabulary embeds statistical regularities of *that* corpus. Models trained on English corpora tokenize Japanese, Python, or numbers coarsely — causing specific, predictable failure modes.

## Prerequisites

- UTF-8 encoding: code points, multi-byte sequences, and byte values 0–255.
- Embedding tables: how a token integer indexes a row in a learned matrix.
- Basic compression intuition: shorter codes for more-frequent symbols (Huffman) and dictionary substitution for repeated substrings (LZ78).

## The core idea

Every LLM tokenizer solves one problem: map an arbitrary string of Unicode text to a sequence of integers in a fixed vocabulary, and back. BPE solves it by treating that mapping as a compression problem. Rather than deciding in advance which character sequences deserve their own token (the linguistic approach), BPE asks: *what merges would minimise the total token count across a large corpus?* The answer is a greedy iterative algorithm that finds the most frequent adjacent pair at each step, creates a new token for it, and repeats. The final vocabulary — 50,257 entries for GPT-2, ~100,000 for GPT-4 — is the codebook this compression algorithm learned.

The result is that common English words like `" the"` or `" is"` (note the leading space) get one token each, while rare or cross-language substrings are left as multiple byte-level tokens. The model always sees the world through this lens.

## Mechanics

### Seed vocabulary: bytes first

BPE starts at the byte level, not the character level. Every possible byte (0–255) is its own token. This is the base vocabulary, and it guarantees that *any* string is representable, even if it costs many tokens. Byte-level BPE, introduced in the GPT-2 paper, avoids the `[UNK]` token that plagued earlier character-level models.

### Training: greedy pair merging

```python
# Pseudocode for BPE training
vocab = {i: bytes([i]) for i in range(256)}
merges = {}            # ordered dict: (pair) → new_id
ids = encode_as_bytes(training_corpus)

for _ in range(vocab_size - 256):
    counts = count_pairs(ids)
    best_pair = max(counts, key=counts.get)
    new_id = len(vocab)
    vocab[new_id] = vocab[best_pair[0]] + vocab[best_pair[1]]
    merges[best_pair] = new_id
    ids = replace_pair(ids, best_pair, new_id)
```

Each iteration:
1. Count every adjacent pair `(a, b)` in the current token sequence.
2. Find the globally most-frequent pair.
3. Assign it a new integer id (`256`, `257`, …).
4. Record the merge rule `(a, b) → new_id`.
5. Replace every occurrence of that pair with `new_id`.

After `V − 256` iterations, the vocabulary has `V` entries and the merge table has `V − 256` ordered rules.

**Key detail — ordering is semantics.** The merge rules are numbered. Rule 0 fires before rule 1. During encoding, the algorithm applies rules in this priority order. Changing the order would produce different tokens for the same string — the merge list is not just metadata, it *is* the tokenizer.

### Encoding (inference)

```python
def encode(text: str) -> list[int]:
    ids = list(text.encode("utf-8"))   # start as bytes
    while True:
        pairs = get_pairs(ids)
        # find the pair with the lowest merge priority
        best = min(pairs, key=lambda p: merges.get(p, float("inf")))
        if best not in merges:
            break
        ids = replace_pair(ids, best, merges[best])
    return ids
```

This is *not* a simple hash lookup. For each string, the algorithm must scan for the merge rule with the lowest index (earliest/highest-priority merge) and apply it. It then rescans. The process terminates when no pairs match any merge rule.

**GPT-4 tiktoken adds regex pre-tokenization.** Before applying merges, tiktoken splits the string with a regex like:

```python
PAT = r"'(?i:[sdmt]|ll|ve|re)|[^\r\n\p{L}\p{N}]?\p{L}+|\p{N}{1,3}|..."
```

This prevents merges across word or punctuation boundaries. `"don't"` becomes two pieces `["don", "'t"]`; each is tokenized independently. Without this, `"don't word"` could accumulate a merge that ties punctuation to the following word in an undesirable way. The regex encodes a soft linguistic prior on top of the purely statistical BPE.

### Decoding

```python
def decode(ids: list[int]) -> str:
    return b"".join(vocab[i] for i in ids).decode("utf-8", errors="replace")
```

Each token maps to a byte sequence (stored in `vocab`); concatenate them and decode UTF-8. The only subtlety: a token can represent a *partial* UTF-8 codepoint (e.g., the first two bytes of a 3-byte Chinese character). Adjacent tokens together form the complete codepoint. This is why joining first and decoding once is correct, but decoding each token independently can fail.

### Where do the merge counts come from?

The same training text that trains the model. GPT-2 used ~40 GB of web text; GPT-4's tokenizer was trained on a much larger multilingual corpus. The vocabulary of the final model *cannot be separated from the data it was trained on* — the two are entangled from the start.

## Where it breaks

**Arithmetic.** The number `1234567` is likely split into `["12", "34", "5", "67"]` or some other arbitrary grouping depending on corpus frequency. The model must learn carry arithmetic across these arbitrary digit boundaries — a feat it handles imperfectly. Adding a leading space changes the split: `" 12345"` might be a different sequence than `"12345"`.

**Non-English and low-resource languages.** A token covers ~4 English characters on average. For Japanese, Korean, or Arabic text, the same 4 bytes might represent only 1–2 characters. Models effectively "spend" 3–5× more of their context window on non-English text, and the embeddings for rare-language tokens are trained on far fewer examples.

**Code.** Python indentation and common keywords get efficient tokens; less-common languages like Cobol or Prolog do not. This is a direct reflection of the training corpus's language distribution.

**Token boundary sensitivity.** `" Python"` and `"Python"` are *different tokens* (leading space or not). Adding or removing whitespace changes behaviour. This is not a bug — it is a faithful reflection of how the token was trained. But it makes prompt engineering fragile.

**The SolidGoldMagikarp / "glitch token" problem.** Some tokens exist in the vocabulary because they appeared frequently in the *tokenizer's* training corpus, but almost never in the LLM's *text* training data. The embedding for that token is barely trained. When the model encounters it, it produces bizarre outputs. This is a data contamination artifact: the vocabulary and the model were trained on different distributions.

**Vocabulary is frozen at training.** Unlike database schemas, you cannot extend the vocabulary after training without retraining embeddings. Adding a new domain (e.g., a new programming language) requires either a full retrain or a continued pretraining step that expands the vocabulary.

## Why it works

The deeper principle is **minimum description length (MDL) applied to vocabulary design**. MDL says the best model for data is the one that compresses it most. BPE's merge criterion — always merge the most frequent pair — is a greedy approximation of minimising the total description length of the corpus under the new vocabulary. Each merge reduces the encoded corpus length by exactly `count(pair) - 1` tokens; maximising that reduction at every step is a greedy MDL strategy.

This unifies several ideas:
- **Huffman coding**: assign short codes to frequent symbols. BPE's merged tokens are "shorter" in the sense that one integer replaces two. The analogy is imperfect (Huffman operates on frequencies, BPE on pair adjacency), but the principle — frequent patterns pay fewer bits — is shared.
- **LZ78 dictionary compression**: build a dictionary of repeated substrings; replace occurrences with dictionary indices. BPE's merge table *is* a dictionary, and encoding is dictionary substitution.
- **Morphology**: in linguistic theory, a morpheme is the smallest unit of meaning. BPE discovers *statistical morphemes* — recurring chunks that often coincide with real morpheme boundaries in inflecting languages, but purely by frequency, not grammar.

The vocabulary is therefore not a neutral encoding of language. It is a compression artifact that encodes the statistical regularities of the training corpus. Any behaviour of the LLM that feels "linguistic" — handling of plurals, verb conjugations, compound nouns — is partly a consequence of tokenization. And any failure — arithmetic, rare-language performance, sensitivity to spacing — is also a tokenization artifact.

## Going deeper

1. **Sennrich et al. 2016 — "Neural Machine Translation of Rare Words with Subword Units"** (arXiv:1508.07909): the paper that introduced BPE to NLP. The original motivation was handling rare words in translation without an `[UNK]` token.

2. **tiktoken** (github.com/openai/tiktoken): OpenAI's production BPE implementation in Rust with a Python interface. Studying its `cl100k_base` encoding rules (GPT-4's tokenizer) shows the regex pre-tokenization and the 100,277-entry vocabulary in practice.

3. **SentencePiece / WordPiece**: Google's alternative subword tokenizers. WordPiece (BERT) merges pairs that maximise the language-model likelihood of the training data, not raw frequency — a Bayesian analogue to BPE's greedy MDL. Comparing the two reveals what the choice of objective function does to the final vocabulary.
