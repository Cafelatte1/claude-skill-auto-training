---
name: progressive-training
description: A manual loop skill that, after a baseline, isolates training into a run-scoped workspace and iteratively trains versioned models, accumulates per-version evaluation/analysis results, and—when the loop ends—compares all versions in the run (plus external models) under the same protocol to produce a run report. Throughout execution, user-defined rules take absolute priority, and on each iteration only a single low-risk improvement action derived from analysis is applied to the next version.
---

## When to use

- Use when the user explicitly enters **`/progressive-training`**, or when `build-baseline` has successfully completed and triggers it.
- Use when you want to progressively improve a model/pipeline in **version steps (v001, v002, …)** after a baseline.
- Use when you want to isolate execution into a run and accumulate version scripts/artifacts/reports only within that run.
- Use when, after the loop, you want to compare **all versions in the current run** and external models with the same protocol to produce a **leaderboard and final report**.

---

## Workflow

### Common

- When running, this skill **must** prioritize **`./user_defined_rules.json`** and **`./user_instruction.md`** above all else.
- Items specified in those two files are treated as **user-defined constraints** and must not be changed arbitrarily across training/evaluation/analysis/next-version authoring.
- Both files must be in the same directory as this SKILL.md. If missing, assume there are **no user-defined constraints**.
- If any change described in `action-extract` conflicts with `user_defined_rules.json`, **`user_defined_rules.json` always wins**.
- Loop iteration count/termination is controlled by `user_defined_rules.json.loop_config`. When `max_versions` is reached, end the loop and run **Step 7 (Final report)** exactly once.
- `cur` is managed as an **integer**. Version naming always uses `version_id = v{cur:03d}`.
  - Example: `cur=1 → v001`, `cur=12 → v012`

#### dataset_id locking rule

- Set `dataset_id` by the following priority, decide it once in **Step 0**, and keep it fixed until the loop ends:
  1. `dataset_id` passed from `build-baseline` upon completion
  2. `user_defined_rules.json.dataset_config.dataset_id`

#### run_id locking rule (core)

- Generate **run_id once** at skill start, and keep it fixed until the loop ends.
- `run_id = YYYY-MM-DD_HH-MM_{counter}`
- If a `Runs/{run_id}` already exists for the same `YYYY-MM-DD_HH-MM`, increment `counter` as `01, 02, 03, ...`
- `run_id` is used in these two areas:
  - **Version script folder (project root):** `./Runs/{run_id}/02_train_v###.py`
  - **Version artifacts/reports (under Models):**
    - `Models/Runs/{run_id}/Artifacts/v###/...`
    - `Models/Runs/{run_id}/Report/...` (no yyyy-mm-dd folder)

#### Path rules (MUST)

- Baseline script: `./02_train_baseline.py` (reference/copy source)
- Baseline artifacts: `Models/Artifacts/Baseline/...` (read-only reference)
- External model root: `Models/External/{model_path}/...` (read-only reference)
- Run script folder: `./Runs/{run_id}/`
  - `./Runs/{run_id}/02_train_v{cur:03d}.py`
  - `./Runs/{run_id}/03_eval.py`
  - `./Runs/{run_id}/04_final_report.py`
- Run artifact root:
  - `Models/Runs/{run_id}/Artifacts/v{cur:03d}/...`
- Run report root:
  - `Models/Runs/{run_id}/Report/...`

#### External model evaluation cache rule (recommended standard)

- Cache external model evaluation results **per run**:
  - `Models/Runs/{run_id}/Artifacts/External/{external_id}/eval_score.json`
  - (Optional) `Models/Runs/{run_id}/Artifacts/External/{external_id}/infer_samples/...`
- By default, `external_id` is a safe-string normalization of `{model_path}`.
  - Example: `Models/External/glm-ocr` → `external_id=glm-ocr`

Example:

```json
{
  "seed": 42,
  "loop_config": { "max_versions": 10 },
  "dataset_config": { "dataset_id": "v1" },
  "training_config": { "epochs": 3, "batch_size": 32 },
  "eval_config": {
    "primary_metric": "loss",
    "metrics": [
      {"name": "loss", "higher_is_better": false}
    ],
    "infer_samples": 10
  }
}
```

---

## External model evaluation interface (contract)

### External model folder rules

* External model folder: `Models/External/{model_path}/`
* `inference_sample.py` (optional):

  * An example implementation for “per-sample inference” of the external model.
  * If present, **use it as a reference to write/generate `evaluation.py`**.
