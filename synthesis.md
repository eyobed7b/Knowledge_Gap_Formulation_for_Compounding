# Week 12 Synthesis — Knowledge Gap Formulation for Compounding

**Author:** Eyobed Feleke · eyobed@10academy.org  
**Date:** 2026-05-10  
**Cohort:** 10Academy TRP Cohort  

---

## What This Week Required

Weeks 10 and 11 produced two systems: an automated lead generation agent and a preference-trained compliance judge. Week 12 required something harder — reading those systems as a hostile reviewer, naming the places where the writing papers over mechanisms I had not actually internalized, and closing those gaps through paired daily research. By Day 5 I had named five gaps in my own work, researched five gaps named by partners, published five blog posts and five tweet threads, and committed concrete edits to six Week 10 and 11 artifacts. What follows is the account of what changed and why.

---

## Five Gaps I Named

**Gap 1 — Day 1: I could not decompose my p95 latency.**

My Week 10 τ²-Bench baseline reported p50=52.5s and p95=203.5s, measured by a single `time.monotonic()` stopwatch in `tau2_harness.py` wrapping three sequential LLM calls. I described this as "pipeline latency" with no attribution to any specific call. The gap: I had no framework for predicting whether a latency optimization should target input length (prefill) or output length (decode). Understanding that decode scales linearly with output tokens — and prefill is highly parallelized — let me predict that the email composition call (max_tokens=500, high output variance) is the p95 bottleneck. Adding per-call `tokens_out` logging confirmed a 0.91 Pearson correlation between output tokens and elapsed time on the composition call. The `memo.md` cost-per-task figure was also corrected from a single number to a prefill+decode decomposition.

**Gap 2 — Day 2: I called my system a "multi-turn agent" without knowing what that means mechanically.**

My Week 10 nurture state machine handles multi-turn prospect engagement through a fixed 8-state machine where reply intent is classified by rule-based keyword matching. The model never sees previous turn content. When a prospect replied mentioning layoffs, my classifier fired `soft_defer` on "timing" and the layoff signal was lost. The gap: I did not know the mechanism by which a model maintains cross-turn context. Understanding prompt accumulation — the messages array as the model's only working memory — let me implement a hybrid reply classifier that invokes the model on ambiguous replies with a two-turn context window. The README was corrected from "multi-turn agent" to "stateful pipeline with model-invoked reply classification."

**Gap 3 — Day 3: I could not choose the right layer for the P-15 disqualification bug.**

My Week 10 `icp_classifier.py` hardcodes `disqualified=False` on every return path. The fix was documented as "1 engineering hour" but I could not say at which layer — model or scaffolding — without understanding what the `tools` parameter actually does at the token level versus what `response_format: json_object` does. Understanding that JSON mode constrains output format while tool-calling generates tool-name tokens as a routing decision made the layer choice clear: the ICP classifier already makes the disqualification decision correctly; no model agency was needed. The scaffolding guard was added to `main.py` and `icp_classifier.py`.

**Gap 4 — Day 4: I could not interpret what pass@1=0.1333 actually says about my agent.**

My τ²-Bench baseline reports pass@1=0.1333 (4/30 tasks) against a published ceiling of ~42%. I cited this as a baseline number without being able to say what pass@1 measures differently from pass@k, or whether my agent's failures were systematic (same tasks always failing) or stochastic (random task failures). Understanding that pass@1 measures single-attempt reliability while pass@k (k>1) measures capability under sampling let me re-examine my traces: the agent fails the same 26 tasks consistently, indicating systematic capability gaps rather than sampling variance. This changes the Week 10 improvement roadmap from "more compute" to "fix specific failure modes."

**Gap 5 — Day 5: I did not know whether prefix caching was active on my OpenRouter calls.**

My Week 10 agent makes up to three sequential LLM calls per lead, each with a static ~500-token system prompt. I reported $0.00345/task cost without knowing whether the system prompt was being cached across calls. Understanding prefix caching mechanics — that the KV cache for a common prompt prefix can be reused across calls if the prefix is identical and the provider supports it — let me verify that OpenRouter routes DeepSeek V3 calls to providers that do not guarantee prefix caching. This means my cost estimate is conservative but the latency estimate may be pessimistic: without prefix caching, the static system prompt is re-processed in prefill on every call.

---

## Five Gaps I Researched

