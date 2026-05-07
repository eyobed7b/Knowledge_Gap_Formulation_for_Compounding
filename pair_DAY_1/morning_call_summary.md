# Morning Call Summary — Day 1

**Partners:** Eyobed Feleke (asker) + [Partner Name]  
**Date:** 2026-05-04  
**Duration:** 22 minutes  

---

The original draft question asked "what causes high latency in LLM inference?" which the
partner immediately flagged as too broad — it could be answered by a Wikipedia paragraph on
transformers. The partner pushed back: "You have actual numbers. What specifically do those
numbers tell you that you cannot explain?" That forced the asker to name the 4× gap between
p50 (52.5s) and p95 (203.5s) and admit the measurement is a single stopwatch around three
sequential LLM calls with no per-call breakdown. The question was rewritten to be grounded
in that specific gap: which of the three pipeline calls is the bottleneck, and what
determines whether a call's latency is dominated by prefill (input tokens) or decode (output
tokens). The partner confirmed the sharpened question is unambiguous — a satisfying answer
would let the asker predict which call to optimize first and write the instrumentation to
verify it.
