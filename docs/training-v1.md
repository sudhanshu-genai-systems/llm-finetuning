# 1. AWS Resource Deployment — Baseline Evaluation, Training, Deployment & Experiment Tracking

## Section 4 — Baseline Evaluation: Resources Needed

| Resource | Purpose | Billing Behavior |
|---|---|---|
| SageMaker Serverless Inference endpoint (temporary) OR SageMaker Studio Notebook kernel (`ml.g5.xlarge`, short-lived) | Load base model, run 20-30 fixed questions | Serverless: per-invocation, scales to zero. Notebook: per-second while kernel is active |
| S3 (`/eval/baseline/`) | Store question, response, timestamp, model version as JSON/CSV | Negligible storage cost |
| IAM execution role | Grants the notebook/endpoint access to S3 and the model artifact | No charge |

## Section 5 — Training & Deployment: Resources Needed

| Resource | Purpose | Billing Behavior |
|---|---|---|
| SageMaker Training Job (HF DLC, `ml.g4dn.xlarge`, Managed Spot) | QLoRA fine-tuning | Per-second while job runs; Spot discount applied automatically |
| S3 (processed data, checkpoints, model artifacts) | Input/output for training | Storage-only cost |
| SageMaker Model Registry | Version tracking of trained models | No separate charge beyond storage |
| SageMaker Serverless Inference endpoint | Post-training evaluation + demo/Q&A bot backend | Per-invocation, scales to zero |
| ECR (if custom training image is used beyond the standard HF DLC) | Container hosting | Storage-only |

## Experiment Tracking — MLflow Decision Point

The earlier cost flag on SageMaker **Managed** MLflow (continuous billing until deleted, ~₹1,600-1,800/day if left running) stands — however, experiment tracking itself is a reasonable requirement given two parallel projects with multiple training iterations. Three options, ranked by cost:

| Option | Cost Profile | Trade-off |
|---|---|---|
| **SageMaker Managed MLflow, created only for active tracking windows, deleted immediately after** | ~₹65-75/hr only while it exists (e.g., a few hours per training session, not continuous) | Fully managed, native SageMaker Model Registry integration; requires discipline to delete after each session |
| **Self-hosted MLflow on a small EC2 instance (e.g., `t3.micro`/`t3.small`) or ECS Fargate task, stopped between sessions** | ~₹1-3/hr, stoppable on demand; backend store on SQLite/S3, artifact store on S3 | More setup effort; full control over start/stop, significantly cheaper for two-project scale |
| **SageMaker Experiments (built-in, no separate tracking server)** | No dedicated server cost — billed only for underlying S3/metadata storage | Native, zero extra infra; less feature-rich UI compared to full MLflow (fewer visualization/comparison capabilities) |

**Recommendation**: self-hosted MLflow on a small EC2 instance (or Fargate task), started before a training/eval session and stopped immediately after, offers the tracking capability desired at a fraction of the Managed MLflow cost — reasonable given both projects will run multiple iterations. Managed MLflow remains viable if it is explicitly provisioned only around active sessions rather than left running.

---

# 2. Training Frameworks — Industry Standard Stack

| Layer | Framework/Library | Role |
|---|---|---|
| Model loading & tokenization | **Hugging Face `transformers`** | Industry-standard model/tokenizer interface |
| Parameter-efficient fine-tuning | **`peft`** (Hugging Face) | LoRA/QLoRA adapter implementation |
| Quantization | **`bitsandbytes`** | 4-bit NF4 quantization for QLoRA |
| Training loop / SFT orchestration | **`trl`** (`SFTTrainer`) | Standard supervised fine-tuning trainer, widely adopted for instruction-tuning workflows |
| Distributed/mixed-precision handling | **`accelerate`** | Underpins `trl`/`transformers`, manages device placement |
| Dataset handling | **`datasets`** (Hugging Face) | Same library already used for preprocessing |
| Experiment tracking integration | **`mlflow`** Python client | Logs directly into the tracking server (Managed or self-hosted) |

This combination (`transformers` + `peft` + `bitsandbytes` + `trl` + `accelerate`) is the most widely adopted, community-supported stack for QLoRA fine-tuning currently, and is natively supported inside the SageMaker Hugging Face DLC — no custom container required.

---

# 3. Local (Cursor IDE) + AWS Hybrid Workflow

**Feasibility**: Yes — local development and testing prior to full-scale AWS training is both possible and a recommended practice, provided the local machine has sufficient resources to run at least a small-scale validation pass.

## Suggested Hybrid Workflow

| Stage | Location | Purpose |
|---|---|---|
| 1. Script development (preprocessing, training script, config) | Local machine, Cursor IDE | Fast iteration, debugging, no AWS cost |
| 2. Smoke test — 1 epoch on a small data sample (few hundred rows) | Local machine (CPU or local GPU if available) | Validates the pipeline logic, config, and data format end-to-end before spending AWS budget |
| 3. Full training run — 20-50 epochs on full sampled dataset | SageMaker Training Job (`ml.g4dn.xlarge`, Managed Spot) | Actual GPU-scale training with checkpointing |
| 4. Evaluation, deployment | SageMaker Serverless Inference | Consistent with the plan already established |

## Making the Same Script Work in Both Environments

- The training entrypoint script should be written to read hyperparameters and data paths via **environment variables / `argparse`**, matching the pattern SageMaker injects automatically (`SM_CHANNEL_TRAIN`, `SM_MODEL_DIR`, `SM_OUTPUT_DATA_DIR`, hyperparameters as CLI args). Locally, these same arguments are simply passed manually or via a `.env`/config file with local paths substituted.
- This means **one script serves both environments** — run as `python train.py --epochs 1 --data_dir ./sample_data` locally, and the identical script is packaged as the SageMaker Training Job entrypoint with `--epochs 50 --data_dir /opt/ml/input/data/train` (SageMaker sets this automatically).
- The **SageMaker Python SDK can be invoked directly from Cursor's terminal** — local development and remote job submission can happen from the same IDE session; no separate console/notebook environment is required to trigger the AWS-side job.
- For closer parity between local and AWS execution, the **Hugging Face DLC image can be pulled and run locally via Docker** (if Docker is available locally) to validate the exact same container environment before submitting to SageMaker — reduces "works locally, fails on SageMaker" discrepancies from library version mismatches.
- Local GPU availability determines how far the "local" stage can go: a local CPU-only machine restricts local testing to very small samples/short runs purely for logic validation (not real training signal); a local machine with a modest GPU (e.g., 8GB+ VRAM) can run a legitimate 1-epoch QLoRA pass on a small model for validation before scaling up on SageMaker.

**Summary recommendation**: develop and debug scripts locally in Cursor with a small data sample and 1 epoch (cost: zero, aside from local compute/time), then submit the identical script unchanged to a SageMaker Training Job for the full 20-50 epoch run — achieved by parameterizing paths/hyperparameters consistently across both environments rather than maintaining two separate versions of the training code.