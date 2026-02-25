---
name: progressive-training
description: A manual loop skill that iteratively trains models in versioned increments after the baseline, evaluates and summarizes each version’s results, and applies exactly one low-risk improvement action to generate the next version’s training code. Throughout execution, it prioritizes user-defined rules above all else. When the loop ends, it re-evaluates all existing versions under the same evaluation protocol and saves a leaderboard and the best artifacts as a standardized report.
---

## When to use

* Use this skill when the user explicitly types **`/progressive-training`**, or after the `build-baseline` workflow has completed successfully.
* Use it to progressively improve the model/pipeline **by version** (`v001`, `v002`, …) after the baseline.
* Use it when you want to accumulate per-version training artifacts and evaluation results in a standard structure and repeatedly connect **analysis → action → next-version code authoring**.
* Use it when you want to re-evaluate **all currently existing versions** under the same protocol and generate a **leaderboard report** after the loop ends.

---

## Workflow

### Common

* On execution, this skill must reference **`./user_defined_rules.json`** and **`./user_instruction.md`** with the **highest priority**.
* Any items specified in these files are treated as **user-defined constraints** and must not be changed arbitrarily across training/evaluation/analysis/next-version authoring.
* Both files must be in the same directory as this SKILL.md. If missing, assume there are **no user-defined constraints**.
* If any changes in `action-extract` conflict with `user_defined_rules.json`, **`user_defined_rules.json` always wins**.
* Loop iterations and termination conditions are controlled by `user_defined_rules.json.loop_config`. When `max_versions` is reached, terminate the loop and run **Step 7 (Final report)** exactly once.
* Maintain `cur` as an **integer**. Always format version IDs as `version_id = v{cur:03d}`.

  * Example: `cur=1 → v001`, `cur=12 → v012`
* File/folder paths must always follow the `v{cur:03d}` convention.

  * Example: `02_train_v001.py`, `Models/Artifacts/v001/`
* Determine `dataset_id` by the following priority, and **fix it once in Step 0 and keep it constant until the loop ends**:

  1. `dataset_id` passed from a completed `build-baseline`
  2. `user_defined_rules.json.dataset_config.dataset_id`
* Record `dataset_id` in each version’s `Models/Artifacts/v{cur:03d}/eval_score.json.dataset_id`.
* `dataset_id` identifies the dataset (prepdata), while `v{cur:03d}` identifies the training version.

Example:

```json
{
  "seed": 42,
  "loop_config": {
    "max_versions": 10
  },
  "dataset_config": {
    "dataset_id": "v1"
  },
  "training_config": {
    "epochs": 3,
    "batch_size": 32
  },
  "eval_config": {
    "primary_metric": "loss",
    "metrics": ["..."],
    "infer_samples": 10
  }
}
```

---

## Step 0) Initialize loop state (Initialize loop state)

Purpose:

* Before starting the loop, **fix `dataset_id`**, decide the current version (`cur`), prepare the initial `v001`, and transition from baseline to versioned loop execution.

Inputs:

* `./user_defined_rules.json`, `./user_instruction.md` (highest priority)
* `dataset/prepdata/{dataset_id}/stats.json`
* Existing list of `02_train_v###.py` files
* `02_train_baseline.py`
* `Models/Artifacts/baseline/spec.json` (if present)
* `Models/Artifacts/baseline/_tests/baseline_smoke_test_report.md` (if present)
* (Optional) query parameter such as `/progressive-training?...&dataset_id=...`

Work:

1. **Fix dataset_id**

   * Determine and lock `dataset_id` once using the common priority rule and keep it fixed until the loop ends.

2. **Decide `cur`**

   * If one or more `02_train_v###.py` exist:

     * Set `cur` to the largest existing version number and start **Step 1 (Train)** from that `cur`.
   * If none exist:

     * Set `cur = 1` (`version_id = v001`) and perform initialization in step (3).

3. **Baseline audit & one-time patch**

   * Read `dataset/prepdata/{dataset_id}/stats.json` to confirm training sample count and `task_type`.
   * Read `Models/Artifacts/baseline/_tests/baseline_smoke_test_report.md` to find `optimal_batch_size`.
   * Verify:

     * data paths follow `dataset/prepdata/{dataset_id}/...`
     * global constants like `DATASET_ID`/`VERSION` match the convention
     * artifact save path follows `Models/Artifacts/baseline/`
     * log path follows `Models/Artifacts/baseline/logs/version_#/`
     * whether `optimal_batch_size` can be applied (apply if possible)
   * If issues are found, apply **only a minimal one-time patch**:

     * patch target: `02_train_baseline.py` (and imported local modules if needed)
     * user constraints (`user_defined_rules.json`, `user_instruction.md`) always take priority
     * changes for “performance improvement” are forbidden; only stabilization/consistency fixes are allowed
   * Record patch details (required):

     * `Models/Artifacts/baseline/_tests/baseline_patch_report.md`

       * must include at least: `issues_found`, `patch_summary`, `files_changed`, `compatibility_notes`

