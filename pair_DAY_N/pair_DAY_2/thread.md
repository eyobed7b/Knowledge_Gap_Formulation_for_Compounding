# Tweet Thread — Day 2

*Ready to publish. Post as a thread under your own identity.*

---

**Tweet 1**
I built a "multi-turn agent" for B2B sales outreach. Then I traced through the code and
found the model starts fresh on every single call — no memory of previous turns whatsoever.

When a prospect replied mentioning layoffs, my keyword classifier said "soft defer" and
the model never saw the content.

Here's how multi-turn context actually works. 🧵

---

**Tweet 2**
The model has no persistent memory. Its entire "knowledge" of previous turns is whatever
you put in the messages array.

The simplest mechanism — **prompt accumulation** — just appends prior messages:

```python
messages = [
  {"role": "system",    "content": SYSTEM_PROMPT},
  {"role": "user",      "content": original_email},
  {"role": "assistant", "content": email_sent},
  {"role": "user",      "content": prospect_reply},  # new turn
]
```

The model can now reason about what the prospect actually wrote.

---

**Tweet 3**
Three patterns for cross-turn context, by cost:

1. **Prompt accumulation** — append all prior messages. Simple. Scales to ~10 turns before
   context window pressure. Right for most B2B sales sequences.

2. **Summarization injection** — compress history into a summary after N turns. Cheaper,
   loses verbatim detail. Good for 10+ turn threads.

3. **External memory store** — vector DB retrieval per turn. High setup cost. Overkill
   for a 4-turn sales sequence.

---

**Tweet 4**
The targeted fix for my system — hybrid classifier:

Keep rule-based for high-confidence signals (hard_no, engaged_booking).
Invoke the model only for unclear/ambiguous replies, with 2-message context:

```python
# Pass what we sent + what they said
messages = [original_email_turn, prospect_reply_turn]
# Returns: {"intent": "soft_defer", "extracted_signals": ["layoff_detected"]}
```

Adds ~$0.001 and ~8s per ambiguous reply. Webhook runs async so it doesn't block.

---

**Tweet 5**
The insight that generalizes:

A stateful pipeline persists state in code. An agent passes state into the model's context.

Both are valid. The difference tells you which bugs are model bugs and which are
scaffolding bugs — and which layer owns the fix.

My layoff-reply miss was a scaffolding bug. The model never had the context to catch it.

---

**Tweet 6**
Full explainer with the prompt accumulation pattern, the three context mechanisms compared,
and the exact `email_reply.py` change:

[link to blog post]

Sources: Park et al. 2023 "Generative Agents" (Stanford/UIST) + Packer et al. 2023
"MemGPT."
