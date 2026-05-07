# Grounding Commit — Day 3

**Asker:** Eyobed Feleke  
**Commit:** [b5473fe — docs: ground Week 12 Day 3 gap — CPO β mechanics and PASS bias diagnosis](https://github.com/eyobed7b/Sales-Evaluation-Bench-trp/commit/b5473fe)  
**Artifacts edited:**
- `week11/Sales-Evaluation-Bench-trp/training/train_simpo.py`
- `week11/Sales-Evaluation-Bench-trp/training/hyperparameters.json`
- `week11/Sales-Evaluation-Bench-trp/scoring_evaluator.py`
- `week11/Sales-Evaluation-Bench-trp/memo.md`

---

## What Changed

**train_simpo.py** — added a 20-line comment block above `CPO_BETA = 2.0` explaining
what β controls (preference-pressure scalar, global reward-scale knob affecting all pairs),
why β=2.0 may over-regularize on sparse disqualification pairs, and the output-length
asymmetry warning (PASS ~11 tokens, FAIL ~54 tokens) that must be addressed before
enabling `loss_type="simpo"`. Added the next-run sweep config as commented-out code:

```python
# config = CPOConfig(beta=1.0, loss_type="simpo", gamma_beta_ratio=0.4, ...)
```

**hyperparameters.json** — added `beta_rationale`, `loss_type_note`, and a
`simpo_next_run` block documenting the prerequisite output-length equalization step and
the proposed next config (`beta=1.0`, `loss_type="simpo"`, `gamma_beta_ratio=0.4`).

**scoring_evaluator.py** — added two new fields to `summary_stats()`:
- `false_pass_rate_on_expected_fail` — fraction of expected-fail tasks the judge
  incorrectly passed; the primary PASS bias metric
- `category_recall_on_expected_fail` — per-category breakdown, enabling comparison
  of ICP misclassification recall against other failure categories

**memo.md** — replaced the vague "PASS bias on ICP misclassification tasks is unresolved"
statement with a mechanistic explanation of two root causes (output-length asymmetry under
raw log-prob reward; β over-regularization on sparse disqualification pairs) and a
four-step diagnostic plan: threshold sweep first, then SimPO loss with length equalization,
then β sweep only if needed.

## Why These Changes

Before reading the peer's explainer, the PASS bias was listed as a known limitation
without a mechanism or a fix path. I knew β=2.0 was the config but could not say what β
physically controls, why the length asymmetry matters, or which lever to pull first. After
understanding that β is a global reward-scale knob and that CPO's raw log-prob reward
creates a structural length confound, every artifact could be updated with a specific,
testable claim. The `scoring_evaluator.py` change is the most immediately useful: it adds
the per-category false-PASS diagnostic that was missing and turns the PASS bias from a
reported observation into a measurable, attributable metric.