4. **Prepare baseline → v001**

   * If `02_train_baseline.py` exists, copy the baseline script (after the audit/patch in step 3) into `02_train_v001.py`.
   * If neither `02_train_baseline.py` nor any versioned training script exists, treat as an error and terminate the loop.

---

## Step 1) Train (Train)

Purpose:

* Run training and produce artifacts for version `v{cur:03d}`.
* Framework-level training logs are allowed, but **do not create evaluation outputs** in this step.

Inputs:

* `user_defined_rules.json` (highest priority)
* `02_train_v{cur:03d}.py`

Work:

* Apply `user_defined_rules.json.training_config` and user constraints **as the highest priority training settings**.
* Run training and save artifacts under `Models/Artifacts/v{cur:03d}/`.
* Must generate `spec.json` for reproducibility.
* Save any pre/post-processing assets required to reproduce inference/evaluation.
* Save checkpoints `best.ckpt` and `last.ckpt`.

#### Failure/exception handling rules for training

* If an exception occurs while running `02_train_v{cur:03d}.py`, **reset the version artifact directory** (delete then recreate):

  * target: `Models/Artifacts/v{cur:03d}/` (entire directory)
* After reset, summarize the cause based on error logs/stack traces and apply **code fixes**:

  * fix target: `02_train_v{cur:03d}.py` (and imported local modules if needed)
  * user constraints always take priority; bypassing constraints is forbidden
* Re-run training with the **same version** `v{cur:03d}`.

  * Do not advance to the next version due to failure (no version skipping).
* (Recommended) record retry history:

  * `Models/Artifacts/v{cur:03d}/_tests/train_retry_report.md`

    * include at least: `attempt`, `error_summary`, `root_cause_hypothesis`, `code_change_summary`, `status(PASS/FAIL)`

Outputs (required):

* `Models/Artifacts/v{cur:03d}/best.ckpt`
* `Models/Artifacts/v{cur:03d}/last.ckpt`
* `Models/Artifacts/v{cur:03d}/spec.json`
* `Models/Artifacts/v{cur:03d}/additional_assets/` (tokenizer/processor/label_map/vocab/normalizer/config as needed)
* `Models/Artifacts/v{cur:03d}/logs/version_#/metrics.csv`
* `Models/Artifacts/v{cur:03d}/logs/version_#/hparams.yaml`

---

## Step 2) Evaluation (Evaluation)

Purpose:

* Evaluate the trained model (`best.ckpt`) on the **validation set only**.
* Produce the final metric scores (`eval_score.json`) and human-checkable inference materials (`infer_samples/`).

Inputs:

* `user_defined_rules.json` (highest priority)
* `02_train_v{cur:03d}.py`
* `Models/Artifacts/v{cur:03d}/best.ckpt`

Work:

1. **Write evaluation script**

   * If `03_eval.py` does not exist, implement it so it can compute metrics (including loss) by referencing `02_train_v{cur:03d}.py` and `user_defined_rules.eval_config.metrics`.
   * Implement inference as if it were used in a real serving environment for this model.

2. **Create `eval_score.json`**

   * Compute metrics over the full validation set.
   * If `eval_config.metrics` is missing, use only the training loss previously used.
   * Save as a **single JSON object per version** with an extensible `metrics: list[dict]`.
   * `eval_config.primary_metric` must be included in `eval_score.json.metrics[].name`. If missing, Step 7 will treat that version as the lowest rank.
   * Must follow this schema (`created_at` is ISO-8601):

   ```json
   {
     "dataset_id": "...",
     "created_at": "...",
     "num_samples": 0,
     "metrics": [
       {"name": "...", "value": 0.0, "higher_is_better": false}
     ],
     "latency_ms": 0
   }
   ```

3. **Create `infer_samples/`**

   * Randomly sample up to `infer_samples` from the validation set, run inference, and write to `infer_samples/`.

     * Use `user_defined_rules.json.eval_config.infer_samples` if present; otherwise default to 10.
     * Use `user_defined_rules.json.seed` if present; otherwise default to 42.
   * Each `{index}` folder must include:

     * `input.*`, `output.*`, `meta.json` (MUST)
     * `gt.*` (optional)
     * Optional task-specific outputs: `viz.*`, `summary.*`

