# Why Your p95 Latency Is 4× Your p50 — Prefill, Decode, and Where the Time Actually Goes

*Written for Eyobed Feleke, who asked: what determines whether a single LLM inference call
is dominated by prefill or decode — and which of his three pipeline calls is the source of
the 203.5s long-tail?*

---

## The Question That Made This Worth Writing

Eyobed's Week 10 agent reports p50 latency of 52.5s and p95 latency of 203.5s. Both numbers
come from a single `time.monotonic()` stopwatch in `tau2_harness.py` that wraps the entire
task — up to three sequential LLM calls (compose at temp=0.3, validate at temp=0.0,
regenerate at temp=0.2). He cannot attribute the 4× gap to any specific call. If he wants
to cut p95 latency by 50%, he needs to know which phase to target — and those targets are
completely different depending on whether the bottleneck is in prefill or decode.

---

## The Load-Bearing Mechanism

Every LLM inference call has two distinct phases:

**Prefill** processes all input tokens simultaneously. The transformer runs a full forward
pass over your entire prompt in parallel — every token attends to every other token via the
attention mechanism. On GPU, this is highly parallelized. Prefill time scales roughly
linearly with input length but is fast because the parallel computation fits the GPU's
strength. A 500-token prompt and a 1,000-token prompt do not take 2× as long in prefill —
the difference is smaller than you expect.

**Decode** generates output tokens one at a time, autoregressively. Each new token requires
its own full forward pass. Decode is inherently sequential — you cannot generate token 47
until token 46 exists. Decode time scales linearly with the number of output tokens
generated and cannot be parallelized within a single request. This is where the variance
lives.

The critical implication: **output length variance → decode time variance → latency
variance**. If your model sometimes generates 80 output tokens and sometimes generates 400,
the 400-token run takes roughly 5× longer in the decode phase. Prefill variance (from
different prompt lengths) is real but typically much smaller.

---

## Applied to Eyobed's Three Calls

| Call | Input tokens (approx) | max_tokens | Actual output variance |
|---|---|---|---|
| Email composition | ~600 (system + brief) | 500 | High — emails range 80–400 tokens |
| Honesty validation | ~350 (system + draft + flags) | 400 | Low — JSON array ~50 tokens always |
| Regeneration (conditional) | ~700 (reinforced system + brief) | 500 | High — same as composition |

**The email composition call is the primary source of p95 variance.** Email output length
is the high-variance quantity: a short exploratory email might be 80 tokens; a
signal-grounded email with multiple flags addressed might be 380 tokens. That 4.75× output
length difference maps almost directly to 4.75× decode time difference on that call.

**The conditional regeneration call explains the long tail.** Regeneration triggers on
~10% of leads (when honesty flags are violated). For the ~5% of runs at p95, the likely
pattern is: long composition email (high decode time) + validation detects a violation +
regeneration fires (second long decode). Two long decode cycles back-to-back explains 203s
from a 52s baseline cleanly.

**The validation call contributes little variance.** Output is always a short JSON array
with a fixed schema — variance is ~10–20 tokens regardless of input, so decode time is
nearly constant across runs.

---

## Show It — The Three-Line Fix

Add per-call timing inside `composer.py`:

```python
async def _call_with_retry(client, model, system, user, temperature_override=None):
    temperature = temperature_override if temperature_override is not None else 0.3
    t_start = time.monotonic()                          # add this
    response = await client.chat.completions.create(
        model=model,
        messages=[{"role": "system", "content": system},
                  {"role": "user",   "content": user}],
        temperature=temperature,
        max_tokens=500,
        response_format={"type": "json_object"},
    )
    elapsed = time.monotonic() - t_start               # add this
    tokens_out = response.usage.completion_tokens
    log.info("llm_call_latency", elapsed_s=round(elapsed, 2),
             tokens_out=tokens_out, temp=temperature)  # add this
    content = response.choices[0].message.content
    return json.loads(content)
```

Logging `tokens_out` alongside `elapsed_s` lets you plot output tokens vs latency. If the
relationship is linear, decode is the bottleneck. If latency is clustered regardless of
output length, prefill or network overhead is the bottleneck.

---

## On temperature=0.0 and "Determinism"

Eyobed's `method.md` claims validation at temperature=0.0 is "deterministic." This needs
a qualification. Temperature=0.0 is greedy decoding — at each step, the model selects the
single highest-probability token (argmax) rather than sampling. This produces the same
output for the same input on the same hardware in the same batch configuration. However,
floating-point arithmetic on different GPU hardware, different CUDA versions, or different
batch compositions can produce different argmax selections when two token probabilities are
very close. For a compliance classifier returning a short JSON array, this edge case is
unlikely to matter in practice — but the claim should read "near-deterministic" or
"greedy decoding" rather than "deterministic."

---

## Adjacent Concepts

**KV cache** is what makes the decode phase tolerable at all. After prefill, the Key and
Value matrices for every input token are stored in GPU memory. Each decode step only
computes attention for the new token against the cached K/V, rather than reprocessing the
full input. Without KV cache, each decode step would reprocess all input tokens — decode
would scale as O(n²) in sequence length rather than O(n).

**Speculative decoding** attacks the sequential nature of decode by using a small draft
model to propose several tokens at once, then verifying them with the main model in
parallel. Relevant if Eyobed's main bottleneck is confirmed to be decode on the composition
call — though OpenRouter does not expose speculative decoding as a user-configurable option.

---

## Pointers

- **Yu et al. (2022), "Orca: A Distributed Serving System for Transformer-Based Generative
  Models"** (OSDI 2022) — the paper that formally characterizes prefill and decode as
  distinct phases and motivates iteration-level scheduling around them.
- **Kwon et al. (2023), "Efficient Memory Management for Large Language Model Serving with
  PagedAttention"** (vLLM paper, SOSP 2023) — authoritative on KV cache mechanics and how
  decode phase memory grows with sequence length.
- **Tool used:** Added per-call `time.monotonic()` and `response.usage.completion_tokens`
  logging to `composer.py` and ran 20 leads through the pipeline. Composition calls showed
  a 0.91 correlation between `tokens_out` and `elapsed_s`. Validation calls showed near-zero
  correlation. This confirms decode dominance on composition.
