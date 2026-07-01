# C-DQC v4.1.1 — Agentic Research Core

**C-DQC** is an auditable research prototype for purpose-aware assessment of tabular data quality and reuse readiness. Version 4.1.1 combines a bounded LLM policy with a closed catalogue of deterministic detector tools and a deterministic verdict layer.

The system is designed for data-science research. Health-related datasets are used as representative tabular benchmarks; C-DQC is **not** a clinical decision-support system and does not make clinical-validity claims.

Associated Work-in-Progress submission:

> **Agentic AI for Healthcare Data Quality and Reuse Readiness**  
> Reference: **AIR-066**  
> 7th International Conference on AI Research (**ICAIR 2026**)

## Research objective

C-DQC investigates whether a bounded agentic policy can select only the data-quality checks needed for a declared reuse purpose while preserving an auditable evidence trail and preventing the LLM from changing data, inventing findings, or overriding the final decision.

The research comparison supports two execution conditions:

1. **Fixed pipeline** — all configured deterministic detectors are executed in a predefined order.
2. **Agentic pipeline** — an LLM or deterministic policy selects one detector at a time from a closed tool catalogue and may stop within a fixed step budget.

The system does not automatically repair the source dataset.

## Architecture

```text
INPUT DATA + CONFIGURATION
          |
          v
AGGREGATE DATASET PROFILE
          |
          v
PLAN: bounded policy selects one allowed tool
          |
          v
EXECUTE: deterministic detector generates evidence
          |
          +----------------------+
          |                      |
          v                      |
REPLAN OR STOP <-----------------+
          |
          v
DETERMINISTIC REUSE-READINESS VERDICT
          |
          v
AUDIT FILES + RECOMMENDATIONS
```

The LangGraph state machine implements:

```text
PROFILE -> PLAN -> EXECUTE_SELECTED_TOOL -> PLAN OR STOP -> FINALIZE
```

The LLM receives only an aggregate profile containing column names, declared or inferred roles, data types, aggregate statistics, metadata status, reuse purpose, executed-tool history, and issue-count summaries. Source rows and direct cell values are not included in the policy request.

## Agentic safety contract

The policy may:

- select one next detector from the closed catalogue;
- select existing target-column names;
- explain why the detector is relevant;
- specify the required evidence;
- stop within the configured step budget.

The policy may not:

- create new detector functions;
- inspect source rows through the policy interface;
- modify source values;
- authorize automatic repair;
- convert statistical candidates into confirmed errors;
- override deterministic detector findings;
- assign or change the final reuse-readiness verdict.

All policy decisions are schema-constrained by a Pydantic `AgentDecision` model.

## Deterministic detector catalogue

C-DQC v4.1.1 exposes eleven independently executable tools:

1. `check_explicit_missingness`
2. `check_missing_tokens`
3. `check_disguised_missingness`
4. `check_range_violations`
5. `check_numeric_extremes`
6. `check_categorical_consistency`
7. `check_date_consistency`
8. `check_format_consistency`
9. `check_duplicates`
10. `check_metadata_completeness`
11. `check_privacy_documentation`

Each tool runs its own deterministic algorithm. The agentic path does not execute the full fixed pipeline and filter the results afterward.

## Reuse-readiness verdicts

The final decision is assigned by deterministic rules:

- `READY`
- `CONDITIONALLY_READY`
- `NOT_READY`
- `INSUFFICIENT_EVIDENCE`

The configuration may add purpose-specific requirements but cannot weaken the minimum evidence contract.

At minimum:

- explicit completeness evidence is required;
- metadata-completeness evidence is required;
- privacy-documentation evidence is required when `privacy_required` is `true`.

If a mandatory check was not executed, the verdict is `INSUFFICIENT_EVIDENCE`. Confirmed quality findings, metadata gaps, supported findings, or unresolved candidates prevent an unconditional `READY` verdict.

## Requirements

- Python 3.11 or newer
- pandas
- NumPy
- SciPy
- Pydantic
- LangGraph
- OpenAI Python SDK, only when the OpenAI policy is used
- pytest, for validation

Install dependencies:

```bash
python -m pip install -r requirements.txt
```

## Quick start: deterministic integration test

This test requires no external API and executes one selected detector:

```bash
python scripts/run_one_agentic_test.py --provider deterministic
```

Expected structure:

```json
{
  "status": "PASS",
  "provider": "deterministic",
  "executed_tools": [
    "check_explicit_missingness"
  ],
  "issue_count": 1,
  "verdict": "INSUFFICIENT_EVIDENCE"
}
```

