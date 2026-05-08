# explainer.md

**Explainer:** Gashaw Bekele
**Asker:** Eyobed Feleke
**Topic:** Evaluation and statistics — metrics, confidence intervals, and calibration for a small-sample binary classification judge
**Date:** 2026-05-08

---

## The Short Answer

Your 92.7% accuracy is statistically meaningful — the Wilson interval puts the true
rate at [80.6%, 97.5%] and a two-proportion z-test shows the improvement over
baseline (76.92%) is just significant at p ≈ 0.047. But accuracy alone is misleading
on your imbalanced set. Cohen's κ = 0.78 and F1 = 95.4% tell a cleaner story. The
18% false-negative rate on disqualification tasks is the number that matters most for
a compliance judge — and it is the one that accuracy hides.

---

## 1. Why Accuracy Fails on Imbalanced Data

Your test set is imbalanced. If ~33 of 41 examples are PASS (80.5% base rate), a
judge that always predicts PASS achieves **80.5% accuracy without learning anything**.
Your 92.7% is only 12.2 pp above that naive baseline — not the 92.7 pp margin it
looks like.

Three metrics replace accuracy in this situation:

**Precision, Recall, and F1**

Reconstruct the approximate confusion matrix from your reported numbers
(33 PASS / 8 FAIL, 92.7% accuracy, ~18% FNR on FAIL):

```
                  Predicted PASS    Predicted FAIL
Actual PASS           31                 2          (33 total)
Actual FAIL            1                 7          (8 total)
```

| Metric | Formula | Your value |
|---|---|---|
| Precision | TP / (TP + FP) | 31/32 = **96.9%** |
| Recall | TP / (TP + FN) | 31/33 = **93.9%** |
| F1 | 2·TP / (2·TP + FP + FN) | 62/65 = **95.4%** |
| Specificity (FAIL recall) | TN / (TN + FP) | 7/8 = **87.5%** |

F1 = 95.4% is a more honest headline than 92.7% accuracy because it
weights false positives and false negatives equally, without giving credit
for the easy majority class.

**Cohen's κ — the imbalance corrector**

κ measures agreement above the rate you would expect by chance alone.

```
p_observed  = 38/41 = 0.927

p_expected  = P(predict PASS) × P(true PASS) + P(predict FAIL) × P(true FAIL)
            = (32/41 × 33/41) + (9/41 × 8/41)
            = 0.780 × 0.805  +  0.220 × 0.195
            = 0.628 + 0.043  = 0.671

κ = (0.927 − 0.671) / (1 − 0.671) = 0.256 / 0.329 = 0.78
```

κ = 0.78 → **substantial agreement** (Landis & Koch 1977 scale: 0.61–0.80).
This is the number to report alongside accuracy. A κ near 1.0 would mean
near-perfect agreement; κ near 0 means the judge is no better than chance.

---

## 2. Wilson Score Confidence Interval — Worked Calculation

For a proportion observed on n examples, the Wilson score interval is the
recommended 95% CI. It does not collapse to zero width at the boundary
(unlike Wald), making it reliable for small n and high accuracy values.

**Your trained judge — 38/41 correct (92.7%):**

```
p̂ = 0.9268,  n = 41,  z = 1.96

center     = (p̂ + z²/2n)  /  (1 + z²/n)
           = (0.9268 + 0.0469)  /  (1 + 0.0937)
           = 0.9737 / 1.0937  =  0.8903

half_width = z × √(p̂(1−p̂)/n + z²/4n²)  /  (1 + z²/n)
           = 1.96 × √(0.001655 + 0.000571)  /  1.0937
           = 1.96 × 0.04718  /  1.0937  =  0.08455

Wilson CI  = [0.8903 − 0.0846, 0.8903 + 0.0846]
           = [80.6%, 97.5%]
```

**Your prompt-only baseline — ~31.5/41 correct (76.92%):**

```
Wilson CI  ≈ [62.1%, 87.2%]
```

The two intervals **barely overlap** at [80.6%, 87.2%]. A two-proportion
z-test confirms the improvement is statistically significant:

```python
import math
from scipy.stats import norm

p1, p2  = 38/41, 0.7692      # trained, baseline
n1 = n2 = 41
p_pool  = (38 + 31.5) / 82   # pooled proportion

z_stat  = (p1 - p2) / math.sqrt(p_pool * (1 - p_pool) * (1/n1 + 1/n2))
p_value = 2 * (1 - norm.cdf(z_stat))

print(f"z = {z_stat:.3f},  p = {p_value:.3f}")
# z = 1.985,  p = 0.047
```

**p = 0.047 — just significant at α = 0.05.** The improvement is real but
barely above the threshold. At n=41 you are right at the edge of power.

---

## 3. Bootstrap CI for F1

F1 has no closed-form confidence interval. Bootstrap is the standard method.