* `evaluation.py` (recommended/standard):

  * An executable entrypoint that performs:

    * full validation evaluation
    * infer_samples generation
    * `eval_score.json` saving
  * This skill delegates external evaluation not to a branch inside `03_eval.py`, but to **running External’s `evaluation.py`**.

### `evaluation.py` inputs (standard)

`evaluation.py` must accept the following via **CLI args or environment variables**, but this skill uses **CLI args** as the standard.

Required inputs (minimum):

* `--dataset_id`
* `--run_id`
* `--external_id`
* `--output_dir`

  * Default: `Models/Runs/{run_id}/Artifacts/External/{external_id}`
* `--seed`
* `--infer_samples` (int, can be 0)

Optional inputs:

* `--metrics` (comma-separated)
* `--primary_metric`
* External-model options like `--device`, `--precision`, etc. (if applicable)

### `evaluation.py` outputs (standard)

Under `output_dir`, it must create:

* `eval_score.json` (MUST, schema identical to Step 2 per-version evaluation)
* (Optional) `infer_samples/{index}/...`
* (Recommended) `eval_notes.md` or `spec.json` (records loading/inference/preprocess assumptions)

---

## Step 0) Initialize run + loop state

Goal:

* Generate `run_id` and create the run folder structure
* Lock `dataset_id`, determine `cur`, prepare the initial v001 script
* Convert the baseline into a “ready-to-train” run-scoped state (without performance-driven changes)

Inputs:

* `./user_defined_rules.json`, `./user_instruction.md` (highest priority)
* `dataset/prepdata/{dataset_id}/stats.json`
* `./02_train_baseline.py`
* `Models/Artifacts/Baseline/spec.json` (if present)
* `Models/Artifacts/Baseline/_tests/baseline_smoke_test_report.md` (if present)
* (Optional) query parameters like `/progressive-training?...&dataset_id=...`

Work:

1. **Create run_id and folders (MUST)**

   * Create:

     * `./Runs/{run_id}/`
     * `Models/Runs/{run_id}/Artifacts/`
     * `Models/Runs/{run_id}/Report/`

2. **Lock dataset_id (MUST)**

   * Decide once using the common rule and keep fixed until loop ends.

3. **Determine cur (MUST)**

   * If one or more `./Runs/{run_id}/02_train_v###.py` exist:

     * Set `cur` to the largest existing version, and start **Step 1 (Train)** from that `cur`.
   * If none exist:

     * Set `cur = 1` (`v001`) and perform 4)–5).

4. **Baseline check**

   * Goal: ensure “run-based path correctness” (do not introduce performance-motivated changes)
   * Minimum checks:

     * Data paths follow `dataset/prepdata/{dataset_id}/...`
     * `DATASET_ID` / global constants match the rules
     * (Important) artifact saving uses the baseline fixed path (`Models/Artifacts/Baseline/`)
     * reflect `optimal_batch_size` found in the smoke test if possible
   * Analyze training process, imported modules, user constraints, etc.; if issues are found, record fixes.
   * Required patch log:

     * `Models/Artifacts/Baseline/_tests/baseline_patch_report.md`

5. **Prepare baseline → v001 script (MUST)**

   * Copy `./02_train_baseline.py` to `./Runs/{run_id}/02_train_v001.py`.
   * Apply any fixes from `Models/Artifacts/Baseline/_tests/baseline_patch_report.md` to `02_train_v001.py`.
   * v001 script must adopt run-based artifact paths (required):

     * `ARTIFACT_DIR = Models/Runs/{run_id}/Artifacts/v001`
     * logs, ckpt, additional_assets, spec.json, _tests must all be created under v001.

Required outputs:

* `./Runs/{run_id}/02_train_v001.py`
* `Models/Runs/{run_id}/run_spec.json`

---

## Step 1) Train

Goal:

* Run training and create artifacts for version `v{cur:03d}`.
* Training framework logging is allowed, but **do not create evaluation outputs** here.

Inputs:

* `user_defined_rules.json` (highest priority)
* `./Runs/{run_id}/02_train_v{cur:03d}.py`

Work:

* Apply `user_defined_rules.json.training_config` and user constraints **as the highest-priority training settings**.
* Run training and save artifacts under:

  * `Models/Runs/{run_id}/Artifacts/v{cur:03d}/...`
* `spec.json` must be created for reproducibility.
* Save pre/post-processing assets.
* Keep checkpoints: `best.ckpt`, `last.ckpt`.

#### Training failure (error/exception) handling rules

* On exception, **reset** (delete and recreate) the version artifact directory.
* Summarize the root cause from logs/stack traces, then retry by modifying only the **same version** script (`v{cur:03d}`) (version skipping is forbidden).
* (Recommended) retry history:

  * `Models/Runs/{run_id}/Artifacts/v{cur:03d}/_tests/train_retry_report.md`