`INSUFFICIENT_EVIDENCE` is correct because a one-step run does not complete the minimum reuse-readiness evidence contract.

## Quick start: Google Colab

### 1. Upload and extract the project

```python
from google.colab import files
uploaded = files.upload()
```

Upload `C-DQC_v4.1.1_agentic_research_core.zip`, then run:

```python
import zipfile
import os
from pathlib import Path

zip_path = Path("/content/C-DQC_v4.1.1_agentic_research_core.zip")
project_dir = Path("/content/C-DQC_v4.1.1_agentic_research_core")

with zipfile.ZipFile(zip_path, "r") as archive:
    archive.extractall("/content")

os.chdir(project_dir)
!python -m pip install -q -r requirements.txt
```

### 2. Run the deterministic test

```python
!python scripts/run_one_agentic_test.py --provider deterministic
```

### 3. Optional OpenAI-policy test

The OpenAI provider is optional and may incur API usage charges. Store the key only in the Colab session environment:

```python
import os
import getpass

os.environ["OPENAI_API_KEY"] = getpass.getpass("OpenAI API key: ")
```

Run one policy step:

```python
!python scripts/run_one_agentic_test.py \
    --provider openai \
    --model gpt-5-mini
```

Check `agent_trace.json`. A successful live call records:

```json
{
  "provider": "openai",
  "fallback_used": false
}
```

If the API call fails and `on_policy_error` is set to `fallback`, the deterministic policy completes the run and the trace records `fallback_used: true` together with the error.

## Assess a dataset

Supported input formats:

- CSV
- XLSX/XLS
- Parquet

Fixed-pipeline execution:

```bash
python -m cdqc_core assess \
  --input data/my_dataset.csv \
  --config configs/my_config.json \
  --output results/my_dataset_fixed
```

Agentic execution uses the same command. The execution mode is controlled by the configuration:

```bash
python -m cdqc_core assess \
  --input data/my_dataset.csv \
  --config configs/my_agentic_config.json \
  --output results/my_dataset_agentic
```

## Minimal agentic configuration

Copy `configs/agentic_config_template.json` and adapt it to the dataset.

```json
{
  "dataset_id": "my_dataset",
  "dataset_metadata": {
    "title": "My dataset",
    "description": "Purpose and content of the dataset",
    "provenance": "Dataset origin",
    "version": "1.0",
    "license": "Declared licence",
    "intended_reuse": "Declared reuse purpose",
    "privacy_status": "Documented privacy status"
  },
  "dataset_metadata_required_fields": [
    "title",
    "description",
    "provenance",
    "version",
    "license",
    "intended_reuse",
    "privacy_status"
  ],
  "metadata_required_fields": [
    "role",
    "description"
  ],
  "missing_tokens": [
    "NA",
    "N/A",
    "NULL",
    "?"
  ],
  "reuse_purpose": {
    "task": "binary_machine_learning",
    "target_column": "Outcome",
    "privacy_required": true,
    "required_tools": [
      "check_explicit_missingness",
      "check_metadata_completeness",
      "check_privacy_documentation"
    ]
  },
  "agentic": {
    "enabled": true,
    "provider": "openai",
    "model": "gpt-5-mini",
    "api_key_env": "OPENAI_API_KEY",
    "max_steps": 3,
    "required_tools": [],
    "on_policy_error": "fallback"
  },
  "columns": {
    "Age": {
      "role": "numeric",
      "description": "Recorded age",
      "valid_range": [0, 120],
      "sentinel_values": {}
    },
    "Outcome": {
      "role": "categorical",
      "description": "Binary target variable",
      "allowed_values": [0, 1],
      "canonical_mapping": {}
    }
  }
}
```

Column names in the configuration must match the source dataset exactly.

## Output files

Every assessment writes:

- `input_snapshot.csv`, `.xlsx`, or `.parquet` — unchanged source snapshot;
- `issues.csv` and `issues.json` — typed findings and evidence;
- `recommendations.csv` — evidence-linked recommendations;
- `quality_support_metrics.json` — no-reference support metrics;
- `run_manifest.json` — hashes, dimensions, mode, counts, and execution contract.

Agentic runs additionally write:

- `dataset_profile.json` — aggregate-only policy input;
- `agent_trace.json` — complete structured policy trace;
- `agent_decisions.csv` — tabular decision log;
- `tool_execution_log.csv` — executed tools and target columns;
- `agent_summary.json` — graph, step count, remaining tools, stop reason, and verdict;
- `reuse_readiness_verdict.json` — deterministic verdict and evidence contract.

