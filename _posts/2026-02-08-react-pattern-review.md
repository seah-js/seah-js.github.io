---
layout: post
title: "ReAct Pattern — Review"
description: "Review session on the ReAct pattern — testing recall on the T-A-O loop, failure modes, and what Observation actually means"
categories: agents
date: 2026-02-08
---

This is a review session between me and [Klover](https://github.com/openclaw/openclaw), my AI assistant. The idea is simple — Klover quizzes me on something I've learned before, I explain it in my own words, and we go deeper on the parts that stick (or don't). This format captures the actual back-and-forth, including the mistakes.

This time: ReAct pattern review, testing whether the core concepts have stuck since the first session.

---

## Empty results — what happens next?

**Klover:** An agent calls a search tool but gets back an empty result. Walk me through what happens next in the ReAct loop — what does the Thought step look like, and what are two ways it could go wrong?

**Me:** The Thought step would analyse what to do next. One failure: it could call the tool again with the same query, get nothing again, and end up in an endless loop.

**Klover:** That's the first one — infinite retry loop. What's the second?

**Me:** I wasn't sure about the second one.

**Klover:** The second is **hallucination**. The Thought step sees no results and instead of trying a different approach, the agent makes up an answer. It "reasons" that it probably knows enough and generates a confident but fabricated response. Dangerous because the user has no idea the info wasn't grounded in actual tool output.

## What should the agent actually do?

**Klover:** So what should a well-designed ReAct agent do when the search returns nothing?

**Me:** The next reasoning step should recognise that something went wrong — maybe the query was too specific or used wrong terms — and try something else. Or if retrying doesn't work, report back to the user that it couldn't find the information.

**Klover:** Exactly. The Thought should explicitly acknowledge the failure and pivot. The key insight — the Thought step is where the agent decides between **adapting** and **giving up gracefully**. Both are valid. Hallucinating is never valid.

## The T-A-O loop

**Klover:** Quick check: what's the correct order of the loop, what does each letter stand for, and where does the agent decide whether to give a final answer or keep going?

**Me:** T — Think, A — Act, O — Observe. In that order. The Think step is where it decides.

**Klover:** Spot on. At each Thought step, the agent either decides to act (picks a tool, executes it, observes the result) or decides it's done (outputs the final answer and exits the loop). That's why Thought is the most critical step.

## Who generates the Observation?

**Klover:** What's the difference between the Action and the Observation? Specifically — who or what generates each one?

**Me:** Action is an external tool call or API. Observation is done by the LLM on the output of that tool call.

**Klover:** Close but not quite. The Action — yes, the LLM decides the tool call, and the tool executes externally. But the **Observation is the raw output returned by the tool/environment**. It's data coming *in*, not the LLM thinking about it. The LLM's analysis of that output happens in the **next Thought step**.

So the full flow:

- **Thought** → LLM reasons
- **Action** → LLM picks a tool, environment executes it
- **Observation** → Environment returns the result (LLM just receives it)
- **Thought** → LLM analyses the observation, decides next move

The Observation is the only part the LLM doesn't generate. It's the real world talking back.

**Me:** Yeah, that makes sense. I was conflating the observation with the reasoning about it.

## Failure modes recall

**Klover:** Name the failure modes we've covered and give a one-line mitigation for each.

**Me:** Context window overflow — mitigate by summarising. Forever loop — limit the total number of tool calls. Attention dilution — same thing, summarise.

**Klover:** Three for three on unprompted recall. Plus you know hallucination from earlier. And you correctly identified the tradeoff — summarisation helps with both context bloat and attention dilution, but it's lossy compression. You might throw away something that matters later.

---

*Review session: Feb 8, 2026. Stays at developing. Recalled 3 of 4 failure modes (missed hallucination — was prompted). Corrected on Observation being environment output, not LLM reasoning. Next review targets unprompted recall of all 4 failure modes.*
