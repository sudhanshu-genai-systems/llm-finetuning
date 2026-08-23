# 1. Guardrails — Bedrock vs. Custom, and Whether Both Are Needed

## What Bedrock Guardrails Covers Natively

| Capability | Coverage |
|---|---|
| Content filters | Hate, insults, violence, sexual content, misconduct — configurable severity thresholds |
| Denied topics | Custom topics defined via natural-language description (e.g., "medication dosage prescriptions," "specific investment/stock recommendations") |
| Sensitive information filters | PII detection and redaction/blocking (names, account numbers, SSN-equivalents, etc.) |
| Word filters | Custom word/phrase blocklists |
| Contextual grounding checks | Flags responses not grounded in the provided source content — useful for reducing hallucinated medical/financial claims |
| Profanity filter | Built-in |

This covers the majority of general-purpose safety requirements for both projects without additional development effort, and is configurable per project (a separate guardrail configuration for healthcare vs. BFSI, each with domain-specific denied topics).

## Where Custom Guardrails Add Value

| Gap | Why Bedrock Alone Is Insufficient | Custom Handling |
|---|---|---|
| Mandatory disclaimers (e.g., "not a substitute for professional medical/financial advice") | Bedrock Guardrails blocks or redacts; it does not append standard disclaimer text | Lightweight Lambda post-processing rule — cheaper than an added Bedrock call for this narrow task |
| Numeric/format validation (e.g., preventing specific dosage figures or account-number-shaped output) | Bedrock's denied-topics/content filters are topic- and content-category based, not fine-grained pattern/regex checks | Simple regex-based Lambda check as a supplementary layer |
| Domain-specific regulatory phrasing (RBI/SEBI/IRDAI or HIPAA-adjacent terminology specific to the fine-tuned model's scope) | Achievable via Bedrock's custom denied-topics, but very narrow/dataset-specific rules are cheaper and faster to iterate on as code than as guardrail configuration | Custom word-list/Lambda check, updated alongside the dataset iteration cycle |
| Cost at scale | Every Bedrock Guardrails call is billed per text unit processed | For high-frequency, simple checks (disclaimer injection, basic regex), a Lambda-only path avoids the added per-call cost |

## Recommendation

A **hybrid layered approach** is preferable to relying on either exclusively:

1. **Bedrock Guardrails** — primary layer for input/output screening (harmful content, PII, denied topics, grounding). This is the layer providing the most safety coverage per unit of implementation effort.
2. **Custom Lambda layer** — thin, fast, low-cost supplementary checks: disclaimer injection, regex-based numeric/format validation, and any dataset-specific rules that would be inefficient to encode purely as Bedrock guardrail configuration.

The orchestrator pattern from the earlier plan remains valid: **Lambda → Bedrock Guardrails (input) → SageMaker endpoint → Custom regex/disclaimer check → Bedrock Guardrails (output) → response**. Bedrock Guardrails alone is sufficient for general safety compliance; the custom layer is a low-cost addition specifically for domain-formatting requirements Bedrock does not natively address.

---

# 2. Drift Analysis — What Applies to This Use Case

An important distinction: both projects involve **generative text output from a fine-tuned LLM**, not a structured/tabular prediction model. Traditional SageMaker Model Monitor Data Quality baselines (numeric feature distributions) do not map directly onto free-text input/output. The relevant drift categories:

| Drift Type | Relevance | Implementation Approach |
|---|---|---|
| **Input distribution drift** | High — detects whether incoming questions are shifting away from the domain the model was fine-tuned on (e.g., BFSI model receiving increasingly off-topic or unrelated queries) | Embedding-based comparison: generate embeddings for live query batches vs. training data distribution, compute distributional distance (e.g., cosine similarity drop, population stability index on embedding clusters) via a scheduled Processing Job |
| **Output/response drift** | Medium — tracks whether generated response characteristics (length, structure, vocabulary) are drifting from the post-fine-tuning baseline | Compare rolling statistics on captured responses (via Data Capture) against the baseline captured during Section 4/5 evaluation |
| **Model quality drift** | Conditional — requires ground-truth or feedback labels | Only actionable if the Q&A bot captures user feedback (thumbs up/down); without a feedback loop, this category is not measurable and should be deferred |
| **Concept drift (domain knowledge aging)** | Lower priority at project scope | Relevant longer-term — e.g., financial regulations or medical guidelines updating over time — not a near-term concern for a bounded project timeline |
| **Adversarial/prompt-injection pattern drift** | Medium — security-oriented, distinct from statistical drift | Monitored via the guardrail layer's block/flag logs rather than Model Monitor directly |

## Recommended Scope for This Project

Given the project is bounded in scale and traffic (not a continuously scaled production system), the practical drift monitoring setup:

1. **Data Capture enabled on the endpoint** (input questions + output responses logged to S3) — foundational, low cost.
2. **Scheduled Processing Job (weekly or per-demo-session, not continuous)** computing embedding-based input drift against the training data distribution — this is the most relevant and actionable drift signal for a fine-tuned domain Q&A model.
3. **Response-length/structure drift** as a secondary, low-effort statistical check on captured outputs.
4. **Model quality drift** deferred unless a feedback mechanism (thumbs up/down) is added to the Q&A bot interface — without it, this category cannot be measured meaningfully.

This scoped approach avoids provisioning continuous/expensive monitoring infrastructure while still covering the drift signal most relevant to a domain-specific fine-tuned LLM: **whether the questions being asked are still within the model's trained domain**, which is the input-distribution-drift category.