# Explainer — β, Margin, and PASS Bias in a Preference-Trained Judge

## The Question

The question is:

> What does β actually control in SimPO’s loss function, how does it interact with length normalisation and the target reward margin, and what concrete config change would reduce PASS bias on disqualification tasks without collapsing the judge’s overall calibration?

In plain English, this is a question about how hard the training objective pushes the model to separate correct verdicts from incorrect verdicts.

For a sales-compliance judge, PASS bias means the model is too willing to say an email is acceptable. That is especially dangerous on disqualification tasks, where the correct behavior is to reject, fail, or escalate the draft. If a disqualification case receives a PASS, the judge has allowed the exact failure it was trained to catch.

---

## The First Repo-Specific Clarification

The first thing to clarify is that the [repo](https://github.com/eyobed7b/Sales-Evaluation-Bench-trp) says “SimPO,” but the training script currently uses TRL `CPOTrainer` with `CPO_BETA = 2.0`. That means the actual implemented system may not yet expose the full SimPO-style target reward margin unless the trainer is configured with SimPO loss settings.

So the question should not be answered as if β alone controls everything. It should be answered as three possible levers:

1. β, which scales preference pressure,
2. the SimPO target margin, often configured through `gamma` or `gamma_beta_ratio`,
3. the inference PASS threshold.

These are not the same thing.

---

## What β Controls

In preference optimisation, the model sees a pair:

```markdown
prompt
chosen response
rejected response
````

For a disqualification task, the pair often looks like:

```markdown
chosen:  FAIL. This email violates the rubric.
rejected: PASS. The email is professional and no obvious violations are present.
```

The model should assign a higher preference score to the `FAIL` answer than the `PASS` answer.

β controls the sharpness or strength of that preference comparison. A simplified loss looks like:

```markdown
loss = -log σ(β × reward_difference)
```

where:

```markdown
reward_difference = reward(chosen) - reward(rejected)
```

If β is low, the model receives a softer training signal. It may learn that FAIL is slightly better than PASS, but not separate them strongly. If β is high, the model receives a sharper signal and pushes harder to separate chosen from rejected. But too much β can make training brittle, especially when preference labels are noisy or ambiguous.

So β is not simply a “make the model stricter” knob. It is a global reward-scale knob. It affects all preference pairs, not only disqualification pairs.

---

## What Length Normalisation Changes

SimPO’s key design is that it uses average log probability as the implicit reward. That means it compares:

```markdown
average log probability per token
```

rather than:

```markdown
total log probability of the whole response
```

This matters because total log probability naturally penalises longer outputs. Every token adds another negative log probability, so longer responses can look worse just because they are longer.

Length normalisation tries to avoid that. It asks: token for token, which response does the model prefer?

For the judge, that means a short `PASS` should not automatically beat a longer but more correct `FAIL` explanation just because it has fewer tokens. The model should prefer the answer that is better on average, not the answer that is shorter.

---

## What the Target Margin Does

The target reward margin says that chosen should not merely beat rejected. It should beat rejected by enough.

Without a margin, the model can satisfy the objective with tiny separations:

```markdown
FAIL reward = 0.51
PASS reward = 0.50
```

That technically ranks correctly, but it is weak. At inference time, weak separation can still become PASS bias.

A margin asks for something stronger:

```markdown
FAIL should beat PASS by at least this much.
```

That is why the target margin is directly relevant to disqualification tasks. If the model is passing too many emails that should fail, the issue may be insufficient separation between correct FAIL and incorrect PASS completions.

---

## The Core Answer

If the judge has PASS bias, do not immediately crank β globally.

A better first move is:

```yaml
keep beta fixed
add or increase gamma_beta_ratio
measure false PASS rate on disqualification tasks
```

Why? Because PASS bias sounds like a margin/calibration problem. The model may understand which answer is better, but not separate FAIL from PASS strongly enough. Increasing the target margin directly tells training: “do not let correct FAIL barely beat incorrect PASS.”

Changing β globally is riskier because it sharpens every training pair, including normal PASS examples. It can reduce PASS bias, but it can also make the judge over-strict, unstable, or worse calibrated across the full benchmark.

---

## The Practical Diagnostic Plan

The right tuning order is:

1. Measure false PASS rate on `expected_pass=false` tasks.
2. Break that down by failure category, especially disqualification categories.
3. Inspect score or reward margins for false PASS examples.
4. Sweep the inference PASS threshold.
5. If margins are weak, add or sweep SimPO-style `gamma_beta_ratio`.
6. Only then run a small β sweep.

A good sweep would be:

```yaml
beta: [2.0, 2.5]
gamma_beta_ratio: [0.3, 0.5, 0.7]
pass_threshold: [0.70, 0.75, 0.80]
```

Track:

```markdown
overall accuracy
false PASS rate
false FAIL rate
disqualification recall
PASS rate
calibration by category
```

The goal is not to maximise strictness. The goal is to reduce false PASS decisions on disqualification tasks while keeping valid PASS tasks from collapsing into false FAIL.

---

## The Recommended Repo-Level Change

The repo currently uses CPO with `CPO_BETA = 2.0`. If the goal is genuinely SimPO-style margin control, the training script should either:

1. explicitly configure TRL’s SimPO mode, if available in the installed TRL version, or
2. rename the method consistently as CPO and tune β plus inference threshold only.

The cleanest implementation path is to add a small experiment config:

```yaml
loss_type: simpo
beta: 2.0
gamma_beta_ratio: 0.5
cpo_alpha: 0.0
```

Then compare it against the current CPO baseline.

---

## Final Takeaway

β controls the strength of the preference push. Length normalisation controls what kind of reward is being compared. The target margin controls how far chosen should beat rejected.

For PASS bias, the safest answer is not “increase β.” The safer answer is:

> First measure false PASS margins. Then adjust the margin or threshold. Only tune β globally if the model still under-separates after margin and threshold diagnostics.

That keeps the judge from becoming over-strict while directly targeting the disqualification failure mode.
