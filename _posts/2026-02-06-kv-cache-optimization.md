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

**Klover:** You already know from MLA that the KV cache is a bottleneck. Quick refresh — why does it exist in the first place?

**Me:** KV cache exists because it's used for inference to generate the next token. It speeds up the process because the key and value vectors don't have to be recalculated every time a new token is generated.

**Klover:** Spot on. Without it, every new token would recompute attention over the entire sequence — quadratic cost. With it, you just compute K and V for the new token and append. Linear.

---

## How Much Memory Does It Use?

**Klover:** What factors make the KV cache bigger?

**Me:** If I'm not wrong, a single number is 4 bits? If your vector is 128 numbers and you have both keys and values, each token would have 4×128×2, and then it scales linearly with tokens.

**Klover:** Close but a few corrections:

- FP16 (standard for inference) is **2 bytes (16 bits)**, not 4 bits
- You got dimension and K+V (×2) right
- But you're missing **number of layers and number of attention heads**

Full formula:

```
2 × num_layers × num_heads × head_dim × seq_length × 2 bytes
```

For Llama 70B (80 layers, 64 heads, head_dim 128) at 4K sequence length — the KV cache alone is **~10GB per single request**. 100 concurrent users = 1TB just for KV cache.

---

## Optimization #1: PagedAttention

**Klover:** The name comes from how operating systems manage RAM. What do you think the connection might be?

**Me:** I don't even know how operating systems manage RAM.

**Klover:** Fair enough. Quick detour:

**The OS problem:** Programs ask for different amounts of memory. If you give each one a big contiguous block "just in case," you waste memory sitting unused. This is **internal fragmentation**.

**The OS solution — paging:** Split memory into small fixed-size **pages**. Allocate on demand. Programs think they have contiguous memory, but the OS maps pages to wherever there's free space.

**Same problem with KV cache:** Each request pre-allocates for the maximum sequence length. A 500-token response with 4096 reserved = 87% wasted.

**PagedAttention (from vLLM):** Splits the KV cache into small fixed-size blocks, allocated on demand as the sequence grows. Result: ~60-80% better memory utilization.

**Klover:** What's the core insight that both OS paging and PagedAttention share?

**Me:** Rationing memory to the services that need it.

**Klover:** Yes — allocate on demand, not upfront. Don't reserve what you might not use.

---

## Optimization #2: Prefix Caching

**Klover:** Think about a chatbot with a system prompt that every conversation starts with. Every request computes the exact same K and V for those tokens. What do you think prefix caching does?

**Me:** Cache the KV for the system prompt even before any user prompt is given.

**Klover:** Exactly. Compute once, reuse across all requests sharing that prefix.

This applies beyond system prompts:
- **Few-shot examples** — same examples prepended to every request
- **Multi-turn chat** — prior conversation already cached, only compute for the new message
- **Parallel requests** — multiple requests sharing the same document

2000-token system prompt across 100 users = 200K tokens of computation eliminated.

---

## Optimization #3: MLA

Already covered in a [previous session](/architecture/2026/02/04/multi-head-latent-attention-mla-review.html). Compresses K and V into a smaller latent space using learned projections. Less memory per token stored.

---

## Putting It All Together

**Klover:** A company is serving an LLM to 1000 concurrent users. Same 1000-token system prompt, but response lengths vary wildly (50 to 4000 tokens). Which optimization helps with which problem?

**Me:** Same 1000-token prompt can use prefix cache across users. Varying response lengths can use PagedAttention. MLA can be used across all LLM settings if you want.

**Klover:** Nailed it. All three correct. And you're right that MLA is a model-level decision baked into the architecture, while PagedAttention and prefix caching are **serving-layer** optimizations you can apply to any model.

| Optimization | Level | When Applied |
|-------------|-------|--------------|
| MLA | Model architecture | Training time |
| PagedAttention | Serving layer | Any model |
| Prefix caching | Serving layer | Any model |

---

## Self-Assessment: 3.8/5

Core concepts make sense. Need another pass to solidify the memory formula and deepen understanding of PagedAttention internals.
