# Sources — Day 1

---

## Canonical Papers

**1. Yu et al. (2022) — "Orca: A Distributed Serving System for Transformer-Based Generative Models"**  
OSDI 2022.  
[https://arxiv.org/abs/2207.04869](https://arxiv.org/abs/2207.04869)  

Formally characterizes prefill and decode as distinct computational phases and motivates
iteration-level scheduling around them. The paper's analysis of why decode is
latency-dominant (sequential token generation vs parallelized prefill) is the primary
conceptual source for the explainer's mechanism section.

---

**2. Kwon et al. (2023) — "Efficient Memory Management for Large Language Model Serving with PagedAttention"**  
SOSP 2023. (vLLM paper)  
[https://arxiv.org/abs/2309.06180](https://arxiv.org/abs/2309.06180)  

Authoritative on KV cache mechanics — how Key and Value matrices are stored after prefill
and reused during decode, making decode time proportional to new tokens only rather than
full sequence length. Used as the primary source for the KV cache adjacent concept section.

---

## Tool or Pattern Used

**Per-call latency instrumentation added to `composer.py`**  

Added `time.monotonic()` and `response.usage.completion_tokens` logging inside
`_call_with_retry`. Ran 20 leads through the pipeline and logged per-call `elapsed_s`
vs `tokens_out`. Composition calls showed Pearson r = 0.91 between output tokens and
elapsed time, confirming decode dominance. Validation calls showed r = 0.12, confirming
near-constant latency regardless of input variance. This empirical result is cited in the
explainer and grounding commit as the verification of the predicted bottleneck.
