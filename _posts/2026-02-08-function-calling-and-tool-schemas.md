---
layout: post
title: "Function Calling & Tool Schemas"
description: "How LLM agents actually talk to tools — schemas, structured output, and why the orchestrator matters"
categories: agents
date: 2026-02-08
---

This is a learning session between me and [Klover](https://github.com/openclaw/openclaw), my AI assistant. The idea is simple — Klover teaches me a concept through back-and-forth dialogue, I explain things in my own words, and we go deeper together. This format captures the actual conversation, including mistakes and corrections.

This time: Function calling and tool schemas — how agents actually interact with external tools.

---

## What is a tool schema?

**Klover:** You know the ReAct loop: Thought → Action → Observation. But when the agent picks an Action, how does it know what tools exist and how to call them? What do you think a tool schema is?

**Me:** I assume it details all the available tools the LLM can call. It exposes a "list" of tools and their purposes.

**Klover:** Good intuition but there's more. A tool schema doesn't just list what tools exist — it defines **how to call them**. Like a function signature in code. For each tool: a name, a description (natural language for the LLM), and typed parameters with required/optional flags.

```
name: get_weather
description: Get current weather for a location
parameters:
  location (string, required): City name
  units (string, optional): "celsius" or "fahrenheit"
```

## Where do schemas live?

**Klover:** Where does this schema live? Part of the model's training, or somewhere else?

**Me:** It's not part of training. The schema lives externally — almost like additional prompts fed to the LLM during every invoke.

**Klover:** Exactly. Schemas are injected into the system prompt or a dedicated tools section at runtime. The model was *trained* to understand and follow the format, but the specific tools change per application. That's why you can give Claude a custom tool it's never seen and it works — it learned the format, not the specific tools.

## The full round trip

**Klover:** When the LLM decides to call a tool, what does that actually look like? Walk me through the full round trip.

**Me:** The model decides to call the weather tool → invokes it via the output → weather tool returns an observation → model thinks about the observation.

**Klover:** Right flow, but let's zoom in. The LLM is a text generator — it can't literally call an API. What does it actually produce?

**Me:** I guess it outputs the actual function call with the parameters, which then calls a function.

**Klover:** Specifically, it outputs **structured JSON**:

```json
{
  "tool": "get_weather",
  "parameters": {
    "location": "Singapore",
    "units": "celsius"
  }
}
```

But the critical part: **the LLM doesn't execute anything**. It produces this JSON and stops. Then your **application code** (the orchestrator) takes over:

1. LLM outputs a tool call → stops generating
2. Your code parses the JSON
3. Your code actually calls the weather API
4. Your code feeds the result back as the Observation
5. LLM continues with the next Thought

The LLM is the brain. Your application is the hands.

## Why the orchestrator matters

**Klover:** Why is it important that the orchestrator sits between the LLM and the tool?

**Me:** So you can add safeguards — limiting total tool calls, ensuring tools are called correctly.

**Klover:** Spot on. The orchestrator is your control layer:

- **Validate** — check parameters before executing
- **Rate limit** — cap tool calls to prevent infinite loops
- **Filter** — block certain tools based on context or permissions
- **Log** — record every call for debugging
- **Sanitize** — clean output before feeding it back

Without that middle layer, the LLM has direct access to your systems. Imagine a prompt injection tricking the model into calling `delete_database` — the orchestrator is where you catch that.

## Schema quality matters

**Klover:** What happens if the schema is poorly written? Vague description, wrong parameter requirements?

**Me:** The tool gets called when it shouldn't be, or called wrongly — resulting in bad or failed results.

**Klover:** Exactly. Bad schemas → bad tool calls:

- **Vague description** → model calls the wrong tool or skips the right one
- **Wrong types/requirements** → malformed requests, crashes, garbage output
- **Missing parameter descriptions** → model guesses what "q" or "id" means

Tool schema design is basically **prompt engineering for tools**. Clear names, precise descriptions, correct types — it's the difference between a reliable agent and a flaky one.

---

*First session: Feb 8, 2026. Status: Exposure. Good intuition throughout — connected naturally to ReAct concepts. Review tomorrow.*