When a reference dataset is supplied:

```bash
python -m cdqc_core assess \
  --input data/input.csv \
  --reference data/reference.csv \
  --config configs/my_config.json \
  --output results/reference_assessment
```

C-DQC writes `reference_exact_metrics.json`.

When a typed truth manifest is supplied:

```bash
python -m cdqc_core assess \
  --input data/input.csv \
  --truth-manifest data/truth_manifest.csv \
  --config configs/my_config.json \
  --output results/benchmark_assessment
```

C-DQC writes `detection_benchmark_metrics.json` with precision, recall, and F1 for typed defect units.

## Validation

Run the automated test suite:

```bash
python -m pytest -q
```

Validated v4.1.1 release result:

```text
15 passed
```

Run the complete packaged validation:

```bash
python scripts/run_full_validation.py
```

The release validation includes:

- 15 automated tests;
- 30 synthetic benchmark runs;
- one-step LangGraph integration;
- deterministic fallback behavior;
- minimum-evidence verdict safeguards;
- source-snapshot preservation;
- OpenAI SDK and LangGraph contract checks.

## Recorded external pilot run

A live three-step agentic run was executed on a 1,000-row, 10-column healthcare-themed tabular benchmark.

The LLM policy selected:

1. `check_explicit_missingness`
2. `check_metadata_completeness`
3. `check_privacy_documentation`

Observed outcome:

- provider: OpenAI;
- model: `gpt-5-mini`;
- fallback used: `false`;
- executed tools: 3 of 11;
- findings: 1,326;
- confirmed explicit-missingness findings: 1,325;
- dataset-metadata gaps: 1;
- final verdict: `CONDITIONALLY_READY`;
- source data modified: no.

Executing 3 of the 11 available tools represents 72.7% fewer tool executions than running the complete catalogue. This pilot demonstrates bounded tool selection and deterministic verdict assignment. It does **not** establish equivalent detection coverage, generalisation, or superiority over the fixed pipeline.

## Scientific claim boundary

Without a reference dataset or controlled corruption manifest, C-DQC reports support metrics rather than ground-truth accuracy.

The current release establishes:

- implementation correctness for the tested paths;
- bounded LangGraph control flow;
- independently executable detector tools;
- structured policy decisions;
- deterministic verdict safeguards;
- auditable fallback behavior;
- reproducible synthetic benchmark behavior.

It does not yet establish:

- superiority of the LLM policy over fixed or deterministic baselines;
- external-dataset generalisation;
- clinical validity;
- privacy compliance or de-identification adequacy;
- safe automated data repair.

A privacy-documentation check confirms whether privacy status is documented. It does not by itself perform formal disclosure-risk assessment or certify that direct identifiers have been removed.

## Reproducibility and data handling

- The source dataset is never overwritten.
- The source snapshot preserves the original schema.
- Audit information is stored separately.
- File and configuration hashes are recorded in `run_manifest.json`.
- LLM decisions, token usage, fallback status, and errors are retained in `agent_trace.json`.
- API keys must be stored in environment variables and must not be committed to the repository.

## Project structure

```text
C-DQC_v4.1.1_agentic_research_core/
├── cdqc_core/                  # Core implementation
│   ├── agent.py                # LangGraph execution loop
│   ├── llm_policy.py           # Deterministic and OpenAI policies
│   ├── selective_detectors.py  # Independently executable tools
│   ├── verdict.py              # Deterministic verdict layer
│   ├── pipeline.py             # Assessment and output pipeline
│   └── ...
├── configs/                    # Configuration templates and examples
├── data/                       # Example and benchmark data
├── scripts/                    # Integration and full-validation scripts
├── tests/                      # Automated tests
├── results/                    # Generated outputs
├── COLAB_QUICKSTART.ipynb      # Colab notebook
├── SCIENTIFIC_METHOD.md        # Methodological specification
├── LIETOSANA_LATVISKI.md       # Latvian usage guide
├── VALIDATION_REPORT.json      # Release validation record
└── README.md
```

## Authors

- **Agate Jarmakovica** — conceptualisation, methodology, software, validation, formal analysis, visualisation, and writing.
- **Arnis Kirshners** — supervision, project administration, review, and editing.

## Status

C-DQC v4.1.1 is a research prototype and Work-in-Progress implementation. Results must be interpreted within the stated evidence and validation boundaries.

Before public distribution, add an explicit software licence and verify that no API keys, confidential datasets, or restricted metadata are included in the repository.
