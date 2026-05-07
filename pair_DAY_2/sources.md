# Sources — Day 2

---

## Canonical Papers

**1. Park et al. (2023) — "Generative Agents: Interactive Simulacra of Human Behavior"**  
Stanford University. UIST 2023.  
[https://arxiv.org/abs/2304.03442](https://arxiv.org/abs/2304.03442)  

Authoritative study of how agents maintain memory and context across long interaction
sequences. Introduces the memory stream, reflection, and retrieval patterns. The paper's
distinction between in-context memory (the messages array) and external memory (retrieved
summaries) is the conceptual foundation for the three-pattern taxonomy in the explainer.

---

**2. Packer et al. (2023) — "MemGPT: Towards LLMs as Operating Systems"**  
UC Berkeley.  
[https://arxiv.org/abs/2310.08560](https://arxiv.org/abs/2310.08560)  

Frames context window management as a memory hierarchy problem analogous to OS virtual
memory — main context as RAM, external storage as disk. Provides the clearest conceptual
model for when prompt accumulation is sufficient vs when summarization injection or
external retrieval is needed. Used to justify prompt accumulation as the right pattern for
Eyobed's scale (30 leads/day, 3–4 turns per prospect).

---

## Tool or Pattern Used

**Hybrid reply classifier — model-invoked path for ambiguous intents**  

Modified `email_reply.py` to add a two-message context window (original email + prospect
reply) passed to the model for `unclear` and low-confidence `soft_defer` classifications.
Ran 10 sample replies through both the keyword matcher and the model-invoked path. The
model correctly extracted `layoff_detected` from 4 replies the keyword matcher had
classified as generic `soft_defer`. The `extracted_signals` field in the model's JSON
output maps directly to the existing `honesty_flags` vocabulary, enabling downstream
flag injection into the next outreach call.
