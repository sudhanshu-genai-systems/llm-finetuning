# S3 Bucket Requirements — Combined Analysis

## Buckets Needed: 2 (recommended) — 1 (minimum viable)

| Approach | Bucket Count | Structure |
|---|---|---|
| **Minimum viable** | 1 shared bucket | Prefixes separate everything: `s3://project-bucket/healthcare/{raw,processed,tmp-processing,models,eval}/` and `s3://project-bucket/bfsi/{raw,processed,tmp-processing,models,eval}/` |
| **Recommended** | 2 buckets (1 per project) | `s3://healthcare-llm-project/{raw,processed,tmp-processing,models,eval}/` and `s3://bfsi-llm-project/{raw,processed,tmp-processing,models,eval}/` |

**Reasoning for the recommended split**: healthcare and BFSI data carry different compliance/handling expectations (even with public/synthetic datasets, the pattern mirrors real HIPAA/RBI-adjacent boundaries). Separate buckets allow independent IAM policies, lifecycle rules, and access boundaries per domain without added cost — S3 buckets themselves are free; only stored data and requests are billed. Lifecycle rules and Intelligent-Tiering apply per-prefix in either approach, so the cost-saving guidance from the earlier plan holds regardless of which structure is chosen.

**Additional (infrastructure-level, not project data) buckets**:

| Bucket | Purpose | Shared or Separate |
|---|---|---|
| Terraform remote state bucket | Stores `.tfstate` for infra (IAM roles, pipeline definitions) | 1 shared bucket across both projects, with state files separated by key/path (e.g., `healthcare/terraform.tfstate`, `bfsi/terraform.tfstate`) |
| CI/CD artifact bucket | Auto-managed by CodePipeline (if used) for build artifacts | Typically 1, auto-provisioned; GitHub Actions does not require this |

**Total distinct buckets for the full setup: 3** — healthcare data, BFSI data, and Terraform state (CI/CD artifact bucket only if CodePipeline is chosen over GitHub Actions).

## Processing Script Storage — S3 or Git?

**Git is sufficient; a dedicated S3 bucket for source code is not needed.**

- Source code (Terraform configs, preprocessing scripts, training scripts) belongs in version control (GitHub/GitLab/CodeCommit) — this is the single source of truth, supports code review, branching, and CI/CD triggers.
- When a SageMaker Processing or Training Job is launched via the SageMaker Python SDK, the `source_dir` (local script folder) is **automatically packaged and staged to S3 by SageMaker itself** at job submission time — this is a transient, SDK-managed upload, not something requiring manual bucket setup or management.
- S3 should hold **data and artifacts** (datasets, model weights, evaluation outputs) — not the codebase.

## Where Else the Project Setup Can Live

| Component | Recommended Location |
|---|---|
| Source code (Terraform, preprocessing/training scripts, CI/CD configs) | Git repository (GitHub/GitLab) — primary source of truth |
| Local/interactive development & testing (before running paid AWS jobs) | Local machine or Docker container — validate preprocessing logic on a data sample at zero AWS cost |
| Interactive exploration against real data | SageMaker Studio Code Editor/JupyterLab (billed only while the app instance runs) |
| Terraform state | S3 bucket + DynamoDB table (state locking) — standard practice, low cost |
| CI/CD orchestration | GitHub Actions (no extra AWS infra needed) or AWS CodePipeline/CodeBuild (if staying fully within AWS-native tooling) |

**Summary**: 2 project-data buckets (healthcare, BFSI) plus 1 shared Terraform state bucket covers the full setup; source code stays in Git and is never persisted to S3 directly, since SageMaker handles script staging automatically per job.