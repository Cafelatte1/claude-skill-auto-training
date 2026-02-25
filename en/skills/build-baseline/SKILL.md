---
name: build-baseline
description: A skill that automatically explores the raw source data to infer sample units and label/path rules, documents them as a fixed standard, and then generates preprocessing code and baseline training code that are immediately runnable and pass smoke tests. Throughout execution, it prioritizes user-defined rules above all else and creates/verifies task-specific data structures and validation rules in a consistent, standardized way.
---

## When to use

* Use this skill **only when the user explicitly types** **`/build-baseline`**.
* Use it when there are multiple source folders under `dataset/rawdata/` and you want the model to explore them, document parsing rules, and generate a standard pipeline/baseline codebase accordingly.
* Use it when you want to write the baseline training script (`02_train_baseline.py`) following the standard rules and ensure it passes a smoke test.
* Use it when you want to manage validation splits with advanced strategies (e.g., stratified/group-based) instead of a simple random split.
* Use it when you want to align data, loader, and schema strictly to `task_format.yaml` to keep task-specific structures consistent.

---

## Workflow

### Common

* On execution, this skill must reference **`./user_defined_rules.json`** and **`./user_instruction.md`** with the **highest priority**.
* Any items specified in `user_defined_rules.json` and `user_instruction.md` are treated as **user-defined constraints** and must not be changed arbitrarily across training/evaluation/analysis/next-version drafting.
* `user_defined_rules.json` and `./user_instruction.md` must be in the same directory as this SKILL.md. If missing, assume there are **no user-defined constraints**.
* `task_format.yaml` must exist at `./task_format.yaml` (same directory as this SKILL.md). If missing, **stop the skill**.
* Steps 0–2 must pass their tests before proceeding to the next step.
* Any operation involving randomness (sampling, split, shuffle, subset extraction, smoke-test sample selection, etc.) must be reproducible using the root `seed` key. If missing, use `42`.
* In this skill, `{version}` always refers to `user_defined_rules.json.dataset_config.dataset_id`.

  * Example: `dataset_id = "v1"` → `dataset/prepdata/v1/...`

Example (partial) `user_defined_rules.json`:

```json
{
  "seed": 42,
  "dataset_config": {
    "dataset_id": "v1",
    "task_type": "ocr-recognition",
    "valid_ratio": 0.05,
    "split_strategy": "stratified",
    "stratify_key": "category",
    "group_key": "group_id",
    "max_profile_samples": 100,
    "max_train_samples": 1000
  },
  "model_config": {
    ...
  },
  "baseline_smoke_test": {
    "train_samples": 100,
    "valid_samples": 100,
    "min_batch_size": 4,
    "max_batch_size": 128
  }
}
```

---

### Step 0) Rawdata Profiling Document + Test (Rawdata profiling)

Role:

* Explore all source folders under `dataset/rawdata/` and analyze files/structures to **freeze** per-source parsing rules and sample-unit definitions as a document.
* If `dataset_config.task_type` is missing, define it via profiling and provide guidelines so the data can be structured accordingly.
* All later pipeline steps must read this document and follow the **exact same rules**.
* Proceed only if documentation and minimum validation (sample load test) pass.

Inputs:

* `user_defined_rules.json` (highest priority)

  * `seed` (default 42)
  * `dataset_config.task_type` (optional)
  * `dataset_config.max_profile_samples` (optional)
* `dataset/rawdata/*` (all sources; not a single specific source)

Work (auto-exploration + documentation):

* Identify profiling target directories under `dataset/rawdata/`.

  * Include: `dataset/rawdata/{source}/...`
  * Exclude: document files like `dataset/rawdata/profiling.md`
* For each `{source}`, perform the steps below and write results into a single consolidated document.
* Limit total scanned/loaded/sampled items during profiling to `max_profile_samples` (default 1,000), selected reproducibly using `seed`.

Minimum required contents of `dataset/rawdata/profiling.md` (MUST):

* `Overview` (overall rawdata overview, list of sources, estimated scale)
* `Sources`

  * One section per source: `## Source: {source_name}`
* Required subsections per source:

  * `Sample unit definition`
  * `Paths & naming rules`
  * `Label schema`
  * `Edge cases`
  * `Extraction rules`
  * `Sanity checks`

Work (test: minimum load validation):

* For each source, load N random samples (recommended N=10) and verify:

  * sample-unit parsing works
  * asset/label matching rules are consistent (if applicable)
  * label schema parsing works
