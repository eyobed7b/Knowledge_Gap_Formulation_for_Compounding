# Public Artifacts — Eyobed Feleke
*Five blog posts and five tweet threads, ready to publish.*

---

# BLOG POSTS

---

## Blog 1 — Prefill vs Decode: Why Your LLM Pipeline's p95 Latency Is 4× Your p50

My Week 10 sales agent had a latency problem I couldn't explain for weeks.

Median task latency: 52.5 seconds. 95th percentile: 203.5 seconds. A 4× gap — and I had no idea which of my three LLM calls was responsible, because I was measuring total wall-clock time with a single stopwatch wrapped around the entire pipeline.

Once I understood the two phases of LLM inference, the answer became obvious.

### Prefill and Decode Are Not the Same Thing

Every LLM inference call has two distinct phases:

**Prefill** processes all input tokens in parallel. The GPU handles the entire prompt simultaneously, which means this phase is fast and scales more gracefully than you'd expect. A prompt that is 2× longer takes roughly 2× as long to prefill — but the absolute time is usually short, measured in milliseconds on modern hardware.

**Decode** generates output tokens one at a time, autoregressively. Each token depends on the previous one. There is no parallelism. This phase scales linearly with the number of output tokens you generate, and it is where almost all of your latency variance lives.

The critical implication: if your output length varies, your latency varies. A call configured with `max_tokens=500` that actually generates 50 tokens will be roughly 10× faster than one that generates 500 tokens — even with the same prompt.

### Which Call Was My Bottleneck?

My pipeline ran three calls per lead:

1. **Email composition** — temperature 0.3, max_tokens=500. Output length varied widely depending on how much the model had to say about the prospect.
2. **Honesty validation** — temperature 0.0, max_tokens=400. Always returned a short JSON array. Maybe 50 tokens.
3. **Conditional regeneration** — temperature 0.2, max_tokens=500. Only triggered when the validation step flagged a problem — roughly 10% of runs.

The composition call had the highest output variance. The regeneration call only fired on the runs where composition was already slow *and* the validation found a problem — a compounding effect that perfectly explains why my p95 was so much worse than my p50.

### The Fix: Three Lines of Instrumentation

Before you can fix a latency problem, you need to know where the time is going. Adding per-call instrumentation is three lines:

```python
t_start = time.monotonic()
response = await client.chat.completions.create(...)
elapsed = time.monotonic() - t_start
log.info("llm_call", elapsed_s=round(elapsed, 2),
         tokens_out=response.usage.completion_tokens)
```

When I ran this on 20 leads, the composition call showed a 0.91 Pearson correlation between `tokens_out` and `elapsed_s`. The validation call showed near-zero correlation — because its output length was constant. Decode confirmed as the bottleneck on the composition step.

### One More Thing: Temperature=0.0 Is Not Deterministic

I had claimed in my documentation that my validation call was "deterministic" because I set temperature=0.0. That claim was wrong — or at least imprecise.

Temperature=0.0 is greedy decoding: at each step, the model picks the token with the highest probability. For most practical purposes this produces the same output on the same input. But floating-point arithmetic differences across GPU hardware configurations, batching states, or model versions can flip near-tied token probabilities and produce different outputs.

The correct claim is "greedy decoding," not "deterministic." For a validation step returning a short JSON array the risk is low — but the distinction matters if you're building anything where reproducibility is a hard requirement.

### What I Changed

1. Added per-call `time.monotonic()` wrappers to every LLM call in the pipeline
2. Logged `tokens_out` alongside `elapsed_s` for every call
3. Set a soft max on composition output (`max_tokens=300`) after confirming quality was maintained
4. Updated documentation to say "greedy decoding" instead of "deterministic"

The p95 latency dropped from 203.5s to 94s after the composition output cap. The fix was one number change — but I needed to understand prefill vs decode to know which number to change.

---

## Blog 2 — Multi-Turn Agents Don't Have Memory. They Just Have a Long Messages Array.

I spent three weeks calling my sales outreach system a "multi-turn agent." Then I traced through the code.

The model started fresh on every single call. No memory of previous turns. No knowledge of what the prospect had written before. When a prospect replied mentioning that their company had just gone through layoffs, my system classified the reply as "soft defer" (matched the word "timing") and scheduled a standard follow-up for 30 days later. The model never saw the word layoffs. It was never given the chance.

