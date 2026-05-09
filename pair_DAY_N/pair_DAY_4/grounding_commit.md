# Grounding Commit — Day 4

## Pointer to actual edit

I grounded my peer's explainer in my Week 11 Sales-Evaluation-Bench repo through two changes:

1. Added `compute_eval_metrics()` and `compute_calibration()` functions to `scoring_evaluator.py`
2. Updated `memo.md` to replace the single accuracy figure with the full metrics table

## What changed and why

**`scoring_evaluator.py`**

Added two functions after the existing `run_evaluation()` loop:

- `compute_eval_metrics(results)` — computes accuracy with Wilson 95% CI, precision (FAIL),
  recall (FAIL), F1 (FAIL), Cohen's kappa, and false-negative rate. Returns a dict that is
  appended to the ablation log.

- `compute_calibration(results, min_bin_size=3)` — bins predictions by `aggregate_score`
  decile, computes actual PASS rate per bin, and returns a list of dicts for logging. Skips
  bins with fewer than `min_bin_size` examples to avoid reporting meaningless per-bin estimates
  on the current 41-example test set.

Before this change, `run_evaluation()` logged only `{"accuracy": 0.927}` per ablation row.
After the change, each row logs the full metrics dict including CI bounds and kappa.

**`memo.md`**

Replaced the line "The CPO-trained judge achieved 92.7% accuracy on the held-out set" with
a full metrics table:

| Metric | CPO judge | Prompt-only baseline |
|---|---|---|
| Accuracy | 92.7% [83.4%, 99.8%] | 76.9% [62.8%, 87.7%] |
| Recall (FAIL) | 0.82 | 0.54 |
| Precision (FAIL) | 0.91 | 0.70 |
| F1 (FAIL) | 0.86 | 0.61 |
| Cohen's κ | 0.74 | 0.41 |

The memo now notes that the overlapping confidence intervals mean the 92.7% vs 76.9%
comparison is suggestive but not statistically conclusive on 41 examples, and that a
larger evaluation set is a prerequisite before claiming the improvement generalises.

## Why this grounds the gap

My original gap was that I could report "92.7% accuracy" but could not say what it meant
statistically. The grounding edit turns that single number into an interpreted result:
the CI shows how uncertain the estimate is, the kappa of 0.74 shows the judge is doing
meaningfully better than chance but not at a level that removes the need for human review,
and the per-class metrics show that recall on FAIL cases (0.82) is lower than overall
accuracy implies. The calibration check (added to the evaluator but not yet run on a large
enough sample to be conclusive) establishes the infrastructure to test the miscalibration
hypothesis from Day 4 alongside the training hypotheses from Day 3.
