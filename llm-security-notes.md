# LLM Security Red Team Notes

> Defensive awareness notes on common LLM failure modes, attack techniques, and practical mitigations.

## Contents

- [Hallucinations](#hallucinations)
- [Direct Prompt Injection](#direct-prompt-injection)
- [Indirect Prompt Injection](#indirect-prompt-injection)
- [Data Exfiltration in Prompt Injection Scenarios](#data-exfiltration-in-prompt-injection-scenarios)
- [Jailbreaking Techniques](#jailbreaking-techniques)
- [Model Bias](#model-bias)
- [Guardrails Builder](#guardrails-builder)
- [Output Validation](#output-validation)
- [Content Classification](#content-classification)
- [Governance, Compliance & Ethics](#governance-compliance--ethics)

---

## Hallucinations

Occurs when the model confidently fabricates information in a response.

Architecturally:

- Next-Token Prediction Mechanism: LLMs predict words sequentially via probabilistic matching. This means the model might output false statements with word patterns that seem plausible.
- Lossy Compression: Massive data sets get compressed during training. During this process, specific facts could get smoothed over, leading the model to "fill in the gaps" with plausible-sounding statements that might not be factual.
- Lack of Internal Fact-Checker: Standard transformer architectures generate responses directly from probability distributions. Usually not hooked up to an external tool/web search, so it can't cross-reference.

**Examples of risks to look out for:**

- Environments where information accuracy is critical, example: legal citations
- Recommended code packages that don't exist. Attackers can take advantage of these hallucinated recommendations by creating those packages, which can lead to exploitation (known as "slopsquatting").

**Mitigations:**

- Output verification against trusted source & human review
- Ask model for its confidence level

**Ways to exploit:**

- Long context + request to "continue": Feeding a lot of information and then asking the model to fill in a gap or extend the pattern.
- Leading questions: "What did the court say in footnote 12 of this case?"
- Asking for specificity beyond what's knowable
- Authority framing: Telling the model it's an expert
- Mixing true claims with a false one: the true ones lend credibility to the false one

---

## Direct Prompt Injection

#1 on OWASP Top 10 for LLM Applications.

Attacker is the user, typing input into chat to try to override system prompt/safety instructions.

Takes advantage of a limitation of the transformer architecture — the model cannot differentiate where input originates.

**Examples of risks to look out for:**

- Chatbots tricked into discounting products
- Browsing agents that clicked through malicious pages and executed embedded instructions

**Ways to exploit:**

- "Ignore previous instructions"
- Role/persona hijacking: take on a new persona that ignores prior instructions/has escalated privileges
- Gradually shift context across several turns instead of just one obvious turn
- Format spoofing: fake tags meant to impersonate a trusted instruction block
- Translation request: output in another language

**Mitigations:**

- Reinforce system prompts (re-assert instructions after user input)
- Instruction-hierarchy training (system prompt holds more weight than user)
- Flag outputs for policy violations

---

## Indirect Prompt Injection

Not directly inputted by the user. Hidden in 3rd-party content the model will later ingest, i.e. webpage, PDF, email body, search result...

Takes advantage of a limitation of the transformer architecture — the model cannot differentiate between "data to summarize" and "instructions to follow," so hidden text could be executed as a command.

**Ways to exploit:**

- Instructions embedded in a webpage that an agent visits and treats as authoritative
- Hidden text in a document meant to be invisible to a human reviewer but readable by the model
- Response payloads that get fed back into the model's context

**Mitigations:**

- Strict privilege separation so retrieved content is always treated as data (never instructions)
- Sanitize content to strip instruction-like patterns
- Sandboxing agent actions

---

## Data Exfiltration in Prompt Injection Scenarios

Leaking sensitive data to an external location that the attacker then accesses.

**Ways to exploit:**

- "Summarize conversation so far"
- Encode exfiltrated data (like base64) to bypass output filters
- Injected instructions asking the model to append hidden context into a URL or markdown link

**Mitigations:**

- Egress filtering and output scanning
- Never store secrets in the model
- Least privilege tool access

---

## Jailbreaking Techniques

Targets the model's safety training instead of its instruction hierarchy. While prompt injection tries to smuggle in new instructions, jailbreaking tries to get the model to apply its existing capabilities to a request it would normally refuse.

**Ways to exploit:**

- Role play/different persona
- Hypothetical wrapping — i.e. framing a harmful request as a story
- Gradual escalation
- Authority/urgency framing
- Find an objective that competes with safety: maybe the model would prioritize "helpfulness" or "honesty" above safety
- Obfuscation, i.e. base64, splitting a harmful term across tokens to evade keyword-level filters

**Mitigations:**

- Train the model to refuse based on underlying intent rather than surface-level phrasing
- Evaluate the whole conversation instead of each turn in isolation
- Red-team testing suites for different framing methods with the same underlying harmful request, to check generalization instead of keyword blocking

**Further resources:**

MITRE ATLAS — for a catalog of adversarial ML tactics/techniques, including jailbreaking patterns, mapped in an ATT&CK-style framework
OWASP Top 10 for LLM Applications (LLM01) — for industry-standard classification of prompt-injection/jailbreak risk

---

## Model Bias

Usually comes from 3 sources: skewed training data, labeling bias, architectural/objective bias

**Ways to test:**

- Run the same prompt template across different names/demographics/pronouns and measure the output variance (i.e. sentiment, competence framing)
- Use an established bias benchmark (BBQ, HELM's fairness suite)
- Record aggregate statistics, not anecdotes

**Mitigations:**

- Diverse and audited training data
- Run bias testing suites before deployment and on a recurring basis
- Include fairness as an objective
- Publish evaluation results to be transparent with users

---

## Guardrails Builder

**Architecture:**

- Input layer: sanitize/classify incoming prompts before they reach the model
- System-prompt hardening: reinforced instructions, instruction hierarchy reminders
- Retrieval isolation: treat ingested external content as data, never instructions
- Output layer: review output before it's returned to the user

Reference: NVIDIA NeMo Guardrails, Guardrails AI, OWASP LLM Top 10

Note: Guardrails are a mitigation layer, not a fix for the underlying model behavior. Attackers adapt to bypass them over time, so this is an ongoing process, not a one-time solution.

---

## Output Validation

Catch policy-violating, incorrect, or unsafe model output before it reaches the user. Be aware of and consider false positives.

**Techniques:**

- Schema/format validation: enforce expected structure before accepting output
- Classifier pass: score the output against policy categories
- Fact check: cross-check against trusted sources
- Consistency checks: flag output that contradicts system instructions or prior conversation state

---

## Content Classification

Label incoming/outgoing content by risk category so the system can route/block/flag it.

**Techniques:**

- Category classifiers: score text against defined risk categories (PII, prompt injection, violence)
- Multi-label scoring: content often falls into more than one category
- Confidence thresholds: tune based on category (some need zero-tolerance, others can tolerate a higher threshold)
- Contextual classification: same text could be harmful/harmless depending on context

Note: classifiers have blind spots because they are trained on finite examples (i.e. obfuscation, multilingual gaps). Classification is based on probability and not a guarantee — do not treat as an absolute verdict.

---

## Governance, Compliance & Ethics

Organizational layer that ensures AI systems are deployed responsibly and legally, adhering to laws.

- Policy definition: document acceptable-use policies, risk tiers, and escalation paths for AI-produced content
- Regulatory mapping: align system behavior with applicable frameworks (EU AI Act risk tiers, sector-specific rules like HIPAA)
- Maintain audit trails
- Determine which decisions need human review/sign-off vs. fully automated handling
- Disclose known limitations and bias evaluation

Note: governance defines accountability, response processes, and legal exposure. It makes technical mitigations enforceable and auditable at the organization level.

---

> This writeup is for defensive and awareness purposes only.