Required outputs:

* `Models/Runs/{run_id}/Artifacts/v{cur:03d}/best.ckpt`
* `Models/Runs/{run_id}/Artifacts/v{cur:03d}/last.ckpt`
* `Models/Runs/{run_id}/Artifacts/v{cur:03d}/spec.json`
* `Models/Runs/{run_id}/Artifacts/v{cur:03d}/additional_assets/`
* `Models/Runs/{run_id}/Artifacts/v{cur:03d}/logs/version_#/metrics.csv`
* `Models/Runs/{run_id}/Artifacts/v{cur:03d}/logs/version_#/hparams.yaml`

---

## Step 2) Evaluation

Goal:

* Evaluate using only the validation set with the trained model (`best.ckpt`).
* Produce the current version’s final metric scores (`eval_score.json`) and inspection materials (`infer_samples/`).
* When `03_eval.py` is created for the first time, **pre-evaluate external models** to establish run-level comparison baselines.
* External model evaluation is not a branch inside `03_eval.py`; it is delegated to **running External’s `evaluation.py`**.

Inputs:

* `user_defined_rules.json` (highest priority)
* `./Runs/{run_id}/02_train_v{cur:03d}.py`
* `Models/Runs/{run_id}/Artifacts/v{cur:03d}/best.ckpt`

Work:

### 1) Prepare evaluation script (MUST)

* If `./Runs/{run_id}/03_eval.py` does not exist, create it.
* `03_eval.py` must support at least:

  * run_id, dataset_id, version_id → execute evaluation
  * evaluate current version model + save `eval_score.json`
  * generate `infer_samples/`
  * (on first creation or when cache is missing) evaluate external models by **calling External’s `evaluation.py`** and saving caches

### 2) Create current version eval_score.json (MUST)

* Compute metrics using the full validation set.
* If `eval_config.metrics` is missing, use only the loss value used during training.
* Save as a **single JSON object** per version (`created_at` is ISO-8601).
* Path:

  * `Models/Runs/{run_id}/Artifacts/v{cur:03d}/eval_score.json`
* Schema (MUST):

```json
{
  "dataset_id": "...",
  "created_at": "ISO-8601",
  "num_samples": 0,
  "metrics": [
    {"name": "...", "value": 0.0, "higher_is_better": false}
  ],
  "latency_ms": 0.0
}
```

### 3) Generate infer_samples (MUST)

* Seed-sample up to `infer_samples` items from the validation set, run inference, and save.
* Path:

  * `Models/Runs/{run_id}/Artifacts/v{cur:03d}/infer_samples/{index}/...`
* Each `{index}` folder MUST contain:

  * `input.*`, `output.*`, `meta.json`
* Optional:

  * `gt.*`, `viz.*`, `summary.*`

### 4) External model pre-evaluation (MUST on first creation / SHOULD when cache is missing)

#### Trigger conditions

* MUST run if `./Runs/{run_id}/03_eval.py` was newly created in this Step.
* OR (recommended) run if there is no cache at `Models/Runs/{run_id}/Artifacts/External/*/eval_score.json`.

#### Targets

* Each external model folder under `Models/External/*`

#### Execution rules (core)

Apply the following priority per external model folder:

1. **Skip if cache exists (MUST)**

* If `Models/Runs/{run_id}/Artifacts/External/{external_id}/eval_score.json` exists, skip evaluation.

2. **If evaluation.py exists, run it (MUST)**

* If `Models/External/{model_path}/evaluation.py` exists, run it to create the cache.

3. **If evaluation.py is missing but inference_sample.py exists: generate evaluation.py then run (MUST)**

* If `Models/External/{model_path}/inference_sample.py` exists:

  * Generate (or enhance a template for) `Models/External/{model_path}/evaluation.py` using it as reference.
  * Run the generated `evaluation.py` to create the cache.

4. **If neither exists: mark as not evaluable (MUST)**

* Record “not evaluable (missing entrypoint)” in `eval_notes.md` and warn in Step 7 that the model may be excluded from the leaderboard.

#### Cache outputs (recommended standard)

* `Models/Runs/{run_id}/Artifacts/External/{external_id}/eval_score.json`
* (Optional) `Models/Runs/{run_id}/Artifacts/External/{external_id}/infer_samples/...`
* Record assumptions/notes (recommended) in one of:

  * `Models/Runs/{run_id}/Artifacts/External/{external_id}/eval_notes.md`
  * or `.../spec.json`

Required outputs:

