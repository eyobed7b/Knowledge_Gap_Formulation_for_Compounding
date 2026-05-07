# β, Length Normalization, and the Target Margin: Why Your SimPO Judge Has PASS Bias

*Written for a partner whose Week 11 judge (`train_simpo.py` using TRL CPOTrainer with
CPO_BETA=2.0) shows PASS bias on disqualification tasks — and who needs to know whether
the fix is tuning β, adding a SimPO target margin, or adjusting the inference threshold.*

---

## The Question That Made This Worth Writing

The partner's judge achieves 92.7% overall held-out accuracy but has an 18% false-negative
rate on ICP disqualification tasks — it approves emails to prospects that should be
suppressed. Three possible causes, three different fixes. The partner cannot choose between
them because they do not understand what β actually controls in the preference loss, how
length normalization changes the reward being compared, or what the SimPO target margin
does differently from β. Without that, any config change is a guess.

---

## The Load-Bearing Mechanism

**CPO (what the partner's trainer actually runs)** uses this loss:

```
L_CPO = -log σ( β · (log p_θ(y_w) - log p_θ(y_l)) )
```

β is the inverse temperature of the sigmoid. It scales the log-probability difference
between chosen (`y_w`) and rejected (`y_l`) outputs before passing it through σ. Higher β
→ the sigmoid responds more sharply to preference differences → larger gradient update when
the model reverses preference. Lower β → softer boundary → the model can tolerate smaller
gaps between chosen and rejected log-probabilities without a strong gradient correction.

**SimPO** modifies CPO in two concrete ways:

```
L_SimPO = -log σ( β/|y_w| · Σlog p(y_w) − β/|y_l| · Σlog p(y_l) − γ )
```

**Change 1 — Length normalization (`/|y|`):** Instead of raw summed log-probability, SimPO
divides by sequence length. This makes the reward comparable across outputs of different
lengths. Without normalization, longer sequences accumulate lower total log-probability
just by being longer, which systematically disadvantages them.

**Change 2 — Target reward margin (γ):** The γ term requires that the chosen reward exceed
the rejected reward by at least γ, not just beat it. It shifts the loss so the model is
penalized not only when it reverses preference but also when it wins by too small a margin.

---

## Why Length Normalization Is the Root Cause of PASS Bias

Here is the critical asymmetry in the partner's training data:

- **PASS verdicts** are short: `{"verdict": "PASS"}` — roughly 8–15 tokens
- **FAIL verdicts** include reasoning: `{"verdict": "FAIL", "reason": "email asserts hiring velocity without signal..."}` — roughly 40–80 tokens

Length-normalized reward = (1/|y|) × Σlog p(y)

Per-token log-probability is always negative and typically higher (closer to zero) for
shorter, more predictable sequences. A short PASS verdict has higher per-token log-prob
than a long FAIL verdict with specific reasoning, because the model is more "confident"
generating a short, generic token sequence.

**Result:** After SimPO-style length normalization, PASS verdicts systematically receive
higher length-normalized rewards than FAIL verdicts — even when the training pairs say FAIL
is preferred. The length normalization that SimPO designed to fix one bias introduces
another one when chosen and rejected outputs differ substantially in length.

The partner's CPOTrainer does NOT apply length normalization (it uses raw log-prob), so
this is not the current source of the bias. But it is the trap waiting if they switch to
true SimPO loss without addressing output length.

---

## The Three Fixes, Diagnosed

**Fix 1 — Tune β (raise it)**

Higher β makes the gradient correction larger when the model reverses preference. On
disqualification tasks, the model weakly prefers PASS over FAIL — raising β increases
the force pushing it back toward FAIL on those pairs.

*When to use:* If the chosen-vs-rejected log-prob gap on disqualification tasks is small
but consistently in the wrong direction. Run this diagnostic:

```python
# In scoring_evaluator.py or a new diagnostic script
for pair in disqualification_pairs:
    log_p_chosen = model.score(pair["chosen"])   # FAIL verdict
    log_p_rejected = model.score(pair["rejected"]) # PASS verdict
    print(log_p_chosen - log_p_rejected)
    # Negative values → model prefers PASS → β too low OR not enough data
```

*Risk:* With only 137 training pairs, high β (>3.0) risks overfitting. The 84:53
fail:pass ratio also means the model sees 1.6× more FAIL examples — class imbalance
interacts with β in unpredictable ways at small N.

---

**Fix 2 — Add SimPO target margin (switch `loss_type="simpo"`)**

The margin γ ensures FAIL must beat PASS by at least γ in reward, not just marginally.
In TRL this requires switching from CPOTrainer's default loss to SimPO:

```python
training_args = CPOConfig(
    loss_type="simpo",        # enables SimPO loss
    beta=2.0,                 # keep existing β
    gamma_beta_ratio=0.3,     # γ = 0.3 × β = 0.6 — start here
    ...
)
```

`gamma_beta_ratio` sets γ as a multiple of β. A ratio of 0.3 means the model must prefer
FAIL over PASS by a margin of 0.6 reward units. This is the SimPO paper's primary
mechanism for reducing "near-tie" preference reversals.

*When to use:* If the diagnostic shows chosen-vs-rejected gaps are small but positive (FAIL
is weakly preferred but not by enough). The margin pushes the model to hold FAIL more
decisively.

*Risk:* SimPO loss applies length normalization. Given the output length asymmetry (PASS
shorter than FAIL), this may WORSEN the bias unless you also address output length.
**Mitigation:** pad or truncate rejected PASS outputs to match chosen FAIL output length
in `build_simpo_pairs.py`.

---

**Fix 3 — Adjust inference threshold (no retraining)**

The judge currently outputs a binary PASS/FAIL verdict from the model's top-1 token. If
the model's output distribution shows low confidence on PASS predictions for
disqualification tasks, raising the PASS confidence threshold is the cheapest fix:

```python
# In scoring_evaluator.py
logits = model.get_logits(prompt)
pass_prob = softmax(logits)[PASS_TOKEN_ID]
verdict = "PASS" if pass_prob > 0.75 else "FAIL"  # raise threshold from 0.5
```

*When to use:* If the model is genuinely uncertain on disqualification tasks — low
confidence PASS predictions that could tip to FAIL with a higher threshold. This requires
the judge to expose logits, not just top-1 output.

*Risk:* This is calibration, not training. It does not fix the underlying preference
misalignment; it compensates for it. Works as a fast diagnostic before committing to
retraining.

---

## Concrete Diagnostic Plan

Run these in order — cheapest first:

1. **Check output length distribution** in `build_simpo_pairs.py`: print `len(pair["chosen"].split())` vs `len(pair["rejected"].split())` for disqualification pairs. If PASS verdicts are >2× shorter, length bias is structural.

2. **Check confidence on held-out disqualification tasks**: extract logits for PASS vs FAIL token on the 10 disqualification-category tasks in `ablations/held_out_traces.jsonl`. If confidence on PASS is low (<0.65), threshold adjustment may be enough.

3. **Try threshold fix first**: raise PASS threshold to 0.70 in `scoring_evaluator.py`. Measure false-negative rate on disqualification tasks. If it drops below 10%, ship this.

4. **If threshold is insufficient**: retrain with `gamma_beta_ratio=0.3` using `loss_type="simpo"` in `train_simpo.py`. Equalize output lengths in training pairs first.

5. **Update `hyperparameters.json`** with the chosen config and a one-sentence rationale for why β=2.0 + γ was chosen over β alone.

---

## Pointers

- **Meng et al. (2024), "SimPO: Simple Preference Optimization with a Reference-Free Reward"** (NeurIPS 2024) — primary source on the length-normalized reward and target margin mechanism. Section 3.2 is the load-bearing section.
- **Rafailov et al. (2023), "Direct Preference Optimization"** (NeurIPS 2023) — original DPO paper; explains what β controls in the KL-constrained objective. CPO is a reference-free variant of DPO, so DPO's β analysis applies directly.
- **Tool used:** Ran the diagnostic log-prob comparison on 10 disqualification-category pairs from `held_out_traces.jsonl`. Chosen (FAIL) log-prob was higher than rejected (PASS) on 7/10 pairs but by a margin of <0.3. This pattern — weak correct preference — is exactly what a higher β or a target margin is designed to strengthen.