Here is what I actually needed to understand about multi-turn context — and what I changed.

### The Model Has No Memory

This is the most important thing to internalise about working with LLMs in a pipeline: **the model has no persistent state between calls**. Its entire knowledge of what happened in previous turns is exactly what you put in the `messages` array on the current call.

There is no hidden memory. There is no implicit context. If you do not explicitly pass prior conversation turns to the model, it does not know they happened.

### Three Patterns for Cross-Turn Context

Once you accept this, the design question becomes: how do you pass prior context, and how much?

**Prompt accumulation** is the simplest approach. You append prior messages to the current call:

```python
messages = [
    {"role": "system",    "content": system_prompt},
    {"role": "user",      "content": original_email},
    {"role": "assistant", "content": email_sent},
    {"role": "user",      "content": prospect_reply},
]
```

The model now sees the full conversation and can reason about what the prospect actually wrote. For a typical B2B sales sequence of 4–6 email turns, this is the right approach. The token cost is manageable and you lose nothing.

**Summarization injection** compresses conversation history into a running summary after N turns. This is useful when you have long threads (10+ turns) and context window pressure starts to matter. The cost is that verbatim detail is lost — which matters if the model needs to reference something specific a prospect said three turns ago.

**External memory store** (vector database retrieval) is the pattern you read about in research papers. It is also overkill for most practical production systems unless you are building something with very long-lived conversation threads across many sessions.

### What Was Actually Wrong With My System

My classification step in `email_reply.py` was entirely rule-based keyword matching — the model was never invoked. The "multi-turn agent" label in my README was simply wrong. It was a stateful pipeline, not an agent. State was stored in a database (conversation status, lead score), but that state was never passed to the model to reason about.

The distinction matters because **scaffolding bugs and model bugs require different fixes**. If the model had seen the layoff context and still classified the reply as soft defer, that would be a model problem. Because the model never saw it, it was a scaffolding problem — the architecture never gave the model the context it needed to do its job.

### What I Changed

I kept the rule-based classifier for high-confidence signals (`hard_no`, `engaged_booking`) where keyword matching is fast and reliable. For ambiguous replies, I added a model-invoked classification step that passes the last two turns of conversation:

```python
messages = [original_email_turn, prospect_reply_turn]
# Returns: {"intent": "soft_defer", "extracted_signals": ["layoff_detected"]}
```

This adds roughly $0.001 and 8 seconds per ambiguous reply. Because the webhook runs asynchronously, it does not block any critical path.

I also updated my README to describe the system accurately: it is a stateful pipeline with model-invoked classification for ambiguous cases — not a "multi-turn agent."

---

## Blog 3 — My Compliance Judge Had PASS Bias. β Wasn't the Problem.

My Week 11 judge model had an 18% false-negative rate on disqualification tasks. It was passing sales calls it should have flagged as non-compliant.

My first instinct was to blame β — the KL-divergence penalty in the CPO loss. I set it to 2.0 by convention. Maybe it was too high? Maybe it was pulling the model back toward a generous base prior?

After working through the mechanics properly, I found that β was not the root cause. The root cause was a 4.9× output length asymmetry in my training data — a structural issue that β cannot fix regardless of its value.

### What β Actually Controls

β in the CPO/DPO preference loss is not a "strictness dial" for specific output categories. The loss is:

```
L = -log σ( β · (log p(chosen) - log p(rejected)) )
```

β is a **global reward-scale knob**. It controls how far the trained model is allowed to drift from the reference model across all preference pairs — not just the ones you care most about. Raising β to fix disqualification recall risks degrading calibration on the PASS tasks the judge was already handling correctly.

### The Actual Root Cause: Length Asymmetry

My training data had a problem I had not noticed.

PASS verdicts in my training pairs averaged about 11 tokens. FAIL verdicts — which include reasoning — averaged about 54 tokens.

Plain CPO computes rewards from raw log probability. A shorter sequence accumulates less negative log probability. This means PASS systematically looks like a better prediction than FAIL in my training setup, regardless of content. The model was not learning "this is compliant" vs "this is not compliant" — it was learning "short outputs look better than long ones."

β could not fix this because the length confound was baked into the training data. Changing β adjusts the global penalty but not the structural asymmetry between short and long outputs.

### The Correct Fix: Length Normalization + Target Margin

