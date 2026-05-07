# Portfolio Update — Week 12 Grounding Commits

**Author:** Eyobed Feleke · eyobed@10academy.org  
**Date:** 2026-05-10  

---

## What This Document Is

This is a one-page summary for a Forward-Deployed Engineer hiring manager. It describes five
grounding commits made during Week 12 that improved the Weeks 10 and 11 portfolio, explains
why each commit matters for a production FDE role, and states what the portfolio demonstrates
as a result.

---

## The Five Grounding Commits

### 1. Per-call latency instrumentation — `tau2_harness.py`

**What changed:** Added per-call `tokens_out` and `time.monotonic()` logging to the three
sequential LLM calls in the Week 10 τ²-Bench harness. Decomposed the `$0.00345/task` cost
figure in `memo.md` into prefill and decode components. Corrected the "deterministic" claim
about temperature=0.0 in `method.md` to "greedy decoding (most probable token at each step;
exact token sequence is identical across runs on the same hardware/batching state)."

**Why it matters:** A baseline report that says "p95=203.5s" with no attribution is not
actionable. This commit turns a number into a diagnosis: output token count on the composition
call has 0.91 Pearson correlation with elapsed time, identifying decode as the bottleneck
before any optimization work begins. An FDE who can instrument first and optimize second
saves engineering time and avoids fixing the wrong thing.

---

### 2. Hybrid reply classifier — `email_reply.py`

**What changed:** Replaced the pure keyword-based reply intent classifier with a hybrid:
keywords handle high-confidence intents (`hard_no`, `engaged_booking`); ambiguous replies
invoke the model with a two-turn context window (original email + reply). Corrected the
Week 10 README from "multi-turn agent" to "stateful pipeline with model-invoked reply
classification."

**Why it matters:** The original system silently dropped signals — a prospect mentioning
layoffs triggered `soft_defer` on the word "timing" and the layoff flag was never surfaced
downstream. Production B2B sales agents fail in exactly this way: they appear to handle
multi-turn context because they respond to each message, but they have no memory of the
conversation. The fix (prompt accumulation with a two-turn messages array) is the minimum
viable multi-turn mechanism and costs zero additional infrastructure.

---

### 3. P-15 scaffolding fix — `icp_classifier.py` and `main.py`

**What changed:** Removed the hardcoded `disqualified=False` on every return path in
`icp_classifier.py` (lines 119 and 136). Added a pre-composition guard in `main.py` that
reads the `disqualified` field and exits the pipeline before the email composition call.

**Why it matters:** This bug caused the agent to draft and send outreach emails to
disqualified prospects — exactly the behavior a compliance system is designed to prevent.
Identifying it required understanding that the model's tool-call output was correct all along;
the scaffolding was discarding the disqualification signal. The commit demonstrates the
debugging instinct most FDE roles require: check Python before changing the model.

---

### 4. PASS bias root-cause documentation — `memo.md`, `scoring_evaluator.py`, `hyperparameters.json`, `train_simpo.py`

**What changed:** Four coordinated edits to the Week 11 training artifacts:
- `memo.md` Skeptic's Appendix: replaced vague "PASS bias" note with mechanistic explanation
  (4.9× output length asymmetry + β=2.0 over-regularization on 10 sparse disqualification
  pairs) and a cheapest-first diagnostic plan
- `scoring_evaluator.py` `summary_stats()`: added `false_pass_rate_on_expected_fail` and
  `category_recall_on_expected_fail` metrics
- `hyperparameters.json`: added `beta_rationale`, `loss_type_note`, and `simpo_next_run`
  block documenting the prerequisite (equalize output lengths before enabling SimPO)
- `train_simpo.py`: added comment above `CPO_BETA=2.0` explaining β as a global reward-scale
  knob and listing the next-run diagnostic sweep

**Why it matters:** A judge that reports 92.7% overall accuracy but has 18% false-negative
rate on disqualification tasks is a liability in production — it approves emails to companies
in active layoffs. Documenting the root cause is not enough; the portfolio now shows the
evaluation instrumentation (`false_pass_rate_on_expected_fail`) needed to detect this bias
in production, the cheapest fix (threshold recalibration before retraining), and the ordered
intervention plan that avoids wasting a training run.

---

### 5. Peer grounding commit — Week 11 model card and decision memo (via Nuhamin's repo)

**What changed:** Grounded Nuhamin's explainer on DPO mechanics in three documentation
commits to the Week 11 artifacts (peer's repo): updated model card to frame the 76.92% →
96.15% accuracy improvement as a decision-boundary and calibration shift rather than a
leaderboard claim, and updated the decision memo to list the empirical checks still needed
before claiming full generalization (train/dev reward margins, held-out margin distributions,
per-source-mode accuracy, failed-pair inspection, threshold sweeps).

**Why it matters:** An artifact that says "DPO beat prompting by 19.2pp" is a result. An
artifact that says "DPO likely moved the decision boundary into adapter weights; here are
five empirical checks before claiming generalization" is an engineering judgment. The peer
grounding commit demonstrates the ability to synthesize a collaborator's explanation into
actionable documentation updates — a core FDE skill when deploying models you did not train.

---

## What the Portfolio Demonstrates

After these five commits, the Weeks 10 and 11 portfolio shows:

| Capability | Evidence |
|---|---|
| Production debugging instinct | P-15 bug found and fixed at correct layer (scaffolding, not model) |
| Latency diagnosis | Per-call instrumentation + decode bottleneck attribution via correlation |
| Evaluation engineering | `false_pass_rate_on_expected_fail` metric and cheapest-first diagnostic plan |
| Post-training literacy | β/SimPO/length-normalization root cause documented with ordered intervention plan |
| Collaborative deployment judgment | Peer grounding commit turns accuracy numbers into mechanism and empirical checks |

The portfolio now shows not just that the systems work (the Week 10 and 11 artifacts already
showed that) but that the engineer who built them can read them as a hostile reviewer, name
the places they are incomplete, and close those gaps with targeted, cost-ordered fixes.
