# Horizon Internal Knowledge Assistant
## Solution Proposal — Client Deliverable
**Prepared for:** Horizon Services Group
**Prepared by:** Eyobed Feleke, AI Engineer
**Date:** May 8, 2026
**Version:** 1.0 — Draft for Review

---

## 1. Executive Summary

Horizon Services Group is growing fast, but internal knowledge has not kept pace. Employees across HR, Finance, Operations, and Delivery are spending unnecessary time searching for policy answers or waiting for a manager to respond — often receiving inconsistent guidance depending on who they ask. This creates friction for employees, overloads managers and HR, and introduces compliance risk as the company scales.

This proposal outlines a practical, low-risk internal AI assistant that gives employees fast, accurate, policy-backed answers to their most frequent questions. The assistant retrieves information from Horizon's own approved policy documents, generates structured responses that cite their source, and clearly signals when a question needs human judgment.

The proposed solution can be delivered as a working first version within six weeks, integrated directly into Slack where employees already spend their working day. It is designed to reduce repetitive HR and Finance queries, improve consistency of internal guidance, and free up manager time — without creating new compliance risks or requiring employees to learn a new tool.

---

## 2. Proposed Solution

### How the assistant works

The assistant uses a Retrieval-Augmented Generation (RAG) architecture. When an employee asks a question, the system searches Horizon's internal policy documents for the most relevant content, then uses a large language model to generate a structured answer grounded in that content. Every answer includes a direct response, a policy-based explanation, the source document it drew from, a confidence level, and a suggested next step.

The system does not guess. If the retrieved content is insufficient, ambiguous, or contains a conflict between sources, the assistant flags this and directs the employee to the appropriate team — HR, Finance, IT, or their manager — rather than producing an unreliable answer.

### What it can answer

The Version 1 assistant covers four policy domains:

- **Leave & Time-Off** — annual leave entitlements, carry-forward rules, sick leave documentation, emergency leave
- **Equipment & Remote Work** — what support is available, eligibility thresholds, who approves requests, lost or damaged equipment
- **Expense Reimbursement** — eligible categories, submission deadlines, required documentation, late submission exceptions
- **Internal Communication & AI Tool Usage** — escalation rules, Slack vs. email norms, permitted AI tool use, confidentiality requirements

### What it will not do

The assistant is a policy information tool, not a decision-making system. It will not:

- Approve leave requests, reimbursements, or equipment orders
- Access or reference individual employee records
- Handle questions outside the four scoped domains
- Process queries that contain confidential client information

---

## 3. Example User Journey

The following example shows how the assistant handles a real question from the sample materials provided.

---

**Employee question:**
*"I work remotely 4 days per week. Can I request internet stipend support?"*

---

**Assistant response:**

**Direct Answer:**
Yes, you are likely eligible to request internet stipend support. Working remotely 4 days per week meets the eligibility threshold in the current policy.

**Explanation:**
According to the Equipment & Remote Work policy, employees working remotely more than 3 days per week may request remote work support, which includes an internet stipend. However, all equipment and stipend requests require direct manager approval before they can be processed.

**Source Reference:**
Policy Note 2 — Equipment & Remote Work

**Confidence Level:**
High — the policy directly addresses this scenario.

**Escalation Needed?**
No

**Suggested Next Step:**
Submit your internet stipend request to your direct manager for approval. If you are unsure how to submit the request, contact HR for the correct process.

---

This response gives the employee a clear, actionable answer in under 10 seconds, without requiring manager involvement for the initial information. The employee now knows they are eligible, understands the condition (manager approval), and knows exactly what to do next.

---

## 4. Key Design Decisions

### Decision 1 — Formal policy notes take precedence over informal Slack guidance

During the review of client materials, several cases were identified where informal Slack messages from managers or HR coordinators contradicted the formal policy notes. For example, an Operations Manager suggested that late-night work meals could be reimbursed, while the Finance policy note explicitly limits food reimbursement to approved travel and approved client meetings.

The assistant is designed to follow formal policy notes in all cases. When a conflict exists, the assistant will flag it to the employee rather than choosing one source over the other. This protects Horizon from a situation where the assistant inadvertently normalises informal overrides of official policy.