* Sample selection must be reproducible via `seed`.
* Record PASS/FAIL at the bottom of each source section.

Outputs (required):

* `dataset/rawdata/profiling.md`

---

### Step 1) Data Pipeline Code + Test (Data pipeline)

Role:

* Read `dataset/rawdata/profiling.md` from Step 0 and parse/normalize rawdata into prepdata following the documented rules.
* Generate `dataset/prepdata/{version}/...` in the structure recommended by `./task_format.yaml` under `supported_tasks[{task_type}]`.
* Proceed only after code is written and tests pass.

Inputs:

* `user_defined_rules.json` (highest priority)

  * `seed` (default 42)
  * `dataset_config.dataset_id` → `{version}`
  * `dataset_config.task_type`
  * `dataset_config.valid_ratio` (optional; default if missing)
  * `dataset_config.split_strategy` (default `random`)
  * `dataset_config.stratify_key` (default `category`)
  * `dataset_config.group_key` (default `group_id`)
  * `dataset_config.max_train_samples` (optional)
* `task_format.yaml` (required)
* `dataset/rawdata/profiling.md` (required)
* `dataset/rawdata/*` (all sources included in profiling.md)

Work (code writing):

#### Apply profiling document rules

* Must parse rawdata strictly according to `dataset/rawdata/profiling.md`.
* Do not introduce new “guessed parsing logic” that conflicts with profiling.md.
* If needed, update profiling.md first, then adjust the pipeline.

#### Build meta table: `df_meta`

* Create `df_meta` based on `Extraction rules` in profiling.md.
* `df_meta` must support multiple sources and include a `source` column.
* If `dataset_config.max_train_samples` exists:

  * first finalize the valid trainable sample pool (after preprocessing/validation)
  * then sample up to `max_train_samples` from that pool reproducibly using `seed`.
* If not specified otherwise, use uniform random sampling across sources.

Minimum columns:

* `sample_id`, `source`, `category` (null if missing), `asset_paths`

Recommended columns:

* `group_id`, `lang`, `length`, `width`, `height`, `timestamp`, `label_summary`, `hash`

Recommended save path:

* `dataset/prepdata/{version}/common/df_meta.parquet`

#### Validation split

* Must be reproducible via `seed`.
* Support strategies by `split_strategy`:

  * `random`
  * `stratified` (default key: `category`)
  * `group` (default key: `group_id`)
* For `stratified`, exact distribution matching is not required; it is acceptable as long as it is not severely skewed.
* Recommended manifests:

  * `dataset/prepdata/{version}/common/train_manifest.parquet`
  * `dataset/prepdata/{version}/common/valid_manifest.parquet`

#### Generate prepdata following `task_format.yaml`

* Must follow `task_format.yaml.supported_tasks[{task_type}]` for `split_layout` and `index_schema`:

  * `dataset/prepdata/{version}/stats.json`
  * `dataset/prepdata/{version}/train/`
  * `dataset/prepdata/{version}/valid/`
  * `dataset/prepdata/{version}/common/`

Work (tests):

* Run consistency checks based on `task_format.yaml.validation.checks`.
* Additional checks per split strategy:

  * `group`: no group leakage
  * `stratified`: do not force PASS/FAIL; record distribution summary in the report
* Required file/folder checks:

  * `stats.json`, `train/`, `valid/`, `common/`
  * `train.jsonl|train.parquet`, `valid.jsonl|valid.parquet`

Outputs (required):

* Data pipeline code: `01_data_pipeline.py`
* `dataset/prepdata/{version}/stats.json`
* `dataset/prepdata/{version}/train/`
* `dataset/prepdata/{version}/valid/`
* `dataset/prepdata/{version}/common/`
* (Recommended) `dataset/prepdata/{version}/common/df_meta.parquet`
* (Recommended) `dataset/prepdata/{version}/_tests/pipeline_test_report.md`

---

### Step 2) Baseline Training Code + Test (Baseline)

Role:

* Write `02_train_baseline.py` and validate that training/validation runs correctly using only a small subset (default 100 train + 100 valid) for debugging/smoke testing.
* Proceed only after code is written and smoke tests pass.

Inputs:

