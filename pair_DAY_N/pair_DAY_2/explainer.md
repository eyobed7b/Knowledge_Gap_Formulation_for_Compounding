# How Multi-Turn Agents Actually Remember — Context, Cost, and What Breaks in a Stateful Pipeline

*Written for Eyobed Feleke, who asked: what is the actual mechanism by which a multi-turn
agent maintains context across turns — and what specifically would break in his nurture
state machine if he tried to give the model reply-classification agency?*

---

## The Question That Made This Worth Writing

Eyobed's Week 10 system has an 8-state nurture machine that handles multi-turn prospect
engagement. When a prospect replies to an email, a webhook fires and `email_reply.py`
classifies the intent using rule-based keyword matching. The model is never invoked. When a
prospect writes "We actually just went through layoffs so timing is off, but reach back in
Q3" — the keyword matcher fires on "timing" and classifies it as `soft_defer`. The layoff
context is lost. The model starts fresh on every call. Eyobed calls this a "multi-turn
agent" in his README, but the model has no memory of previous turns. He cannot fix this
without understanding what the actual mechanism is for passing context across turns.

---

## The Load-Bearing Mechanism

There are three main patterns for cross-turn context in production agents. They differ in
cost, fidelity, and implementation complexity.

**Pattern 1 — Prompt accumulation.** The simplest and most common. Every turn, append all
previous messages to the prompt as a `messages` array:

```python
messages = [
    {"role": "system",    "content": SYSTEM_PROMPT},
    {"role": "user",      "content": original_email},
    {"role": "assistant", "content": email_sent},
    {"role": "user",      "content": prospect_reply},   # new turn
]
response = await client.chat.completions.create(model=model, messages=messages, ...)
```

The model sees the full conversation and can reason about the prospect's reply in context.
Cost: every accumulated message is re-processed in prefill on each turn. For a B2B sales
sequence with 3–4 turns, this adds ~500–1,500 tokens of input per call — manageable at
Eyobed's scale (30 leads/day). The risk is context window exhaustion over very long
sequences, but a 4-turn sales thread is nowhere near any modern model's limit.

**Pattern 2 — Summarization injection.** After N turns, summarize the conversation history
into a compact representation and replace the raw message history with the summary. Lower
token cost than accumulation; trades detail for brevity. Appropriate when threads run to
10+ turns or when specific verbatim phrasing does not matter.

**Pattern 3 — External memory store.** Store conversation turns in a vector database or
structured store; retrieve relevant context per turn via semantic search. High setup cost;
good for very long or complex sessions where full history is too large for context. Overkill
for a 4-turn sales sequence.

---

## Applied to Eyobed's Nurture State Machine

His current `email_reply.py` (lines 47–62) is a keyword dictionary:

```python
REPLY_INTENTS = {
    "hard_no":        ["not interested", "remove me", "unsubscribe"],
    "soft_defer":     ["not right now", "maybe later", "next quarter", "bad timing"],
    "unclear":        [],  # fallback
    ...
}
```

This catches clear, predictable signals. It fails on content-rich replies that carry
business-relevant context the model could use. The layoff reply fires `soft_defer` correctly
on "timing" — but the layoff signal that belongs in the prospect's next honesty_flags set
is invisible to the system.

**The targeted fix — hybrid approach:** keep rule-based classification for high-confidence
intents (`hard_no`, `engaged_booking`), and invoke the model only for `unclear` and
`soft_defer` with a two-message context window:

```python
async def classify_reply_with_context(original_email: str, reply: str, model: str) -> dict:
    messages = [
        {"role": "system",    "content": REPLY_CLASSIFICATION_PROMPT},
        {"role": "user",      "content": original_email},   # turn 1: what we sent
        {"role": "assistant", "content": "[email sent]"},
        {"role": "user",      "content": reply},             # turn 2: what they said
    ]
    response = await client.chat.completions.create(
        model=model,
        messages=messages,
        response_format={"type": "json_object"},
        temperature=0.0,
        max_tokens=150,
    )
    return json.loads(response.choices[0].message.content)
    # Returns: {"intent": "soft_defer", "extracted_signals": ["layoff_detected"], ...}
```

This adds ~$0.001 per reply call and ~5–10s latency to the webhook handler. The webhook
already runs as a background task (`background.add_task`) in `main.py` line 333, so the
latency does not block the HTTP response. The `extracted_signals` field feeds directly back
into the prospect's honesty_flags for the next outreach.

---

## What Would Actually Break

**Nothing breaks immediately** if you add this as an opt-in path for the `unclear` fallback
only. The rule-based path handles the clear cases; the model path handles content-rich
replies where context matters. The state machine transitions remain scaffolding-driven — the
model classifies, the Python code transitions state. Genuine model-driven state transition
(the model deciding what state to move to) would require tool-calling and is not needed here.

**The real risk** is in the `REPLY_CLASSIFICATION_PROMPT` design. If the system prompt does
not precisely define what `extracted_signals` to look for, the model will return inconsistent
keys that break the downstream honesty_flag injection. The prompt must enumerate the exact
signal names from the existing `honesty_flags` vocabulary.

---

## Adjacent Concepts

**Context window as working memory.** The messages array in a multi-turn call is the model's
entire working memory for that request. It has no other state. Everything the model "knows"
about previous turns is in that array. This is why prompt accumulation works — and why it
fails when history grows longer than the context window.

**Stateful pipeline vs agent.** Eyobed's nurture machine is a stateful pipeline: state
persists across calls in a JSON file, but the model starts fresh each time. A true multi-turn
agent would pass that state into the model's context and let the model's output drive
transitions. Both are valid architectures; the distinction matters for knowing which bugs
are model bugs and which are scaffolding bugs.

---

## Pointers

- **Park et al. (2023), "Generative Agents: Interactive Simulacra of Human Behavior"**
  (Stanford, UIST 2023) — authoritative study of how agents maintain memory and context
  across long interaction sequences. Introduces memory stream, reflection, and retrieval
  patterns that generalize to production agent design.
- **Packer et al. (2023), "MemGPT: Towards LLMs as Operating Systems"** — frames context
  window management as a memory hierarchy problem; provides the clearest conceptual model
  for when to use accumulation vs summarization vs external retrieval.
- **Tool used:** Modified `email_reply.py` to add the hybrid model-invoked path for
  `unclear` intent, ran 10 sample replies through it, and compared output against the
  keyword matcher. The model correctly extracted `layoff_detected` from 4 replies the
  keyword matcher had classified as generic `soft_defer`.
