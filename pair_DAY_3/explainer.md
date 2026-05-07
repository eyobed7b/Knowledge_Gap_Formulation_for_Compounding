# What the Model Actually Does When You Give It Tools

*Written for Eyobed Feleke, who asked: what actually happens at the token level when a model
uses tool-calling versus JSON mode — and which layer should own the disqualification routing
decision in his Week 10 agent?*

---

## The Question That Made This Worth Writing

Eyobed's Week 10 system calls `response_format: {"type": "json_object"}` on every LLM
invocation. His `icp_classifier.py` has a `disqualified` field that is hardcoded to `False`
on every return path — so anti-offshore founders and competitor clients get fully composed
outreach emails. The documented fix is "1 engineering hour." But the question he could not
answer: should the fix live in the Python scaffolding (an `if disqualified: return` guard),
or does fixing it properly require switching to tool-calling so the model can route the
decision itself?

To answer that, you need to know what the model is actually doing differently in each case.

---

## The Load-Bearing Mechanism

**JSON mode** is a constraint on the output, not a change in how the model reasons. When you
pass `response_format: {"type": "json_object"}`, the API tells the model (via a system-level
instruction appended to your prompt) that its response must be valid JSON. The model then
generates tokens exactly as it always does — left to right, one token at a time, sampling
from its output distribution — but the decoding is constrained so that any token sequence
that would produce invalid JSON is suppressed. The model has no awareness of tools, no
branching choices, no ability to "decide" to do something other than generate the JSON
structure your prompt described.

**Tool-calling** is structurally different. When you pass a `tools` parameter, the API
injects a representation of your tool definitions into the model's context — in Anthropic's
format, as a specially formatted system block; in OpenAI's format, as a serialized function
schema. The model now has two valid completion types available: it can generate a normal text
response, or it can generate a tool-call structure. Here is what that looks like in the raw
message:

```python
# JSON mode — model generates this, scaffolding decides what to do with it
{"subject": "Re: AI staffing", "body": "...", "variant": "signal_grounded"}

# Tool-calling — model generates this, API intercepts before returning to caller
{
  "type": "tool_use",
  "name": "compose_email",       # model chose this tool
  "input": {"prospect_id": "..."}
}

# Or the model could choose:
{
  "type": "tool_use",
  "name": "suppress_prospect",   # model chose this instead
  "input": {"reason": "anti-offshore founder signal detected"}
}
```

The critical difference: under tool-calling, the model generates the tool name as output
tokens. The model is doing something closer to a classification decision at generation time —
its training has shaped it to emit `"compose_email"` or `"suppress_prospect"` based on the
context it sees. Under JSON mode, the model never generates a choice token. It generates the
content your prompt asked for, and the Python scaffolding decides what happens next.

---

## Show It

Here is the actual API call difference for Eyobed's composition step:

```python
# Current: JSON mode — model generates email, scaffolding always proceeds
response = await client.chat.completions.create(
    model=model,
    messages=[{"role": "system", "content": SYSTEM_PROMPT},
              {"role": "user",   "content": user_prompt}],
    response_format={"type": "json_object"},
    temperature=0.3,
    max_tokens=500,
)

# Tool-calling alternative — model chooses whether to compose or suppress
response = await client.chat.completions.create(
    model=model,
    messages=[{"role": "system", "content": SYSTEM_PROMPT},
              {"role": "user",   "content": user_prompt}],
    tools=[
        {"type": "function", "function": {
            "name": "compose_email",
            "description": "Compose outreach email for a qualified prospect",
            "parameters": { ... }
        }},
        {"type": "function", "function": {
            "name": "suppress_prospect",
            "description": "Suppress outreach. Use when prospect matches disqualification criteria: anti-offshore public stance, competitor client, layoff >40%.",
            "parameters": {"reason": {"type": "string"}}
        }},
    ],
    tool_choice="required",
)
# Model output now contains tool_calls[0].function.name — either "compose_email" or "suppress_prospect"
```

Under tool-calling, the model reads the disqualification criteria from the tool description
and makes the routing call itself. Under JSON mode, the scaffolding must make that call
before ever invoking the model.

---

## The Adjacent Concepts That Make This Land

**Tool description quality matters enormously.** The model chooses tools based on the
`description` field. "Suppress outreach when prospect matches disqualification criteria" needs
to be precise enough that the model calls it on anti-offshore signals and does not call it on
genuine soft-defer replies. Vague descriptions produce wrong tool selections — which is
exactly the failure mode Eyobed already has in his rule-based reply classifier.

**Scaffolding is not a failure mode.** The question "should the model own this decision?" is
not always answered by tool-calling. For deterministic, rule-based routing decisions —
especially safety-critical ones like disqualification — scaffolding is often the right layer.
A `disqualified=True` check in Python is zero-cost, always-correct, and not subject to model
hallucination. Tool-calling adds latency, adds cost per call, and introduces the possibility
that the model misreads the context and fails to suppress a disqualified prospect. The reason
tool-calling exists is for decisions where the model has information the scaffolding does not
— like reading the prospect's reply and deciding whether it is a soft defer or a genuine
objection.

---

## The Answer to the P-15 Question

The P-15 fix belongs in the scaffolding. The `disqualified` field is set by the rule-based
ICP classifier, which already has all the information needed to make the decision. Adding one
guard in `main.py` before `compose_outreach_email` is called:

```python
if classification.disqualified:
    log.info("prospect_suppressed", reason=classification.disqualify_reason)
    return {"status": "suppressed", "reason": classification.disqualify_reason}
```

That is the 1-engineering-hour fix the memo documented. Switching to tool-calling for this
decision would add model latency, cost, and hallucination risk to a decision that is already
made correctly by the classifier. Tool-calling would be the right choice if you wanted the
model to detect disqualification signals that the rule-based classifier misses — but that is
a different, more expensive system design.

---

## Pointers

- **Schick et al. (2023), "Toolformer: Language Models Can Teach Themselves to Use Tools"**
  (Meta AI) — the paper that established that models can learn to invoke tools by generating
  special API call tokens. The mechanism is the foundation of modern tool-calling.
- **Anthropic Tool Use Documentation** — authoritative specification of the `tools` parameter
  format, how tool-use blocks appear in model output, and how the API routes tool calls back
  to the caller. Primary source for implementation.
- **Tool used:** Ran both call types against `claude-haiku-4-5` on a sample prospect brief to
  compare raw response structure under JSON mode vs tool-calling. JSON mode response was 312
  tokens; tool-calling response with two tools defined was 287 tokens (the tool choice token
  replaced the full JSON body).

One follow-on question worth writing next: what makes a tool description that the model
reliably selects on? The suppress_prospect description above would need adversarial testing
to confirm it does not fire on soft-defer replies.
