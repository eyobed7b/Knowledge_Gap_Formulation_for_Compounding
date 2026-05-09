# Knowledge Gap Formulation for Compounding
**10 Academy — Week 12 Capstone**
**Author:** Eyobed Feleke | eyobed@10academy.org

---

## Overview

This repository documents four days of structured knowledge gap formulation work, paired with a full Week 12 capstone AI engineer client delivery simulation. Each pair day follows a fixed format: a precise technical question grounded in Week 11 artifacts, a peer-written explainer, a tweet thread, a sources file, an evening call debrief, and a grounding commit.

---

## Repository Structure

```
├── pair_DAY_1/          # Inference-time mechanics (prefill vs decode, latency)
├── pair_DAY_2/          # Agent and tool-use internals (multi-turn context)
├── pair_DAY_3/          # Training and post-training mechanics (β, SimPO, threshold)
├── pair_DAY_4/          # Evaluation and statistics (Wilson CI, calibration, kappa)
├── capstone/            # Week 12 client delivery simulation — Horizon Services Group
│   ├── part1_problem_framing.md
│   ├── part2_ai_system_proposal.md
│   ├── part3_client_deliverable.md
│   ├── part4a_presentation_script.md
│   ├── part4b_ai_reflection.md
│   └── presentation_dashboard.html
├── public_artifacts.md  # Full text of all 5 blog posts and 5 tweet threads
└── README.md
```

---

## Public Artifacts

### Blog Posts

| # | Title | Platform | Link |
|---|-------|----------|------|
| 1 | Prefill vs Decode: Why Your LLM Pipeline's p95 Latency Is 4× Your p50 | Medium | [Read →](https://medium.com/p/a3ff60b46315?postPublishedType=initial) |
| 2 | Multi-Turn Agents Don't Have Memory — They Just Have a Long Messages Array | Medium | [Read →](https://medium.com/@eyobed7b/multi-turn-agents-dont-have-memory-they-just-have-a-long-messages-array-fa1b3452c41b) |
| 3 | My Compliance Judge Had PASS Bias. β Wasn't the Problem. | Medium | [Read →](https://medium.com/@eyobed7b/the-technical-deep-dive-best-for-linkedin-goal-showcases-professional-leadership-92150c84a3f6) |
| 4 | Four Questions to Ask Before Your Next Model Retraining | Substack | [Read →](https://eyobedfeleke.substack.com/p/four-questions-to-ask-before-your?r=50a2c3) |
| 5 | Stop Reporting Accuracy: What to Say Instead When You Have 41 Examples and Class Imbalance | *(pending)* | — |

### Tweet Threads

| # | Topic | Link |
|---|-------|------|
| 1 | Prefill vs decode — the source of your p95 long-tail | [View thread →](https://x.com/i/status/2052834814462402749) |
| 2 | Multi-turn agents and context across turns | [View thread →](https://x.com/i/status/2052835245087461838) |
| 3 | PASS bias, β, and SimPO length asymmetry | [View thread →](https://x.com/i/status/2052835485676941575) |
| 4 | Stop reporting accuracy on 41 imbalanced examples | [View thread →](https://x.com/i/status/2053177323210338653) |
| 5 | 4 questions to ask before your next retraining run | [View thread →](https://x.com/i/status/2053177489782878701) |

---

## Pair Day Summaries

### Day 1 — Inference-Time Mechanics
**Gap:** What determines whether a single LLM inference call is dominated by prefill or decode, and which of three sequential pipeline calls causes a 4× p50→p95 latency gap?
**Outcome:** Identified decode phase on the email composition call as the bottleneck. Added per-call instrumentation. Corrected a "deterministic" temperature=0.0 claim to "greedy decoding."

### Day 2 — Agent and Tool-Use Internals
**Gap:** What is the actual mechanism by which a multi-turn agent maintains context across turns, and what breaks when a rule-based classifier replaces model reasoning in a reply-classification step?
**Outcome:** Understood the three context mechanisms (prompt accumulation, summarization, external memory). Added model-invoked hybrid classification for ambiguous replies. Corrected the README from "multi-turn agent" to "stateful pipeline."

### Day 3 — Training and Post-Training Mechanics
**Gap:** If a held-out judge shows PASS bias on disqualification tasks, is the root cause a β problem, a missing SimPO target-margin problem, or an inference-threshold problem?
**Outcome:** Identified output length asymmetry (11 vs 54 tokens) as the structural root cause — not β. Established a diagnostic ordering: threshold calibration first, then SimPO + length equalization, then β sweep.

### Day 4 — Evaluation and Statistics
**Gap:** Is 92.7% accuracy on 41 imbalanced examples a reliable estimate, and what metrics and confidence intervals should replace it for a compliance judge?
**Outcome:** Added Wilson CI [80.6%, 97.5%], Cohen's κ = 0.78, F1 = 95.4%, and bootstrap CIs to `scoring_evaluator.py`. Added calibration check for `aggregate_score`. Updated memo to report full metrics table.

---

## Week 12 Capstone — Horizon Services Group

A four-part client delivery simulation designing an internal AI assistant for a 350-person services company.

| Part | Deliverable |
|------|-------------|
| Part 1 | Problem Framing & Clarification Brief |
| Part 2 | AI System Proposal (RAG architecture, prompts, guardrails, MVP scope) |
| Part 3 | Client-Ready Structured Report |
| Part 4A | Presentation Script + Interactive Dashboard |
| Part 4B | AI Usage Reflection |

The interactive presentation dashboard can be opened directly: [`capstone/presentation_dashboard.html`](capstone/presentation_dashboard.html)
