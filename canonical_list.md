# Canonical Reading List — Week 12 Contribution

**Author:** Eyobed Feleke · eyobed@10academy.org  
**Date:** 2026-05-10  
**Cohort:** 10Academy TRP Cohort  

*Annotated papers, tools, and patterns worth every Forward-Deployed Engineer reading before
shipping an LLM-based system. Organized by the gap they close, not by date.*

---

## Papers

### Inference and Serving

**Yu et al. 2022 — "Orca: A Distributed Serving System for Transformer-Based Generative Models" (OSDI)**  
*Gap closed: Why p95 latency is 4× p50 in sequential LLM pipelines.*  
Orca gives the clearest published treatment of the prefill/decode phase split. Prefill processes
all input tokens in parallel (fast, scales sub-linearly); decode generates one token at a time
(slow, scales linearly with output length). The key insight for agents: decode time is
controlled by `max_tokens` ceiling and actual output variance, not input length. Any pipeline
where output length varies widely (email composition, open-ended generation) will show
long-tail latency from decode, not prefill. Read Section 3 for the phase characterization;
skip the cluster scheduling sections unless you run your own serving infrastructure.  
*Practical action: Add per-call `tokens_out` logging. Compute Pearson correlation with elapsed
time per call. The call with ρ > 0.85 is your p95 bottleneck.*

**Kwon et al. 2023 — "Efficient Memory Management for Large Language Model Serving with PagedAttention" (SOSP)**  
*Gap closed: What prefix caching is and why it may or may not be active on your provider.*  
PagedAttention introduced the KV cache management scheme that underlies most prefix caching
implementations. The relevant insight: prefix caching requires (a) identical token sequences in
the prefix across calls, (b) a provider that implements the cache (not all OpenRouter backends
do), and (c) the cached prefix to still be in memory when the next call arrives. For agents
with static system prompts sent on every call: if prefix caching is active, you pay prefill
cost only on the first call; subsequent calls skip it. If it is not active, your cost estimate
is conservative but latency estimate is pessimistic. Read Section 2 (background) and Section 4
(PagedAttention design). Skip Section 5 (implementation details) unless you are writing a
serving engine.  
*Practical action: Check whether your provider documents prefix caching. If not, measure:
send the same prompt twice with a 1-second gap and compare token counts in the usage field.*

---

### Multi-Turn Memory and Agent Architecture

**Park et al. 2023 — "Generative Agents: Interactive Simulacra of Human Behavior" (UIST)**  
*Gap closed: What "multi-turn agent" means mechanically, and what a system needs to actually
maintain cross-turn context.*  
The Generative Agents paper is the clearest demonstration that agents without an explicit memory
mechanism repeat themselves, lose prior signals, and fail at tasks requiring cross-turn
coherence. The architecture separates memory stream (append-only log), retrieval (recency +
importance + relevance), and reflection (synthesized summary). For B2B sales agents: a prospect
who mentioned layoffs in turn 1 will have that signal lost by turn 3 unless the agent
explicitly stores and retrieves it. Read Section 3 (agent architecture) and Section 5
(evaluation). The virtual-world simulation is the demo, not the point.  
*Practical action: Audit your state machine for signals that are captured in turn N but not
surfaced to the model in turn N+1. "Multi-turn" without prompt accumulation is just sequential
single-turn inference.*

**Packer et al. 2023 — "MemGPT: Towards LLMs as Operating Systems"**  
*Gap closed: How to manage context when conversation history exceeds the context window.*  
MemGPT frames the context window as main memory and external storage as disk, with explicit
paging operations. For most B2B sales agents (3-4 turns per prospect), prompt accumulation
(appending all turns to the messages array) is sufficient and far simpler. MemGPT becomes
relevant when threads span many turns or when the same agent must maintain persistent knowledge
across hundreds of prospects in the same session. Read the introduction and Section 2
(memory hierarchy); the OS analogy is load-bearing, not decorative.  
*Practical action: Estimate your max messages array size per prospect thread. If it fits in
16K tokens, use prompt accumulation. If not, use MemGPT's summarization approach.*

---

### Tool Use and Structured Output

**Schick et al. 2023 — "Toolformer: Language Models Can Teach Themselves to Use Tools" (NeurIPS)**  
*Gap closed: What tool-calling is at the token level, and why the layer choice (model vs
scaffolding) matters for debugging.*  
Toolformer shows that tool invocation emerges from training the model to generate special
tokens (`[API call]`, `[API return]`) within the generation stream. Modern tool-calling APIs
work the same way: the model generates a tool-name token as a routing decision; the scaffolding
catches it and executes the tool. The key FDE insight: if your model is generating the wrong
tool calls, the bug is in training or prompting (model layer). If the model generates the right
call but your scaffolding ignores the result or hardcodes a value, the bug is in Python (your
layer). Read Section 2 (data augmentation for tool use) and Section 4 (experiments). Skip
the self-supervised training procedure unless you are building a Toolformer variant.  
*Practical action: When debugging a tool-calling pipeline, first print the raw model output
before JSON parsing. If the tool name is correct in the raw output and wrong after parsing,
the bug is in your scaffolding.*

---

### Preference Optimization

