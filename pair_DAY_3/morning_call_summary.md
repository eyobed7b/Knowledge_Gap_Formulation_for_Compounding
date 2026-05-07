# Morning Call Summary — Day 3

**Partners:** Eyobed Feleke (asker) + [Partner Name]  
**Date:** 2026-05-07  
**Duration:** 25 minutes  

---

The original draft question was too broad — it asked "what is the difference between JSON
mode and tool-calling" without naming a specific failure the answer would fix. The partner
pushed back: "You're asking a comparison question. What breaks in your system because you
don't know the answer?" That interrogation surfaced the P-15 bug: `disqualified` is hardcoded
to `False` in `icp_classifier.py` lines 119 and 136, and the question was really about
whether fixing it requires switching the model to tool-calling (model-layer fix) or just
adding a Python guard before `compose_outreach_email` is called (scaffolding-layer fix).
The question was rewritten to be diagnostic: it now names the token-level mechanism as the
load-bearing gap, because without understanding what the model generates differently under
`tools` vs `response_format`, the asker cannot choose the right fix layer. Both partners
confirmed the sharpened question is unambiguous and answerable in one explainer.
