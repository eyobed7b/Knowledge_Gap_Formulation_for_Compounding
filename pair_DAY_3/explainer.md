# Explainer: β, SimPO Margin, and Inference Threshold in Preference-Trained Judges

*Context: Week 11 `Sales-Evaluation-Bench-trp`. A judge model was trained on preference
pairs to label sales conversation outcomes as `PASS` or `FAIL`. The script is named
`train_simpo.py` but runs `CPOTrainer` with `CPO_BETA = 2.0` — no SimPO-specific config
is applied. The held-out judge shows PASS bias on disqualification tasks. The question is
whether this is a β problem, a missing SimPO target-margin problem, or an
inference-threshold problem.*

---

## The Setup

Training pairs for failing tasks are built as:

- `chosen` = correct `FAIL` verdict
- `rejected` = over-generous `PASS` verdict

The goal: teach the model that `FAIL` should outscore `PASS`. Overall held-out accuracy
improved, but disqualification tasks still produce too many false PASSes. Three things
could be responsible: the KL penalty β, the absence of a SimPO reward margin, or the
inference PASS threshold in `scoring_evaluator.py`.

---

## What β Controls in the Preference Loss

β is the **KL-divergence penalty**. It controls how far the trained policy is allowed to
drift from the reference model during preference learning.

- **High β (e.g., 2.0):** The model is pulled back toward the reference at every update.
  Strong prior behavior — such as a base model's tendency to be generous with PASS — is
  hard to override. The `FAIL > PASS` preference signal competes against this pull and
  may not win, especially on sparse categories like disqualification tasks.
- **Low β (e.g., 0.5):** Larger updates are allowed. The preference signal is stronger,
  but β too low risks collapsing calibration on the PASS tasks the judge already handles
  correctly.

At `CPO_BETA = 2.0`, over-regularization is a plausible explanation for lingering PASS
bias. Lowering β is the lowest-cost hypothesis to test: it requires only a config change
and a retrain.

---

## CPO vs SimPO: Two Structural Gaps

The script name implies SimPO but the config runs plain CPO. The two differ in exactly the
ways that matter for disqualification bias.

### Gap 1 — Length-Normalized Reward

CPO computes rewards from **raw log probability**. A longer `PASS` response accumulates
higher raw log probability than a shorter `FAIL` response purely because it has more
tokens. This introduces a length bias that works against the preference signal: the model
may learn to prefer verbose outputs rather than correct ones.

SimPO uses **length-normalized average log probability** — total log probability divided
by token count. Length stops being a confound. The model must learn quality differences
between `FAIL` and `PASS`, not length differences.

### Gap 2 — Target Reward Margin (γ)

CPO only requires that `chosen` beats `rejected` by *any positive amount*, even 0.001.
A model can satisfy this constraint while still producing nearly identical scores for
`FAIL` and `PASS`. At inference time, a small score perturbation or a generous threshold
is then enough to flip the prediction to PASS.

SimPO adds a **target reward margin**: `chosen` must beat `rejected` by at least `γ/β`.
This enforces a minimum separation robust enough to survive inference noise. For
disqualification tasks — where the surface forms of `FAIL` and `PASS` verdicts may look
similar — this margin is often the difference between the judge being reliable and not.

The `gamma_beta_ratio` hyperparameter sets γ/β directly. A typical starting range is
0.3–0.5.

**Config change required:**

```python
# in train_simpo.py — current state
trainer = CPOTrainer(
    model=model,
    args=CPOConfig(beta=2.0, ...),
    ...
)

# target state
trainer = CPOTrainer(
    model=model,
    args=CPOConfig(
        beta=1.0,
        loss_type="simpo",
        gamma_beta_ratio=0.4,
        ...
    ),
    ...
)
```

---

## The Inference Threshold Problem

Even a well-trained judge emits a continuous score. The threshold at which `scoring_evaluator.py`
calls a score `PASS` vs `FAIL` is a post-hoc calibration decision, not a property of the
model weights. If that threshold is too generous — or if the same value is applied uniformly
across all task categories — disqualification tasks produce false PASSes regardless of how
well the model was trained.

This is the fastest hypothesis to test (no retrain required), but also the most dangerous
to apply prematurely: raising the threshold on an under-trained model hides the root cause
rather than resolving it.

---

## Diagnostic Plan: How to Distinguish the Three

Run in this order:

**Step 1 — Check reward margins on held-out disqualification pairs**

Compute `chosen_reward − rejected_reward` for every disqualification pair in the held-out
set. If the median margin is < 0.5, the model has not learned a robust `FAIL > PASS`
separation. This points to β or missing SimPO margin — fix training first.

**Step 2 — Check per-category recall**

Compare recall on `expected_pass=false` vs overall recall. If disqualification recall is
disproportionately low while other categories hold, the threshold may be applying a
uniform standard to a harder category. This opens the threshold hypothesis.

**Step 3 — Retrain with SimPO config (hold β constant)**

Add `loss_type="simpo"` and `gamma_beta_ratio=0.4`. If disqualification recall improves,
the missing margin was the root cause.

**Step 4 — Retrain with lower β (hold everything else constant)**

Set β = 0.5. If disqualification recall improves without precision collapse on PASS tasks,
β was over-regularizing.

**Step 5 — Adjust threshold only if Steps 3–4 do not close the gap**

Raise the PASS threshold specifically for `disqualification` category tasks in
`scoring_evaluator.py`. Confirm that overall calibration on other categories does not
degrade.

---

## Summary: Three Levers, One Ordering

| Lever | Config location | When to pull it |
|---|---|---|
| Lower β | `hyperparameters.json` → `CPO_BETA` | Margins are small; model is over-regularized |
| Add SimPO margin | `train_simpo.py` → `loss_type`, `gamma_beta_ratio` | Margins are small; length bias suspected |
| Raise PASS threshold | `scoring_evaluator.py` | Training is correct; threshold is uniformly too generous |

---

## Artifacts

- [training/train_simpo.py](training/train_simpo.py) — add `loss_type="simpo"`, `gamma_beta_ratio`, tune β
- [training/hyperparameters.json](training/hyperparameters.json) — record final swept values
- [training_data/build_simpo_pairs.py](training_data/build_simpo_pairs.py) — add more disqualification pairs if the category is data-sparse
- [ablations/ablation_results.json](ablations/ablation_results.json) — log one ablation row per hypothesis
- [scoring_evaluator.py](scoring_evaluator.py) — adjust PASS threshold only after confirming hypothesis 3
- [memo.md](memo.md) — record decision and rationale
