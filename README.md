# jjushycsh-noshow-analytics

Predicting and reducing missed referral appointments at Jigjiga University Sheikh Hassan Yabare
Comprehensive Specialized Hospital (JJUSHYCSH). BAN6800 capstone - Nexford University.

| Module | Increment in this repo |
|---|---|
| 1 | Vision, ethical AI charter (`docs/vision_document.md`) |
| 2 | Architecture, data dictionary, RAID log, privacy plan |
| **3 (Sprint 1)** | **Data pipeline: DAG, Great Expectations suites, anonymization, bias check, audit log, Docker** |
| 4 | Model, SHAP, Fairlearn (next) |
| 5 | Dashboard and stakeholder validation (next) |

## Data
Development uses the public *Medical Appointment No Shows* dataset (Hoppen, 2016; 110,527 rows, 14 columns).
Download `KaggleV2-May-2016.csv` from Kaggle and place it in `data/external/`. Data files are never
committed (see `.gitignore`); DVC tracks them. A future JJUSHYCSH extract follows the same data contract
(`docs/data_dictionary.md`) after ethics and DPO approval.

## Pipeline stages
`ingest -> validate_raw (GX) -> clean -> anonymize -> build_features -> validate_processed (GX) -> publish_outputs + bias_check -> governance_report`

| Stage | Code |
|---|---|
| Ingestion + immutable snapshot | `src/data_prep/ingest.py` |
| Cleaning + quality report | `src/data_prep/clean.py` |
| Anonymization (PII removal) | `src/data_prep/anonymize.py` |
| Features + patient history | `src/features/build_features.py` |
| Integration, splits, dashboard feed | `src/data_prep/integrate.py` |
| Validation suites | `src/validation/gx_suite.py` (Great Expectations); `src/validation/pandas_suite.py` (same checks, no install) |
| Bias detection | `src/validation/bias_check.py` |
| Access control / retention | `src/governance/access_control.py` |
| Privacy audit log | `src/utils/audit_log.py` -> `logs/privacy_audit.jsonl` |
| Orchestration | `dags/noshow_etl_dag.py` (Apache Airflow) |

## Run it
```bash
# 1. local  (use `python run_pipeline.py --skip-gx` if Great Expectations cannot be installed)
pip install -r requirements-pipeline.txt
export NOSHOW_HASH_SALT="<secret from the data officer>"
pytest -q tests
python run_pipeline.py
# Data Docs: open gx/uncommitted/data_docs/local_site/index.html

# 2. Docker (tests run first, then the pipeline)
docker build -t noshow-pipeline:0.3.0 .
docker run --rm -e NOSHOW_HASH_SALT=$NOSHOW_HASH_SALT \
  -v $(pwd)/data:/opt/pipeline/data -v $(pwd)/reports:/opt/pipeline/reports \
  -v $(pwd)/logs:/opt/pipeline/logs -v $(pwd)/gx:/opt/pipeline/gx noshow-pipeline:0.3.0

# 3. Airflow: copy dags/ into $AIRFLOW_HOME/dags, set PIPELINE_ROOT to this repo, trigger jjushycsh_noshow_etl
```

## Outputs (input to Module 4)
- `data/processed/appointments_model_table.parquet` - one row per appointment, pseudonymized
- `data/processed/train|validation|test.parquet` - patient-grouped 70/15/15 split, seed 42
- `data/processed/dashboard_daily_noshow.parquet` - aggregate-only feed for the Module 5 dashboard
- `reports/` - data-quality report, GX results, bias tables and chart

## Versioning
Git Flow (`main` / `develop` / `feature/*`), SemVer tags (this release: `v0.3.0`), DVC for data.
