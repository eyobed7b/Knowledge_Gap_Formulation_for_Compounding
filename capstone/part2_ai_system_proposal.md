# Part 2 — AI System Proposal
**Client:** Horizon Services Group
**Prepared by:** Eyobed Feleke
**Date:** May 8, 2026

---

## 1. System Overview

### What the assistant does

The Horizon Internal Knowledge Assistant is a conversational AI system that answers employee questions about internal company policies. It retrieves relevant information from a curated knowledge base of approved policy documents, generates a structured answer grounded in that content, and clearly communicates its confidence level. When a question is ambiguous, involves conflicting sources, or requires human judgment, the assistant flags this and routes the employee to the appropriate person or team.

### Who it serves

All Horizon Services Group employees — including office-based, hybrid, remote, and field operations staff — who need quick, reliable answers to internal policy questions without waiting for a manager or HR response.

### What types of questions it should answer

- Leave and time-off entitlements (annual leave balance, carry-forward rules, sick leave documentation)
- Equipment and remote work support (what is available, who approves it, what to do if equipment is lost)
- Expense reimbursement (eligible categories, submission deadlines, required documentation)
- Internal communication and escalation norms (when to use Slack vs. email, blocker escalation rules)
- AI tool usage policies (what is permitted, what is prohibited, when to seek manager guidance)

### What types of questions it should NOT answer

- Questions that require a final approval decision (e.g., "Can you approve my leave request?")
- Questions involving confidential employee data (e.g., salaries, performance ratings, disciplinary records)
- Legal or compliance advice requiring professional judgment
- Questions outside the four scoped domains (HR, Finance, Operations, AI Tool Usage)
- Questions that contain confidential client information pasted directly into the query

---

## 2. High-Level Architecture

| Component | Description | Technology Choice (MVP) |
|-----------|-------------|------------------------|
| **User Interface** | Where employees submit questions and receive answers. A Slack bot is the MVP choice since all employees already use Slack, minimising adoption friction. | Slack Bot (Bolt SDK) |
| **Document / Knowledge Base** | Stores the approved policy documents as chunked, indexed text. Only formally approved policy notes are ingested — Slack messages and informal guidance are excluded to maintain accuracy. | Vector database (Pinecone or Chroma) |
| **Retrieval Layer** | Takes the employee's question, converts it into a search query, and retrieves the most relevant policy chunks from the knowledge base using semantic similarity. | Embedding model (OpenAI `text-embedding-3-small` or equivalent) + cosine similarity search |
| **LLM Layer** | Receives the retrieved policy chunks and the original question, then generates a structured, policy-grounded answer in the required response format. | Claude claude-sonnet-4-6 (via Anthropic API) |
| **Guardrails** | Checks the input for confidential data, checks the output for overconfident or unsupported claims, and enforces the structured response format. Also detects when the retrieved context is insufficient to answer confidently. | Pre- and post-processing rules + confidence scoring in the prompt |
| **Escalation Path** | When the guardrails flag a question as ambiguous, conflicting, or out of scope, the assistant tells the employee clearly what it cannot answer and directs them to the correct human contact (HR, Finance, IT, or their manager). | Rule-based routing logic based on topic classification |
| **Logging / Feedback** | Every query, retrieved chunk, generated answer, and employee feedback signal (thumbs up/down) is logged. This data is used to identify gaps, retune retrieval, and flag outdated policies. | Structured log store (e.g., a simple database table or cloud logging service) |

### Simple Architecture Diagram

```
Employee (Slack)
      |
      v
[User Interface — Slack Bot]
      |
      v
[Query Pre-processor]
  - Sanitise input
  - Check for confidential data
  - Classify topic domain
      |
      v
[Retrieval Layer]
  - Embed query
  - Search knowledge base
  - Return top-k policy chunks
      |
      v
[LLM Layer — Claude]
  - System prompt + retrieved context + user question
  - Generate structured answer
      |
      v
[Guardrails / Confidence Check]
  - Is context sufficient?
  - Is answer grounded in retrieved text?
  - Does the response stay within scope?
      |
      +--- [High confidence] --> Structured answer to employee
      |
      +--- [Low confidence / conflict / out of scope] --> Escalation message to employee
      |
      v
[Logger]
  - Log query, context, answer, confidence, feedback
```

