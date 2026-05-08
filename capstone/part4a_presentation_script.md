# Part 4A — Final Presentation Script & Slide Guide
**Presenter:** Eyobed Feleke
**Duration:** 5–7 minutes
**Audience:** Horizon Services Group stakeholders (mixed technical and non-technical)

---

> **How to use this document:**
> Each section below is one slide. Read the speaker notes aloud in your own words — do not read them verbatim. The bullet points are what appear on screen. Aim for a natural, confident delivery.

---

## Slide 1 — Title Slide

**On screen:**
```
Horizon Internal Knowledge Assistant
A Proposed AI Solution for Faster, Consistent Policy Answers

Prepared by: Eyobed Feleke
Date: May 8, 2026
```

**Speaker notes (say this):**
"Thank you for your time. Today I'm presenting a proposed AI assistant designed specifically for Horizon Services Group. The goal is simple: help your employees find accurate policy answers quickly, without having to chase managers or HR for information that should be self-service."

---

## Slide 2 — The Problem

**On screen:**
- 350 employees across HR, Finance, Operations, Delivery, and IT
- Policy information spread across documents, Slack, and manager decisions
- Employees receive inconsistent answers depending on who they ask
- Managers and HR spend time on repetitive, low-value questions
- Problem grows as the company continues to scale

**Speaker notes:**
"Horizon has grown rapidly over the last 18 months, and that growth has exposed a knowledge gap. Employees can't find reliable answers. Two people ask the same leave question and get two different answers. HR and managers are answering the same questions over and over. This is a productivity problem, a consistency problem, and — left unchecked — a compliance risk."

---

## Slide 3 — The Proposed Solution

**On screen:**
- An internal AI assistant that answers policy questions in Slack
- Draws only from Horizon's approved policy documents
- Returns structured, source-cited answers in seconds
- Knows when to escalate — never guesses when it's unsure
- MVP delivers in 6 weeks

**Speaker notes:**
"The solution is a conversational AI assistant, accessed directly from Slack — where your employees already work. Employees ask a question, the assistant searches your own policy documents, and returns a clear answer that cites exactly which policy it used. Critically, when a question is ambiguous or involves a conflict, the assistant flags it and directs the employee to the right person — it does not make up an answer."

---

## Slide 4 — How It Works (Architecture)

**On screen:**
```
Employee asks question in Slack
         ↓
Query is cleaned and classified
         ↓
Knowledge base is searched (Policy Notes 1–5)
         ↓
LLM generates structured answer
         ↓
Confidence check — High/Medium/Low
         ↓
Answer delivered  OR  Escalated to HR/Finance/Manager
```

**Speaker notes:**
"Here's the flow at a high level. The employee types a question. The system cleans the input, searches the knowledge base — which contains your five approved policy notes — and generates an answer using an LLM. Before that answer reaches the employee, the system checks its own confidence. High confidence means the answer is delivered. Low confidence means the employee is told to speak to the right human contact. Nothing is guessed."

---

## Slide 5 — Example: What an Answer Looks Like

**On screen:**
```
Question: "I work remotely 4 days a week. Can I request internet stipend support?"

Direct Answer: Yes — 4 days/week meets the eligibility threshold.

Explanation: Policy allows remote work support for employees working
remotely more than 3 days/week. Manager approval is required.

Source: Policy Note 2 — Equipment & Remote Work

Confidence: High

Next Step: Submit request to your direct manager for approval.
```

**Speaker notes:**
"This is what the employee sees. A direct answer, a brief explanation, the exact policy it came from, and a confidence level. The next step tells them what to do — in this case, they now know they're eligible and they need to go to their manager. The whole interaction takes under 10 seconds, and the employee didn't need to ping HR."

---

## Slide 6 — Key Design Decisions and Trade-offs

**On screen:**
- **Formal policy over Slack messages** — consistency and audit-readiness
- **No approval decisions** — the assistant informs, never decides
- **Escalate rather than guess** — accuracy over coverage in Version 1
- **Slack as the interface** — zero new tool adoption required

