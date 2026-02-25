---
name: build-baseline
description: A skill that automatically explores raw source data, infers sample-level and label/path rules on its own, documents those rules as a fixed reference, and produces preprocessing data plus baseline training code in an immediately runnable state—passing a smoke test. Throughout execution, it prioritizes user-defined rules above all else and generates/verifies task-specific data structures and validation rules under a consistent standard.
---

## When to use

- Use **only** when the user explicitly enters **`/build-baseline`**.
- Use when there are multiple source folders under `dataset/rawdata/` and you want the model to explore them directly, document parsing rules, and then generate a standard pipeline/baseline codebase.
- Use when you want to write the baseline training script (`02_train_baseline.py`) according to a standard rule set and pass a smoke test.
- Use when you want to manage validation splits with advanced strategies (e.g., stratified/group-based) rather than simple random splits.
- Use when you want to align data/loader/schema to `task_format.yaml` to keep task-specific data structures consistent.

---

## Workflow

### Common

- When running, this skill **must** prioritize **`./user_defined_rules.json`** and **`./user_instruction.md`** above all else.
- Items specified in `user_defined_rules.json` and `user_instruction.md` are treated as **user-defined constraints** and must not be changed arbitrarily across the entire process (training/evaluation/analysis/next-version writing).
- `user_defined_rules.json` and `./user_instruction.md` must be in the same directory as this SKILL.md. If missing, assume there are **no user-defined constraints**.
- `task_format.yaml` must exist at `./task_format.yaml` in the same directory as this SKILL.md. If missing, **stop** this skill.
- Steps 0–2 must pass tests after writing code before proceeding to the next step.
- Any operation involving “randomness” (sampling, split, shuffle, subset extraction, smoke-test sample selection, etc.) must be reproducible using the root `seed` value. If `seed` is missing, use 42.
- In this skill, `{dataset_id}` always refers to `user_defined_rules.dataset_config.dataset_id`.
  - Example: `dataset_id = "v1"` → `dataset/prepdata/v1/...`

#### Script/Artifact save rules (this skill)

- Save the data pipeline script at the project root:
  - `./01_data_pipeline.py`
- Save the baseline training script at the project root:
  - `./02_train_baseline.py`
- Save baseline artifacts under the fixed path:
  - `Models/Artifacts/Baseline/...`

#### progressive-training integration rules (reference)

- `build-baseline` does not create a run-scoped (dataset_id loop) folder.
- The follow-up skill `progressive-training` generates `run_id = YYYY-MM-DD_HH-MM` once at start, and uses:
  - Version training script: `./Runs/{run_id}/02_train_v###.py`
  - Version artifacts: `Models/Runs/{run_id}/Artifacts/v###/...`
  - Report: `Models/Runs/{run_id}/Report/...`
  `build-baseline` only verifies that baseline outputs are valid according to this convention.

Example excerpt of `user_defined_rules.json`:

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

### Step 0) Rawdata profiling doc creation & test (Rawdata profiling)

Role:

* Directly explore every source folder under `dataset/rawdata/` and analyze internal files/structure to fix per-source parsing rules and the sample-unit definition as documentation.
* If `dataset_config.task_type` is missing, define it via profiling and provide guidelines so the data can be structured to match `dataset_config.task_type`.
* All later data-pipeline steps must read this document and follow the same rules.
* Proceed only after the document is written and minimum validation (sample-load test) passes.
* If `profiling.md` already exists, skip this step.

Inputs:

* `user_defined_rules.json` (highest priority)

  * `seed` (default 42 if missing)
  * `dataset_config.task_type` (optional)
  * `dataset_config.max_profile_samples` (optional)
* `dataset/rawdata/*` (the entire folder, not a single source)

Work (auto exploration + documentation):

* Identify profiling target directories under `dataset/rawdata/`.

  * Include: `dataset/rawdata/{source}/...`
  * Exclude: documentation files like `dataset/rawdata/profiling.md`
* For each `{source}`, do the following and consolidate results into a single document.
* Cap the total number of samples actually loaded/scanned/sampled during profiling to `max_profile_samples` (default 1,000). Target selection must be reproducible via `seed`.

Minimum required structure for `dataset/rawdata/profiling.md` (MUST):

* `Overview` (overall rawdata overview, source list, size estimates)
* `Sources`

  * A section per source: `## Source: {source_name}`
* Required contents per source section:

  * `Sample unit definition`
  * `Paths & naming rules`
  * `Label schema`
  * `Edge cases`
  * `Extraction rules`
  * `Sanity checks`

Work (test: minimum load validation):