**Gap 6 — Day 1 (Partner's question): How does speculative decoding produce speedup without changing output quality?**

My partner's Week 10 agent used GPT-4o-mini for latency-sensitive steps but could not explain why a smaller draft model speeds up inference from a larger model. Researching this required understanding autoregressive decode at the token level: the draft model proposes k tokens in parallel; the large model verifies them in a single forward pass (also parallel); accepted tokens are kept, rejected tokens are resampled. The speedup comes from amortizing the large model's forward pass over multiple tokens when the draft model is correct. For agents with predictable output patterns (structured JSON, fixed-format verdicts), draft acceptance rates are high and speedups of 2-3× are achievable.

**Gap 7 — Day 2 (Partner's question): What does LLM-as-a-judge position bias actually look like in practice and how do you detect it?**

My partner's Week 11 judge evaluation compared models by presenting model A first in the prompt. Understanding position bias — judges systematically favor whichever response appears first — required running a real swap experiment: present A-B and B-A for the same pair and measure agreement rate. Agreement below ~85% signals position bias. For my Week 11 Tenacious-Bench judge, this revealed that the LLM judge dimension in `scoring_evaluator.py` was always presenting the prospect brief before the email body — a fixed ordering that may systematically favor certain email structures.

**Gap 8 — Day 3 (Partner's question — Nuhamin): Why does β=2.0 in CPO produce PASS bias on disqualification tasks?**

My partner's question forced me to understand CPO β as a global reward-scale knob, not a per-category strictness setting. The structural root cause of the PASS bias was output length asymmetry: PASS verdicts (~11 tokens) accumulate higher per-token log-probability than FAIL verdicts (~54 tokens), creating a confound the raw CPO loss does not correct. The fix is not simply raising β — it requires either switching to SimPO's length-normalized reward with a target margin, or equalizing output lengths in training pairs before enabling length normalization. This research grounded four concrete edits to the Week 11 training artifacts.

**Gap 9 — Day 4 (Partner's question): How does LoRA rank interact with task complexity, and when does higher rank stop helping?**

My partner had set LoRA rank=32 for a generation task without understanding what the rank bound actually constrains. Researching this required understanding that LoRA decomposes ΔW=BA where rank r bounds how many independent update directions the adapter can express. For classification tasks (like my Week 11 judge), rank=8 is typically sufficient; rank=16 adds no measurable benefit because the decision boundary shift requires few directions. The partner's rank=32 was borrowed from a reasoning-task recommendation and was wasting adapter parameters that could have been better spent on more training pairs.

**Gap 10 — Day 5 (Partner's question): What does "structured output" actually do at the token level?**

My partner's production system used constrained decoding for structured output but could not explain the difference between regex-based token masking and grammar-based constrained decoding. Understanding that regex masking suppresses invalid tokens at each step (local constraint, can produce locally valid but globally invalid JSON) versus grammar-based decoding (globally valid by construction) let the partner replace a brittle regex constraint with outlines-based grammar decoding, eliminating a class of malformed output bugs.

---

## The Most Surprising Thing I Learned

The most surprising thing this week was that **the most consequential bugs in Weeks 10 and 11 were scaffolding bugs, not model bugs** — and I had been misattributing them to model limitations.

P-15 (disqualified hardcoded to False) was a scaffolding bug. The PASS bias in the CPO judge was a training data construction bug (output length asymmetry in `build_simpo_pairs.py`), not a model inference problem. The rule-based reply classifier missing layoff signals was a data coverage bug, not a model capability problem. In every case the model was doing exactly what the scaffolding and training data told it to do. Week 12 forced me to read the systems at the layer where the actual decision was being made — and that layer was almost always Python, not the transformer.

---

## Canonical Reading List and Tool List

*Full annotated entries in `canonical_list.md`.*

**Papers:**
- Yu et al. 2022 "Orca" (OSDI) — prefill/decode phase characterization
- Kwon et al. 2023 "PagedAttention/vLLM" (SOSP) — KV cache mechanics
- Park et al. 2023 "Generative Agents" (UIST) — multi-turn memory mechanisms
- Packer et al. 2023 "MemGPT" — context window as memory hierarchy
- Schick et al. 2023 "Toolformer" (NeurIPS) — tool invocation as generation tokens
- Rafailov et al. 2023 "DPO" (NeurIPS) — β in preference optimization
- Meng et al. 2024 "SimPO" (NeurIPS) — length normalization and target margin

**Tools and patterns:**
- Per-call `tokens_out` + `time.monotonic()` logging — decode bottleneck diagnosis
- Prompt accumulation with two-turn messages array — cheapest multi-turn context
- `false_pass_rate_on_expected_fail` metric in scoring evaluator — PASS bias quantification
- Output length diagnostic in training pair builder — SimPO length normalization prerequisite
- Swap experiment for LLM-as-judge position bias detection
