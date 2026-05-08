# Tweet Thread — Day 4

*Ready to publish. Post as a thread under your own identity.*

---

**Tweet 1**
My Week 11 judge hit 92.7% accuracy on 41 held-out examples.

I reported it as if it meant something. It didn't — not without context.

Here's what I learned about evaluating a classification judge on a small, imbalanced test set. 🧵

---

**Tweet 2**
First problem: accuracy on an imbalanced test set is the wrong metric.

If 80% of your test cases are PASS, predicting PASS every time gives 80% accuracy.
My judge's 92.7% is only ~12 points above that naive baseline.

I had never checked the class distribution of my test set.

---

**Tweet 3**
The three metrics that actually matter for a binary judge:

**Precision (FAIL):** Of the calls I flagged as non-compliant, how many really were?
**Recall (FAIL):** Of all truly non-compliant calls, how many did I catch?
**Cohen's κ:** Agreement with ground truth, corrected for the class baseline.

Accuracy tells you nothing about which class you're failing on.

---

**Tweet 4**
Second problem: 92.7% on 41 examples is not a reliable point estimate.

Wilson 95% CI on 92.7%, n=41:

→ Lower bound: 83.4%
→ Upper bound: 99.8%

The baseline was 76.92%. The improvement is plausibly real — but I cannot claim statistical significance. I need ~150 balanced examples before accuracy estimates are tight enough to trust.

---

**Tweet 5**
Third problem: calibration.

My judge uses `aggregate_score >= 0.7` as the PASS threshold. But I never checked whether a score of 0.8 actually means "80% chance of PASS."

A simple calibration check: group predictions by score bin, compute the actual PASS rate per bin. If the score says 0.75 but the actual rate is 0.55, the threshold is miscalibrated — and that explains false negatives without touching model weights.

---

**Tweet 6**
The minimal fix: add one function to `scoring_evaluator.py` that computes:

- Accuracy + Wilson CI
- Precision, Recall, F1 (FAIL class)
- Cohen's kappa
- Calibration check

Every evaluation run should log this dict to `ablation_results.json`.

Reporting a single accuracy number from 41 examples is not evaluation. It's a number.

Sources: Wilson (1927) "Probable inference", Guo et al. (2017) "On Calibration of Modern Neural Networks" (ICML).