* `user_defined_rules.json` (highest priority)

  * `seed` (default 42)
  * `dataset_config.dataset_id` → `{version}` and `DATASET_ID`
  * `dataset_config.task_type`
  * `baseline_smoke_test.epoch` (optional; default 1)
  * `baseline_smoke_test.train_samples` (default 100)
  * `baseline_smoke_test.valid_samples` (default 100)
  * `baseline_smoke_test.min_batch_size` (optional)
  * `baseline_smoke_test.max_batch_size` (optional)
  * (Optional) `training_config.*`, `model_config.*`
* `task_format.yaml` (required)
* `dataset/prepdata/{version}/train/`
* `dataset/prepdata/{version}/valid/`
* `dataset/prepdata/{version}/common/` (optional depending on task)

Work (code writing):

#### Generate baseline training script

* Write `02_train_baseline.py`.
* Must include a global variable `DATASET_ID: str`, and it must equal `{version}` (=`dataset_config.dataset_id`).
* Data loading must treat `dataset/prepdata/{version}` as the single source of truth and follow structure/schema required by `task_format.yaml.supported_tasks[{task_type}]`.
* Subset selection for smoke testing must be reproducible via `seed`.
* Must create minimal artifact/log structure:

  * `Models/Artifacts/baseline/spec.json`
  * `Models/Artifacts/baseline/additional_assets/`
  * `Models/Artifacts/baseline/logs/version_0/{metrics.csv,hparams.yaml}`
* Must support smoke-test mode:

  * sample N/M items from train/valid (default 100 each)
  * run only `baseline_smoke_test.epoch` (default 1) to validate execution

#### Find optimal batch size

* Search batch sizes in the range `min_batch_size ~ max_batch_size` using powers of two:

  * candidates: `bs ∈ { 2^k | min_batch_size ≤ 2^k ≤ max_batch_size }`
* Run only 10 steps per candidate.
* Start from the largest batch size; stop immediately if the largest succeeds.
* Choose the largest successful batch size as `optimal_batch_size`.
* Success criteria:

  * PASS if forward+backward completes for 10 steps without OOM/RuntimeError.
* Must log results into `_tests/baseline_smoke_test_report.md`.

#### Validate `expected_*` contracts

* Treat every key in `user_defined_rules.json.model_config` starting with `expected_` as a contract to validate.
* Prefer computing observed values using a single dummy forward pass (batch=1, minimal input).
* Minimal input must satisfy model input constraints (H_TARGET, W_BUCKETS, etc.) if any.

Validation procedure (MUST):

1. Collect all `expected_*` entries into `expected_map`.
2. If `expected_map` is empty, **SKIP** this validation step (not a FAIL).
3. Build `observed_map` from “verifiable actual values” produced/loaded by the training code.

   * `observed_map` must use the same suffix naming:

     * `expected_stride` ↔ `observed_stride`, `expected_channels` ↔ `observed_channels`
4. Decide PASS/FAIL per key (apply type rules below).

Type comparison rules (MUST):

* Scalar (number/string/bool): `observed == expected` else FAIL
* List/tuple: length, elements, and order must match
* Dict: all keys must exist and all values must match
* Float: exact match unless `model_config.expected_tolerance` exists; then require `abs(observed-expected) <= tolerance`

Handling unobservable values (MUST):

* If an `observed_*` value cannot be produced, mark FAIL.
* Exception: if `model_config.expected_allow_missing_keys: true`, record `SKIP(allow_missing)` and do not fail overall.

On failure (MUST):

* If any FAIL occurs, stop immediately and do not run batch search/smoke run.
* Re-run only after fixing config (`expected_*` etc.) or code (observation logic).

Report logging (MUST):

* Include the following in `Models/Artifacts/baseline/_tests/baseline_smoke_test_report.md`:

  * `expected_map`
  * `observed_map`
  * `validation_table` (key, expected, observed, status PASS/FAIL/SKIP, reason)
  * `overall_result: PASS|FAIL`

#### Smoke run

* Purpose: ensure the script runs without errors and produces artifacts.
* Run smoke training only with the selected `optimal_batch_size`.
* Run with train N + valid M samples.
* Minimum pass conditions:

  * training starts and finishes without errors
  * `best.ckpt` and `last.ckpt` are created
  * `spec.json`, `additional_assets/`, and logs are created

#### Failure/exception handling rules for training

* If an exception occurs while running `02_train_baseline.py`, fully reset (delete and recreate) the artifact directory:

  * target: entire `Models/Artifacts/baseline/`