* For each source, load N random samples (recommended 10) and verify:

  * Sample-unit parsing works
  * Asset/label matching rule consistency (if applicable)
  * Label schema parsing works
* Sample selection must be reproducible via `seed`.
* Record PASS/FAIL at the bottom of each source section.

Required outputs:

* `dataset/rawdata/profiling.md`

---

### Step 1) Data pipeline code & test (Data pipeline)

Role:

* Read `dataset/rawdata/profiling.md` from Step 0 and parse/normalize rawdata to generate prepdata according to those rules.
* Create `dataset/prepdata/{dataset_id}/...` in the structure recommended by `./task_format.yaml` under `supported_tasks[{task_type}]`.
* Proceed only after code is written and tests pass.
* If `01_data_pipeline.py` already exists:

  * Delete that script and `dataset/prepdata/{dataset_id}/...`, then proceed.

Inputs:

* `user_defined_rules.json` (highest priority)

  * `seed` (default 42 if missing)
  * `dataset_config.dataset_id` → `{dataset_id}`
  * `dataset_config.task_type`
  * `dataset_config.valid_ratio` (use default if missing)
  * `dataset_config.split_strategy` (default `random` if missing)
  * `dataset_config.stratify_key` (default `category` if missing)
  * `dataset_config.group_key` (default `group_id` if missing)
  * `dataset_config.max_train_samples` (optional)
* `task_format.yaml` (required)
* `dataset/rawdata/profiling.md` (required)
* `dataset/rawdata/*` (all sources included in profiling.md)

Work (code writing):

* Apply profiling document rules

  * Always parse rawdata based on `dataset/rawdata/profiling.md`.
  * Do not invent new “guessed parsing logic” that conflicts with profiling.md.
  * If needed, update profiling.md first, then adjust the pipeline.

* Create meta table: `df_meta`

  * Build `df_meta` according to profiling.md `Extraction rules`.
  * Include a `source` column so multiple sources can be managed together.
  * If `dataset_config.max_train_samples` exists: first determine the full pool of trainable samples that pass preprocessing/validation, then reproducibly sample up to `max_train_samples` from that pool using `seed` to form `df_meta`.
  * If there is no extra user instruction, apply uniform random sampling by source.
  * Minimum columns:

    * `sample_id`, `source`, `category` (null if missing), `asset_paths`
  * Recommended columns:

    * `group_id`, `lang`, `length`, `width`, `height`, `timestamp`, `label_summary`, `hash`
  * Recommended save:

    * `dataset/prepdata/{dataset_id}/common/df_meta.parquet`

* Validation split

  * Split must be reproducible via `seed`.
  * Support the following via `split_strategy`:

    * `random`
    * `stratified` (default `category`)
    * `group` (default `group_id`)
  * Recommended: save split results as manifests:

    * `dataset/prepdata/{dataset_id}/common/train_manifest.parquet`
    * `dataset/prepdata/{dataset_id}/common/valid_manifest.parquet`

* Generate prepdata (must follow `task_format.yaml`)

  * Generate outputs following `task_format.yaml.supported_tasks[{task_type}]` `split_layout` and `index_schema`:

    * `dataset/prepdata/{dataset_id}/stats.json`
    * `dataset/prepdata/{dataset_id}/train/`
    * `dataset/prepdata/{dataset_id}/valid/`
    * `dataset/prepdata/{dataset_id}/common/`

Work (tests):

* Run consistency checks based on `task_format.yaml.validation.checks`.
* Additional checks by split strategy:

  * `group`: no group leakage
  * `stratified`: only record a distribution summary in the report
* Required files/folders:

  * `stats.json`, `train/`, `valid/`, `common/`
  * `train.jsonl|train.parquet`, `valid.jsonl|valid.parquet`

Required outputs:

* Data pipeline code: `./01_data_pipeline.py`
* `dataset/prepdata/{dataset_id}/stats.json`
* `dataset/prepdata/{dataset_id}/train/`
* `dataset/prepdata/{dataset_id}/valid/`
* `dataset/prepdata/{dataset_id}/common/`
* (Recommended) `dataset/prepdata/{dataset_id}/common/df_meta.parquet`
* (Recommended) `dataset/prepdata/{dataset_id}/_tests/pipeline_test_report.md`

---

### Step 2) Baseline training code & test (Baseline)

Role:

* Write `./02_train_baseline.py`, and verify training/validation runs correctly with only minimal samples (train/valid 100 each) for debug/smoke-test purposes.
* Proceed only after code is written and the smoke test passes.
* If `02_train_baseline.py` already exists:

  * Delete that script and `Models/Artifacts/Baseline/...`, then proceed.

