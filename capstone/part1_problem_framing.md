# Part 1 — Problem Framing & Clarification Brief
**Client:** Horizon Services Group
**Prepared by:** Eyobed Feleke
**Date:** May 8, 2026

---

## 1. Problem Interpretation

### What problem is the client trying to solve?

Horizon Services Group has grown rapidly over the past 18 months, but its internal knowledge infrastructure has not kept pace with that growth. Policy information is scattered across formal policy documents, Slack messages, informal manager decisions, and team-specific notes. Employees have no single reliable place to look for answers, so they resort to asking managers and HR directly — repeatedly, for the same questions.

The client wants to reduce this burden by deploying an internal AI assistant that can answer employee questions about leave, reimbursements, equipment, communication norms, and AI tool usage — drawing from the company's own policy documents.

### Why does this problem matter?

When employees cannot find policy answers quickly and consistently, several things break down:

- **Productivity drops** — employees waste time searching or waiting for a response from a manager.
- **Decisions become inconsistent** — different managers give different answers to the same question, creating fairness and compliance risks.
- **Manager bandwidth is consumed** by repetitive, low-value questions that could be self-served.
- **Trust erodes** — employees who receive conflicting guidance lose confidence in internal processes.

At 350 employees across hybrid and remote setups, the problem will only grow as the company continues to scale.

### Who is affected?

- **All employees** (hybrid, remote, and field staff) who need policy answers quickly
- **Managers and HR** who are flooded with repetitive questions
- **Finance and Operations** who deal with inconsistent reimbursement and approval requests
- **IT** who receives ad hoc equipment requests without clear process

### What could go wrong if this is not solved?

- Policy violations increase as employees make uninformed decisions
- Compliance and audit risk rises if financial or leave policies are applied inconsistently
- Employee frustration and disengagement grow, particularly among remote workers who cannot easily access informal guidance
- Manager time is increasingly diverted from strategic work to answering basic procedural questions

---

## 2. Clarification Questions

The following questions would be asked before finalizing the solution design. They are grouped by theme.

### Users and Use Cases

1. Which employee groups should have access to the assistant in Version 1 — all 350 employees, or a specific team (e.g., HR, Finance, Operations first)?
2. Are there employees whose roles or clearance levels should restrict what information the assistant can show them? For example, should a junior operations staff member see the same policy details as an HR lead?
3. Will the assistant be accessed via a web browser, a Slack bot, or another interface? Does the company already have a preferred internal tool where this should live?

### Document Sources

4. Beyond the five policy notes provided, what other documents exist that should be included — for example, onboarding guides, department-specific handbooks, or approval templates?
5. Who owns and maintains each policy document? Is there a designated person or team responsible for keeping them up to date?
6. How often are policies updated? Is there a review cycle, and how would the assistant be updated when policies change?

### Accuracy and Uncertainty

7. When a policy question has no clear documented answer — for example, emergency leave decisions made case by case — how should the assistant respond? Should it say "I don't know" and direct the employee to HR, or should it attempt a best-effort answer with a caveat?
8. How much tolerance does the client have for incorrect answers? Is a factually wrong response worse than a response that says "I'm not sure, please check with HR"?

### Confidentiality and Security

9. Are any of the policy documents classified as confidential or restricted to specific roles? For example, are salary banding or performance management policies included in scope?
10. What are the data residency or security requirements for the system? Does employee data need to stay on-premise, or is a cloud-hosted solution acceptable?

### Escalation Rules

11. When the assistant identifies a conflict between a Slack message and a formal policy (as seen in the materials), which source should it treat as authoritative? Should it flag the conflict to the employee, or always defer to the formal policy note?
12. If an employee asks a question that requires manager approval to answer (e.g., "Can I get an exception on the reimbursement deadline?"), what should the assistant do — refuse to answer, provide policy context only, or route the request to the appropriate person?

### Success Metrics

13. How will success be measured for Version 1? For example: reduction in HR query volume, employee satisfaction scores, percentage of questions answered without escalation, or response accuracy rates?
14. Is there a specific target for the proportion of questions the assistant should be able to answer without escalating to a human?

### Timeline and Priorities

15. Are there specific policy areas the client considers highest priority for the first version — for example, leave and reimbursement questions are most frequent, while AI tool usage questions are less urgent?

---

## 3. Key Assumptions

If the client cannot answer all clarification questions before work begins, the following assumptions will be used as the starting basis for design.

---