* Summarize root cause based on error logs/stack traces and apply **code fixes**.

  * fix targets: `02_train_baseline.py` (and local modules it imports if needed)
  * user constraints (`user_defined_rules.json`, `user_instruction.md`) always take priority; bypassing constraints is forbidden
* Re-run using the same version (`baseline`).
* (Recommended) record retry history in:

  * `Models/Artifacts/baseline/_tests/baseline_smoke_test_report.md`

    * include at least: `attempt`, `error_summary`, `root_cause_hypothesis`, `code_change_summary`, `status(PASS/FAIL)`

#### Artifact saving

* Even for smoke runs, the following files must be created:

  * `Models/Artifacts/baseline/best.ckpt`
  * `Models/Artifacts/baseline/last.ckpt`
* Recommended best checkpoint criterion: choose best by `valid_loss` (loss-based is fine; document the criterion in `spec.json`).
* If checkpoint saving is impossible for the given task/framework, stop and document an alternative rule (do not omit arbitrarily).

Outputs (required):

* `02_train_baseline.py`
* `Models/Artifacts/baseline/spec.json`
* `Models/Artifacts/baseline/best.ckpt`
* `Models/Artifacts/baseline/last.ckpt`
* `Models/Artifacts/baseline/additional_assets/`
* `Models/Artifacts/baseline/logs/version_0/{metrics.csv,hparams.yaml}`
* `Models/Artifacts/baseline/_tests/baseline_smoke_test_report.md`

---

### Step 3) Final check (Final check)

Role:

* Perform a final verification that all outputs from Steps 0–2 exist and their tests passed.
* If the final check passes, trigger the next skill: **`progressive-training`**.

Inputs:

* All artifacts from Steps 0–2

Work:

* Verify using the checklist:

  * `dataset/rawdata/profiling.md` exists and contains PASS records per source
  * `dataset/prepdata/{version}/` contains `stats.json, train/, valid/, common/`
  * Step 1 tests passed (confirm PASS if report exists)
  * `02_train_baseline.py` exists
  * `Models/Artifacts/baseline/` contains `best.ckpt, last.ckpt, spec.json, logs/, additional_assets/, _tests/baseline_smoke_test_report.md`
  * Step 2 smoke test passed (confirm PASS if report exists)

Result:

* If final check passes:

  * Read **`.claude/skills/progressive-training/SKILL.md`** and continue follow-up work by passing the `dataset_id` used in this run
* If final check fails:

  * Write detailed failure reasons and conclude the work

---

## Directory structure (example)

```text
dataset/
├── rawdata/
│   ├── profiling.md
│   ├── source_a/
│   └── source_b/
└── prepdata/
    └── v1/
        ├── stats.json
        ├── train/
        ├── valid/
        ├── common/
        │   ├── df_meta.parquet
        │   ├── train_manifest.parquet
        │   └── valid_manifest.parquet
        └── _tests/
            └── pipeline_test_report.md
Models/
└── Artifacts/
    └── baseline/
        ├── spec.json
        ├── additional_assets/
        ├── best.ckpt
        ├── last.ckpt
        ├── logs/version_0/
        │   ├── metrics.csv
        │   └── hparams.yaml
        └── _tests/
            └── baseline_smoke_test_report.md
```

---

## Rules

### MUST

* On execution, read and apply `./user_defined_rules.json` with the highest priority.
* All randomness must use the root `seed` (default 42).
* Profiling scan/load caps must be controlled by `user_defined_rules.dataset_config.max_profile_samples`.
* Training sample caps must be controlled by `user_defined_rules.dataset_config.max_train_samples`, applied after preprocessing/validation finalizes the trainable pool.
* Do not run if `task_format.yaml` is missing.
* Step 0 must auto-explore all source folders under `dataset/rawdata/` and write `dataset/rawdata/profiling.md`.
* Step 1 must parse rawdata strictly by reading `dataset/rawdata/profiling.md`.
* Step 2 must generate `02_train_baseline.py` and pass the smoke test.
* Step 2 must generate **both** `best.ckpt` and `last.ckpt`.
* Step 3 final check must satisfy all conditions to be considered ready.

### SHOULD

* Save `df_meta.parquet` and split manifests (train/valid) under `common/` to improve data management and reproducibility.
* For `stratified` split, document the distribution checks and acceptable tolerance.
* Keep Step 0–2 test reports under `_tests/` as a persistent validation record.
