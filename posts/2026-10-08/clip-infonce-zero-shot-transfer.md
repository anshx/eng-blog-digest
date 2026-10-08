---
title: "Learning Transferable Visual Models From Natural Language Supervision (CLIP)"
source: https://arxiv.org/abs/2103.00020
author: Alec Radford, Jong Wook Kim, Chris Hallacy, Aditya Ramesh, Gabriel Goh, Sandhini Agarwal, Girish Sastry, Amanda Askell, Pamela Mishkin, Jack Clark, Gretchen Krueger, Ilya Sutskever
company: OpenAI
date_posted: 2021-02-26
date_digested: 2026-10-08
---

# Learning Transferable Visual Models From Natural Language Supervision (CLIP)

## What's new to learn

1. **InfoNCE / Noise-Contrastive Estimation (NCE) as a training objective**: A loss that replaces an intractable softmax over all possible labels with a tractable discrimination between one true pair and N−1 in-batch negatives, while provably lower-bounding mutual information between the two modalities. Understanding why this works unlocks the logic behind Word2Vec negative sampling, SimCLR, dense retrieval, and DPO.

2. **Zero-shot transfer via text prompting**: At inference time, encode each candidate class name as a natural language sentence ("a photo of a {class}") and compare its embedding to the query image — no fine-tuning, no labeled examples needed. The trained encoders act as an open-vocabulary classifier.

3. **Natural language as open-vocabulary supervision**: Fixed label sets create a closed conceptual ceiling (1,000 ImageNet classes); free-form text descriptions give the model access to any concept expressible in language, which is why representations learned this way transfer to novel tasks the model has never seen.

## Prerequisites

- What a vector embedding is and what cosine similarity measures.
- The transformer architecture at a high level (attention, [EOS] token as sequence summary).
- Cross-entropy loss for classification.
- Basic probability: conditional distribution, marginal distribution.

The mutual information lower-bound interpretation is the richest part — it helps to have seen the phrase before, but the mechanics of CLIP are fully learnable without it.

## The core idea

CLIP trains two encoders — one for images, one for text — to embed their inputs into a shared vector space such that matched (image, text) pairs have high cosine similarity and unmatched pairs have low cosine similarity. Given a batch of N image-text pairs, the loss asks every image to pick its paired text from the N candidates, and every text to pick its paired image. After training on 400 million noisy (image, alt-text) pairs scraped from the internet, the embeddings capture a rich shared conceptual vocabulary.

Zero-shot classification then works as open-vocabulary lookup: describe each candidate class in text, embed it, embed the query image, and return the most similar class. No class-specific fine-tuning, no output head — just nearest-neighbor retrieval in a shared concept space.

## Mechanics

### Architecture

CLIP learns two independent encoders:

- **Image encoder**: ResNet or Vision Transformer (ViT). A global average pool (ResNet) or the [CLS] token (ViT) produces a single vector, linearly projected to d dimensions (d = 512 for the base ViT-B/32 variant; up to 768 for larger models).
- **Text encoder**: A GPT-2–style transformer. The embedding of the special [EOS] token at the end of the tokenized string becomes the text representation, linearly projected to the same d dimensions.

Both projections produce L2-normalized embeddings on the unit hypersphere.

### Training objective: InfoNCE / symmetric cross-entropy

For a batch of N (image, text) pairs:

1. Compute the N×N cosine similarity matrix:

   ```
   S[i,j] = (image_embed_i · text_embed_j) / τ
   ```

   where τ is a learnable temperature parameter (initialized to 0.07, roughly 1/14).

2. **Row cross-entropy**: Each row i is treated as a K=N-class classification problem — image i must identify its paired text j=i from all N texts. The label is 1 on the diagonal, 0 elsewhere.

3. **Column cross-entropy**: Same in reverse — each text must identify its paired image.

4. Total loss = (row CE loss + column CE loss) / 2.

The N diagonal entries are the positive pairs; the 2(N−1) off-diagonal entries in each row/column are the negatives. The loss pushes diagonal entries high and off-diagonals low, simultaneously.

**Why batch size matters**: The InfoNCE lower bound on mutual information is:

```
I(image; text) ≥ log(N) − L_InfoNCE
```

Larger N tightens the bound and provides harder negatives. CLIP uses a batch size of **32,768** — each sample is compared against 32,767 negatives. The enormous compute cost (592 V100 GPUs × 18 days for ViT-B, 63 days for ViT-L) is driven by this batch size requirement at least as much as by encoder depth.

**Temperature τ**: Smaller τ sharpens the softmax (model must be more certain); larger τ flattens it (more forgiving). CLIP makes τ a learnable parameter clamped to [0.01, 100], allowing the model to self-calibrate the confidence required during training.

### Training data

WIT (WebImageText): 400 million (image, alt-text) pairs assembled by querying Common Crawl with ~500,000 search terms. Notably, the alt text is **noisy** — it often describes the page or context rather than the image directly. Yet noisy web text at 400M scale outperforms carefully curated 15M-pair datasets like Conceptual Captions. Scale beats curation.

### Zero-shot inference