**Why:** Consistency and audit-readiness require a single authoritative source. Slack messages are informal, may be outdated, and do not carry the same authority as a reviewed policy document.

### Decision 2 — The assistant must never imply an approval decision

A risk specific to policy assistants is that employees may interpret a policy-accurate answer as confirmation that their request will be approved. For example, "Yes, you are eligible to request an internet stipend" does not mean the request has been approved — it means the employee meets the criteria to submit one.

Every response includes an explicit next step that reminds the employee of the human approval step required. The system prompt that governs the assistant prohibits language that implies a decision has been made.

**Why:** If employees act on a policy answer as though it were an approval, Horizon could face situations where employees have made financial or operational decisions based on a misunderstanding — creating disputes and compliance exposure.

### Decision 3 — Low confidence answers escalate rather than guess

The assistant uses a three-tier confidence scale (High, Medium, Low). Answers rated Low confidence are automatically escalated to a human contact rather than delivered as a direct answer. This threshold was set conservatively for Version 1, with the expectation that it will be tuned based on actual usage data after launch.

**Why:** In a policy context, a confidently wrong answer is more damaging than a cautious escalation. Employees who receive a wrong answer may act on it; employees who receive an escalation will seek human guidance. The first version should prioritise accuracy over coverage.

---

## 5. Risks and Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Assistant produces a confident but incorrect answer** | Employee acts on wrong policy guidance; financial or HR dispute follows | LLM is constrained to only use retrieved policy content. Claims not supported by the retrieved text trigger a confidence downgrade before delivery. |
| **Policy documents become outdated** | Assistant continues to give correct-sounding but stale answers after a policy change | Each policy chunk is tagged with a version date and source owner. A monthly review process confirms currency before each review cycle. |
| **Policy conflicts produce misleading guidance** | Employee receives one-sided answer on a genuinely disputed point | Conflict detection runs before answer generation. Conflicted responses are always flagged as Low confidence and escalated. |
| **Employee submits confidential client data in a query** | Client data reaches an external AI system, violating data agreements | Input pre-processing blocks queries containing patterns associated with confidential data before the query reaches the LLM. |
| **High escalation rate overwhelms HR and managers** | The assistant fails to reduce the repetitive question burden it was built to solve | Escalation rate is monitored weekly. If it exceeds 40%, retrieval quality and knowledge base coverage are reviewed and improved before the next sprint. |

---

## 6. Next Steps

### Immediate next steps (Weeks 1–2)

1. **Client sign-off on policy documents** — HR and Finance confirm that all five policy notes are current and authoritative. Any outdated content is updated before ingestion.
2. **Resolve known policy conflicts** — Horizon's leadership or legal team clarifies the authoritative position on the four identified conflicts (food expenses, AI tool use, remote equipment threshold, blocker escalation). The assistant cannot handle these correctly until the official position is documented.
3. **Confirm Slack bot integration permissions** — IT confirms that a Slack bot can be deployed to the company workspace and that all employees have active accounts.

### What the client needs to provide

- Written confirmation that the five policy notes are approved and current
- Official resolution of the four policy conflicts, in writing
- IT access to configure the Slack bot in the company workspace
- A designated policy owner for each domain (HR for leave/equipment, Finance for reimbursements, IT for AI tool usage) who will be the escalation contact

### What the engineering team builds first (Weeks 2–6)

| Week | Milestone |
|------|-----------|
| 2 | Knowledge base setup — ingest, chunk, and index all five policy notes |
| 3 | Retrieval pipeline — embed queries, test retrieval quality against the 12 sample questions |
| 4 | Answer generation — integrate LLM with prompts, test structured output format |
| 5 | Guardrails and escalation — conflict detection, confidence scoring, escalation routing |
| 6 | Slack integration, logging, feedback prompt — deploy to a small pilot group (10–15 employees) |

Following the pilot, a two-week review period will assess accuracy, escalation rate, and employee satisfaction before broader rollout.

---

*This document is a proposal for discussion. Final scope, technology choices, and timelines are subject to alignment with Horizon Services Group stakeholders and the engineering team.*
