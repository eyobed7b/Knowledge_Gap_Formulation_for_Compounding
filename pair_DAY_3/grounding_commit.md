# Grounding Commit

## Pointer to actual edit

I grounded my peer’s explainer in my Week 11 Tenacious-Bench repo through three documentation commits:

1. [docs: add peer explainer blog](https://github.com/nuhaminae/The-Conversion-Engine-Ground-Truth/commit/f2cb19c2b0d684886dff856a3aa5af2a50e44aa4)  
2. [docs: update model card with peer explanation](https://github.com/nuhaminae/The-Conversion-Engine-Ground-Truth/commit/2824b15a2af4d877b542af819fd448fad53049b8)  
3. [docs(report): update decision memo with peer explanation section](https://github.com/nuhaminae/The-Conversion-Engine-Ground-Truth/commit/1f33433262c697459b9029350da21e284888b9fa)

## What changed and why

I updated my Week 11 documentation to reflect my peer’s explanation of what DPO changed compared with prompting. Before this grounding edit, my report mainly stated the metric result: the prompt-engineered judge reached 76.92% strict pairwise accuracy, while the DPO judge reached 96.15%. After the edit, the result is interpreted mechanistically: prompting kept the Tenacious rubric in the model’s context window, but left the base model’s internal accept/reject boundary unchanged; DPO moved the preference signal into the LoRA adapter weights by training on `prompt/chosen/rejected` pairs and using chosen-vs-rejected log-prob margins against a frosen reference model. The model card and decision memo now frame the improvement as a likely decision-boundary and calibration shift, while still acknowledging that small-data overfitting or benchmark-style memorisation remains a risk that should be tested with train/dev reward margins, held-out margin distributions, per-source-mode accuracy, failed-pair inspection, threshold sweeps, and same-base prompted comparison.

## Why this grounds the gap

My original gap was that I could report “DPO beat prompting,” but I could not explain what changed inside the model behavior. These edits turn that result from a simple leaderboard claim into a post-training interpretation: the DPO judge did not necessarily learn a completely new Tenacious-specific reasoning ability from 65 pairs; more plausibly, it calibrated an already capable base model by moving the acceptable-vs-bad decision boundary into the adapter weights. This makes the Week 11 artifact stronger because the model card and decision memo now state both the mechanism I understand and the empirical checks still needed before claiming full generalisation.
