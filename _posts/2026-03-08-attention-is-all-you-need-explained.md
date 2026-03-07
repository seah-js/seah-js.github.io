---
layout: post
title: "Attention Is All You Need — Full Paper Breakdown"
description: "A complete visual walkthrough of the Transformer paper that changed everything — self-attention, multi-head attention, positional encoding, and why it replaced RNNs."
categories: architecture
date: 2026-03-08
---

I made a video breaking down the original Transformer paper — "Attention Is All You Need" (Vaswani et al., 2017). This is the paper that kicked off the entire modern AI wave: GPT, BERT, LLMs, all of it traces back here.

{% include youtube.html id="C7YiPaUYo1k" %}

---

## What's covered

The video walks through the full paper, section by section:

- **Why Transformers replaced RNNs** — the parallelization problem with sequential models
- **The Encoder-Decoder architecture** — how the two stacks work together
- **Self-attention mechanism** — queries, keys, values, and scaled dot-product attention
- **Multi-head attention** — why multiple attention heads beat a single one
- **Positional encoding** — how the model knows word order without recurrence
- **Feed-forward networks** — the often-overlooked other half of each layer
- **Training details** — optimizer, learning rate schedule, regularization

## Why this paper matters

Every major language model today — GPT-4, Claude, Gemini, Llama — is built on the architecture introduced in this paper. Understanding the Transformer from the original source gives you the foundation to understand everything that came after.

---

*This is part of my [AI explainer series](https://www.youtube.com/@jayseah) — visual breakdowns of key papers and concepts.*
