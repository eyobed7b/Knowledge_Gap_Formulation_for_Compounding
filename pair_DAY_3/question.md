# Day 3 Question — Agent and Tool-Use Internals

**Asker:** Eyobed Feleke  
**Date:** 2026-05-07  
**Topic:** Agent and tool-use internals  

---

## The Question

My Week 10 system is described as an "agent" that enriches prospects, classifies them,
composes outreach emails, validates honesty flags, and routes to HubSpot and Cal.com. But
examining the code, every step except email composition and honesty validation is executed by
scaffolded Python — fixed-order, no model input. The model is invoked with
`response_format={"type": "json_object"}` on two steps only; no `tools` or `functions`
parameter is ever passed. HubSpot and Cal.com are triggered unconditionally by the
scaffolding, regardless of what the model returns. Bug P-15 (`disqualified` hardcoded to
`False` in `icp_classifier.py` lines 119 and 136) is invisible to the model — it cannot
detect or route around it.

**What actually happens at the token level when a model uses tool-calling (the `tools`
parameter) versus JSON mode (`response_format: json_object`), and what would I concretely
need to change in my Week 10 pipeline to give the model genuine agency over the
disqualification routing decision that bug P-15 currently hard-codes away?**

---

## Why This Gap Matters in My Work

I call my system an "agent" in the README and memo, but the code shows it is a linear
orchestrator: enrichment → classify → compose → validate → route, with the model only
producing text at steps 4 and 5. The scaffolding decides everything else.

The clearest symptom is P-15: the disqualification check is implemented as a Pydantic field
in `ICPClassification` but the field is never set to `True` on any code path. A prospect
flagged as anti-offshore or already a competitor client still receives a fully composed,
honesty-flag-compliant email — because the scaffolding never checks `disqualified` before
calling `compose_outreach_email`.

I do not know whether fixing this requires:

- Switching from `response_format: json_object` to a `tools` call where the model explicitly
  selects a "suppress_prospect" tool vs a "compose_email" tool
- Or simply adding an `if classification.disqualified: return` guard in the Python scaffolding
  (which requires no model change at all)

I cannot reason about this trade-off because I do not know what the model is actually doing
differently under tool-calling vs JSON mode — whether it is generating different tokens,
whether the API intercepts something mid-generation, or whether tool-calling is just a
different prompt format that produces a structured response the same way JSON mode does.

This matters beyond P-15: if I ever want the model to decide *which* channel to use, or
*whether* to route to Cal.com at all based on signal confidence, I need to know what mechanism
gives the model genuine branching agency rather than just producing text the scaffolding acts on.

---

## Connection to Existing Artifacts

| Artifact | Location | What depends on understanding this |
|---|---|---|
| JSON mode usage | `week10/.../agent/outreach/composer.py` line 142 | No `tools` param — is this a real limitation or just a style choice? |
| P-15 bug | `week10/.../agent/qualification/icp_classifier.py` lines 119, 136 | Fix requires knowing if model or scaffolding should own routing |
| P-15 documentation | `week10/.../memo.md` lines 106–116 | States fix is "1 engineering hour" but doesn't say at which layer |
| Pipeline structure | `week10/.../README.md` lines 4–35 | Calls this an "agent" — does the architecture justify that label? |
| Cal.com trigger | `week10/.../agent/main.py` lines 124–128 | Triggered unconditionally — no model input on whether to include |
| HubSpot trigger | `week10/.../agent/main.py` lines 70–81 | Triggered unconditionally — always writes regardless of disqualification |

The concrete edit this would enable: implement the P-15 fix at the correct layer (scaffolding
guard vs model tool-call) with a defended rationale in `memo.md`, and correct the README
description of the system from "agent" to whatever label accurately reflects the
scaffolding-to-model ratio.

---

## What Would Constitute a Satisfying Answer

An explainer that closes this gap would let me:

1. Describe what tokens the model actually generates differently when a `tools` parameter is
   present vs when only `response_format: json_object` is set
2. State when tool-calling gives the model genuine branching agency and when it is still just
   prompt formatting with the scaffolding making the real decision
3. Implement the P-15 fix at the right layer with one defended code change
4. Write one accurate sentence in the README that describes what kind of system this actually
   is — pipeline, orchestrator, or agent — and why

The answer does not need to cover MCP servers or reasoning-trace tokens in depth — those are
adjacent to the load-bearing gap.