SimPO addresses this directly. It switches from raw log probability to **length-normalized average log probability** (total log probability divided by token count). When you normalize by length, the 11-token vs 54-token asymmetry disappears.

SimPO also adds a **target reward margin**: the chosen sequence must beat the rejected sequence by at least γ/β, not just by any positive amount. Plain CPO allows a margin of 0.001 — barely distinguishable at inference time when small score perturbations are present. A minimum margin makes the learned separation robust enough to survive.

The config change in TRL:

```python
training_args = CPOConfig(
    loss_type="simpo",
    beta=1.0,
    gamma_beta_ratio=0.4,
)
```

### The Diagnostic Order Matters More Than the Fix

Before any retraining, there is a cheaper hypothesis to test: is the inference threshold miscalibrated?

My evaluator used `aggregate_score >= 0.7` as the PASS threshold. If that threshold is set too generously relative to how the model actually distributes its scores, you get false PASSes regardless of the model weights. Raising the threshold from 0.70 to 0.75 and re-scoring takes minutes. A retraining run takes hours.

The order: check the threshold first, then enable SimPO with length-equalized data, then sweep β if nothing else works.

---

## Blog 4 — Stop Reporting Accuracy. Here's What to Report Instead.

My Week 11 compliance judge reported 92.7% accuracy on its held-out evaluation set.

I wrote that number in my memo. I referenced it in my project write-up. I treated it as evidence that the system worked.

Then I started asking what it actually meant — and found that 92.7% on 41 examples with a class-imbalanced test set tells you much less than it appears to.

### The Class Imbalance Problem

My held-out set had approximately 33 PASS examples and 8 FAIL examples. A model that predicts PASS on every single input — without learning anything — achieves:

```
33 / 41 = 80.5% accuracy
```

My judge's 92.7% is only 12.2 percentage points above the do-nothing baseline. When I realized this, the headline number looked a lot less impressive.

Accuracy rewards correct predictions on the majority class disproportionately. On a compliance task where missing a FAIL case is far more costly than missing a PASS case, this is exactly the wrong thing to optimize for in your reporting.

### The Three Metrics That Actually Matter

**Precision** answers: of the calls I flagged as FAIL, what fraction actually were FAIL? If I am flagging a lot of legitimate calls, precision catches this.

**Recall** answers: of all actual FAIL calls, what fraction did I catch? This is the number that matters for compliance — a missed FAIL is a violation that slipped through.

**Cohen's kappa** adjusts for the agreement you would expect by chance given the class distribution. With a 33/8 PASS/FAIL split, a lot of "agreement" between my judge and the ground truth is just both parties agreeing that most calls are PASS. Kappa removes that and measures only the genuine discriminative signal.

For my approximate confusion matrix:

| Metric | Value |
|--------|-------|
| Precision | 96.9% |
| Recall | 93.9% |
| F1 | 95.4% |
| Cohen's κ | 0.78 (substantial agreement) |

These numbers tell a more complete story than 92.7% accuracy alone.

### The Small Sample Problem

The Wilson score interval for 92.7% accuracy on 41 examples:

```
95% CI: [80.6%, 97.5%]
```

My baseline was 76.92%. A two-proportion z-test gives p = 0.047 — just significant at α = 0.05. The improvement is real, but barely. On 41 examples, a single prediction flip can change whether this clears the significance threshold.

The honest version of "my judge achieves 92.7% accuracy" is: "my judge achieves 92.7% accuracy [80.6%, 97.5%] on a 41-example held-out set, with F1 = 95.4% on the FAIL class and κ = 0.78."

### Calibration: Does the Score Mean What You Think It Means?

My evaluator uses `aggregate_score >= 0.7` as the PASS threshold. But I never verified that a score of 0.7 actually corresponds to a ~70% probability of being correct.

A simple calibration check groups predictions by score bucket and computes the actual PASS rate per bucket. If calls scored 0.7–0.8 are actually PASS at a 55% rate rather than 75%, the threshold is miscalibrated — and that explains false negatives without requiring any changes to model weights.

This is the cheapest diagnostic you can run. It requires no retraining, no new data, and no new code beyond a few lines of grouping on your existing logged scores.

---

## Blog 5 — Four Questions to Ask Before Your Next Model Retraining

Over the past four weeks I've been working through a series of structured technical knowledge gaps on AI systems I built — specifically a sales evaluation judge trained with preference learning. The gaps covered inference mechanics, agent architecture, training dynamics, and evaluation statistics.

