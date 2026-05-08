# Day 4 Question — Evaluation and Statistics

**Asker:** Eyobed Feleke
**Date:** 2026-05-08
**Topic:** Evaluation and statistics for a small-sample classification judge

---

## The Question

My Week 11 judge reports 92.7% overall accuracy and an 18% false-negative rate on disqualification tasks, measured on a 41-example held-out set. I cannot say whether 92.7% is a statistically reliable estimate of true performance, whether the improvement over the prompt-only baseline (76.92%) is significant or within noise, or whether overall accuracy is even the right metric given that my test set is class-imbalanced. What metrics should replace or supplement accuracy for a binary classification judge on a small, imbalanced test set, how do I compute confidence intervals on those metrics, and how do I measure whether my judge's `aggregate_score` is calibrated — meaning that a score of 0.8 actually corresponds to roughly 80% chance of being correct?

---

## Why This Gap Matters in My Work

My Week 11 judge (`Qwen2.5-0.5B-Instruct`, LoRA r=16, trained on 137 preference pairs) is
evaluated on a held-out set of 41 examples. The `memo.md` reports:

- **92.7% overall accuracy** (CPO-trained judge)
- **76.92% overall accuracy** (prompt-only baseline)
- **18% false-negative rate on disqualification tasks**

These numbers are presented as if they are reliable point estimates. They are not:

**Problem 1 — Class imbalance makes accuracy misleading.**
If the 41-example test set contains 33 PASS cases and 8 FAIL cases (a realistic split given that
most sales calls pass the basic compliance screen), a model that always predicts PASS achieves
80.5% accuracy without learning anything. My 92.7% figure may not be meaningfully above this
naive baseline, but I have never checked.

**Problem 2 — 41 examples is too small to trust a point estimate.**
With 41 examples, a 95% confidence interval for an observed accuracy of 92.7% spans roughly
±8 percentage points (using the Wilson interval). The "improvement" from 76.92% to 92.7%
may or may not be statistically distinguishable from noise. I have been treating the difference
as meaningful without checking.

**Problem 3 — The `aggregate_score` is used as a confidence proxy but never validated.**
`scoring_evaluator.py` uses `aggregate_score >= 0.7` as the PASS/FAIL threshold. But I have
never checked whether the score is calibrated — i.e., whether calls scored 0.8 actually pass
at ~80% rate in reality, or whether the score is systematically over- or under-confident. If
the score is uncalibrated, the 0.7 threshold is arbitrary and may explain the disqualification
false-negative rate as much as the model weights do.

---

## Connection to Existing Artifacts

| Artifact | Location | What depends on understanding this |
|---|---|---|
| Metric reporting | `memo.md` — "92.7% accuracy" | No CI, no class-adjusted metric, no baseline comparison |
| Score threshold | `scoring_evaluator.py` line 223 | `aggregate_score >= 0.7` — threshold not calibrated |
| Held-out results | `ablations/ablation_results.json` | Per-example results logged but no statistical analysis run |
| Class distribution | `training_data/build_simpo_pairs.py` | PASS vs FAIL ratio in training data is known but never checked against test set |
| False-negative report | `memo.md` Skeptic's Appendix | 18% stated without denominator, confidence interval, or comparison to PASS false-negative rate |

---

## What Would Constitute a Satisfying Answer

An explainer that closes this gap would let me:

1. Name the two or three metrics that replace or supplement accuracy for an imbalanced binary
   classification judge — specifically what precision, recall, F1, and Cohen's kappa each
   measure and why they are more informative than accuracy in this context

2. Compute a 95% confidence interval for my observed 92.7% accuracy on 41 examples using the
   Wilson score interval, and assess whether the 92.7% vs 76.92% comparison is statistically
   meaningful given those intervals

3. Explain what a bootstrap confidence interval is and when it is preferable to the Wilson
   interval — specifically for metrics like F1 that do not have a closed-form CI formula

4. Define calibration for a scoring judge and describe a simple calibration check I can run
   using the existing `aggregate_score` values already logged in `ablations/ablation_results.json`

5. Propose the minimal additions to `scoring_evaluator.py` that would compute these metrics
   automatically on every evaluation run — ideally a function or block I can add directly
   to the existing script

The answer does not need to cover ROC curves, AUC, or multi-class evaluation in depth. The
load-bearing gap is specifically about binary classification evaluation under small-sample,
class-imbalance, and score-calibration conditions, grounded in the concrete numbers and files
from my Week 11 repo.
