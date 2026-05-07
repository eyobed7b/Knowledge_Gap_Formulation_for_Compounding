**Asker:** Nuhamin Alemayehu

**Gap closure status:** Partially closed.

**What I understand now that I didn't before:**

The gap is partially closed because I now understand the main mechanism my peer identified: prompting keeps the rubric in the context window, while DPO changes the adapter weights so the model’s preference boundary moves toward the `chosen` examples and away from the `rejected` examples. I also understand that the reference model in DPO acts as a regulariser and that the chosen-minus-rejected log-prob margin is the training signal that prompting alone cannot create. What is not fully closed yet is the empirical attribution: the explainer gives a strong hypothesis that the 76.92% → 96.15% improvement came mostly from decision-boundary shift and calibration improvement, but I still need to run the proposed diagnostics — train/dev reward margins, held-out margin distribution, per-source-mode accuracy, failed-pair inspection, and threshold sweeps — before I can rule out benchmark-style memorisation or overfitting.