Inputs:

* `user_defined_rules.json` (highest priority)

  * `seed` (default 42 if missing)
  * `dataset_config.dataset_id`
  * `dataset_config.task_type`
  * `baseline_smoke_test.epoch` (optional, default 1 if missing)
  * `baseline_smoke_test.train_samples` (default 100 if missing)
  * `baseline_smoke_test.valid_samples` (default 100 if missing)
  * `baseline_smoke_test.min_batch_size` (optional)
  * `baseline_smoke_test.max_batch_size` (optional)
  * (Optional) `training_config.*`, `model_config.*`
* `task_format.yaml` (required)
* `dataset/prepdata/{dataset_id}/train/`
* `dataset/prepdata/{dataset_id}/valid/`
* `dataset/prepdata/{dataset_id}/common/` (optional, depending on task)

Work (code writing):

#### Generate baseline training script

* Write `./02_train_baseline.py`.
* `./02_train_baseline.py` must include a global variable `DATASET_ID: str`, and it must equal `{dataset_id}` (= `dataset_config.dataset_id`).
* Data loading must treat `dataset/prepdata/{dataset_id}` as the single source of truth and follow the structure/schema in `task_format.yaml.supported_tasks[{task_type}]`.
* Smoke-test subset selection must be reproducible via `seed`.

#### Baseline artifact/log structure (MUST)

* Even for a smoke test, outputs **must** be generated with the fixed structure below:

  * `Models/Artifacts/Baseline/spec.json`
  * `Models/Artifacts/Baseline/additional_assets/`
  * `Models/Artifacts/Baseline/best.ckpt`
  * `Models/Artifacts/Baseline/last.ckpt`
  * `Models/Artifacts/Baseline/logs/version_0/{metrics.csv,hparams.yaml}`
  * `Models/Artifacts/Baseline/_tests/baseline_smoke_test_report.md`

* When `progressive-training` starts a run from this baseline:

  * It copies `./02_train_baseline.py` to `./Runs/{run_id}/02_train_v###.py`.
  * It references baseline artifacts in-place from `Models/Artifacts/Baseline/`.

Work (search for optimal batch size):

* From `baseline_smoke_test.min_batch_size ~ baseline_smoke_test.max_batch_size`, construct power-of-two batch size candidates and search:

  * Run the batch-size search for only 10 steps.
  * Candidate rule: `bs ∈ { 2^k | min_batch_size ≤ 2^k ≤ max_batch_size }`
  * Start from the largest batch size; if it succeeds, stop immediately.
  * Choose the largest successful batch size as `optimal_batch_size`.
  * Success criterion:

    * PASS if forward+backward completes for 10 steps without OOM/RuntimeError.
  * Always record results in `_tests/baseline_smoke_test_report.md`.

Work (expected_* validation):

* Treat all keys in `user_defined_rules.model_config` with the `expected_` prefix as validation contracts.

  * Examples: `expected_stride`, `expected_channels`, `expected_out_shape`, `expected_dtype`, ...

* Prefer deriving observations via a single dummy forward pass (batch=1, minimal input).

* Minimal input must satisfy model input constraints in `model_config` (e.g., H_TARGET, W_BUCKETS).

* Validation procedure (MUST):

  1. Collect all `expected_*` items from `model_config` into `expected_map`.
  2. If `expected_map` is empty, **SKIP** this validation stage (not a FAIL).
  3. Build `observed_map` from “verifiable actual values” produced/loaded by the current training code.

     * `observed_map` must use the same suffix names as `expected_*`.

       * Example: `expected_stride` ↔ `observed_stride`, `expected_channels` ↔ `observed_channels`
  4. Decide PASS/FAIL per `expected_*` (apply type rules below).

* Type-specific comparison rules (MUST):

  * Scalars (number/string/bool): PASS only if `observed == expected`
  * Lists/tuples: PASS only if length/items/order all match
  * Dicts: PASS only if all keys exist and values match
  * Floats: if `model_config.expected_tolerance` (optional) is missing, require exact match; otherwise require `abs(observed-expected) <= tolerance`

* Handling non-observable values (MUST):

  * If an `observed_*` value cannot be produced, mark as FAIL.
  * Exception: if `model_config.expected_allow_missing_keys: true`, record as `SKIP(allow_missing)` and do not fail overall.

* Failure behavior (MUST):

  * If any FAIL occurs, stop immediately without running training (batch search/smoke run).
  * Re-run only after fixing configs (`expected_*` or related) or code (observation logic).

