---
layout: post
title: "Function Calling & Tool Schemas — Review"
description: "First review of function calling concepts — what stuck, what didn't, and the orchestrator gaps"
categories: agents
date: 2026-02-09
---

This is a learning session between me and [Klover](https://github.com/openclaw/openclaw), my AI assistant. Spaced repetition review — testing what stuck from the [first session]({% post_url 2026-02-08-function-calling-and-tool-schemas %}).

---

## What does the LLM actually output?

**Klover:** When an LLM "calls a function," what does it actually output, and what does it NOT do?

**Me:** It outputs a JSON table with all the relevant parameters which will be passed to an orchestrator which will parse the JSON and call the actual tool.

**Klover:** Core is right — the LLM outputs structured JSON, it does NOT execute anything itself. The orchestrator handles actual execution. Small note: "JSON table" — it's more accurately a JSON object (key-value pairs), not a table. Minor terminology thing but worth being precise.

## Where do schemas come from?

**Klover:** Where do tool schemas come from? Are they baked into the model during training, or something else? And what does a schema typically contain?

**Me:** It is external to the model. It is passed like a system prompt to the model. A schema usually contains the description of the tool, the required and optional parameters.

**Klover:** Solid. Schemas are injected at runtime, not baked in during training. They contain the tool name, description, and typed parameters (required + optional). ✅

## Why schema quality matters

**Klover:** Why does the quality of your schema descriptions matter so much? What happens if you write vague or poor descriptions?

**Me:** It might cause the LLM to use the tool wrongly which might result in useless or non-relevant outputs from the tool called.

**Klover:** Right — poor descriptions lead to wrong tool selection or bad parameter values. The framing: schema quality is basically **prompt engineering for tools**. Same principle — garbage in, garbage out. ✅

## The orchestrator's role

**Klover:** Beyond just executing the tool call, what else does the orchestrator do? Think about why it's a critical control layer.

**Me:** It acts as a security layer. It ensures that the tool called is allowed for this model or something. Also, it prevents prompt injection from directly calling a tool.

**Klover:** Security is a big one — validation and access control. But I missed the full picture. The complete list:

1. **Validation** — are the params correct/allowed?
2. **Rate limiting** — preventing abuse (throttling API call frequency)
3. **Security/filtering** — blocking unauthorized calls
4. **Logging** — audit trail of what was called, when, with what params

I got 2 out of 4. Missed rate limiting and logging. Rate limiting prevents runaway agents from hammering APIs. Logging is critical for debugging agent behavior and compliance — if something goes wrong 10 steps into an agent loop, logs are how you trace it.

## Connecting to ReAct

**Klover:** How does function calling relate to the ReAct pattern? Where do tool calls fit in the Thought-Action-Observation loop?

**Me:** Tool calls fit in the Action part which is the actual execution of the tool. The Observation is basically the output from the tool call.

**Klover:** Spot on. The Action is where the LLM outputs the tool call JSON, the orchestrator executes it, and the Observation is the raw result fed back to the LLM for the next Thought. ✅

---

## Review verdict: Pass → Developing

**What stuck:** Core concepts solid — LLM outputs JSON (doesn't execute), schemas injected at runtime, schema quality = prompt engineering for tools, clean ReAct connection.

**What I missed:** Rate limiting and logging from the orchestrator responsibilities. Need to lock in all four: validation, rate limiting, security/filtering, logging.

Next review: February 12.
