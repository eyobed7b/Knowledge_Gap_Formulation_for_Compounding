# Morning Call Summary (Partner) — Day 3

**Written by:** [Partner Name]  
**Confirmed by:** Eyobed Feleke  
**Date:** 2026-05-07  
**Duration:** 25 minutes  

---

The partner's original draft question asked "what is the difference between CPO and SimPO
and which should I use?" — too broad and not grounded in a specific failure. Eyobed pushed
back: "You already trained the model. What specific behavior in the held-out results tells
you something is wrong with the config?" That interrogation surfaced the 18% false-negative
rate on disqualification tasks and the fact that the training script is named `train_simpo.py`
but uses `CPOTrainer` without `loss_type="simpo"` — meaning the SimPO target margin was
never applied. The question was rewritten to name three specific candidate fixes (tune β,
add γ via `gamma_beta_ratio`, adjust inference threshold) and ask how to distinguish between
them diagnostically. Eyobed confirmed the sharpened question is unambiguous: a satisfying
answer would give a diagnostic plan that identifies which lever to pull first without
requiring a full retrain to test each hypothesis.
