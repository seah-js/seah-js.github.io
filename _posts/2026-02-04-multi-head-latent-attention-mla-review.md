---
layout: post
title: "Multi-head Latent Attention (MLA) — Review"
description: "Review session on MLA — testing recall on KV cache compression, learned projections, and the memory vs compute tradeoff"
categories: architecture
date: 2026-02-04
---

This is a review session between me and [Klover](https://github.com/openclaw/openclaw), my AI assistant. The idea is simple — Klover quizzes me on something I've learned before, I explain it in my own words, and we go deeper on the parts that stick (or don't). This format captures the actual back-and-forth, including the mistakes.

This time: Multi-head Latent Attention (MLA), the mechanism behind DeepSeek's efficient inference.

---

## What does MLA solve?

**Klover:** Let's start with the basics. What problem does MLA solve?

**Me:** It saves on KV cache. Instead of storing the full key and value vectors for every token, MLA stores a smaller latent vector. That latent vector gets decoded back into keys and values during inference when you actually need them.

**Klover:** Good — you've got the core idea. Two follow-ups. First: how does that compression actually work, mechanically? And second: if the goal is to reduce memory, why not just use fewer attention heads?

## Compression mechanism and why not fewer heads

**Me:** The compression is matrix multiplication. You encode the full KV into a smaller latent space, then decode it back when needed. As for fewer heads — different heads attend to different aspects of the input. If you just cut heads, you lose those different perspectives entirely. MLA preserves the multi-head relationships but stores everything compactly.

**Klover:** Right. The key detail: MLA uses **learned linear projections**. There's a down-projection matrix (W_c) that compresses the KV into the latent vector, and up-projection matrices (W_uk, W_uv) that reconstruct keys and values per head during attention.

The distinction that matters — this is a **learned** compression, not a post-hoc one. The model learns *how* to compress during training, so the latent vector captures what actually matters for attention. That's fundamentally different from just truncating or dropping heads after the fact.

## Memory vs compute

**Klover:** Next question: does MLA save compute during training?

**Me:** No, it's actually more compute. You have extra encode and decode steps — the compression and decompression aren't free.

**Klover:** Correct. The savings are about **memory**, not compute. During inference, the KV cache is the main bottleneck — it grows linearly with sequence length and batch size, and it's what limits how many tokens you can process or how many users you can serve. MLA shrinks that cache dramatically.

## Pushback on training memory

**Me:** Wait — isn't the KV cache during training a win too? You still have KV stored in memory during the forward pass.

**Klover:** Good catch, actually. During training, the activations from the forward pass need to be stored for the backward pass. If you store the latent vectors instead of the full KV, that's less activation memory — similar in spirit to gradient checkpointing. So yes, there is a training memory benefit.

But the relative win is much smaller than inference. During training, you have a fixed batch and sequence length, and the activation memory is just one part of the total memory budget (model parameters, optimizer states, gradients all compete for space). During inference, the KV cache is often *the* dominant memory cost, especially for long sequences. That's where MLA really shines.

## What's the risk?

**Klover:** Last one. What's the risk of this approach?

**Me:** It's lossy compression. You're squeezing KV into a lower-dimensional space, so some information gets lost. Attention quality could drop.

**Klover:** Exactly. The latent dimension is a tuning knob — smaller means more compression and less memory, but more information loss. Too aggressive and your attention patterns degrade. The art is finding the sweet spot where you get major memory savings without meaningful quality loss.

---

*Review session: Feb 4, 2026. Exposure → Developing. All core concepts retained, good pushback on training memory savings.*
