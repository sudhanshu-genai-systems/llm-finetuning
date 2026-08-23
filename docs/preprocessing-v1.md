# Preprocessing: Python Framework, Libraries & Code Structure

## Libraries

| Purpose | Library |
|---|---|
| Dataset loading | `datasets` (Hugging Face) |
| Data manipulation | `pandas` |
| Tokenization/length filtering | `transformers` (`AutoTokenizer`) |
| Dedup (near-duplicate detection, optional) | `datasketch` (MinHash) or simple hash-based exact dedup via `pandas`/`hashlib` |
| S3 I/O | `boto3` or `s3fs` (works directly with `pandas.to_parquet()`) |
| Config management | `pydantic` or simple `argparse`/YAML config |
| SageMaker job orchestration (from CI/CD side) | `sagemaker` Python SDK |

## Single File vs. Multi-File (OOP) — Recommendation

**Multi-file, lightly-OOP structure is preferable**, for the following reasons:

| Reason | Explanation |
|---|---|
| **Reusability across both datasets** | Healthcare and BFSI preprocessing share ~70% of logic (load, dedup, token-length filter, split, save-to-S3) — a shared base class avoids duplicating this across two projects |
| **SageMaker Processing entrypoint constraint** | The job needs one clear entrypoint script; multi-file works fine as long as all files are packaged together in the `source_dir` passed to the Processing Job |
| **Testability** | Individual components (cleaning, formatting, sampling) become unit-testable in isolation, which a single monolithic script discourages |
| **CI/CD fit** | Cleaner diffs, easier code review, and modular versioning align better with a CI/CD pipeline than one large file |

## Suggested Structure

```
preprocessing/
├── entrypoint.py              # SageMaker Processing Job entrypoint — parses args, calls pipeline
├── base_preprocessor.py       # Abstract base class: load, dedup, token-filter, split, save (shared logic)
├── healthcare_preprocessor.py # Subclass: ChatDoctor-specific formatting (Alpaca template, medical Q/A structure)
├── finance_preprocessor.py    # Subclass: Finance-Instruct-specific formatting (sub-task stratified sampling)
├── utils/
│   ├── s3_io.py                # S3 read/write helpers
│   ├── tokenizer_utils.py      # Token-length filtering helpers
│   └── dedup.py                 # Dedup logic
├── config.yaml                 # Dataset-specific parameters (sample size, HF dataset ID, target model tokenizer)
└── requirements.txt
```

**Pattern**: an abstract `BasePreprocessor` class handles shared steps (load → clean → dedup → filter → split → save), with `HealthcarePreprocessor` and `FinancePreprocessor` overriding only the dataset-specific formatting method. `entrypoint.py` selects the correct subclass based on a config/CLI argument, keeping one Processing Job definition reusable for both datasets rather than maintaining two separate jobs.

This structure can be committed to the repository, with the CI/CD pipeline packaging the `preprocessing/` directory as the `source_dir` for the SageMaker Processing Job.