* `./Runs/{run_id}/03_eval.py`
* `Models/Runs/{run_id}/Artifacts/v{cur:03d}/eval_score.json`
* `Models/Runs/{run_id}/Artifacts/v{cur:03d}/infer_samples/{index}/...` (up to infer_samples)

---

## Step 3) Result analysis

Inputs:

* `Models/Runs/{run_id}/Artifacts/v{cur:03d}/eval_score.json`
* `Models/Runs/{run_id}/Artifacts/v{cur:03d}/infer_samples/{index}/...`
* (Optional) `Models/Runs/{run_id}/Artifacts/v{cur:03d}/logs/version_*/metrics.csv`

Work:

* Summarize metric deltas vs the previous version.
* Identify plateaus/regressions and trade-offs.
* Using `infer_samples` as evidence, summarize failure patterns and derive causal hypotheses.

Required outputs:

* `Models/Runs/{run_id}/Artifacts/v{cur:03d}/result-analysis.md`
* Minimum structure (MUST):

  * `Summary`
  * `Metric deltas (v{cur-1:03d} → v{cur:03d})`
  * `Evidence`
  * `Hypotheses`
  * `Next focus candidates`

---

## Step 4) Action extract

Inputs:

* `Models/Runs/{run_id}/Artifacts/v{cur:03d}/result-analysis.md`

Work:

* Select **exactly one** improvement action to apply to the next version.
* Selection criteria:

  * low implementation risk
  * high likelihood of improving performance
  * narrow scope so cause/effect tracking is feasible
* If the action conflicts with constraints in `user_defined_rules.training_config`:

  * revise the action to satisfy constraints, or discard it and choose the next best alternative.

Required outputs:

* `Models/Runs/{run_id}/Artifacts/v{cur:03d}/action-extract.md`
* Minimum structure (MUST):

  * `Selected Action (one)`: `Goal`, `Change`, `Where`, `Expected impact`, `Risk`, `Verification`
  * `Deferred Candidates` (optional)

---

## Step 5) Loop termination check

Work:

* If `cur >= user_defined_rules.loop_config.max_versions`, end the loop and run Step 7 once.
* Otherwise proceed to Step 6.

---

## Step 6) Next version authoring

Inputs:

* `./Runs/{run_id}/02_train_v{cur:03d}.py`
* `Models/Runs/{run_id}/Artifacts/v{cur:03d}/action-extract.md`

Work:

* Copy `./Runs/{run_id}/02_train_v{cur:03d}.py` to `./Runs/{run_id}/02_train_v{cur+1:03d}.py`.
* Apply only the change described in `Selected Action`.
* Update artifact output paths to match v{cur+1:03d}:

  * `Models/Runs/{run_id}/Artifacts/v{cur+1:03d}/...`
* Add a change summary at the top of the file.

Outputs:

* `./Runs/{run_id}/02_train_v{cur+1:03d}.py`
* Increment `cur += 1`, then loop back to Step 1.

---

## Step 7) Final report after loop ends

Inputs:

* `user_defined_rules.json` (highest priority)
* All `v###` under `Models/Runs/{run_id}/Artifacts/`
* `./Runs/{run_id}/02_train_v###.py`
* (Optional) external caches: `Models/Runs/{run_id}/Artifacts/External/*/eval_score.json`

Work:

1. **Fix report location (MUST)**

* Use `Models/Runs/{run_id}/Report/` (do not create yyyy-mm-dd folders).

2. **Create and run 04_final_report.py (MUST)**

* Script location: `./Runs/{run_id}/04_final_report.py`
* Outputs:

  * `Models/Runs/{run_id}/Report/leaderboard.csv`
  * `Models/Runs/{run_id}/Report/final-analysis.md`

3. **leaderboard.csv generation rules (MUST)**

* Collect version evals:

  * `Models/Runs/{run_id}/Artifacts/v###/eval_score.json`
* Collect external evals:

  * Prefer caches: `Models/Runs/{run_id}/Artifacts/External/*/eval_score.json`
  * If caches are missing, you may run supplemental evaluation via `./Runs/{run_id}/03_eval.py` (recommended),
    but it must do so by calling External’s `evaluation.py`.
* Flatten all items in `eval_score.json.metrics` into columns:

  * Column name rule: `metric__{name}`
  * If name collides, uniquify as `metric__{name}__{k}`
* Allowed columns only (MUST):

  * `version_id`, `dataset_id`, `created_at`, `latency_ms`, `artifact_relpath`, `metric__{name}...`
