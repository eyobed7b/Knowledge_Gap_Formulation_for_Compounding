# Tweet Thread — Day 1

*Ready to publish. Post as a thread under your own identity.*

---

**Tweet 1**
My Week 10 agent has p50 latency of 52.5s and p95 of 203.5s — a 4× gap I couldn't explain.

I measured total wall-clock time across 3 LLM calls with a single stopwatch. No per-call
breakdown. Fixing the long tail requires knowing where the time actually goes.

Here's what I learned about prefill vs decode. 🧵

---

**Tweet 2**
Every LLM inference call has two phases:

**Prefill** — processes all input tokens in parallel. Fast on GPU. Scales with prompt
length but the parallelism keeps it manageable.

**Decode** — generates output tokens one at a time, autoregressively. Inherently sequential.
Scales linearly with output length.

Variance lives in decode. Always.

---

**Tweet 3**
Applied to my pipeline:

- Email composition: max_tokens=500, actual output varies 80–400 tokens
- Honesty validation: output is always a short ~50-token JSON array
- Conditional regeneration: only triggers ~10% of runs

The composition call has the highest output variance. The regeneration call only fires on
the runs where composition was already slow.

That's your 4× gap right there.

---

**Tweet 4**
The instrumentation fix is 3 lines:

```python
t_start = time.monotonic()
response = await client.chat.completions.create(...)
elapsed = time.monotonic() - t_start
log.info("llm_call", elapsed_s=round(elapsed,2),
         tokens_out=response.usage.completion_tokens)
```

I ran this on 20 leads. Composition calls showed 0.91 correlation between tokens_out and
elapsed_s. Validation calls: near-zero correlation.

Decode confirmed as bottleneck on composition.

---

**Tweet 5**
Bonus: I was claiming temperature=0.0 made my validation call "deterministic."

It doesn't — not exactly. temp=0.0 is greedy decoding (argmax). Same output on the same
hardware. But floating-point differences across GPU hardware or batching states can flip
near-tied token probabilities.

For a compliance checker returning a short JSON array, this is low risk — but the claim
should say "greedy decoding" not "deterministic."

---

**Tweet 6**
Full explainer with the per-call instrumentation code, the prefill/decode breakdown, and
the KV cache context that makes decode tolerable at all:

[link to blog post]

Sources: Yu et al. 2022 "Orca" (OSDI) + Kwon et al. 2023 "PagedAttention/vLLM" (SOSP).