**Assumption 1: Formal policy notes take precedence over Slack messages**

- **Why reasonable:** Formal policy documents are reviewed and approved by the relevant department or leadership. Slack messages represent informal or individual guidance that may be outdated or role-specific.
- **Risk it creates:** The assistant may contradict guidance employees have already received informally, causing confusion or pushback.
- **How to validate:** Confirm with HR and Finance whether formal policy notes have been officially approved and are considered binding.

---

**Assumption 2: The assistant will be deployed as a Slack bot for MVP**

- **Why reasonable:** Slack is already the primary internal communication tool at Horizon. Employees are already in Slack, reducing adoption friction. A Slack bot is faster to deploy than a custom web application.
- **Risk it creates:** Slack bots have limited UI flexibility; structured responses (tables, citations) may render awkwardly. Some field staff may have limited Slack access.
- **How to validate:** Confirm with IT whether a Slack bot integration is technically and contractually permissible, and whether all employees have active Slack accounts.

---

**Assumption 3: All five policy notes are current and authoritative**

- **Why reasonable:** The client provided these as the source materials, implying they reflect current policy.
- **Risk it creates:** If any policy note is outdated, the assistant may provide incorrect guidance. This is particularly risky for financial policies (reimbursement deadlines, amounts) and leave entitlements.
- **How to validate:** Request that HR and Finance sign off on each policy note as current before ingesting them into the knowledge base.

---

**Assumption 4: The MVP will cover HR, Finance, Operations, and AI Tool Usage only**

- **Why reasonable:** The client materials explicitly scope these four areas for the first version. Delivery and IT-specific questions are out of scope for now.
- **Risk it creates:** Employees may ask questions that fall outside these four areas and receive no useful answer, which could frustrate early adopters.
- **How to validate:** Track out-of-scope question volume in the first month of deployment to decide whether to expand scope in Version 2.

---

**Assumption 5: The assistant will not make approval decisions**

- **Why reasonable:** Any decision involving manager approval, exception handling, or individual circumstances requires human judgment and accountability. An AI assistant making approvals creates legal and compliance risk.
- **Risk it creates:** Employees may misinterpret a policy-accurate answer as an approval. For example, "Policy allows you to carry forward 5 days" is not the same as "Your request is approved."
- **How to validate:** Add explicit disclaimers to all responses that require formal action, and test with a sample of employees to confirm they understand the distinction.

---

**Assumption 6: A low-confidence or conflicting answer should escalate to a human, not guess**

- **Why reasonable:** The client explicitly stated that overconfident answers are a risk. In a policy context, a wrong answer can lead to real financial or compliance harm for the employee.
- **Risk it creates:** Too many escalations will defeat the purpose of the assistant and continue to overload HR and managers.
- **How to validate:** Monitor escalation rate during the first 4 weeks and tune confidence thresholds based on actual question distribution.

---

**Assumption 7: The assistant will support English only in Version 1**

- **Why reasonable:** All provided policy documents are in English. Building multi-language support in the first version significantly increases complexity.
- **Risk it creates:** Field operations staff who are not fluent in English may be underserved by the assistant.
- **How to validate:** Survey the operations and delivery teams to understand the language needs of field staff before finalising MVP scope.

---

## 4. Initial Success Criteria

The following criteria define what a successful first version of the assistant looks like. These are measurable and realistic within a 6-week build window.

| Criterion | Target | Measurement Method |
|-----------|--------|--------------------|
| Question coverage | The assistant answers ≥70% of submitted questions without escalation | Log of queries vs. escalations over first 30 days |
| Policy accuracy | ≥90% of answered questions are factually consistent with the source policy note | Spot-check by HR and Finance reviewer on a sample of 50 answers |
| Response time | Answers delivered within 5 seconds of question submission | System performance logs |
| Escalation quality | When escalated, the assistant correctly identifies the right team (HR, Finance, IT, manager) ≥80% of the time | Manual review of escalated cases |
| Employee satisfaction | ≥60% of employees who use the assistant in the first month rate their experience as "useful" or "very useful" | Short in-assistant feedback prompt (thumbs up/down) |
| Conflict detection | All four known policy conflicts (food expenses, AI tool use, remote equipment, blocker escalation) are flagged correctly when triggered by a relevant question | Manual test with the 12 sample employee questions |

Version 1 is considered successful if it demonstrably reduces repetitive HR and Finance queries, provides consistent policy-backed answers, and does not produce harmful or confidently incorrect responses.
