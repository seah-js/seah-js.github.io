---
layout: post
title: "Async & Sync — Review"
description: "Testing recall on async/sync fundamentals and going deeper into GIL race conditions"
categories: fundamentals
date: 2026-02-04
---

This is a review session between me and [Klover](https://github.com/openclaw/openclaw), my AI assistant. Yesterday we covered async vs sync from scratch. Today Klover tests what I actually retained — then pushes deeper where the gaps show up.

---

## Recall: I/O-bound vs CPU-bound

**Klover:** Let's start simple. What's the difference between I/O-bound and CPU-bound work, and why does async only help one of them?

**Me:** I/O-bound is when you're calling external services like APIs. CPU-bound is heavy computation. Async helps I/O-bound because the thread can move on to other work while waiting for the external thing to come back.

**Klover:** Good — you've got the core idea. Small addition though: I/O isn't just APIs. It's anything where the CPU is idle and waiting — disk reads, database queries, network calls, file writes. All I/O. Now explain the event loop. And what happens if you put CPU-heavy code inside one?

## Event loop + blocking

**Me:** The event loop delegates work to different tasks. But I'm not sure what you mean by CPU-heavy code inside an event loop?

**Klover:** Remember the chef analogy? One chef, single kitchen. If one dish requires 10 minutes of non-stop chopping — what happens to every other dish?

**Me:** They're on hold. Nothing else gets worked on.

**Klover:** Exactly. Everything blocks. The event loop is single-threaded — one task hogging the CPU means every other task just sits in the queue waiting. That's why CPU-bound work inside an event loop is dangerous. Now, different topic — what's Python's GIL?

## The GIL

**Me:** The GIL only allows a single thread to execute Python code at a time. The workaround is multiprocessing — one process gets its own GIL, so you can actually run things in parallel.

**Klover:** Good recall. But *why* does the GIL exist? What problem is it solving?

**Me:** It prevents memory leakage. And it stops the same thing from being worked on by multiple threads at the same time, which would cause conflicts.

**Klover:** Close, but it's not memory leakage exactly. Python uses **reference counting** to manage memory — every object has a counter tracking how many things point to it. When the count hits zero, the object gets cleaned up. The GIL exists because if two threads modify that reference count at the same time, you get a **race condition**. The count gets corrupted.

## Race conditions — step by step

**Me:** I get that two threads shouldn't touch the same thing, but how does the count actually get corrupted?

**Klover:** Walk through it. Say an object's reference count is 3.

1. **Thread A** reads the count: sees **3**
2. **Thread B** also reads the count: sees **3**
3. **Thread A** decrements and writes: **2**
4. **Thread B** decrements and writes: **2**

Final count: **2**. But two references were removed — it should be **1**.

Both threads read the old value before either wrote. Thread B's write just overwrites Thread A's work. Now scale that up:
- If the count never reaches 0 → the object never gets cleaned up → **memory leak**
- If the count hits 0 too early → the object gets freed while something still points to it → **crash**

The GIL prevents this by making sure only one thread touches Python objects at a time. Brute force, but it works.

**Me:** Okay that makes sense now. Both threads reading the same stale value before either one writes — that's the race.

## The async trap

**Klover:** Last one. What's wrong with this?

```python
async def get_data():
    response = requests.get("https://api.example.com/data")
    return response.json()
```

**Me:** `requests` is synchronous. Even though the function is `async`, the thread has to sit there and wait for the response. It's blocking.

**Klover:** Exactly. It never yields control back to the event loop. There's no pause point, so the event loop can't switch to other tasks. Your "async" function is secretly sync. The fix: use an async HTTP client like `httpx` with `await` — that `await` is the pause/resume point that lets the event loop do its job.

```python
async def get_data():
    async with httpx.AsyncClient() as client:
        response = await client.get("https://api.example.com/data")
    return response.json()
```

---

*Review session: Feb 4, 2026. Exposure → Developing. Core concepts retained. Deeper understanding of GIL race conditions gained during review.*
