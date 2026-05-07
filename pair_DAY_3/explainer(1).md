# What the Model Sees When You Give It Tools vs JSON Mode

*Written by [Partner Name] for Eyobed Feleke, who asked: what actually happens at the
token level when a model uses tool-calling vs JSON mode — and which layer should own the
disqualification routing decision that bug P-15 hard-codes away?*

---

## The Question

Eyobed's Week 10 agent passes `response_format: {"type": "json_object"}` on every LLM
call. No `tools` parameter is ever used. His `icp_classifier.py` has a `disqualified`
field hardcoded to `False` on every return path (lines 119 and 136), meaning disqualified
prospects still receive outreach emails. The documented fix is "1 engineering hour" — but
at which layer? Model or scaffolding? Without understanding what tool-calling actually does
differently at the token level, you cannot reason about this trade-off.

---

## The Mechanism

**JSON mode** constrains output format. The API appends a system-level instruction telling
the model its output must parse as valid JSON. The model then generates tokens normally —
left to right, one at a time — but the decoding process suppresses any token that would
break valid JSON. The model has no awareness of branching choices, tools, or routing. Your
Python scaffolding receives the JSON and decides everything that happens next.

**Tool-calling** works fundamentally differently. When you pass a `tools` parameter, the
API injects your tool definitions into the model's context. The model now has two valid
completion types: a regular text response, or a special `tool_use` block where the model
generates the *name of a tool* as output tokens:

```json
{
  "type": "tool_use",
  "name": "suppress_prospect",
  "input": {"reason": "anti-offshore founder signal detected"}
}
```

That tool name is a token the model chose to generate. The model's training has shaped it
to emit `"suppress_prospect"` vs `"compose_email"` based on the context it reads —
including the tool descriptions you provide. The API intercepts this before returning to
your Python code; your scaffolding receives a routing decision, not a JSON body.

---

## The P-15 Answer

The P-15 fix belongs in the scaffolding — one Python guard before `compose_outreach_email`
is called:

```python
if classification.disqualified:
    log.info("prospect_suppressed", reason=classification.disqualify_reason)
    return {"status": "suppressed", "reason": classification.disqualify_reason}
```

This is the right layer because the disqualification decision is already made correctly by
the rule-based ICP classifier. The classifier has all the signal data. Switching to
tool-calling would give the model routing agency over a decision the scaffolding already
makes correctly — adding model latency (~5–15s), token cost (~$0.0005/call), and the
possibility of the model hallucinating a routing decision based on ambiguous context.

Tool-calling belongs on decisions where the model has information the scaffolding does not.
P-15 is not that case. The scaffolding fix is the right engineering call.

---

## When to Use Each

| Mechanism | Use when |
|---|---|
| JSON mode | Model generates content; scaffolding decides what to do with it |
| Tool-calling | Model needs to choose between actions based on context it reads |

The line between "pipeline" and "agent" is roughly this boundary. Eyobed's system is a
pipeline — the model generates content, the scaffolding routes. Making it a true agent
would require tool-calling for decisions where the model's reading of content matters.

---

## Pointers

- **Schick et al. (2023), "Toolformer"** (NeurIPS 2023) — foundational paper on models
  generating API-call tokens mid-generation. Explains why tool invocation can be
  represented as regular tokens in the output stream.
- **Anthropic Tool Use Documentation** — authoritative specification of the `tools`
  parameter format, `tool_use` content block structure, and how the API routes tool calls.
