**Asker:** Gashaw Bekele

**Gap closure status:** Closed.

**What I understand now that I didn't before:**

Before reading the explainer I was reporting 92.7% accuracy as a reliable performance estimate and treating my false-negative problem as a model training issue. The explainer closed both misunderstandings. I now understand that accuracy on my ~33/8 class-imbalanced test set is only 12.2 percentage points above the naive always-PASS baseline, and that Cohen's κ = 0.78 — not 92.7% — is the honest headline because it removes the agreement attributable to class frequency alone. I understand that the Wilson score interval puts my 92.7% at [80.6%, 97.5%] on 41 examples, making the improvement over the 76.92% baseline barely significant at p = 0.047 and not the stable claim I had been treating it as. I also understand that the 18% false-negative rate on disqualification tasks may be a calibration problem rather than a weight problem: if `aggregate_score` is over-confident in the 0.7–0.8 range, calls are pushed above the PASS threshold regardless of the model weights, and a bin-accuracy check on my logged scores can distinguish this from a training failure before any retraining is attempted.
