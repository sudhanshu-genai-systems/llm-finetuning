# Healthcare Fine-Tuning Project — Pipeline Overview

## 1. Process Overview

1. Source, clean, and sample the dataset (10-20K rows)
2. Establish baseline — capture base model outputs on 20-30 test questions prior to fine-tuning
3. Format data into instruction/chat template (Alpaca or ChatML)
4. Upload processed data to S3
5. Execute SageMaker Training Job (QLoRA, PEFT, bitsandbytes via HF container)
6. Evaluate fine-tuned model against the same 20-30 questions and compare with baseline
7. Merge LoRA adapters (optional) and register the model in SageMaker Model Registry
8. Deploy to a SageMaker endpoint (Serverless or Async Inference)
9. Attach Model Monitor (drift detection) and guardrails to the inference pipeline
10. Connect the endpoint to a Q&A bot via Lambda and API Gateway

## 2. Data Preparation — Serverless-First AWS Stack

| Task | Service | Rationale |
|---|---|---|
| Ingest raw HF dataset | SageMaker Studio Notebook (small `ml.t3.medium`) or AWS Lambda (for chunks under 250MB/15-min limit) | Avoids GPU cost for pure CPU/text work |
| Clean, dedupe, filter, tokenize-length checks | AWS Glue (serverless PySpark) or Lambda + Pandas for smaller volumes | Serverless, pay-per-job-second, suited for one-off ETL |
| Format to instruction JSON/Parquet | Same Glue job or notebook | Output directed to S3 |
| Orchestration (optional, for repeatability) | Step Functions triggering Glue → SageMaker Processing → Training | Maintains a serverless, repeatable pipeline |

Cost note: Glue/Lambda processing for a dataset of this scale (~100K rows) is expected to cost a nominal amount, not GPU-hour pricing.

## 3. Data Storage — Cost Considerations

- S3 Standard for active working data (train/val/test splits) — cost is negligible at this scale.
- Parquet format is preferable to raw JSON/CSV — reduces size and improves read speed for SageMaker Processing/Training.
- S3 Intelligent-Tiering should be enabled on the bucket to auto-migrate untouched raw/intermediate files to lower-cost tiers.
- Lifecycle rules should be set to expire intermediate processing artifacts after final splits are confirmed (e.g., 7-day expiry on a `/tmp-processing/` prefix).
- EBS/EFS storage is not recommended here — S3 is the lower-cost option and integrates directly with SageMaker.

## 4. Baseline Evaluation (Pre-Fine-Tuning)

- A dedicated training GPU is not required for this step. SageMaker Serverless Inference (scales to zero, billed per invocation) or a short-lived notebook kernel is sufficient to load the base model and run the fixed set of 20-30 medical questions.
- Outputs to be saved: question, base model response, timestamp, model version — stored as JSON/CSV in S3 (`/eval/baseline/`).
- This becomes the before/after comparison artifact; the same question set should be reused post-fine-tuning.

## 5. Training on SageMaker and Deployment

**Training:**
- SageMaker Training Job using the Hugging Face DLC (Deep Learning Container), which supports `transformers`, `peft`, `bitsandbytes`, and `trl` natively.
- Instance: `ml.g4dn.xlarge`, with SageMaker Managed Spot Training enabled — this provides Spot pricing with automatic checkpointing/resume built in.
- QLoRA configuration: 4-bit NF4 quantization, LoRA rank 16-32, target modules set to attention projections.
- Checkpoints are written to S3 automatically as part of standard SageMaker behavior.

**Deployment — decision point:**

| Option | Best suited for | Trade-off |
|---|---|---|
| SageMaker Serverless Inference | Light or bursty traffic (testing/demo) | No idle cost, scales to zero; higher cold-start latency |
| Real-time endpoint (`ml.g4dn.xlarge`) | Consistent low-latency needs | Billed hourly regardless of usage; requires manual stop/start discipline to control cost |

Given the project's budget constraints, Serverless Inference is the more cost-efficient default, with a real-time endpoint reserved for periods of active demo/testing.

Model versions should be registered in SageMaker Model Registry for traceability.

## 6. Data Drift and Guardrails on Inference

- **Drift detection**: SageMaker Model Monitor, with Data Capture enabled on the endpoint. A baseline is established from training data statistics, with periodic drift-check jobs scheduled via serverless Processing jobs.
- **Guardrails — decision point**:

| Option | Description | Trade-off |
|---|---|---|
| Amazon Bedrock Guardrails (standalone API) | Can be invoked independently of Bedrock models to screen input/output text for PII, harmful content, denied topics (e.g., medication dosage requests) | More robust, additional per-call cost |
| Custom guardrail Lambda | Regex/keyword filtering, optionally a lightweight classifier | Lower cost, less comprehensive coverage |

Recommended approach: a Lambda orchestrator that calls Bedrock Guardrails for input screening, invokes the SageMaker endpoint, and calls Bedrock Guardrails again for output screening — a fully serverless, pay-per-call structure.

## 7. Q&A Bot Using the Same Endpoint

- Backend: API Gateway + Lambda (the guardrail orchestrator described in Section 6) — fully serverless.
- Frontend — decision point:

| Option | Best suited for | Trade-off |
|---|---|---|
| Static React/Streamlit app on S3 + CloudFront | Public-facing or shareable demo | Near-zero hosting cost, requires initial setup |
| Gradio app on a small notebook instance | Internal/quick testing | Faster to stand up, incurs instance runtime cost while active |

The same SageMaker endpoint from Section 5 is reused — no additional model hosting cost is incurred, only the orchestration layer.

## Cost Summary

| Component | Estimated Cost |
|---|---|
| Data preparation (Glue/Lambda) | ₹50-100 |
| Baseline evaluation (serverless inference, 30 calls) | ₹20-50 |
| Training (QLoRA, g4dn.xlarge Spot, multiple iterations) | ₹150-300 |
| Serverless inference (testing/demo traffic) | ₹50-150 |
| Model Monitor + Guardrails calls | ₹50-100 |
| **Total (Healthcare project)** | **Under ₹1,000** |