Given K candidate classes (e.g., ImageNet's 1,000):

1. For each class c, construct a text prompt: `"a photo of a {c}."` (or an ensemble of several prompt templates for better accuracy).
2. Encode each of the K prompts with the text encoder → K text embeddings.
3. Encode the query image → image embedding.
4. Compute K cosine similarities; apply softmax; return the argmax.

The soft-probability output can be used for calibrated confidence. Ensembling ~80 prompt templates (e.g., "a photo of a big {c}", "a photo of a small {c}", "a close-up of a {c}") improves accuracy by ~3.5 points over using the class name alone.

### Linear probing vs. zero-shot

An additional result: the CLIP image encoder, frozen with no fine-tuning, used as features fed into a logistic regression classifier ("linear probe") achieves ImageNet performance competitive with fully fine-tuned ResNet-50s from the pre-CLIP era. This is the strongest evidence that the representations are rich and general — the encoder captures genuine semantics, not overfitted patterns.

## Where it breaks

**Fine-grained spatial and counting tasks**: CLIP cannot reliably count objects ("there are three dogs"), reason about spatial relationships ("the cat is to the left of the box"), or compare visual attributes within an image. The contrastive objective aligns high-level image-level concepts with high-level sentence-level concepts, not sub-image regions with clauses.

**Out-of-distribution visual domains**: Radiology images, satellite imagery, histopathology slides, handwritten digits (MNIST zero-shot: 76%, vs. fully supervised 99.7%) — domains where web captions are rare or use very different language patterns. Web supervision instills a web-distribution bias.

**Adversarial robustness**: CLIP's embedding space can be perturbed by adversarial noise that preserves perceptual similarity while flipping the nearest-neighbor class. Embeddings learned from noisy web data are not robust to this.

**Caption-image misalignment**: A bicycle alt-text might say "a great deal on bikes" rather than describing what is in the image. The model learns some useful signal but also absorbs the co-occurrence statistics of search queries, not image semantics.

**No in-context adaptation**: Unlike a language model, CLIP cannot be given a few labeled examples at test time and improve. Zero-shot is its only mode; few-shot requires fine-tuning.

**Data and compute inefficiency**: Matching a ResNet-50 trained on 1.28M labeled ImageNet examples requires 400M noisy web pairs and ~600 GPU-days. Per-example efficiency is far worse than supervised training; scale compensates.

## Why it works

The deepest principle is that **InfoNCE is Noise Contrastive Estimation — and NCE converts intractable partition functions into tractable classification problems**.

In generative modeling, the probability of a data point is p(x) = f(x) / Z where Z = Σ f(x') is the partition function (normalizing constant). Computing Z requires summing over all possible inputs — intractable for images or text.

NCE (Gutmann & Hyvarinen, 2010) sidesteps this: instead of computing Z, train a binary classifier to distinguish real data points from "noise" samples drawn from a known distribution. The classifier's learned weights implicitly encode Z. Crucially, the partition function never has to be computed directly — it cancels in the log-odds ratio.

Word2Vec negative sampling (2013 archive: Word2Vec) is exactly NCE applied to word co-occurrence: discriminate the true (word, context) pair from k randomly sampled words. The vocabulary partition function never appears.

DPO (2026-08-07 in this archive) is NCE for preferences: discriminate the preferred response from the rejected one; the per-context partition function Z(x) cancels in the pairwise cross-entropy because it appears in both numerator and denominator of the log-odds.

InfoNCE (Oord et al., CPC, 2018) is NCE with a special noise distribution: the marginals of the joint distribution, estimated by the in-batch negatives. Maximizing the InfoNCE objective therefore:

1. Pushes matched embeddings together (mutual information alignment), and  
2. Pushes all embeddings toward uniform distribution on the sphere (implicit uniformity pressure from the denominator), which prevents representation collapse.

Wang & Isola (2020) formalize this as the **alignment + uniformity decomposition**: every contrastive loss can be written as a weighted sum of an alignment term (positive pairs should be identical) and a uniformity term (all embeddings should be uniformly distributed on the hypersphere). Together, these forces produce representations that are maximally informative about the other modality without being redundant.

The "oh, so X is just Y" insight: **CLIP is word2vec running at image–sentence granularity instead of word–context granularity**, scaled 400× in data and run simultaneously over two very different input modalities. The objective, the trick, and the intuition are identical; only the modality and scale differ.

Why natural language supervision outperforms fixed labels: fixed labels bake in the taxonomic choices of whoever labeled the dataset (the 1,000 ImageNet categories). Text descriptions encode the natural language concept structure (hierarchical, relational, open-vocabulary). Representations learned against text supervision therefore organize themselves according to how language structures meaning — which happens to be how humans want to query them.

## Going deeper

1. **SimCLR** (Chen et al., 2020): The same NCE objective applied to self-supervised visual representation learning — two augmented views of the same image are the positive pair, all other batch images are negatives. No text. Shows that the contrastive objective works within a single modality; understanding CLIP and SimCLR together isolates exactly what natural language adds.

2. **"Understanding Contrastive Representation Learning through Alignment and Uniformity on the Hypersphere"** (Wang & Isola, 2020, arXiv:2005.10242): Proves that the two forces driving contrastive training are alignment (positive pairs should converge) and uniformity (all embeddings should be evenly spread on the sphere). Provides analytic gradients for each term and shows that optimizing both separately achieves better representations than the joint InfoNCE loss alone.

3. **DALL-E 2** (Ramesh et al., 2022, arXiv:2204.06125): Uses CLIP image embeddings directly as the conditioning signal for a diffusion model. Since CLIP has aligned the image–text embedding spaces, you can generate images by specifying a target CLIP text embedding — the text→image direction is "free" once the embeddings are aligned. This is why CLIP is the invisible backbone of most text-to-image systems.
