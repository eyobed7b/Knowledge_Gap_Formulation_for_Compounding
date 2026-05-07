# Sources — Day 3

---

## Canonical Papers

**1. Schick et al. (2023) — "Toolformer: Language Models Can Teach Themselves to Use Tools"**  
Meta AI Research. Published at NeurIPS 2023.  
[https://arxiv.org/abs/2302.04761](https://arxiv.org/abs/2302.04761)  

Establishes the foundational mechanism by which language models learn to generate special
API-call tokens during text generation, which is the basis of modern tool-calling. The paper
shows that tool invocation can be represented as tokens within the model's normal generation
stream — the mechanism the explainer references when explaining how tool-name tokens differ
from JSON body tokens.

---

**2. Anthropic Tool Use Documentation — "Tool Use (Function Calling)"**  
Anthropic Developer Documentation (authoritative, primary source).  
[https://docs.anthropic.com/en/docs/tool-use](https://docs.anthropic.com/en/docs/tool-use)  

Authoritative specification of how the `tools` parameter is formatted, how tool definitions
are injected into the model's context, what a `tool_use` content block looks like in the
model's raw response, and how the API routes the tool call back to the caller. Used as the
primary implementation reference for the API call comparison in the explainer.

---

## Tool or Pattern Used

**Anthropic Python SDK — direct `tools` parameter comparison**  

Ran both call types (JSON mode and tool-calling with two tools: `compose_email` and
`suppress_prospect`) against `claude-haiku-4-5` using a sample prospect brief from the
Week 10 trace log. Inspected the raw response structure in both cases. JSON mode returned
a `choices[0].message.content` string of 312 tokens (full email JSON body). Tool-calling
returned a `content[0]` block of type `tool_use` with `name: "suppress_prospect"` and an
`input` dict — 287 tokens total, with the routing decision encoded in the tool name rather
than the response body. This concrete output comparison is what grounded the explainer's
claim about token-level differences.
