# Tweet Thread — Day 3

*Ready to publish. Post as a thread under your own identity.*

---

**Tweet 1**
My Week 11 compliance judge hit 92.7% overall accuracy — but had an 18% false-negative rate
on disqualification tasks. It was passing emails it should have flagged.

I assumed β=2.0 was the problem. It wasn't the root cause. Here's what I learned about where
PASS bias actually comes from. 🧵

---

**Tweet 2**
β in the CPO/DPO loss is not a "strictness dial" for specific categories.

The loss is:
`L = -log σ( β · (log p(chosen) - log p(rejected)) )`

β is a global reward-scale knob. Higher β pulls the trained policy back toward the reference
model's prior on *all* tasks. Raising it to fix disqualification recall risks degrading
calibration everywhere else.

---

**Tweet 3**
The real root cause in my case was **output length asymmetry**.

PASS verdicts in my training data averaged ~11 tokens.
FAIL verdicts (with reasoning) averaged ~54 tokens.

Plain CPO uses total log-probability as the reward. Shorter sequences accumulate less
negative log-prob — so PASS systematically looks like a better prediction than FAIL,
regardless of content.

β didn't cause this. β can't fix it either.

---

**Tweet 4**
SimPO fixes this structurally — but only if you prepare the data first.

SimPO switches to **length-normalized reward** (average log-prob per token) and adds a
**target margin γ** requiring chosen to beat rejected by at least γ, not just marginally.

But if you enable length normalization with a 4.9× length asymmetry still in the training
pairs, the normalization amplifies the bias rather than correcting it.

Equalize output lengths in the pair builder *before* enabling SimPO.

---

**Tweet 5**
The diagnostic order matters more than the fix itself.

Cheapest first:
1. Raise the inference threshold from 0.70 → 0.75 and re-score. If false-negative rate on
   disqualification drops below 10%, no retraining needed.
2. If not: equalize PASS/FAIL output lengths, then retrain with `loss_type="simpo"` and
   `gamma_beta_ratio=0.4`.
3. Only sweep β (try 1.0) if step 2 doesn't close the gap.

A full retrain to test β first wastes compute if the threshold alone would have worked.

---

**Tweet 6**
The metric that makes this measurable:

```python
false_pass_rate = sum(
    1 for r in results
    if r.get("expected_pass") is False and r.get("passed") is True
) / len(expected_fail_tasks)
```

Add this to your evaluator before claiming your judge works. Overall accuracy hides
category-level failures.

Full explainer with the CPO loss derivation, length asymmetry diagnosis, and SimPO config:

[link to blog post]

Sources: Rafailov et al. 2023 "DPO" (NeurIPS) + Meng et al. 2024 "SimPO" (NeurIPS).
