---
layout: post
title: "ReAct Pattern"
description: "How ReAct agents interleave reasoning and acting, failure modes, CoT as computation, and agent pattern tradeoffs"
categories: agents
date: 2026-02-04
---

## What is ReAct?

**Klover:** ReAct stands for **Reasoning + Acting** — explicitly interleaved. The idea is simple: instead of just having an LLM answer a question in one shot, you let it think out loud, take actions (like calling tools), observe the results, and then think again. It's a loop.

**Me:** so it's like chain-of-thought but with actual tool use mixed in?

**Klover:** Exactly. The loop looks like this:

1. **Thought** — the model reasons about what to do next
2. **Action** — the model calls a tool (search, calculator, API, etc.)
3. **Observation** — the system injects the real result from that tool
4. Repeat until the model decides it has enough info → **Final Answer**

The key word is "interleaved." It's not "think everything through then act" — it's think a little, act a little, observe, adjust.

**Me:** how does the system know when the model wants to call a tool vs just talking?

**Klover:** Stop sequences. The model is trained (or prompted) to output in a specific format — like `Action: search["weather in Singapore"]`. The system watches for that pattern, intercepts it, actually runs the tool, then injects the result back as `Observation: 32°C, sunny`. The model never actually executes anything. It just writes text that looks like a tool call, and the system does the rest.

**Me:** so the LLM doesn't know it's calling real tools? it just thinks it's generating text?

**Klover:** Right. From the model's perspective, it's just doing next-token prediction. The magic is in the scaffolding around it — the system that intercepts, executes, and injects.

## A practical example

**Klover:** Say you ask a ReAct agent: "Should I go to the beach this Saturday?"

The agent might do:

- **Thought:** I need to check the weather for Saturday.
- **Action:** `weather_forecast("Saturday")`
- **Observation:** Saturday: 34°C, sunny, UV index 11, no rain.
- **Thought:** Weather looks great, but UV is very high. Let me check tide times too.
- **Action:** `tide_info("Saturday")`
- **Observation:** High tide at 2pm, low tide at 8am.
- **Thought:** Good beach weather, but go early to avoid peak UV and catch low tide. I have enough info now.
- **Final Answer:** Yes! Saturday looks perfect — sunny, 34°C. Go in the morning around 8-9am when the tide is low and UV isn't brutal yet.

Multiple steps, each one informed by the last. The model decides when it has enough.

**Me:** got it. so it's basically an LLM in a loop.

**Klover:** That's the most concise way to put it. An LLM in a loop, with tool access and the ability to decide when to stop. That's what an "agent" is at its core.

**Me:** wait — are you a ReAct agent right now?

**Klover:** Pretty much. I reason about what you're asking, I can call tools (search, read files, run code), I observe the results, and I decide what to say next. You're watching the pattern in action.

## Failure modes

**Me:** what goes wrong with this?

**Klover:** Two big failure modes.

**First: infinite loops.** The model might keep calling tools without converging on an answer. It searches for something, doesn't find what it wants, rephrases, searches again, rephrases again… forever. The fix is simple — set a **max iteration limit**. After N steps, force it to give a final answer with whatever it has.

**Second: context overflow.** Every Thought, Action, and Observation gets appended to the conversation. After 10-15 steps, that's a LOT of tokens. Eventually you hit the context window limit and the model just breaks. But the real problem starts before you even hit the limit.

## Context degradation

**Me:** what do you mean "before the limit"?

**Klover:** The context window might be 128K tokens, but performance degrades long before you fill it. Think about how attention works — the softmax function spreads attention weight across all tokens. More tokens = each one gets a thinner slice of attention.

**Me:** like a pie with more slices.

**Klover:** Exactly. A pie cut into 10 slices vs 1000 slices. Each individual slice becomes almost meaningless. The model can still "see" all the tokens technically, but its ability to actually focus on any one of them gets diluted.