**Speaker notes:**
"Three decisions shaped the design. First, when informal Slack guidance conflicts with formal policy — and we found four such conflicts in your materials — the assistant always follows the formal policy and flags the conflict. Second, the assistant never implies an approval. Telling an employee they're eligible is not the same as approving their request. Third, Version 1 is built to be cautious. If confidence is low, the assistant escalates rather than guessing. This may mean some questions go to humans that could eventually be automated — but it protects against the larger risk of wrong answers being trusted."

---

## Slide 7 — Key Risks and How They Are Managed

**On screen:**
| Risk | Mitigation |
|------|------------|
| Wrong answer delivered confidently | LLM constrained to retrieved content only |
| Policy documents go out of date | Monthly review process with source owners |
| Policy conflicts mislead employees | Conflict detection flags and escalates |
| Confidential data enters the system | Input filter blocks sensitive queries |
| Too many escalations overwhelm HR | Weekly monitoring with tuning threshold |

**Speaker notes:**
"No AI system is risk-free, and I want to be honest about that. The five main risks are hallucination, stale content, policy conflicts, confidential data, and escalation overload. Each has a concrete mitigation built into the design. The most important one for your context is the policy conflict issue — we identified four specific conflicts in the materials provided, and the system handles each one by flagging the conflict rather than picking a side."

---

## Slide 8 — MVP Scope and Timeline

**On screen:**
**Version 1 includes:**
- 5 policy domains (Leave, Equipment, Reimbursement, Communication, AI Tools)
- Slack bot interface
- Structured answers with source citations
- Conflict detection and escalation routing
- Logging and feedback collection

**Version 1 excludes:**
- Live HR system integration
- Multi-language support
- Analytics dashboard

**6-week build plan:**
Weeks 1–2: Policy sign-off + knowledge base | Weeks 3–4: Retrieval + answer generation | Weeks 5–6: Guardrails + Slack + pilot

**Speaker notes:**
"Version 1 is deliberately scoped to what is achievable in six weeks with a small team. We cover the five policy areas from your materials. We do not connect to live HR systems or build a custom dashboard — those come in Version 2 once we've validated the core experience. The six-week plan ends with a controlled pilot of 10 to 15 employees, giving us real usage data before broader rollout."

---

## Slide 9 — What We Need From Horizon to Move Forward

**On screen:**
- Written confirmation that the 5 policy notes are current and authoritative
- Official resolution of the 4 identified policy conflicts
- IT permission to deploy a Slack bot to the company workspace
- Designated policy owners per domain (HR, Finance, IT)

**Speaker notes:**
"Before the engineering team can build, we need four things from your side. The most important is resolution of the policy conflicts. The assistant cannot handle ambiguous guidance correctly until you have a documented official position on each conflict. This is not a technical blocker — it is an organisational one, and it needs to be resolved before or alongside the build."

---

## Slide 10 — Recommended Next Steps

**On screen:**
1. Horizon confirms policy documents are current — **Week 1**
2. Policy conflicts resolved and documented — **Week 1–2**
3. IT enables Slack bot — **Week 2**
4. Engineering begins build — **Week 2**
5. Pilot with 10–15 employees — **Week 6**
6. Full rollout decision based on pilot results — **Week 8**

**Speaker notes:**
"To close — this is a practical, low-risk solution that can be live within six weeks. The technology is proven. The risk mitigations are built in from day one. The main dependencies are on your side: confirming the policy documents and resolving the four conflicts. I'm happy to take questions and discuss any aspect of the design in more detail. Thank you."

---

## Presentation Tips

- **Total time target:** 6 minutes (roughly 35 seconds per slide)
- **Do not read slides verbatim** — glance at the bullet points, speak from the speaker notes in your own voice
- **Slide 5 (the example)** is your strongest visual — pause here and walk through each field slowly
- **Slide 7 (risks)** shows maturity — don't rush past it; assessors value honest risk awareness
- **End confidently** — the last line of Slide 10 is your close, deliver it directly to camera