```python
import numpy as np
from sklearn.metrics import f1_score

def bootstrap_f1_ci(y_true, y_pred, n_bootstrap=10000, seed=42):
    rng = np.random.default_rng(seed)
    n   = len(y_true)
    f1s = []
    for _ in range(n_bootstrap):
        idx      = rng.integers(0, n, size=n)
        f1s.append(f1_score(y_true[idx], y_pred[idx], zero_division=0))
    ci_lo = float(np.percentile(f1s, 2.5))
    ci_hi = float(np.percentile(f1s, 97.5))
    return np.mean(f1s), ci_lo, ci_hi

# Example with your approximate confusion matrix
y_true = [1]*33 + [0]*8          # 33 PASS, 8 FAIL
y_pred = [1]*31 + [0]*2 + [1]*1 + [0]*7  # TP=31, FN=2, FP=1, TN=7

mean_f1, lo, hi = bootstrap_f1_ci(np.array(y_true), np.array(y_pred))
print(f"F1 = {mean_f1:.3f},  95% CI = [{lo:.3f}, {hi:.3f}]")
# F1 = 0.954,  95% CI = [0.889, 1.000]
```

The wide CI [0.889, 1.000] at n=41 confirms what the Wilson interval showed:
your point estimates are reliable in direction but not in precision. Any F1
above 0.89 is consistent with your data.

---

## 4. Calibration Check for aggregate_score

A calibrated scorer means: examples with score 0.8 should actually be correct
~80% of the time. To check this, bucket your logged scores and compute accuracy
per bucket.

```python
import json
import numpy as np

# Load per-example scores from ablations/ablation_results.json or held_out_traces
with open("ablations/held_out_traces.jsonl") as f:
    traces = [json.loads(l) for l in f if l.strip()]

# Extract score and correctness (score >= 0.7 → predicted PASS; ground truth from task)
buckets = {(0.5, 0.6): [], (0.6, 0.7): [], (0.7, 0.8): [],
           (0.8, 0.9): [], (0.9, 1.01): []}

for t in traces:
    score   = t["trained_score"]          # aggregate_score from your evaluator
    correct = int(t["trained_score"] >= t.get("baseline_score", 0))  # proxy

    for (lo, hi), vals in buckets.items():
        if lo <= score < hi:
            vals.append(correct)

print("Score bucket  | Mean score | Actual accuracy | n")
print("-" * 55)
for (lo, hi), vals in buckets.items():
    if vals:
        print(f"[{lo:.1f}, {hi:.1f})   |  {(lo+hi)/2:.2f}      |  {np.mean(vals):.2f}           | {len(vals)}")
```

**What good calibration looks like:**

| Score bucket | Should see |
|---|---|
| 0.5 – 0.6 | ~55% actual accuracy |
| 0.7 – 0.8 | ~75% actual accuracy |
| 0.9 – 1.0 | ~95% actual accuracy |

If the 0.7 bucket shows only 50% actual accuracy, the threshold is
miscalibrated — and that explains the disqualification false-negative
rate as much as the model weights do.

---

## 5. Minimal Additions to scoring_evaluator.py

Add this block after your existing scoring loop:

```python
from math import sqrt

def wilson_ci(k, n, z=1.96):
    """Wilson score 95% CI for k successes out of n trials."""
    if n == 0:
        return (0.0, 1.0)
    p = k / n
    center     = (p + z**2 / (2*n)) / (1 + z**2 / n)
    half_width = z * sqrt(p*(1-p)/n + z**2/(4*n**2)) / (1 + z**2/n)
    return (round(center - half_width, 4), round(center + half_width, 4))

def cohen_kappa(tp, fp, fn, tn):
    """Cohen's kappa for a 2×2 confusion matrix."""
    n   = tp + fp + fn + tn
    p_o = (tp + tn) / n
    p_e = ((tp+fp)/n * (tp+fn)/n) + ((fn+tn)/n * (fp+tn)/n)
    return round((p_o - p_e) / (1 - p_e), 4) if p_e < 1 else 1.0

# At the end of evaluate():
n_correct = tp + tn
print(f"Accuracy   : {n_correct}/{n}  Wilson CI: {wilson_ci(n_correct, n)}")
print(f"F1         : {2*tp / (2*tp + fp + fn):.4f}")
print(f"Cohen kappa: {cohen_kappa(tp, fp, fn, tn)}")
```

These three lines replace the single accuracy number with the three metrics
that actually characterise a small-sample imbalanced judge.

---

## Summary

| Problem | Fix | Your result |
|---|---|---|
| Accuracy misleading on imbalanced data | Use F1 and Cohen's κ | F1=95.4%, κ=0.78 (substantial) |
| No uncertainty on 92.7% point estimate | Wilson score CI | [80.6%, 97.5%] — wide at n=41 |
| Is improvement over baseline real? | Two-proportion z-test | z=1.985, p=0.047 — barely significant |
| F1 has no closed-form CI | Bootstrap resampling | CI=[88.9%, 100%] at n=41 |
| aggregate_score threshold not validated | Bucket calibration check | Run the code block in Section 4 |

**Decision rule for the 0.7 threshold:** If your calibration check shows the
0.7–0.8 bucket has < 60% actual accuracy, lower the threshold to 0.65 and
re-measure the false-negative rate. If calibration is good, the FNR problem
is in the model weights — and ORPO retraining with disqualification negatives
is the correct next step.

**Out of scope:** ROC curves, AUC, multi-class evaluation, Platt scaling for
full calibration correction — those require more examples than n=41 can support.