---

## 3. Workflow

The following steps describe what happens from the moment an employee submits a question to the moment they receive a response.

**Step 1 — Employee submits question**
The employee types a question into the Horizon Knowledge Bot in Slack (e.g., "I worked remotely 4 days this week. Can I request an internet stipend?").

**Step 2 — Input pre-processing**
The system checks the query for:
- Obvious confidential content (e.g., client names, employee IDs, salary figures). If found, the system refuses to process the query and asks the employee to rephrase without sensitive data.
- Topic domain classification — is this a leave, equipment, reimbursement, communication, or AI tool usage question? If it is clearly outside all five domains, the system responds immediately with an out-of-scope message and directs the employee to the relevant team.

**Step 3 — Query reformulation**
The original question is rewritten into a clean, keyword-rich search query optimised for semantic retrieval (e.g., "remote work equipment support internet stipend eligibility criteria"). This improves retrieval accuracy when employee phrasing is informal or incomplete.

**Step 4 — Knowledge base retrieval**
The reformulated query is embedded into a vector representation and compared against all indexed policy chunks in the knowledge base. The top 3–5 most semantically similar chunks are retrieved along with their source labels (e.g., "Policy Note 2 — Equipment & Remote Work").

**Step 5 — Conflict detection**
Before passing the retrieved chunks to the LLM, the system checks whether multiple chunks from different sources provide conflicting guidance on the same topic. If a conflict is detected (e.g., a Slack excerpt contradicts a formal policy note), the conflict flag is set to True and will affect the confidence level and escalation decision in the output.

**Step 6 — Answer generation**
The LLM receives a structured prompt containing:
- The system instructions and response format
- The retrieved policy chunks with their source labels
- The original employee question
- The conflict flag (if set)

The LLM generates a structured answer following the required format: Direct Answer, Explanation, Source Reference, Confidence Level, Escalation Needed, and Suggested Next Step.

**Step 7 — Output guardrail check**
Before the answer is sent to the employee, the system checks:
- Is the answer grounded in the retrieved chunks? (If the LLM made a claim not supported by the context, confidence is downgraded.)
- Does the answer avoid making approval decisions or giving legal advice?
- Is the response format correctly structured?

**Step 8 — Delivery to employee**
The structured answer is returned to the employee via Slack. If escalation is needed, the message includes a clear directive (e.g., "Please contact HR at hr@horizonservices.com for further guidance on this.").

**Step 9 — Feedback collection**
After receiving the answer, the employee is shown a simple thumbs up / thumbs down prompt. Their feedback, along with the full query-context-answer record, is written to the log store.

**Step 10 — Periodic review**
On a weekly basis, a designated reviewer (HR or an AI engineer) reviews logged escalations, low-confidence answers, and negative feedback signals to identify knowledge gaps and update the knowledge base as needed.

---

## 4. Prompt Strategy

### Prompt 1 — Query Reformulation Prompt

**Purpose:** Convert informal employee questions into clean search queries that improve retrieval accuracy from the knowledge base.

**Prompt text:**
```
You are a query reformulation assistant for an internal HR and policy knowledge base.

Your task is to rewrite the employee's question as a short, keyword-rich search query.
The search query will be used to find relevant policy documents.

Rules:
- Remove personal details, names, and specific dates
- Focus on the policy topic and key terms
- Output only the reformulated search query — no explanation

Employee question: {employee_question}

Reformulated search query:
```

**Expected output format:** A single plain-text search query of 5–15 words.

