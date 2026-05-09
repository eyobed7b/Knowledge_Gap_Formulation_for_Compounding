# Day 3 Question — Training and Post-Training Mechanics

**Asker:** Eyobed Feleke  
**Date:** 2026-05-06  
**Topic:** Training and post-training mechanics  

---

## The Question

If the held-out judge shows PASS bias on disqualification tasks, should I treat that as a β problem, a missing SimPO target-margin problem, or an inference-threshold problem? More specifically, what does β actually control in the preference loss, how does it interact with length-normalized rewards and the implicit target reward margin, and what concrete configuration change would reduce false PASS decisions on disqualification tasks without collapsing the judge's overall calibration?

---

## Why This Gap Matters in My Work

My Week 11 SimPO judge (`Qwen2.5-0.5B-Instruct`, LoRA r=16, trained on 137 preference pairs)
reports an **18% false-negative rate on held-out disqualification tasks** — it passes emails
it should flag as non-compliant. The `hyperparameters.json` records `beta=2.0` under the CPO
loss, and the `memo.md` notes a PASS bias without being able to explain its structural cause.

The gap: I set β=2.0 by convention and noted the bias, but I cannot say whether:

- **β is the root cause** — if too high, it pulls the policy back toward the base model's
  generosity prior, insufficient to overcome the FAIL > PASS preference signal on sparse
  disqualification pairs (~10 training examples)
- **The missing SimPO target margin is the root cause** — my loss is plain CPO
  (`loss_type="cpo"` in `training/train_simpo.py` despite the filename), which has no
  requirement that the chosen log-prob margin exceeds a minimum γ. PASS verdicts (~11 tokens)
  accumulate higher total log-probability than FAIL verdicts (~54 tokens), creating a length
  confound that a target margin would partially correct
- **The inference threshold is the root cause** — the current `score_task()` in
  `scoring_evaluator.py` uses `aggregate_score >= 0.7` as the PASS threshold, and recalibrating
  this number upward might eliminate false passes without any retraining

Without understanding what β actually controls at the loss level — and how it interacts with
length normalization and the target margin — I cannot prioritize the cheapest intervention.
A β sweep is more expensive than threshold recalibration; enabling SimPO length normalization
requires equalizing output lengths in `build_simpo_pairs.py` before retraining. Getting the
order wrong wastes a training run.

---

## Connection to Existing Artifacts

| Artifact | Location | What depends on understanding this |
|---|---|---|
| CPO training config | `week11/.../training/train_simpo.py` line ~35 | `CPO_BETA = 2.0` — set by convention, not analysis |
| Hyperparameters | `week11/.../training/hyperparameters.json` | `"beta": 2.0`, `"loss_type": "cpo"` — β rationale not documented |
| Scoring threshold | `week11/.../scoring_evaluator.py` line 223 | `aggregate_score >= 0.7` — not calibrated against disqualification recall |
| PASS bias report | `week11/.../memo.md` Skeptic's Appendix | Names bias, no structural cause, no prioritized fix |
| Pair builder | `week11/.../training/build_simpo_pairs.py` | PASS verdicts avg 11 tokens, FAIL avg 54 tokens — asymmetry not corrected |

The concrete edit this would enable: document the structural cause (length asymmetry) in
`memo.md`, add the `false_pass_rate_on_expected_fail` metric to `scoring_evaluator.py`,
add the cheapest-first diagnostic plan (threshold → SimPO + length equalization → β sweep)
to `hyperparameters.json`, and correct the β=2.0 comment in `train_simpo.py` to explain
what β controls and why it is not the first knob to turn.

---

## What Would Constitute a Satisfying Answer

An explainer that closes this gap would let me:

1. State what β controls in the CPO loss formula — specifically the difference between
   β as a KL-penalty scalar and β as a reward-scale multiplier, and why raising it hurts
   sparse disqualification coverage
2. Explain how length-normalized rewards (SimPO) interact with the target margin γ, and
   why the 4.9× output length asymmetry (11 vs 54 tokens) causes structural PASS bias
   under plain CPO that persists regardless of β value
3. Rank the three interventions (threshold calibration, SimPO + length equalization, β sweep)
   by cost and expected impact, and give the concrete configuration change for the first step
4. State what `gamma_beta_ratio=0.4` means in TRL's `CPOConfig` and what value to start with

The answer does not need to cover DPO or RLHF in full depth — the load-bearing gap is
specifically CPO/SimPO mechanics for a binary classification judge on sparse training data.