* `artifact_relpath` rule:

  * Write paths relative to the report folder (`Models/Runs/{run_id}/Report/`).
  * Examples:

    * Trained version: `../Artifacts/v010/best.ckpt`
    * External model: `../Artifacts/External/glm-ocr/` (folder path is allowed)

4. **Sort/select by primary metric (MUST)**

* Primary metric name: `user_defined_rules.json.eval_config.primary_metric`
* Sort direction follows that metric’s `higher_is_better`.
* Versions missing the primary metric go to the bottom and must be warned about in `final-analysis.md`.
* Best version = rank #1.

5. **Copy best artifacts (MUST)**

* Copy the best version’s artifacts to:

  * `Models/Runs/{run_id}/Report/Artifacts/{best_version_id}/{best.ckpt, spec.json, additional_assets/...}`

6. **final-analysis.md content (MUST)**

* For the top 3 leaderboard entries (including best):

  * analyze which implementation changes contributed to rank improvement
* Recommended references:

  * `./Runs/{run_id}/02_train_v###.py`
  * `Models/Runs/{run_id}/Artifacts/v###/action-extract.md`
  * `Models/Runs/{run_id}/Artifacts/v###/result-analysis.md`

Required outputs:

* `./Runs/{run_id}/04_final_report.py`
* `Models/Runs/{run_id}/Report/final-analysis.md`
* `Models/Runs/{run_id}/Report/leaderboard.csv`
* `Models/Runs/{run_id}/Report/Artifacts/{best_version_id}/...`

Example (`leaderboard.csv`, 3 rows):

```csv
version_id,dataset_id,created_at,latency_ms,artifact_relpath,metric__loss,metric__wer
v008,v1,2026-02-23T22:01:10,87,../Artifacts/v008/best.ckpt,0.142,0.091
v009,v1,2026-02-23T22:01:10,86,../Artifacts/v009/best.ckpt,0.137,0.095
v010,v1,2026-02-23T22:01:10,85,../Artifacts/v010/best.ckpt,0.131,0.089
```

---

## Directory structure

```text
./Runs/{run_id}/
├── 02_train_v001.py
├── 02_train_v002.py
├── ...
├── 03_eval.py
└── 04_final_report.py

Models/
├── Artifacts/
│   └── Baseline/...
├── External/
│   └── {model_path}/
│       ├── inference_sample.py         (optional)
│       └── evaluation.py               (recommended / generated)
└── Runs/
    └── {run_id}/
        ├── Artifacts/
        │   ├── v001/...
        │   ├── v002/...
        │   └── External/
        │       └── {external_id}/
        │           ├── eval_score.json
        │           └── infer_samples/{index}/...
        └── Report/
            ├── leaderboard.csv
            ├── final-analysis.md
            └── Artifacts/{best_version_id}/...
```

---

## Rules

### MUST

* Run only when the user enters `/progressive-training` or it is auto-triggered by `build-baseline` Step 3.
* On execution, **read and apply `./user_defined_rules.json` with the highest priority**.
* Generate `run_id = YYYY-MM-DD_HH-MM_{counter}` once in Step 0 and keep it fixed until the loop ends.
* Version scripts must be saved under `./Runs/{run_id}/02_train_v###.py`.
* Version artifacts must be saved under `Models/Runs/{run_id}/Artifacts/v###/`.
* Reports must be saved under `Models/Runs/{run_id}/Report/` (do not create date folders).
* Baseline is read-only and must use `./02_train_baseline.py` and `Models/Artifacts/Baseline/`.
* External models are read-only and must use `Models/External/{model_path}/...`.
* `02_train_v###.py` must not create/modify `eval_score.json` or `infer_samples/`.
* `eval_score.json` must be saved as a **single JSON object**.
* `infer_samples` must be sampled from the validation set up to `infer_samples`, using `user_defined_rules.json.seed` (default 42 if missing).
* Next-version changes must apply **only one Selected Action** derived from `result-analysis.md` → `action-extract.md`.
* For external model comparison in the final report, prefer run caches under `Artifacts/External/...`.
* External model evaluation must be performed by **running External’s `evaluation.py`**, not by branching inside `03_eval.py`.
* If an external model folder has `inference_sample.py` but no `evaluation.py`, you must generate (or enhance) `evaluation.py` and run it to create `eval_score.json`.

### SHOULD

* Store run metadata (run_id, dataset_id, seed, max_versions, external model list) in `Models/Runs/{run_id}/run_spec.json`.
* For external evaluations, record assumptions and the executed command in `Models/Runs/{run_id}/Artifacts/External/{external_id}/eval_notes.md`.
* Include sample identifiers/conditions/config summary/seed in `infer_samples/{index}/meta.json`.