And there's another effect on top of this — **"lost in the middle."** Research shows models disproportionately attend to the beginning and end of context, and neglect the middle. So if your key reasoning happened in step 5 of a 15-step chain, it might as well not exist.

**Me:** so long ReAct chains get hit twice — diluted attention AND lost-in-the-middle?

**Klover:** Exactly. Double hit. Which is why real systems don't just naively append everything.

## The fix: summarize between steps

**Klover:** The practical fix is to **compress context between steps**. Instead of keeping the raw Thought/Action/Observation history, you periodically summarize it.

Think of it as a sliding window — you keep the last few exchanges verbatim for detail, and everything before that gets compressed into a summary. LangChain has `ConversationSummaryBufferMemory` that does exactly this: recent messages stay raw, older ones get summarized by an LLM.

**Me:** but the summary is lossy right? the summarizer might drop something important.

**Klover:** Yes. That's the tradeoff. It's lossy compression. The summarizer might decide a detail was unimportant and drop it, but then 10 steps later that detail turns out to be critical. There's no perfect solution — it's a tradeoff between context freshness and information retention.

## Why the Thought step matters — CoT as computation

**Me:** why does the explicit Thought step matter so much? couldn't the model just go straight to Action?

**Klover:** This is one of the deeper insights. Chain-of-thought isn't just a prompting trick — it's **extra computation**.

Without CoT, the model gets one forward pass to produce an answer. That's it — the entire computation happens in the fixed depth of the transformer network. With CoT, every token the model generates becomes new context for the next token. Each generated token is the product of a full forward pass, and it feeds into the next one. You're effectively chaining multiple forward passes together.

**Me:** but isn't each forward pass independent? the only difference is it has new context from the previous tokens. it's not like the model "remembers" — it just sees more text.

**Klover:** You're right about the mechanism — yes, each pass is independent, and the only connection is through the context. But think about the effect: you've turned a fixed-depth computation (one pass through N transformer layers) into a **variable-depth computation** (multiple passes, each informed by the output of the last). The mechanism is context. The effect is more computation. You're chaining fixed-depth into variable-depth.

**Me:** so the extra computation is encapsulated in the generated tokens themselves.

**Klover:** Yes — each token is a compressed artifact of a full forward pass. All those layers of attention and feedforward processing get distilled into a single token, which then becomes context for the next pass. The output tokens are essentially **working memory** — scratch space where the model does its intermediate computation. Without CoT, there's no scratch space. One shot, one pass.

## ReAct vs other patterns

**Me:** are there alternatives to ReAct?

**Klover:** Two main ones worth comparing.

**Pure tool calling** — no explicit Thought step. The model just emits tool calls directly. Faster, fewer tokens, but the model doesn't reason about what to do before doing it. Works fine for simple tasks, falls apart for complex multi-step reasoning.

**Plan-first** — the model creates a full plan upfront (step 1, step 2, step 3…), then executes it. Advantage: it can see dependencies and think ahead. "I need X before I can do Y." Disadvantage: plans are brittle. They don't survive contact with reality. What if step 2 returns unexpected data that invalidates step 3?

**Me:** so ReAct is more adaptive but short-sighted, and plan-first thinks ahead but breaks easily?

**Klover:** That's a clean summary. ReAct is **adaptive but myopic** — it only thinks one step ahead. Plan-first is **strategic but brittle**.

The practical sweet spot is a **hybrid: plan and replan**. Make a rough plan, start executing, but be willing to replan when observations don't match expectations. This is essentially what frameworks like LangGraph enable — you define a graph of steps with conditional edges, so the agent can branch and loop based on what actually happens.

**Me:** so nobody actually uses pure ReAct in production?

**Klover:** Not in its textbook form, no. Real systems take the core insight — interleave reasoning with acting — and add structure around it. Planning, memory management, guardrails, fallbacks. But the ReAct loop is the foundation all of it builds on.

---

*Session: Feb 4, 2026. First exposure — covered full ReAct pattern including failure modes, CoT mechanics, and agent pattern comparisons.*