**Why it is designed this way:** Employee questions are often conversational and contain personal context that does not help retrieval (e.g., "I was working late last Tuesday…"). Stripping these and focusing on the policy topic significantly improves the quality of chunks returned by the vector search. This step also normalises phrasing so that semantically identical questions from different employees retrieve the same relevant content.

---

### Prompt 2 — Answer Generation Prompt

**Purpose:** Generate a structured, policy-grounded answer to the employee's question based on the retrieved policy chunks.

**Prompt text:**
```
You are the Horizon Services Group internal policy assistant.
Your role is to help employees find accurate answers to questions about internal company policies.

Rules:
- Answer only using the policy context provided below. Do not use outside knowledge.
- If the context does not contain enough information to answer, say so clearly.
- Never make final approval decisions on behalf of a manager.
- Never include or infer confidential employee data.
- If the context contains conflicting information, flag it clearly rather than choosing one side.
- Always cite the source policy note in your answer.

Respond using exactly this format:

**Direct Answer:** [One or two sentences directly answering the question]

**Explanation:** [Policy-based explanation in 2–4 sentences]

**Source Reference:** [Name of the policy note(s) used]

**Confidence Level:** [High / Medium / Low]
- High: The context directly and clearly answers the question.
- Medium: The context is relevant but incomplete or requires interpretation.
- Low: The context is insufficient, ambiguous, or contains a conflict.

**Escalation Needed?** [Yes / No]

**Suggested Next Step:** [What the employee should do next]

---

Policy context:
{retrieved_policy_chunks}

Conflict detected: {conflict_flag}

Employee question: {employee_question}
```

**Expected output format:** Structured response following the six-field format above.

**Why it is designed this way:** The strict format ensures every response is consistent and auditable. Anchoring the model to only the retrieved context prevents hallucination. The explicit confidence scale and conflict flag ensure the model communicates uncertainty rather than guessing. Prohibiting approval decisions keeps the assistant within its intended scope.

---

### Prompt 3 — Uncertainty / Escalation Prompt

**Purpose:** Generate a clear, helpful escalation message when the assistant cannot confidently answer the question — either because the knowledge base has insufficient content, a conflict was detected, or the question is out of scope.

**Prompt text:**
```
You are the Horizon Services Group internal policy assistant.
You have determined that you cannot provide a confident, policy-backed answer to the employee's question.

Reason for escalation: {escalation_reason}
Possible values:
  - "insufficient_context": The knowledge base does not contain relevant information.
  - "policy_conflict": Two or more sources provide conflicting guidance on this topic.
  - "approval_required": The question requires a manager or HR decision, not a policy lookup.
  - "out_of_scope": The question falls outside the assistant's current coverage area.
  - "confidential_data": The question appears to involve confidential information.

Write a short, professional, and helpful message to the employee that:
1. Acknowledges their question without repeating it back verbatim.
2. Explains briefly why the assistant cannot answer it directly.
3. Tells them exactly who to contact or what to do next.
4. Keeps a friendly, professional tone — do not make the employee feel penalised for asking.

Use plain language. Maximum 4 sentences.

Escalation reason: {escalation_reason}
Relevant team or contact (if known): {escalation_target}
```

**Expected output format:** A short paragraph of 2–4 sentences addressed directly to the employee.

**Why it is designed this way:** Escalation messages that simply say "I don't know" frustrate employees and waste the interaction. This prompt ensures the assistant always gives the employee a clear next step, which preserves trust in the tool even when it cannot answer. Specifying the escalation reason as a typed parameter makes the output predictable and testable.

---

## 5. Risk & Guardrail Plan

