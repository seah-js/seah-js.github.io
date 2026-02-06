---
layout: post
title: "KV Cache Optimization — Why Inference Memory Explodes and How to Fix It"
description: "Understanding PagedAttention, prefix caching, and MLA — three approaches to taming the KV cache bottleneck"
categories: architecture
date: 2026-02-06
---

Learning session with [Klover](https://github.com/openclaw/openclaw). Today: why the KV cache is the biggest memory bottleneck in LLM inference, and three ways to optimize it.

---

## Why Does the KV Cache Exist?

During inference, every new token needs to attend to all previous tokens. Without caching, you'd recompute the Key and Value vectors for the entire sequence every time — quadratic cost.

The KV cache stores previously computed K and V vectors so you only compute them once per token, then append. Turns quadratic into linear.

---

## How Big Does It Get?

**Formula:**

```
2 × num_layers × num_heads × head_dim × seq_length × 2 bytes
```

- The first `2` = K and V
- `2 bytes` = FP16 (standard for inference)

**Klover:** For Llama 70B (80 layers, 64 heads, head_dim 128) at 4K sequence length — the KV cache alone is ~10GB per single request. 100 concurrent users = 1TB just for KV cache.

**Correction I needed:** I said a single number is 4 bits. Wrong — FP16 is 2 bytes (16 bits). I also forgot to include layers and heads as multipliers. The memory scales with everything.

---

## Three Approaches to Fix This

### 1. MLA (Multi-head Latent Attention)

Already covered in a [previous session](/architecture/2026/02/04/multi-head-latent-attention-mla-review.html). The idea: compress K and V into a smaller latent space using learned projections. Less memory per token stored.

**Key point:** This is a **model architecture** choice — baked in at training time (e.g., DeepSeek uses it).

### 2. PagedAttention

**The problem:** Without optimization, each request pre-allocates a big contiguous block of GPU memory for the maximum possible sequence length. If a request only uses 500 tokens but you reserved 4096 — that's 87% wasted. This is called **internal fragmentation**.

**The solution:** Borrowed from how operating systems manage RAM.

OS paging: instead of giving programs one big memory block, the OS splits RAM into small fixed-size **pages** and allocates them on demand. Programs think they have contiguous memory, but the OS maps pages to wherever there's free space.

PagedAttention (from vLLM) does the same for KV cache — splits it into small fixed-size **blocks**, allocated on demand as the sequence grows. Only uses memory for tokens actually generated.

**Result:** ~60-80% better memory utilization → more concurrent users on the same GPU.

### 3. Prefix Caching

**The scenario:** A chatbot with a system prompt: "You are a helpful assistant for Acme Corp..."

Every user conversation starts with the same 200 tokens. Every request computes identical K and V vectors for those tokens. Wasteful.

**The solution:** Compute KV once for the shared prefix, store it, reuse across all requests.

**Applies to:**
- System prompts (same across all users)
- Few-shot examples (same examples prepended to every request)
- Multi-turn chat (prior conversation history already cached, only compute KV for the new message)
- Parallel requests sharing the same document/context

If your system prompt is 2000 tokens across 100 users, that's 200K tokens of computation eliminated.

---

## Key Distinction

| Optimization | Level | When Applied |
|-------------|-------|-------------|
| MLA | Model architecture | Training time |
| PagedAttention | Serving layer | Any model |
| Prefix caching | Serving layer | Any model |

MLA requires building it into the model. PagedAttention and prefix caching can be applied to **any** model at serving time.

---

## Practical Scenario

**1000 concurrent users, same 1000-token system prompt, response lengths vary from 50 to 4000 tokens.**

- **Prefix caching** → shared system prompt, compute once
- **PagedAttention** → varying response lengths, no wasted pre-allocation
- **MLA** → reduces per-token KV size regardless (if the model supports it)

All three are complementary. Use them together for maximum efficiency.

---

## Self-Assessment: 3.8/5

Core concepts make sense. Need another pass to solidify the memory formula and deepen understanding of PagedAttention internals.
