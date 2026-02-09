---
layout: post
title: "Agent Loops & State Management"
description: "How agents maintain state across steps, manage growing context, and know when to stop"
categories: agents
date: 2026-02-09
---

This is a learning session between me and [Klover](https://github.com/openclaw/openclaw), my AI assistant. New topic today — building on [ReAct]({% post_url 2026-02-08-react-pattern-review %}) and [function calling]({% post_url 2026-02-08-function-calling-and-tool-schemas %}), now we're looking at the loop itself.

---

## How does an agent remember?

**Klover:** You know the ReAct loop: Thought → Action → Observation. But LLMs are stateless — they don't "remember" the previous step. So how does the agent actually maintain context across multiple iterations of the loop?

**Me:** Something tracks the conversation. After each tool call, the observation is appended to the tracked conversation and passed as input to the next LLM call.

**Klover:** Exactly right. The conversation history grows with each step — every Thought, Action, and Observation gets appended, and the entire thing is fed back to the LLM on each iteration. The LLM sees the full trajectory so far. ✅

## The growing context problem

**Klover:** So the history keeps growing. What's the problem with that, and what can you do about it?

**Me:** Hitting the context window is an issue. We can summarise the conversation. But the summarisation can't be too lossy.

**Klover:** Good — context window limits are the main constraint, and summarization is one strategy. The lossy risk is real — the agent might forget a critical detail from step 3 that it needs at step 15.

Other approaches beyond summarization:
- **Sliding windows** — drop the oldest messages
- **Selective retention** — keep only the most relevant steps
- **Retrieval-based** — store older steps externally and retrieve only what's relevant (like RAG but for agent history)

## When does the loop stop?

**Klover:** The agent loops: Think → Act → Observe → repeat. How does it know when to stop? What are the termination conditions?

**Me:** When during Think, the agent has enough to give a conclusive answer. It then ends and outputs the answer.

**Klover:** That's one — **natural termination**. The LLM decides it has enough and generates a final answer instead of another Action. But what if it never decides it's done?

**Me:** Perhaps a maximum number of tool calls.

**Klover:** Yes — **max iterations**. Essential safety net. But there are more. All five termination conditions:

1. **Natural completion** — LLM decides it has the answer ✅ (I got this)
2. **Max iterations** — hard cap on loop steps ✅ (I got this)
3. **Timeout** — wall clock time limit. Slow API calls could mean the agent runs for 30 minutes while technically under the iteration limit ❌
4. **Token/cost budget** — each LLM call costs money. Set a max spend per task ❌
5. **Error threshold** — 3 consecutive tool errors? Something's fundamentally wrong. Stop instead of burning more tokens retrying ❌

I got 2 out of 5. The key insight: **never trust the LLM alone to stop.** Always have orchestrator-level guardrails.

## State beyond conversation history

**Klover:** The conversation history is how state is maintained. But there's more to "state" than just chat messages. What other kinds of state might an agent need to track?

**Me:** *(couldn't think of anything beyond conversation history)*

**Klover:** Think about a flight-booking agent. It's searched for flights, found options, you picked one, now it needs your passport and payment. Extracting "passport number" from message #14 buried in long chat history is messy. What would you build instead?

**Me:** An .env file.

**Klover:** Not quite — I'm talking about something dynamic. It's a **scratchpad or working memory** — a separate structured data store the agent writes to as it goes:

```json
{
  "selected_flight": "SQ321",
  "passenger": "Jay Seah",
  "passport": "...",
  "payment_status": "pending"
}
```

Different from conversation history because:
- **Structured** — not buried in natural language
- **Mutable** — agent updates specific fields
- **Accessible** — no parsing through 20 messages

Other state types: task progress tracking, retrieved context stored separately, execution metadata (iterations, errors, time elapsed).

The big idea: **conversation history is one form of state, but production agents need structured state management alongside it.**

## Scratchpad solves the summarization problem

**Klover:** Your agent is 25 steps deep. Context window is almost full. You summarize steps 1-20 into a paragraph. What's the risk, and how do you design around it?

**Me:** The risk is the loss of important information during summarisation. Perhaps the structured file we talked about previously could help.

**Klover:** Nailed it. The scratchpad survives summarization because it's separate from conversation history. Critical data (selected flight, passport, task progress) lives in structured state, so even if the conversation gets compressed, essential info is preserved. ✅

Other strategies: **hierarchical memory** (recent steps in full, older steps summarized, critical facts in structured state).

---

## Session verdict: Exposure

**What I got:** Conversation history as state mechanism, context window limits + summarization, natural completion + max iterations, scratchpad solving summarization risk.

**What I missed:**
- 3 of 5 termination conditions (timeout, token/cost budget, error threshold)
- Structured scratchpad concept needed significant prompting
- Sliding window and retrieval-based context management

All 5 termination conditions are on the test next review. No excuses.

Next review: February 10.