* Report content (MUST):

  * Include the following in `Models/Artifacts/Baseline/_tests/baseline_smoke_test_report.md`:

    * `expected_map`
    * `observed_map`
    * `validation_table` (key, expected, observed, status(PASS/FAIL/SKIP), reason)
    * `overall_result: PASS|FAIL`

Work (smoke run):

* The smoke test verifies: “the training script runs without error and produces outputs in the artifact path.”
* Run the smoke training only with `optimal_batch_size` selected above.
* Run with train N + valid M samples.
* Minimum pass conditions:

  * Training starts and finishes without errors
  * `best.ckpt` and `last.ckpt` are created
  * `spec.json`, `additional_assets/`, and logs are created

#### Training failure (error/exception) handling rules

* If an exception occurs while running `./02_train_baseline.py`, **reset** (delete and recreate) the baseline artifact directory:

  * Target: `Models/Artifacts/Baseline/` (entire directory)
* After reset, summarize the cause based on logs/stack traces and apply a **code fix**:

  * Target: `./02_train_baseline.py` (and local modules imported by it, if needed)
  * User-defined constraints (`user_defined_rules.json`, `user_instruction.md`) always take priority; bypassing constraints is forbidden.
* After fixes, re-run training with the same **Baseline**.
* (Recommended) include retry history in:

  * `Models/Artifacts/Baseline/_tests/baseline_smoke_test_report.md`

    * Minimum: `attempt`, `error_summary`, `root_cause_hypothesis`, `code_change_summary`, `status(PASS/FAIL)`

Required outputs:

* `./02_train_baseline.py`
* `Models/Artifacts/Baseline/spec.json`
* `Models/Artifacts/Baseline/best.ckpt`
* `Models/Artifacts/Baseline/last.ckpt`
* `Models/Artifacts/Baseline/additional_assets/`
* `Models/Artifacts/Baseline/logs/version_0/{metrics.csv,hparams.yaml}`
* `Models/Artifacts/Baseline/_tests/baseline_smoke_test_report.md`

---

### Step 3) Final check

Role:

* Perform a final check that all outputs from Steps 0–2 exist and tests passed.
* If the final check passes, trigger the next skill: **`progressive-training`**.

  * `progressive-training` generates `run_id` at runtime (`YYYY-MM-DD_HH-MM`) and uses:

    * Version scripts: `./Runs/{run_id}/02_train_v###.py`
    * Version artifacts: `Models/Runs/{run_id}/Artifacts/v###/...`
    * Report: `Models/Runs/{run_id}/Report/...`

Inputs:

* All outputs from Steps 0–2

Work:

* Verify via checklist:

  * `dataset/rawdata/profiling.md` exists and contains per-source PASS records
  * `dataset/prepdata/{dataset_id}/` contains `stats.json, train/, valid/, common/`
  * Step 1 test passed (confirm PASS if a report exists)
  * `./02_train_baseline.py` exists
  * `Models/Artifacts/Baseline/` contains `best.ckpt, last.ckpt, spec.json, logs/, additional_assets/, _tests/baseline_smoke_test_report.md`
  * Step 2 smoke test passed (confirm PASS if a report exists)

Result:

* If final check passes:

  * Reset Todo/Todos items
  * Read **`.claude/skills/progressive-training/SKILL.md`** and continue with `progressive-training`
  * Pass the dataset_id used in this skill into `progressive-training` unchanged
* If final check fails:

  * Write a detailed failure reason and end the work

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

./
├── 01_data_pipeline.py
└── 02_train_baseline.py

Models/
└── Artifacts/
    └── Baseline/
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
* Any operation involving randomness must use the root `seed` value; if missing, use 42.
* Control profiling scan/load limits via `user_defined_rules.dataset_config.max_profile_samples`.
* Control the training sample cap via `user_defined_rules.dataset_config.max_train_samples`, applied after fixing the pool of trainable samples that pass preprocessing/validation.
* Do not run if `task_format.yaml` is missing.
* Step 0 must auto-explore all source folders under `dataset/rawdata/` and write `dataset/rawdata/profiling.md` (skip only if profiling.md already exists).
* Step 1 must read `dataset/rawdata/profiling.md` and parse rawdata strictly by those rules.
* Step 2 must generate `./02_train_baseline.py` and pass the smoke test.
* Step 2 must **always** produce `best.ckpt` and `last.ckpt`.
* Baseline artifacts must use `Models/Artifacts/Baseline/`.
* Step 3 final check must satisfy all conditions to be considered ready.

### SHOULD

* Save `df_meta.parquet` and split manifests (train/valid) under `common/` to improve management and reproducibility.
* Keep test reports for Steps 0–2 under `_tests/` to preserve validation history.
