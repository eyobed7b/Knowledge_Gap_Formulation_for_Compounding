# Tweet Thread — Day 3

*Ready to publish. Post as a thread under your own identity.*

---

**Tweet 1**
I called my Week 10 system an "agent." Then I found a bug where a hardcoded `False`
meant disqualified prospects were getting outreach emails. To fix it, I had to understand
something I'd been glossing over: what does the model actually *do* differently with
`tools` vs `response_format: json_object`?

The answer changed how I thought about where bugs like this belong. 🧵

---

**Tweet 2**
JSON mode (`response_format: json_object`) is a **constraint on output**, not a decision
mechanism.

The model generates tokens exactly as normal. The API just suppresses any token that would
break valid JSON. The model never "chooses" anything — your scaffolding gets the JSON and
decides what to do with it.

Your Python code is making all the routing decisions.

---

**Tweet 3**
Tool-calling is structurally different.

When you pass a `tools` parameter, the model sees your tool definitions in context and can
generate a **tool-call token** instead of a text response:

```
{"type": "tool_use", "name": "suppress_prospect", "input": {...}}
```

The model chose that tool name as an output token. The API intercepts it. Your scaffolding
never saw a JSON body — it saw a routing decision the model made.

---

**Tweet 4**
So which layer should own a disqualification routing decision?

The scaffolding — if the decision is rule-based and deterministic (classifier already has
the answer).

The model via tool-calling — if the decision requires reading context the classifier
can't see (e.g., the prospect's reply email).

My P-15 bug was a scaffolding bug. One Python guard fixed it:

```python
if classification.disqualified:
    return {"status": "suppressed"}
```

No tool-calling needed. No model latency added.

---

**Tweet 5**
The insight that generalizes:

Tool-calling gives the model genuine branching agency — but only for decisions where the
model has information the scaffolding doesn't.

For deterministic, safety-critical routing (disqualification, compliance suppression),
scaffolding is cheaper, faster, and not subject to model hallucination.

Knowing the difference is what separates a pipeline from an agent.

---

**Tweet 6**
Full explainer with the API call comparison, the Toolformer paper that established the
mechanism, and the exact fix to the P-15 bug:

[link to blog post]

Sources: Schick et al. 2023 "Toolformer" (Meta AI) + Anthropic Tool Use docs.