Outputs (required):

* `Models/Artifacts/v{cur:03d}/eval_score.json`
* `Models/Artifacts/v{cur:03d}/infer_samples/{index}/...` (up to `infer_samples`, default 10)

---

## Step 3) Result analysis (Result analysis)

Inputs:

* `user_defined_rules.json` (highest priority)
* `Models/Artifacts/v{cur:03d}/eval_score.json`
* `Models/Artifacts/v{cur:03d}/infer_samples/{index}/...`
* (Optional) `Models/Artifacts/v{cur:03d}/logs/version_0/metrics.csv`

  * If `version_0` does not exist, use the latest `logs/version_*/metrics.csv`.

Work:

* Summarize metric deltas vs the previous version.
* Identify plateau/regression segments and trade-offs.
* Use `infer_samples` as evidence to describe failure patterns and derive hypotheses.

Outputs (required):

* `Models/Artifacts/v{cur:03d}/result-analysis.md`

Minimum required structure of `result-analysis.md` (MUST):

* `Summary`
* `Metric deltas` (v{cur-1:03d} → v{cur:03d})
* `Evidence` (summary backed by infer_samples)
* `Hypotheses`
* `Next focus candidates`

---

## Step 4) Action extract (Action extract)

Inputs:

* `user_defined_rules.json` (highest priority)
* `Models/Artifacts/v{cur:03d}/result-analysis.md`

Work:

* Select **exactly one** improvement action to apply to the next version.
* Selection criteria:

  * low implementation risk
  * high potential for performance gain
  * narrow change scope so cause/effect is traceable
* If the action conflicts with constraints in `user_defined_rules.training_config`, either:

  * modify the action to satisfy constraints, or
  * discard it and pick the next-best alternative.

Outputs (required):

* `Models/Artifacts/v{cur:03d}/action-extract.md`

Minimum required structure of `action-extract.md` (MUST):

* `Selected Action (only 1)`

  * `Goal`
  * `Change`
  * `Where`
  * `Expected impact`
  * `Risk`
  * `Verification`
* `Deferred Candidates` (optional)

---

## Step 5) Loop termination check (Loop)

Inputs:

* current `cur`

Work:

* If `cur >= user_defined_rules.json.loop_config.max_versions`, stop the loop and run Step 7 exactly once.
* Otherwise proceed to Step 6.

---

## Step 6) Next version authoring (Next version authoring)

Inputs:

* `user_defined_rules.json` (highest priority)
* `02_train_v{cur:03d}.py`
* `Models/Artifacts/v{cur:03d}/action-extract.md`

Work:

* Copy `02_train_v{cur:03d}.py` to `02_train_v{cur+1:03d}.py`.
* Apply **only** the changes described in `Selected Action`.
* Always prioritize user constraints in `user_defined_rules.json`.
* Leave a change summary at the top of the file.

Outputs:

* `02_train_v{cur+1:03d}.py`
* Increment `cur += 1` and return to Step 1.

---

## Step 7) Final report after loop ends (Final report after loop ends)

Inputs:

* `user_defined_rules.json` (highest priority)
* All `v###` directories under `Models/Artifacts/`
* `02_train_v###.py`

Work:

* Run exactly once after the loop ends.
* Save reports to **`Models/Report/YYYY-MM-DD_HH-MM/`**.
* Write `04_final_report.py` to generate `leaderboard.csv` and `final-analysis.md`.
* Generate `leaderboard.csv` by parsing `eval_score.json` from each version.
* Flatten all entries in `eval_score.json.metrics` into columns:

  * column naming: `metric__{name}` (e.g., `metric__loss`, `metric__wer`)
  * if `{name}` duplicates occur, uniquify with suffix `metric__{name}__{k}`
  * if a metric is missing for a version, leave the cell empty
* If external models exist under `Models/external/`, evaluate them using the same metrics and add to `leaderboard.csv` (consult `inference_sample.py` to match their inference method).

  * Treat the external model folder name as `version_id` (e.g., `Models/external/glm-ocr` → `version_id=glm-ocr`)
* `leaderboard.csv` must include **only** the following columns (do not create others):

  * `version_id`, `dataset_id`, `created_at`, `latency_ms`, `artifact_relpath`, `metric__{name}...`
