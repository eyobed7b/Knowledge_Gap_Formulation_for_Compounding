# Morning Call Summary — Day 4

**Partners:** Eyobed Feleke + Gashaw Bekele
**Date:** 2026-05-08

During the morning call, Gashaw and Eyobed narrowed the Day 4 topic from the broad area of
"evaluation and statistics" to a concrete, artifact-grounded question about what the 92.7%
accuracy figure on Eyobed's held-out set actually means — and whether it means anything at all
given the test set size and class distribution.

The original draft question was too general: "what evaluation metrics should I use for a
classification judge?" Gashaw pushed back during the call, pointing out that the real
confusion was not about which metrics exist, but about three specific things Eyobed could
not answer from his existing numbers: whether 92.7% is a reliable point estimate, whether
the improvement over the 76.92% baseline is statistically significant, and whether the
`aggregate_score` used as a confidence proxy in `scoring_evaluator.py` is calibrated or
arbitrary.

The question was sharpened around these three gaps. Gashaw agreed to write the explainer
covering: (1) why accuracy on a ~33/8 class-imbalanced set is misleading and what precision,
recall, F1, and Cohen's kappa correct for, (2) how to compute the Wilson score CI on the
92.7% figure and run a two-proportion z-test against the baseline, and (3) a concrete
calibration check that can be added to `scoring_evaluator.py` without restructuring the
existing evaluation loop.

Both partners also agreed that the question was connected to Day 3: if the calibration check
reveals that `aggregate_score` is miscalibrated in the 0.7–0.8 range, that is a cheaper
explanation for the false-negative rate on disqualification tasks than any of the training
hypotheses explored on Day 3, and should have been checked first.
