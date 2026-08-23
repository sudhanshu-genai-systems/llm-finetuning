# Unified Q&A Bot — Multi-Domain Website Architecture

## 1. Single Website, Domain-Filtered (React.js + HTML/CSS)

## Architecture Overview

```
┌─────────────────────────────────────────┐
│  React.js Frontend (S3 + CloudFront)     │
│  - Domain selector: Healthcare | BFSI    │
│  - Chat UI, suggested questions panel    │
└───────────────┬───────────────────────────┘
                │ API calls (domain param included)
                ▼
┌─────────────────────────────────────────┐
│  API Gateway (single API, routed by      │
│  domain parameter or separate resource   │
│  paths: /healthcare/chat, /bfsi/chat)    │
└───────────────┬───────────────────────────┘
                ▼
┌─────────────────────────────────────────┐
│  Lambda Orchestrator (domain-aware)      │
│  - Reads domain from request             │
│  - Routes to correct guardrail config    │
│  - Routes to correct SageMaker endpoint  │
└───────────────┬───────────────────────────┘
                ▼
   Healthcare Endpoint  |  BFSI Endpoint
   (separate SageMaker serverless endpoints)
```

## Key Design Decisions

| Component | Approach |
|---|---|
| Single React app | One codebase, one deployment, with a domain toggle/tab (Healthcare / BFSI) at the top level of the UI — state managed via React Context or a simple query param (`?domain=healthcare`) |
| Routing to correct model | Domain selection is passed as a parameter in the API request body/header; the Lambda orchestrator uses this to select the correct SageMaker endpoint name and the correct Bedrock Guardrail configuration (healthcare denied-topics vs. BFSI denied-topics) |
| Component structure | Shared chat UI components (message list, input box, loading state) reused across both domains; domain-specific elements (suggested questions, disclaimer text, color theme) rendered conditionally |
| Hosting | Static build (`npm run build`) deployed to S3, served via CloudFront — same low-cost pattern noted in the original plan |

## Suggested Folder Structure

```
frontend/
├── src/
│   ├── components/
│   │   ├── ChatWindow.jsx        # shared
│   │   ├── DomainSelector.jsx    # tab/toggle: Healthcare | BFSI
│   │   ├── SuggestedQuestions.jsx
│   │   └── MessageBubble.jsx
│   ├── config/
│   │   ├── healthcare.config.js  # suggested questions, disclaimer, theme
│   │   └── bfsi.config.js
│   ├── api/
│   │   └── chatApi.js            # single API client, domain passed as param
│   └── App.jsx
```

---

## 2. Referencing the Same Deployed Endpoints

- Each domain's fine-tuned model remains deployed as its **own separate SageMaker Serverless Inference endpoint** (as established in Section 5) — one endpoint per project, not a shared endpoint, since the two models have different fine-tuned weights.
- The **Lambda orchestrator holds a mapping** (environment variable or a small config table, e.g., DynamoDB or a static JSON in the Lambda package):

```
{
  "healthcare": "sagemaker-endpoint-healthcare-v1",
  "bfsi": "sagemaker-endpoint-bfsi-v1"
}
```

- The frontend never calls SageMaker directly — it only calls API Gateway with a `domain` field; the Lambda resolves which endpoint and which guardrail configuration to apply. This keeps the frontend fully decoupled from backend model infrastructure and allows either endpoint to be updated/redeployed independently without frontend changes.

---

## 3. Authentication, Caching, Suggested Questions

### Authentication

| Option | Fit |
|---|---|
| **Amazon Cognito** (User Pools) | Standard, serverless-native choice — integrates directly with API Gateway (Cognito Authorizer), supports email/password or social login, no server management |
| API Gateway usage plans + API keys | Simpler alternative if only basic access control (not full user identity) is required — suited for an internal demo rather than public-facing use |

Recommended: Cognito if the bot is intended for broader/public access with user-level tracking; API key-based access control is sufficient for an internal/demo-stage deployment given project scope.

### Caching

| Layer | Purpose |
|---|---|
| **CloudFront** (frontend) | Caches static assets (JS/CSS/images) — standard, near-zero additional cost |
| **API Gateway response caching** OR **DynamoDB-based semantic cache** | Caches repeated/common question-answer pairs to avoid redundant SageMaker invocations — directly reduces both latency and per-invocation cost |
| **Semantic cache (recommended)** | Store embeddings of previously asked questions alongside their responses in DynamoDB or a lightweight vector store; incoming questions are checked for high similarity against cached entries before invoking the endpoint — effective for domain Q&A bots where many users ask overlapping questions |

### Suggested Questions

- Curated list per domain (5-8 example questions), stored in the frontend config files (`healthcare.config.js`, `bfsi.config.js`) — displayed as clickable chips above the input box.
- Optionally sourced dynamically from the **most frequently asked cached questions** (via the DynamoDB cache from above), giving a self-updating suggestion list based on real usage patterns rather than a static hardcoded set.

---

## 4. Improving LLM Response Latency

| Technique | Impact | Applicability Here |
|---|---|---|
| **Semantic caching** (above) | Eliminates endpoint invocation entirely for repeated/similar questions | High — direct latency win for common queries |
| **Avoid cold starts on Serverless Inference** | Serverless Inference has cold-start latency (model load on first invocation after idle) | Consider a **provisioned concurrency** setting on the serverless endpoint if demo/traffic patterns are predictable, or accept cold-start trade-off given budget constraints |
| **Smaller base model** | Directly reduces inference time | Already addressed via the 3B base model choice made earlier — smaller models infer faster than 7B+ alternatives |
| **Quantized inference (not just training)** | 4-bit/8-bit inference reduces compute per token | Deploy the merged fine-tuned model in a quantized format (e.g., via `bitsandbytes` or AWQ) rather than full precision at inference time |
| **Reduced max output token length** | Shorter generations complete faster | Cap `max_new_tokens` appropriately for expected answer length (Q&A responses rarely need very long generations) |
| **Streaming responses** | Improves perceived latency, not actual total time | SageMaker real-time endpoints support response streaming; displaying tokens as they generate improves user experience even if total generation time is unchanged |
| **Guardrail call optimization** | Each Bedrock Guardrails call adds latency | Run input and output guardrail checks in parallel with independent, non-blocking paths where possible, and keep custom Lambda regex checks lightweight (avoid heavy computation in the orchestrator) |
| **Regional proximity** | Network latency | Ensure the SageMaker endpoint, Lambda, and API Gateway are deployed in the same AWS region as the primary user base (e.g., `ap-south-1` for India-based usage) to minimize round-trip latency |

**Priority order for this project's scale**: semantic caching (highest impact, lowest cost) → smaller/quantized model at inference → provisioned concurrency only if cold-start latency proves problematic during demo/testing (adds cost, so applied selectively) → streaming for perceived responsiveness.