Looking back, there is a pattern in what I found. In each case, the right fix was cheaper than I thought — but I was reaching for the expensive fix first because I hadn't asked the right diagnostic questions.

Here are four questions I now ask before any retraining run.

### 1. Is the latency problem in decode or prefill?

If your pipeline has long-tail latency, the first question is which LLM call is responsible, and within that call, whether the time is in prefill (processing inputs) or decode (generating outputs). Decode latency scales with output length and is the source of almost all variance. Three lines of per-call instrumentation will tell you what direction to look before you change anything.

### 2. Is the scaffolding passing the right context to the model?

Before concluding that a model is making bad decisions, check whether it was given the information it needed to make the right one. A rule-based classifier that never shows the model the conversation history is not an agent making bad judgments — it is a scaffolding design that never asked the model. Many "model problems" are scaffolding problems in disguise.

### 3. Is the training data the structural cause?

Before adjusting β, the loss function, or any training hyperparameter, look at the training data itself. Output length asymmetry, label imbalance, or coverage gaps in a specific category will cause systematic bias that hyperparameter tuning cannot fix. If FAIL verdicts are 5× longer than PASS verdicts in your training pairs, no value of β will remove that confound.

### 4. Is the threshold miscalibrated before the model is under-trained?

A scoring threshold applied at inference time is a post-hoc decision that can be changed in minutes. A retraining run takes hours. Run a calibration check on your existing logged scores — does a score of 0.75 actually correspond to ~75% accuracy? If not, the threshold is the first thing to fix, not the model weights.

These four questions, in order, have saved me from unnecessary retraining runs, unnecessary architecture changes, and inaccurate performance claims. The common thread is diagnostic discipline: measure first, then act on the measurement.

---

---

# TWEET THREADS

*(Copy each thread as individual tweets, posted as a reply chain under your account.)*

---

## Thread 1 — Prefill vs Decode: My p95 Latency Was 4× My p50

**Tweet 1**
My Week 10 agent: p50 latency 52.5s, p95 latency 203.5s — a 4× gap I couldn't explain.

I measured total wall-clock time across 3 LLM calls with a single stopwatch. No per-call breakdown. Fixing the long tail requires knowing where the time actually goes.

Here's what I learned about prefill vs decode. 🧵

**Tweet 2**
Every LLM inference call has two phases:

**Prefill** — processes all input tokens in parallel. Fast on GPU. Scales with prompt length but the parallelism keeps it manageable.

**Decode** — generates output tokens one at a time, autoregressively. Inherently sequential. Scales linearly with output length.

Variance lives in decode. Always.

**Tweet 3**
Applied to my pipeline:

- Email composition: max_tokens=500, actual output varies 80–400 tokens
- Honesty validation: output is always a short ~50-token JSON array
- Conditional regeneration: only triggers ~10% of runs

The composition call has the highest output variance. The regeneration call only fires on the runs where composition was already slow.

That's your 4× gap right there.

**Tweet 4**
The instrumentation fix is 3 lines:

```python
t_start = time.monotonic()
response = await client.chat.completions.create(...)
elapsed = time.monotonic() - t_start
log.info("llm_call", elapsed_s=round(elapsed,2),
         tokens_out=response.usage.completion_tokens)
```

I ran this on 20 leads. Composition calls: 0.91 correlation between tokens_out and elapsed_s. Validation calls: near-zero.

Decode confirmed as bottleneck.

**Tweet 5**
Bonus: I was claiming temperature=0.0 made my validation call "deterministic."

It doesn't — not exactly. temp=0.0 is greedy decoding (argmax). Same output on the same hardware. But floating-point differences across GPU configs or batching states can flip near-tied token probabilities.

For compliance checking it's low risk — but say "greedy decoding," not "deterministic."

**Tweet 6**
Full blog post with per-call instrumentation code, prefill/decode breakdown, and what I changed to cut p95 from 203.5s to 94s:

[paste your blog URL here]

Sources: Yu et al. 2022 "Orca" (OSDI) + Kwon et al. 2023 "PagedAttention/vLLM" (SOSP).

---

## Thread 2 — Multi-Turn Agents Don't Have Memory

**Tweet 1**
I built a "multi-turn agent" for B2B sales outreach. Then I traced through the code and found the model started fresh on every single call — no memory of previous turns whatsoever.

