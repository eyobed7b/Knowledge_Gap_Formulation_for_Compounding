# Sign-Off — Day 3

**Asker:** Eyobed Feleke  
**Gap closure status:** CLOSED  

---

Before reading the explainer I could not distinguish between what `response_format:
json_object` and the `tools` parameter actually cause the model to do differently at the
token level. I was using both terms interchangeably in my README and memo, and I had no
basis for deciding whether the P-15 disqualification bug required a model-layer fix or a
scaffolding-layer fix. I would have reached for tool-calling because it sounded more
"agentic" — adding latency and cost to a decision the rule-based classifier already makes
correctly.

After reading the explainer, I understand that JSON mode constrains output format without
changing the model's generation process or giving it any routing capability, while
tool-calling injects tool definitions into the model's context and allows it to generate a
tool-name token as a first-class routing decision. The distinction clarifies exactly when
each mechanism is appropriate: tool-calling belongs on decisions where the model has
context the scaffolding cannot access; scaffolding guards belong on deterministic
rule-based decisions. P-15 is the second kind. I can now defend the one-line Python fix to
a senior engineer and explain why it is the correct layer — not a shortcut.
