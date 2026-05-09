# Day 1 Question — Inference-Time Mechanics

**Asker:** Eyobed Feleke  
**Date:** 2026-05-04  
**Topic:** Inference-time mechanics  

---

## The Question

My Week 10 agent (`deepseek/deepseek-chat-v3-0324` via OpenRouter) runs up to three sequential
LLM calls per lead: email composition (temp=0.3, max_tokens=500), honesty validation
(temp=0.0, max_tokens=400), and conditional regeneration (temp=0.2, max_tokens=500). My
τ²-Bench baseline reports p50 latency of 52.5s and p95 latency of 203.5s — a 4× gap I cannot
explain. I measure total wall-clock time in `tau2_harness.py` with a single `time.monotonic()`
wrap around the entire task; I do not instrument individual call latencies.

**What determines whether a single LLM inference call's latency is dominated by the prefill
phase (processing input tokens) or the decode phase (generating output tokens), and which of
my three pipeline calls is most likely the source of the 203.5s long-tail — and how would I
instrument my pipeline to confirm it?**

---

## Why This Gap Matters in My Work

My latency measurement in [tau2_harness.py](../../../week10/Automated_Lead_Generation_and_Conversion_System/tau2_harness.py)
is a single stopwatch around the full task:

```python
start = time.monotonic()
# ... entire pipeline runs ...
latency = time.monotonic() - start
```

The 4× gap between p50 (52.5s) and p95 (203.5s) means something in ~5% of runs takes
dramatically longer. My candidates are:

- **Decode phase dominance on the composition call** — max_tokens=500 for email drafting,
  but actual output length varies widely. Long emails = more decode steps = more latency.
- **Conditional regeneration call** — only triggered ~10% of the time (when honesty flags
  are violated), which matches the frequency of p95 events. If regeneration adds another
  ~150s, that would explain the gap.
- **Prefill cost on long prospect briefs** — some prospect briefs are significantly longer
  than others (companies with more signals). Prefill scales with input length.
- **OpenRouter routing variance** — different provider backends may have different cold-start
  or queue latencies for the same model.

I cannot currently tell which of these is true because I have no per-call instrumentation.
This matters because if I wanted to cut p95 latency by 50% (from 203.5s to ~100s), I would
target the fix completely differently depending on which phase is the bottleneck.

I also set temperature=0.0 for the validation call and claim it is "deterministic" in
[method.md](../../../week10/Automated_Lead_Generation_and_Conversion_System/method.md).
But I do not actually know what temperature=0.0 does at the token level — whether it is
truly deterministic across different hardware configurations or batching states, and whether
that matters for my validation reliability claim.

---

## Connection to Existing Artifacts

| Artifact | Location | What depends on understanding this |
|---|---|---|
| Latency measurement | `week10/.../tau2_harness.py` lines 57, 128, 168–169 | Single wrap — cannot attribute latency to any specific call |
| Baseline report | `week10/.../baseline.md` | Claims p50=52.5s, p95=203.5s with no decomposition |
| Three-call pipeline | `week10/.../method.md` lines 170–212 | Compose → validate → regenerate, measured as one number |
| Validation temp claim | `week10/.../method.md` line 317 | "validation must not drift" — asserts determinism without mechanism |
| Cost per task | `week10/.../memo.md` line 21 | $0.00345 not split into prefill vs decode cost |

The concrete edit this would enable: add per-call latency instrumentation to `tau2_harness.py`,
decompose the $0.00345 cost figure into prefill and decode components in `memo.md`, and
correct or qualify the "deterministic" claim about temperature=0.0 in `method.md`.

---

## What Would Constitute a Satisfying Answer

An explainer that closes this gap would let me:

1. State what prefill and decode phases are, what each costs in time and money, and which
   scales with which variable (input tokens vs output tokens)
2. Predict which of my three calls is the most likely p95 bottleneck and why
3. Write the three lines of instrumentation code that would confirm the prediction
4. Correct my claim about temperature=0.0 determinism with a mechanistically accurate
   statement

The answer does not need to cover speculative decoding or KV cache in depth — those are
adjacent topics, not the load-bearing gap.
