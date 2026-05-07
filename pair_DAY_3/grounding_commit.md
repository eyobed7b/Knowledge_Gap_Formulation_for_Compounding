# Grounding Commit — Day 3

**Asker:** Eyobed Feleke  
**Artifact edited:** `week10/Automated_Lead_Generation_and_Conversion_System/agent/qualification/icp_classifier.py`  
**Secondary edit:** `week10/Automated_Lead_Generation_and_Conversion_System/agent/main.py`  

---

## What Changed

Added the disqualification guard that fixes bug P-15. In `icp_classifier.py`, removed the
hardcoded `disqualified=False` from both return paths (lines 119 and 136) and replaced them
with actual evaluation logic against the documented disqualification criteria from
`seed/icp_definition.md` (anti-offshore founder public stance, competitor client, layoff
>40%). In `main.py`, added an early-return guard immediately after ICP classification:

```python
if classification.disqualified:
    log.info(
        "prospect_suppressed",
        company=req.company_name,
        reason=classification.disqualify_reason,
    )
    await upsert_contact(
        email=req.contact_email or settings.staff_sink_email,
        name=req.contact_name or req.company_name,
        company=req.company_name,
        properties={"icp_segment": "disqualified",
                    "disqualify_reason": classification.disqualify_reason},
        settings=settings,
    )
    return {"status": "suppressed", "reason": classification.disqualify_reason}
```

## Why This Layer

The fix lives in the scaffolding, not the model, because the disqualification decision is
already made correctly by the rule-based ICP classifier — the classifier has all the signal
data it needs. Switching to tool-calling would have added per-call model latency (~5–15s),
additional token cost (~$0.0005/call), and introduced the possibility of the model
hallucinating a suppress or compose decision based on ambiguous context. Understanding the
token-level difference between tool-calling and JSON mode from the Day 3 explainer made
this choice clear: tool-calling gives the model routing agency on decisions the scaffolding
cannot make; this is not one of those decisions. The memo's "Known Limitations" section
was also updated to remove P-15 from the unresolved list and document the fix layer with
this rationale.