* `artifact_relpath` must be recorded as a relative path from `Models/Report/YYYY-MM-DD_HH-MM/`.
* **Primary-metric-based ranking/selection**

  * Use `user_defined_rules.json.eval_config.primary_metric` as the primary metric name.
  * Find the metric entry in `eval_score.json.metrics` where `name == primary_metric` and use its `value`.
  * Sorting direction follows `higher_is_better`:

    * `higher_is_better=true` → descending (higher ranks first)
    * `higher_is_better=false` → ascending (lower ranks first)
  * Versions missing the primary metric must be pushed to the bottom and logged as a warning in `final-analysis.md`.
* Define the best version as **rank #1** by the primary metric rule.
* Copy best version artifacts to:

  * `Models/Report/YYYY-MM-DD_HH-MM/Artifacts/{best_version_id}/{best.ckpt, spec.json, additional_assets/...}`
* In `final-analysis.md`, analyze why the top 3 versions (including best) ranked high by referencing:

  * (recommended) each of the top 3 versions’ `02_train_v{version_id}.py`, `action-extract.md`, and `result-analysis.md`

Outputs (required):

* `04_final_report.py`
* `Models/Report/YYYY-MM-DD_HH-MM/final-analysis.md`
* `Models/Report/YYYY-MM-DD_HH-MM/leaderboard.csv`
* `Models/Report/YYYY-MM-DD_HH-MM/Artifacts/{best_version_id}/{best.ckpt, spec.json, additional_assets/...}`

`leaderboard.csv` schema (MUST):

* `version_id`
* `dataset_id`
* `created_at`
* `latency_ms`
* `artifact_relpath` (e.g., `Artifacts/v010/best.ckpt`)
* `metric__{name}` (one or more flattened metric columns)

Example (`leaderboard.csv`, 3 rows):

```csv
version_id,dataset_id,created_at,latency_ms,artifact_relpath,metric__loss,metric__wer
v008,v1,2026-02-23T22:01:10,87,Artifacts/v008/best.ckpt,0.142,0.091
v009,v1,2026-02-23T22:01:10,86,Artifacts/v009/best.ckpt,0.137,0.095
v010,v1,2026-02-23T22:01:10,85,Artifacts/v010/best.ckpt,0.131,0.089
```

---

## Directory structure

### Version artifacts

```text
Models/Artifacts/v001/
├── spec.json
├── additional_assets/
│   ├── tokenizer.json
│   └── processor.json
├── best.ckpt
├── last.ckpt
├── eval_score.json
├── result-analysis.md
├── action-extract.md
├── infer_samples/
│   └── 0001/
│       ├── input.*
│       ├── output.*
│       ├── gt.*            # optional
│       ├── viz.*           # optional
│       └── meta.json
└── logs/version_#/
    ├── metrics.csv
    └── hparams.yaml
```

### Final report

```text
Models/Report/YYYY-MM-DD_HH-MM/
├── leaderboard.csv
├── final-analysis.md
└── Artifacts/
    └── v###/
        ├── best.ckpt
        ├── spec.json
        └── additional_assets/
            └── ...
```

---

## Rules

### MUST

* Execute only when the user types `/progressive-training` or it is triggered automatically from `build-baseline` Step 3.
* On execution, read and apply **`./user_defined_rules.json` with highest priority**.
* Do not apply changes that conflict with user-defined constraints in `user_defined_rules.json`.
* Always format versions as `v{cur:03d}`.
* Decide `dataset_id` in Step 0 and keep it fixed until the loop ends.
* All versions must follow the `Models/Artifacts/v###/` structure.
* `02_train_v###.py` must not create or modify `eval_score.json` or `infer_samples/`.
* `eval_score.json` must be stored as a **single JSON object**.
* `infer_samples` must be randomly sampled from the validation set up to `infer_samples` (default 10) using a **fixed seed** (`user_defined_rules.json.seed`, else 42).
* The next version change must reflect **only one Selected Action** derived from `result-analysis.md` → `action-extract.md`.
* Loop count is controlled by `user_defined_rules.json.loop_config.max_versions`; upon reaching it, stop and run Step 7 once.
* Save the final report to **`Models/Report/YYYY-MM-DD_HH-MM/`**.
* Each version’s artifacts must include any required pre/post-processing assets needed to reproduce inference/evaluation. If they are not required for the task, they may be omitted, but `spec.json` must explicitly mark them as `not_required`.

### SHOULD

* Include sample identifiers, conditions, config summaries, and the used seed in `infer_samples/{index}/meta.json`.