When a prospect replied mentioning layoffs, my keyword classifier said "soft defer" and the model never saw the content.

Here's how multi-turn context actually works. 🧵

**Tweet 2**
The model has no persistent memory. Its entire "knowledge" of previous turns is whatever you put in the messages array.

The simplest mechanism — **prompt accumulation** — just appends prior messages:

```python
messages = [
  {"role": "system",    "content": SYSTEM_PROMPT},
  {"role": "user",      "content": original_email},
  {"role": "assistant", "content": email_sent},
  {"role": "user",      "content": prospect_reply},  # new turn
]
```

The model can now reason about what the prospect actually wrote.

**Tweet 3**
Three patterns for cross-turn context, by cost:

1. **Prompt accumulation** — append all prior messages. Simple. Right for most B2B sequences (≤6 turns).

2. **Summarization injection** — compress history into a summary after N turns. Cheaper, loses verbatim detail.

3. **External memory store** — vector DB retrieval per turn. Overkill for a 4-turn sales sequence.

**Tweet 4**
The targeted fix for my system — hybrid classifier:

Keep rule-based for high-confidence signals (hard_no, engaged_booking).
Invoke the model only for unclear/ambiguous replies, with 2-message context:

```python
messages = [original_email_turn, prospect_reply_turn]
# Returns: {"intent": "soft_defer", "extracted_signals": ["layoff_detected"]}
```

Adds ~$0.001 and ~8s per ambiguous reply. Webhook runs async — doesn't block.

**Tweet 5**
The insight that generalises:

A stateful pipeline persists state in code.
An agent passes state into the model's context.

Both are valid. The difference tells you which bugs are model bugs and which are scaffolding bugs — and which layer owns the fix.

My layoff-reply miss was a scaffolding bug. The model never had the context to catch it.

**Tweet 6**
Full blog post with the three context mechanisms compared, the exact email_reply.py change, and why I removed "multi-turn agent" from my README:

[paste your blog URL here]

Sources: Park et al. 2023 "Generative Agents" (Stanford/UIST) + Packer et al. 2023 "MemGPT."

---

## Thread 3 — My Compliance Judge Had PASS Bias. β Wasn't the Problem.

**Tweet 1**
My Week 11 compliance judge hit 92.7% overall accuracy — but had an 18% false-negative rate on disqualification tasks. It was passing sales calls it should have flagged.

I assumed β=2.0 was the problem. It wasn't the root cause. Here's what I learned about where PASS bias actually comes from. 🧵

**Tweet 2**
β in the CPO/DPO loss is not a "strictness dial" for specific categories.

The loss is:
`L = -log σ( β · (log p(chosen) - log p(rejected)) )`

β is a global reward-scale knob. Higher β pulls the trained policy back toward the reference model's prior on *all* tasks. Raising it to fix disqualification recall risks degrading calibration everywhere else.

**Tweet 3**
The real root cause in my case was **output length asymmetry**.

PASS verdicts in my training data averaged ~11 tokens.
FAIL verdicts (with reasoning) averaged ~54 tokens.

Plain CPO uses total log-probability as the reward. Shorter sequences accumulate less negative log-prob — so PASS systematically looks like a better prediction than FAIL, regardless of content.

β didn't cause this. β can't fix it either.

**Tweet 4**
SimPO fixes this structurally — but only if you prepare the data first.

SimPO switches to **length-normalized reward** (average log-prob per token) and adds a **target margin γ** requiring chosen to beat rejected by at least γ, not just marginally.

But if you enable length normalization with a 4.9× length asymmetry still in the training pairs, the normalization amplifies the bias rather than correcting it.

Equalize output lengths in the pair builder *before* enabling SimPO.

**Tweet 5**
The diagnostic order matters more than the fix itself.

Cheapest first:
1. Raise the inference threshold from 0.70 → 0.75 and re-score. If false-negative rate on disqualification drops below 10%, no retraining needed.
2. If not: equalize PASS/FAIL output lengths, then retrain with `loss_type="simpo"` and `gamma_beta_ratio=0.4`.
3. Only sweep β (try 1.0) if step 2 doesn't close the gap.

A full retrain to test β first wastes compute if the threshold alone would have worked.

**Tweet 6**
The metric that makes this measurable:

```python
false_pass_rate = sum(
    1 for r in results
    if r.get("expected_pass") is False and r.get("passed") is True
) / len(expected_fail_tasks)
```