**Rafailov et al. 2023 — "Direct Preference Optimization" (NeurIPS)**  
*Gap closed: What β controls in the preference loss, and why it is a global knob not a
per-category strictness setting.*  
DPO derives β from the KL-constrained RLHF objective. The loss is:
`L = -log σ( β · (log p_θ(chosen) - log p_θ(rejected)) - β · (log p_ref(chosen) - log p_ref(rejected)) )`  
β is the inverse temperature of the implicit reward model — higher β pulls the trained policy
closer to the reference model's prior. The critical FDE insight: β is not a category-specific
strictness setting. Raising it does not make the judge stricter on disqualification tasks
specifically; it makes the judge more conservative on all tasks, which may help overall but
cannot fix a structural length confound. Read Section 2 (deriving DPO from RLHF) and Section 3
(understanding the DPO objective). Section 4 (experiments) is less relevant for practitioners.  
*Practical action: Before changing β, run the cheapest diagnostic first — recalibrate the
inference threshold. β is expensive to sweep because it requires retraining.*

**Meng et al. 2024 — "SimPO: Simple Preference Optimization with a Reference-Free Reward" (NeurIPS)**  
*Gap closed: Why plain CPO produces PASS bias on tasks with asymmetric output lengths, and how
length normalization and a target margin fix it.*  
SimPO replaces DPO's reference-model reward with a length-normalized reward (average log-prob
per token) and adds a target margin γ requiring chosen to beat rejected by at least γ, not
just marginally. The practical consequence for a binary judge trained on short PASS verdicts
(~11 tokens) and long FAIL verdicts (~54 tokens): plain CPO's summed log-prob reward
systematically over-rewards PASS because shorter sequences accumulate less negative
log-probability per token. Length normalization corrects this; γ prevents the model from
satisfying the loss with minimal margins. Read Section 3 (SimPO objective) and Section 4
(connection to DPO). The empirical results in Section 5 are on chat tasks, but the length
normalization argument applies to any binary classifier trained on outputs of unequal length.  
*Practical action: Before enabling SimPO, equalize output lengths in your training pairs:
pad or truncate FAIL verdicts to match PASS length, or explicitly check that your pair builder
enforces length balance. Enabling length normalization without this step can make bias worse.*

---

## Tools and Patterns

**Per-call `tokens_out` + `time.monotonic()` logging — decode bottleneck diagnosis**  
Three lines per LLM call in your pipeline:
```python
t0 = time.monotonic()
response = client.chat.completions.create(...)
elapsed = time.monotonic() - t0
tokens_out = response.usage.completion_tokens
```
Compute Pearson correlation between `tokens_out` and `elapsed` across 30+ runs. ρ > 0.85
on a specific call identifies the decode bottleneck. This is the cheapest instrument for
explaining p95 latency gaps. Add to any pipeline before claiming a latency number.

**Prompt accumulation with two-turn messages array — cheapest multi-turn context**  
For 3-4 turn prospect threads, the cheapest multi-turn memory mechanism is appending all
prior turns to the messages array:
```python
messages = [
    {"role": "system", "content": SYSTEM_PROMPT},
    {"role": "user", "content": original_email},
    {"role": "assistant", "content": "[email sent]"},
    {"role": "user", "content": reply},
]
```
The model's only working memory is the messages array. No external store needed for short
threads. Prefill cost grows linearly with history length; monitor for context window overflow
at scale.

**`false_pass_rate_on_expected_fail` metric in scoring evaluator — PASS bias quantification**  
Primary diagnostic for any binary judge that shows excessive PASS decisions on tasks that
should fail:
```python
expected_fail = [r for r in valid if r.get("expected_pass") is False]
false_pass_rate = sum(1 for r in expected_fail if r.get("passed") is True) / len(expected_fail)
```
Break down by category to identify which task types drive the bias. A judge with overall
accuracy 92% can still have 18% false-negative rate on a specific category if that category
is small. Run this before any retraining.

**Output length diagnostic in training pair builder — SimPO length normalization prerequisite**  
Before enabling length normalization in any preference-trained classifier, compute average
output lengths for positive (PASS) and negative (FAIL) training pairs:
```python
pass_lengths = [len(tokenizer.encode(p["chosen"])) for p in pairs if is_pass_verdict(p)]
fail_lengths = [len(tokenizer.encode(p["rejected"])) for p in pairs if is_fail_verdict(p)]
ratio = mean(pass_lengths) / mean(fail_lengths)  # target: < 1.5×
```
A ratio > 2× indicates structural length asymmetry. Equalize before training or the length-
normalized reward will amplify the bias rather than correct it.

**Swap experiment for LLM-as-judge position bias detection**  
Present the same pair in A-B and B-A order and measure the fraction of cases where the judge
disagrees with itself:
```
disagreement_rate = |A-B judgments ≠ B-A judgments| / total_pairs
```
Disagreement above ~15% indicates position bias. For rubric-based evaluators: fix presentation
order in all evaluations and document it. For pairwise preference judges: only trust results
where disagreement rate is below 15%.

**Scaffolding-first debugging protocol — layer attribution before model changes**  
When an LLM pipeline produces wrong outputs, check Python before touching model config:
1. Print raw model output before any parsing or routing
2. Check hardcoded values, fixed return paths, and unconditional assignments in scaffolding
3. Verify that the model's actual output matches what the scaffolding is supposed to act on

Most "model bugs" in production systems are scaffolding bugs. Model changes are expensive
(retraining) or slow (prompt iteration); scaffolding fixes take minutes. Attribute the bug
to the correct layer before acting.
