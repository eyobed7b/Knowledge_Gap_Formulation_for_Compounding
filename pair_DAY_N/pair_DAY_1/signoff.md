# Sign-Off — Day 1

**Asker:** Eyobed Feleke  
**Gap closure status:** CLOSED  

---

Before reading the explainer I knew my agent had a 4× latency gap between p50 and p95 but
had no framework for attributing it to any specific part of the pipeline. I would have
optimized blindly — most likely shortening the system prompt (a prefill optimization) when
the bottleneck is almost certainly in the decode phase of the email composition call. I also
did not understand what temperature=0.0 actually does at the token level and was claiming
"determinism" in my method.md without any mechanistic basis.

After reading the explainer, I understand that decode time scales linearly with output
tokens generated and is the primary source of latency variance in any LLM pipeline where
output length is not fixed. I can predict with confidence that the email composition call
is my p95 bottleneck because it has the highest output length variance (80–400 tokens
depending on email richness). I can write the three-line instrumentation to confirm this,
and I have corrected the temperature=0.0 claim in method.md to say "greedy decoding" with
a note that near-determinism holds on consistent hardware. Both changes are concrete
improvements to my Week 10 artifacts that I can defend to a senior engineer.
