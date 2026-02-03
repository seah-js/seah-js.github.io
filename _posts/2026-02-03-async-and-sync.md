---
layout: post
title: "Async & Sync"
description: "What async actually means, event loops, threads, the GIL, and why using requests inside async code is a trap"
categories: fundamentals
date: 2026-02-03
---

This is a Socratic-style learning session between me and [Klover](https://github.com/openclaw/openclaw), my AI assistant. Instead of reading docs or watching tutorials, I learn by explaining concepts in my own words and getting challenged on the gaps. The conversation format is the point — it captures the actual "aha" moments as they happen.

In this session, we start from the basics of async vs sync and work our way through event loops, threads, processes, the GIL, and a common async pitfall that silently kills Python web servers.

---

## The basics

**Me:** Async is after initiating a process, a worker can then attend to another process while waiting for the initial process to complete. Sync is when the worker has to wait for the process to complete fully before it can move on to something else.

**Klover:** Solid foundation. But what is that "worker" in practice? When you write async Python or JavaScript code, what's actually doing the work and deciding "oh, this task is waiting, let me switch to something else"?

**Me:** Sounds like there is something delegating the tasks and the worker is something like a processing engine?

**Klover:** Good instinct — there IS something delegating. But first, think about what kinds of tasks would benefit from async. When you call an API, query a database, or read a file — what do all these have in common?

**Me:** They might take a long time for something to return? Or maybe the job itself doesn't require the worker.

**Klover:** Second answer is the key insight. These are all **I/O operations** — the CPU isn't doing any work while waiting. It's just sitting there waiting for a network response, a disk to return data, or a database to finish a query. The CPU literally has nothing to do. That's wasted time in sync mode.

## I/O-bound vs CPU-bound

**Klover:** There are two types of "slow" tasks:
1. **I/O-bound** — waiting on something external (network, disk, database)
2. **CPU-bound** — actually doing heavy computation (training a model, image processing, crunching numbers)

For which type does async make sense, and why wouldn't it help the other?

**Me:** Async makes sense when it is I/O-bound. Since it is waiting on something, it can attend to something else. Async would not help the other as it is literally doing the job itself and has to take time to crunch out the numbers.

**Klover:** Exactly. This is a really important distinction that a lot of people miss — they think async magically makes everything faster. It doesn't. It only helps when the bottleneck is waiting, not computing.

## The event loop

**Klover:** In JavaScript and Python, the thing doing the delegating is the **event loop**. Picture a single chef in a kitchen. The chef can only do one thing at a time (single-threaded). But when they put something in the oven, they don't stand there staring at it — they go chop vegetables, prep the next dish, etc. When the oven timer dings, they go back and handle it.

The event loop is that chef. It:
1. Runs a task until it hits an I/O wait
2. Parks that task and picks up the next one
3. When the I/O comes back, puts it back in the queue

So if there's only ONE chef (one thread), what happens if one of your tasks is CPU-bound? Like one "dish" requires 10 minutes of non-stop chopping?

**Me:** Everything else pauses.

**Klover:** That's exactly why CPU-bound work in an async single-threaded environment is dangerous. One heavy task **blocks** the entire event loop. You've probably experienced this — ever had a web page freeze when some JavaScript was doing heavy work? That's the event loop being blocked.

## Threads and processes

**Me:** So if async only solves I/O-bound problems, what about CPU-bound work? More threads?

**Klover:** Yes — more threads or more processes. But Python has a quirk called the **GIL** (Global Interpreter Lock).

Python objects in memory have a reference counter that tracks how many things point to them. If two threads modify the same object simultaneously, that counter can get corrupted. The GIL is a brute-force fix: lock the entire interpreter so only one thread runs Python code at a time.

**Me:** So you mean only 1 thread can execute code at one time? What exactly is a thread?

**Klover:** A thread is a sequence of instructions that the CPU can execute — a to-do list the CPU follows step by step. All threads within a process share the same memory. That's why the GIL is needed — shared memory + multiple threads = potential corruption.

Compare:
- **Threads** — multiple to-do lists, sharing one desk (memory). Lightweight but need to be careful.
- **Processes** — completely separate desks, separate memory. Heavier but can't interfere with each other.

**Me:** So multiprocessing means there will be more GILs able to work? More actual threads doing computing?

**Klover:** Spot on. Each process gets its own Python interpreter, its own GIL. 4 processes = 4 GILs = 4 threads executing simultaneously on 4 cores. True parallelism.

The tradeoff: processes don't share memory, so passing data between them requires serialization. Slower than threads just reading the same variable.

## Threading vs async for I/O

**Klover:** Both threading and async solve I/O-bound problems. But what happens if you try to create 10,000 threads for 10,000 concurrent tasks?

**Me:** You can't create 10,000 threads right? It will hit some sort of limit.

**Klover:** Right. Each thread consumes ~1-8MB of stack memory. 10,000 threads = potentially gigabytes of memory.

Async with an event loop? One thread. 10,000 tasks are just lightweight objects in a queue — barely any memory.

Rule of thumb:
- **Few concurrent I/O tasks (tens to hundreds)** → threading is fine
- **Massive concurrent I/O (thousands+)** → async wins
- **CPU-bound** → neither helps, use multiprocessing

## The practical gotcha

**Klover:** You're building a FastAPI endpoint that queries a database and calls an external API. Should it be sync or async?

**Me:** Async. Because most of the actions are done externally. While waiting, it can attend to other tasks.

**Klover:** Now, what's the problem with this code?

```python
async def get_data():
    response = requests.get("https://api.example.com/data")  # sync requests
    return response.json()
```

vs this:

```python
async def get_data():
    async with httpx.AsyncClient() as client:
        response = await client.get("https://api.example.com/data")  # async httpx
    return response.json()
```

**Me:** The first one will have problems. Does `await` only run when the previous dependency is done?

**Klover:** Not quite. `requests.get()` is a **blocking** call. It doesn't release control back to the event loop. The entire event loop freezes while waiting on that one API call. Your FastAPI server that should handle thousands of users is stuck.

`await` is actually a **pause/resume point**. It tells the event loop: "I'm pausing here, go handle other stuff, come back when this is ready." Without `await` (sync calls), there's no pause point. The event loop never gets a chance to switch.

This is one of the most common mistakes in async Python — using sync libraries inside async code. It silently kills your concurrency.

---

*Session: Feb 3, 2026. First exposure — lots of new concepts. Review scheduled for tomorrow.*
