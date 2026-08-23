# Preprocess Job Setup: Terraform + CI/CD for SageMaker Processing Job

## Terraform Resources Required

| Resource | Purpose |
|---|---|
| `aws_iam_role` + `aws_iam_role_policy` | Execution role for the Processing Job (S3 read/write, ECR pull, CloudWatch logs) |
| `aws_s3_bucket` (or reference existing) | Input/output data location |
| `aws_ecr_repository` (if using custom container) | Hosts the preprocessing image, or reference the public HF DLC directly |
| `aws_sagemaker_processing_job` — **not natively supported as a persistent Terraform resource** (Processing Jobs are ephemeral/run-once, not long-lived infra) | Handled differently — see below |

## Key Design Point
SageMaker Processing Jobs are **transient, run-to-completion jobs**, not standing infrastructure — Terraform is not the ideal tool to "launch" the job itself (a job run has no meaningful ongoing state for Terraform to manage). The practical split:

| Layer | Tool | Reason |
|---|---|---|
| **Infrastructure** (IAM roles, S3 buckets, ECR repo, SageMaker Pipeline definition) | **Terraform** | Declarative, versioned, reused across both projects (healthcare + BFSI) |
| **Job execution** (triggering the actual Processing/Training run) | **CI/CD pipeline step** (GitHub Actions/CodePipeline) calling AWS CLI, Boto3, or `sagemaker-python-sdk` | Jobs are invoked per pipeline run, not declared as static Terraform state |

## Recommended CI/CD Flow

```
Terraform (infra: IAM role, S3 bucket, ECR repo, SageMaker Pipeline resource)
        ↓
CI/CD trigger (GitHub Actions / CodePipeline on push to main)
        ↓
Build & push preprocessing container to ECR (if custom image)
        ↓
Boto3 / SageMaker SDK call → starts Processing Job (or triggers SageMaker Pipeline execution)
        ↓
Output written to S3 → triggers next stage (Training Job)
```

Terraform can, however, declare a **`aws_sagemaker_pipeline`** resource (the pipeline *definition*, including the processing + training steps as a DAG) — this is the more Terraform-appropriate object, since the pipeline definition itself is persistent infrastructure, while individual *executions* of it are triggered by CI/CD, not Terraform.