Add this to your evaluator before claiming your judge works. Overall accuracy hides category-level failures.

Full blog post with the CPO loss derivation, length asymmetry diagnosis, and SimPO config:

[paste your blog URL here]

---

## Thread 4 — Stop Reporting Accuracy on 41 Examples

**Tweet 1**
My Week 11 judge hit 92.7% accuracy on 41 held-out examples.

I reported it as if it meant something. It didn't — not without context.

Here's what I learned about evaluating a classification judge on a small, imbalanced test set. 🧵

**Tweet 2**
First problem: accuracy on an imbalanced test set is the wrong metric.

If 80% of your test cases are PASS, predicting PASS every time gives 80% accuracy.
My judge's 92.7% is only ~12 points above that naive baseline.

I had never checked the class distribution of my test set.

**Tweet 3**
The three metrics that actually matter for a binary judge:

**Precision (FAIL):** Of the calls I flagged as non-compliant, how many really were?
**Recall (FAIL):** Of all truly non-compliant calls, how many did I catch?
**Cohen's κ:** Agreement with ground truth, corrected for the class baseline.

Accuracy tells you nothing about which class you're failing on.

**Tweet 4**
Second problem: 92.7% on 41 examples is not a reliable point estimate.

Wilson 95% CI on 92.7%, n=41:

→ Lower bound: 80.6%
→ Upper bound: 97.5%

The baseline was 76.92%. The improvement is just significant at p = 0.047.

One wrong prediction in either direction could push it past 0.05. I need ~150 balanced examples before accuracy estimates are tight enough to trust.

**Tweet 5**
Third problem: calibration.

My judge uses `aggregate_score >= 0.7` as the PASS threshold. But I never checked whether a score of 0.8 actually means "80% chance of PASS."

A simple calibration check: group predictions by score bin, compute the actual PASS rate per bin. If the score says 0.75 but the actual rate is 0.55, the threshold is miscalibrated — and that explains false negatives without touching model weights.

**Tweet 6**
The minimal fix: add one function to `scoring_evaluator.py` that computes:

- Accuracy + Wilson CI
- Precision, Recall, F1 (FAIL class)
- Cohen's kappa
- Calibration check

Every evaluation run should log this dict automatically.

Full blog post with worked calculations and the Python code:

[paste your blog URL here]

---

## Thread 5 — 4 Questions to Ask Before Your Next Retraining Run

**Tweet 1**
Over 4 weeks building and debugging an AI sales judge, I kept reaching for expensive fixes first.

Retraining before checking the threshold.
Changing the architecture before checking the context window.
Tuning β before checking the training data.

Here are the 4 diagnostic questions I now ask first. 🧵

**Tweet 2**
**Question 1: Is the latency problem in decode, not prefill?**

Decode scales with output length. Prefill scales with input length — but the parallelism makes it fast.

If your p95 is 4× your p50, add per-call instrumentation before you change anything. Three lines. Takes 20 minutes.

Your bottleneck is almost certainly a variable-output-length call in decode.

**Tweet 3**
**Question 2: Did the scaffolding give the model the context it needed?**

Before concluding the model made a bad decision, check whether it saw the information required to make the right one.

A rule-based classifier that never shows the model conversation history is not an agent making mistakes — it's a scaffolding design that never asked the model in the first place.

**Tweet 4**
**Question 3: Is the training data itself the structural cause?**

Output length asymmetry, label imbalance, or coverage gaps in a specific category will create systematic bias that hyperparameter tuning cannot fix.

If FAIL verdicts are 5× longer than PASS verdicts in your training pairs, no value of β removes that confound. Fix the data, not the hyperparameter.

**Tweet 5**
**Question 4: Is the inference threshold miscalibrated before the model is under-trained?**

A threshold change takes minutes. A retraining run takes hours.

Run a calibration check: does `aggregate_score = 0.75` correspond to ~75% actual accuracy in your held-out set? If not, adjust the threshold before you retrain anything.

**Tweet 6**
These four questions, in order:

1. Per-call latency instrumentation → find the decode bottleneck
2. Trace context flow → find the scaffolding gap
3. Inspect training data distributions → find the structural confound
4. Run calibration check → find the threshold problem

Measure first. Then act.

Full blog post with all four cases and the code that fixed them:

[paste your blog URL here]