| Risk | Why It Matters | Mitigation |
|------|---------------|------------|
| **Hallucination — the assistant generates a confident answer not supported by the policy documents** | Employees may act on incorrect guidance, leading to denied reimbursements, leave disputes, or policy violations. | Constrain the LLM to only use retrieved context (grounding instruction in the system prompt). Post-generation check: if the answer contains a specific number or rule not present in the retrieved chunks, confidence is automatically downgraded to Low. |
| **Policy conflict produces misleading answer** | Four known conflicts exist in the current materials (food expenses, AI tool use, remote equipment, blocker escalation). The assistant choosing one side without flagging the conflict could entrench the wrong behaviour. | Conflict detection layer runs before answer generation. If conflicting chunks are retrieved, the conflict flag is set to True, the confidence level is forced to Low, and the response explicitly states the conflict and escalates. |
| **Employee pastes confidential client data into query** | Public or cloud-hosted LLMs must not receive client-identifiable information. Violation could breach client contracts or data protection obligations. | Input pre-processing scans for patterns associated with confidential data (proper names in context, email addresses, project codes). If detected, the query is blocked before reaching the LLM and the employee is prompted to rephrase. |
| **Assistant provides guidance that implies approval** | Employees may interpret a policy-accurate answer (e.g., "you are eligible to request this") as a formal approval, then act on it without going through the proper channel. | All responses include a disclaimer in the Suggested Next Step field clarifying that the assistant provides information only and does not replace manager or HR approval. The system prompt explicitly prohibits the model from using language that implies a decision has been made. |
| **Knowledge base becomes outdated after policy changes** | If a policy is updated but the knowledge base is not, the assistant will continue to give answers based on the old policy. This is particularly risky for financial policies (reimbursement amounts, deadlines). | Every policy chunk is stored with a version date and a source owner. A monthly review process is established where HR and Finance confirm whether each policy note is still current. Outdated chunks are flagged in the admin dashboard and re-ingested after approval. |
| **High escalation rate defeats the purpose of the assistant** | If too many questions are escalated, managers and HR are not relieved of the repetitive question burden. | Monitor the escalation rate weekly for the first month. If escalation exceeds 40% of queries, investigate whether retrieval quality or knowledge base coverage is the cause. Tune the confidence thresholds and expand the knowledge base accordingly before declaring the MVP a success. |
| **Field operations staff are underserved** | Some field staff may have limited Slack access or low digital literacy, meaning the assistant reaches only office and hybrid employees. | Track which employee groups are using the assistant in the first month. If field staff adoption is low, consider a secondary access method (e.g., SMS or a simple web form) in Version 2. |

---

## 6. MVP Scope

### What to include in Version 1

- Knowledge base ingestion of all five current policy notes (Leave, Equipment, Reimbursement, Communication, AI Tool Usage)
- Slack bot user interface accessible to all employees
- Semantic retrieval over the five policy notes
- Structured answer generation using the three-prompt pipeline
- Conflict detection for the four known policy conflicts
- Escalation routing with clear contact directions (HR, Finance, IT, manager)
- Basic logging of queries, answers, confidence levels, and escalations
- Simple thumbs up/down feedback prompt after each answer
- Input sanitisation to block queries containing obvious confidential data

### What to exclude from Version 1

- Integration with live HR systems (leave balance lookup, approval workflows) — requires significant IT involvement and security review
- Multi-language support — all policy documents are currently in English
- Personalisation based on employee role or department — adds complexity and requires role data integration
- Voice interface or mobile app — Slack covers the majority of use cases first
- Automated policy update pipeline — manual review process is sufficient at this scale
- Analytics dashboard — raw logs are sufficient for the first review cycle; a dashboard can be built in Version 2
- Coverage of Delivery team or IT-specific policies — out of scope per client constraint

### Why this scope is reasonable

The five policy notes cover the questions employees ask most frequently, based on the 12 sample questions provided by the client. All four known policy conflicts are handled. The Slack interface minimises deployment complexity since no new tool needs to be introduced. The logging and feedback mechanisms provide enough signal to drive meaningful improvement after launch.

A small AI engineering team can build, test, and deploy this MVP within the 6-week timeline. Keeping integrations minimal and the knowledge base static for Version 1 means the team can focus on getting the retrieval quality and prompt behaviour right before adding complexity.

Version 2 priorities should include: live HR system integration, expanded policy coverage (Delivery, IT), analytics dashboard, and personalised responses based on employee role.
