# Day 2 Question — Agent and Tool-Use Internals

**Asker:** Eyobed Feleke  
**Date:** 2026-05-06  
**Topic:** Agent and tool-use internals  

---

## The Question

My Week 10 system handles multi-turn prospect engagement through a fixed 8-state machine in
`nurture.py` (states: new → email_sent → email_replied → sms_sent → call_booked → qualified
→ stalled → closed). When a prospect replies to an email, a webhook fires, and
`email_reply.py` (lines 47–62) classifies the intent using rule-based keyword matching into
one of seven categories (`hard_no`, `soft_defer`, `objection`, `engaged_booking`, `engaged`,
`curious`, `unclear`). The model is never invoked during state transitions. It never sees
conversation history. The scaffolding reads the classification, advances the state, and
decides the next action — without the model reasoning about what the prospect actually wrote.

**What is the actual mechanism by which a multi-turn agent maintains and uses conversation
context across turns — is it prompt accumulation, external memory injection, or something
else — and what specifically would break in my `nurture.py` architecture if I tried to give
the model genuine reasoning agency over reply classification instead of the current
rule-based keyword matcher?**

---

## Why This Gap Matters in My Work

My reply classification in `email_reply.py` is entirely rule-based:

```python
# email_reply.py lines 47–62 (rule-based intent classification)
REPLY_INTENTS = {
    "hard_no":        ["not interested", "remove me", "unsubscribe", "stop"],
    "soft_defer":     ["not right now", "maybe later", "next quarter", "bad timing"],
    "objection":      ["too expensive", "already have", "not a priority"],
    "engaged_booking":["book", "schedule", "calendar", "let's talk"],
    "engaged":        ["tell me more", "interested", "sounds good"],
    "curious":        ["how does", "what is", "can you explain"],
    "unclear":        [],  # fallback
}
```

When a prospect writes "We actually just went through a round of layoffs so the timing
isn't great, but reach back out in Q3" — my classifier puts this in `soft_defer` (matched
"timing"). The model never sees the layoff context, never updates the honesty flags, and
the scaffolding will send a standard follow-up in 30 days with no adjustment.

I describe this as a "multi-turn agent" in my README, but the model has no memory of what
was said in previous turns. Each LLM call starts from a fresh prompt. I do not know:

- Whether "multi-turn" in a real agent means the full conversation history is appended to
  every prompt, or whether there is a separate memory mechanism that summarizes and injects
  context selectively
- What the cost and context-window implications of accumulating full conversation history
  are at Week 10 scale (30 leads/day, potentially 5–7 email turns per lead)
- Whether giving the model reply-classification agency would actually improve outcomes, or
  whether scaffolding is the right layer for this decision and the gap is just in my
  rule-based keyword list

This matters because the nurture state machine is where the system makes decisions that
have real business consequences — routing a `soft_defer` vs `objection` incorrectly means
either burning a warm prospect or wasting follow-up budget on a dead one.

---

## Connection to Existing Artifacts

| Artifact | Location | What depends on understanding this |
|---|---|---|
| State machine definition | `week10/.../agent/outreach/nurture.py` lines 9–22 | 8 states, 7 transitions — all transitions are scaffolding decisions |
| Rule-based reply classifier | `week10/.../agent/webhooks/email_reply.py` lines 47–62 | Keyword matching with no model input |
| Webhook handler | `week10/.../agent/main.py` lines 327–339 | Background task — async, model never called |
| README architecture diagram | `week10/.../README.md` lines 4–35 | Labels this a multi-turn "agent" |
| Memo failure modes | `week10/.../memo.md` | Does not identify the rule-based classifier as a failure mode |

The concrete edit this would enable: either (a) add a model-invoked reply-reasoning step to
`email_reply.py` with a defended description of how conversation history is passed, or (b)
keep the rule-based classifier but document in `memo.md` why the scaffolding layer is the
correct place for this decision and what its known failure cases are — with the layoff-reply
example as a named case.

---

## What Would Constitute a Satisfying Answer

An explainer that closes this gap would let me:

1. Describe the two or three main mechanisms real multi-turn agents use to maintain context
   across turns (prompt accumulation, summarization injection, external memory store) and
   what each costs in tokens and latency
2. Predict whether my 30-leads/day scale makes prompt accumulation viable or whether I need
   a different approach
3. Write the specific change to `email_reply.py` that would invoke the model on incoming
   replies — with the conversation history passed correctly — versus leave it rule-based with
   an honest scope statement
4. Update the README to accurately describe whether this is a multi-turn agent or a
   stateful pipeline

The answer does not need to cover MCP servers or reasoning-trace tokens — the load-bearing
gap is specifically about how context crosses turn boundaries.
