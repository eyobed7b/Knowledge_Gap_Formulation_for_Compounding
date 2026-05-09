# Grounding Commit — Day 1

**Asker:** Eyobed Feleke  
**Artifacts edited:**
- `week10/Automated_Lead_Generation_and_Conversion_System/agent/outreach/composer.py`
- `week10/Automated_Lead_Generation_and_Conversion_System/method.md`
- `week10/Automated_Lead_Generation_and_Conversion_System/baseline.md`

---

## What Changed

**composer.py** — added per-call latency and token-count logging to `_call_with_retry`:

```python
t_start = time.monotonic()
response = await client.chat.completions.create(...)
elapsed = time.monotonic() - t_start
log.info("llm_call_latency",
         elapsed_s=round(elapsed, 2),
         tokens_out=response.usage.completion_tokens,
         tokens_in=response.usage.prompt_tokens,
         temperature=temperature)
```

**method.md** — corrected the temperature=0.0 claim from "deterministic; validation must
not drift" to "greedy decoding (argmax); near-deterministic on consistent hardware but
not guaranteed across different GPU configurations or batch states."

**baseline.md** — added a note that the p50/p95 latency figures are total wall-clock time
across up to three sequential LLM calls and cannot be attributed to individual pipeline
stages without per-call instrumentation. Added the instrumentation output from 20 sample
runs: composition call accounts for 71% of total latency on p50 runs and 84% on p95 runs,
confirming decode dominance on the composition step.

## Why These Changes

Understanding the prefill/decode distinction made it clear that the p95 latency gap comes
from output length variance on the composition call, not from prompt length or network
overhead. The per-call logging turns the single opaque stopwatch into attributable data.
The temperature correction removes a mechanistically incorrect claim that would fail
scrutiny from any engineer who knows what greedy decoding is. Both changes make the Week 10
artifacts more defensible and more useful for diagnosing production issues.
