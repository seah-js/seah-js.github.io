---
layout: post
title: "Race Conditions, Asyncio Locks & Concurrency Patterns"
description: "S3 race conditions in async pipelines, asyncio.Lock, thread pools, and the difference between sequential initiation and concurrent waiting"
categories: fundamentals
date: 2026-02-04
---

This is a Socratic-style learning session between me and [Klover](https://github.com/openclaw/openclaw), my AI assistant. Instead of reading docs or watching tutorials, I learn by explaining concepts in my own words and getting challenged on the gaps. The conversation format is the point — it captures the actual "aha" moments as they happen.

In this session, I hit a real bug in an AWS pipeline and used it to understand race conditions, asyncio locks, thread pools, and what "concurrent" actually means in single-threaded async code.

---

## The bug

**Me:** I've got a pipeline on AWS that processes invoices. There's a shared manifest file in S3 that tracks processing status. Multiple async LLM API calls run at once, and each one updates the manifest after it finishes. Problem: if two LLM calls return at nearly the same time, they both read the old manifest, both write their update, and one overwrites the other. Lost data.

**Klover:** Classic **read-modify-write race condition**. Same fundamental pattern as why Python needs the GIL for reference counting — two things reading the same value, modifying independently, then writing back. One update gets stomped.

The key issue: S3 doesn't lock files for you. There's no built-in mechanism that says "someone else is editing this, wait your turn." It's just object storage — you read bytes, you write bytes.

**Me:** But this is single-threaded async. I thought race conditions were a multithreading thing?

## Why single-thread async still has races

**Klover:** Common misconception. The shared state here isn't in-memory — it's external, sitting in S3. Single thread protects in-memory variables (only one line of Python runs at a time), but your event loop switches tasks at every `await` point.

Here's what happens:
1. Task A finishes its LLM call, reads manifest from S3 (`await s3_read`)
2. At that `await`, the event loop can switch. Task B also finishes, reads the **same old manifest** from S3
3. Task A writes its update to S3
4. Task B writes its update to S3 — **overwriting A's changes**

Single thread doesn't help because the "shared state" isn't protected by the thread. It's a file on a remote server.

**Me:** So even though only one line of code runs at a time, the damage happens because both tasks saw a stale version of the manifest before either wrote back.

**Klover:** Exactly. The race window is between the read and the write. Any `await` in that window is an opportunity for another task to slip in.

## The naive fix vs the right fix

**Me:** Would making the S3 reads and writes synchronous fix it?

**Klover:** Technically yes — if everything is sync, there's no `await` for the event loop to switch at. But you'd kill all your concurrency. Your LLM calls, which are pure I/O waiting, would run one at a time. For a batch of invoices, that's brutal.

Better approach: **`asyncio.Lock()`**. Keep the LLM calls concurrent, but serialize just the manifest updates.

**Me:** What is asyncio.Lock exactly?

**Klover:** Same concept as any lock — only one task can hold it at a time. The difference from `threading.Lock` is that it's async-friendly. When a task is waiting to acquire the lock, it **yields back to the event loop** instead of blocking the thread. Other tasks keep running while you wait.

```python
manifest_lock = asyncio.Lock()

async def update_manifest(invoice_result):
    async with manifest_lock:
        # only one task in here at a time
        manifest = await read_manifest_from_s3()
        manifest[invoice_result.id] = invoice_result.status
        await write_manifest_to_s3(manifest)
```

**Me:** So all the S3 read/writes for the manifest queue up one at a time?

**Klover:** Yes. Task A grabs the lock → reads → updates → writes → releases. Task B was waiting → grabs the lock → reads the manifest **with A's update already in it** → updates → writes. No lost updates. Meanwhile, all your LLM API calls are still running concurrently outside the lock.

## Single-thread async for I/O pipelines

**Me:** Does single-thread async actually work well for an I/O-heavy pipeline like this? Feels like threading would be more powerful.

**Klover:** Single-thread async is *ideal* for I/O-heavy work. You're mostly waiting on network responses — LLM APIs, S3 reads/writes. One thread juggling thousands of I/O waits uses almost no memory and has zero thread-switching overhead.

It only breaks down when you have CPU-heavy steps in the pipeline — like parsing giant PDFs or running local models. The event loop can't switch during CPU work since there's no `await` to pause at.

**Me:** So what if my pipeline has a mix — some I/O calls and some heavy parsing?

## Threading for CPU-bound work

**Klover:** For mixed workloads, you offload the CPU parts to a **thread pool** using `run_in_executor`. The async event loop stays responsive for I/O, and CPU work runs on separate threads.

**Me:** What's a thread pool?

**Klover:** Pre-created threads sitting idle, waiting for work. Think of a taxi stand — the taxis are already there, engines running. When a job comes in, a taxi picks it up immediately instead of having to spawn a brand new car from scratch.

Python gives you a default one:

```python
result = await loop.run_in_executor(None, cpu_heavy_fn, data)
```

**Me:** Break that down for me.

**Klover:**
- `loop` — the event loop (your async scheduler)
- `run_in_executor` — hand this work off to a thread pool
- `None` — use Python's default thread pool
- `cpu_heavy_fn` — the function to run on a thread
- `data` — argument passed to that function
- `await` — pause here, let the event loop do other stuff, resume when the thread finishes

You stay in async-land. The CPU work happens on a thread without blocking your event loop.

**Me:** Got it. But for my invoice pipeline where it's all API calls and S3 — just async with the lock is enough. No need for threads.

**Klover:** Correct. Don't add threading complexity when you don't need it.

## Sequential initiation vs concurrent waiting

**Me:** Let me make sure I understand the full picture. For batch invoice processing: the event loop fires off all the LLM calls, they wait concurrently, and when each returns, it hits the asyncio lock to update the manifest one at a time.

**Klover:** That's the right mental model. One nuance — `asyncio.gather` vs a `for` loop with `await`:

```python
# concurrent waiting — all tasks run at the same time
results = await asyncio.gather(
    process_invoice(inv1),
    process_invoice(inv2),
    process_invoice(inv3),
)

# sequential — each one finishes before the next starts
for inv in invoices:
    result = await process_invoice(inv)
```

With `gather`, all three tasks are kicked off and wait concurrently. With `for + await`, you wait for each one to finish before starting the next.

**Me:** Wait — even with `gather`, the event loop is single-threaded. So it still *initiates* the tasks one by one, right? It's the **waiting** that's concurrent, not the initiation.

**Klover:** That's a sharp catch, and you're right. I should be more precise. The event loop is single-threaded, so it does start each coroutine sequentially. But initiation is near-instant — it's just setting up the task and sending the network request. The heavy part is the waiting, and that's fully concurrent. So practically:

- **Initiation** — sequential, but microseconds per task
- **Waiting** — truly concurrent, all tasks in-flight at the same time
- **Manifest updates** — serialized by the lock

For 100 invoices, you're not waiting 100× the LLM latency. You're waiting roughly 1× (plus lock overhead for manifest writes).

**Me:** That's the bit that makes async powerful for this use case. Cool — I think I've got it.

---

*Impromptu session: Feb 4, 2026. Practical async patterns for real-world pipelines — race conditions, locks, and concurrency.*
