---
name: build-baseline
description: 원천 데이터를 자동으로 탐색해 샘플 단위와 라벨/경로 규칙을 스스로 파악하고, 이를 문서화한 기준에 따라 전처리 데이터와 베이스라인 학습 코드가 즉시 실행 가능한 상태로 만들어 스모크 테스트까지 통과시키는 스킬입니다. 실행 과정 전반에서 사용자 정의 규칙을 최우선으로 적용하며, 태스크별 데이터 구조와 검증 규칙을 일관된 표준으로 생성·검증합니다.
---

## When to use

* 반드시 사용자가 **`/build-baseline`** 를 입력했을 때만 사용한다.
* `dataset/rawdata/` 아래 여러 source 폴더가 존재하고, 모델이 직접 탐색해 파싱 규칙을 문서화한 뒤 표준 파이프라인/베이스라인 코드를 만들고 싶을 때 사용한다.
* 베이스라인 학습 스크립트(`02_train_baseline.py`)를 표준 규칙에 맞게 작성하고, 스모크 테스트까지 통과시키고 싶을 때 사용한다.
* validation split을 단순 랜덤이 아니라 층화/그룹 기반 등 고급 전략으로 관리하고 싶을 때 사용한다.
* 태스크별 데이터 구조를 일관되게 유지하기 위해 `task_format.yaml`을 기준으로 데이터/로더/스키마를 맞추고 싶을 때 사용한다.

---

## Workflow

### 공통

* 본 스킬은 실행 시 **`./user_defined_rules.json`** 및 **`./user_instruction.md`**을 최우선으로 참조한다.
* `user_defined_rules.json` 및 `user_instruction.md`에 명시된 항목은 **사용자 정의 제약**으로 간주하며, 학습/평가/분석/다음 버전 작성 전 과정에서 임의로 변경하지 않는다.
* `user_defined_rules.json` 및 `./user_instruction.md`은 본 SKILL.md와 동일 디렉토리에 있어야 한다. 없으면 **사용자 정의 제약**이 없는 것으로 간주한다.
* `task_format.yaml`은 본 SKILL.md와 동일 디렉토리의 `./task_format.yaml`에 있어야 한다. 없으면 본 스킬을 중단한다.
* Step 0~2는 코드 작성 후 테스트를 통과해야 다음 step으로 넘어간다.
* 전체 과정에서 “랜덤성”이 개입되는 모든 작업(샘플링, split, 셔플, subset 추출, 스모크 테스트 샘플 선택 등)은 **루트의 `seed` 키** 값을 사용해 재현 가능해야 한다. `seed`가 없으면 42를 사용한다.
* 본 스킬에서 `{version}`은 항상 `user_defined_rules.json.dataset_config.dataset_id`를 의미한다.
  * 예: `dataset_id = "v1"` → `dataset/prepdata/v1/...`

`user_defined_rules.json` 예시(일부):

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

### Step 0) Rawdata 자동 탐색 프로파일링 문서 작성 및 테스트 (Rawdata profiling)

역할:

* `dataset/rawdata/` 아래 존재하는 모든 source 폴더를 직접 탐색하고 내부 파일/구조를 분석하여, source별 파싱 규칙과 샘플 단위 정의를 문서로 고정한다.
* `dataset_config.task_type`이 없으면 프로파일링을 통해 직접 정의하고, `dataset_config.task_type`에 맞게 데이터가 구성될 수 있는 가이드라인을 제공한다.
* 이후 데이터 파이프라인 단계는 반드시 이 문서를 읽고 동일 규칙으로 처리해야 한다.
* 이 단계는 문서 작성과 최소 검증(샘플 로드 테스트)을 통과해야 다음 단계로 진행한다.

입력:

* `user_defined_rules.json` (최우선)

  * `seed` (없으면 42)
  * `dataset_config.task_type` (선택)
  * `dataset_config.max_profile_samples` (선택)
* `dataset/rawdata/*` (대상은 특정 source가 아니라 전체 폴더)

작업 (자동 탐색 + 문서 작성):

* `dataset/rawdata/` 하위에서 프로파일링 대상 디렉토리를 식별한다.

  * 포함: `dataset/rawdata/{source}/...`
  * 제외: `dataset/rawdata/profiling.md` 같은 문서 파일
* 각 `{source}`에 대해 아래를 수행하고, 결과를 하나의 문서에 통합 기록한다.
* 프로파일링 단계에서 실제로 로드/스캔/샘플링하는 총 샘플 수는 `max_profile_samples`(없으면 1,000) 로 제한한다. (대상 선택은 seed 기반으로 재현 가능해야 한다).

`dataset/rawdata/profiling.md` 최소 구성(MUST):

* `Overview` (전체 rawdata 개요, source 목록, 규모 추정)
* `Sources`

  * 각 source별 섹션: `## Source: {source_name}`
* 각 source 섹션의 필수 내용:

  * `Sample unit definition`
  * `Paths & naming rules`
  * `Label schema`
  * `Edge cases`
  * `Extraction rules`
  * `Sanity checks`

작업 (테스트: 최소 로드 검증):

* 각 source별로 임의 샘플 N개(권장 10개)를 로드해 다음을 확인한다.

  * 샘플 단위 파싱 가능
  * asset/label 매칭 규칙 일관성(해당 시)
  * 라벨 스키마 파싱 가능
* 샘플 선택은 `seed` 기반으로 재현 가능해야 한다.
* 결과를 source 섹션 하단에 PASS/FAIL로 기록한다.

결과(필수):

* `dataset/rawdata/profiling.md`

---

### Step 1) 데이터 파이프라인 코드 작성 및 테스트 (Data pipeline)

역할:

* Step 0에서 생성된 `dataset/rawdata/profiling.md`를 읽고, 그 규칙에 따라 rawdata를 파싱/정규화하여 prepdata를 생성한다.
* `./task_format.yaml`의 `supported_tasks[{task_type}]`가 권장하는 구조로 `dataset/prepdata/{version}/...`를 생성한다.
* 이 단계는 코드 작성과 테스트 통과가 완료되어야 다음 단계로 진행한다.

입력:

* `user_defined_rules.json` (최우선)

  * `seed` (없으면 42)
  * `dataset_config.dataset_id`  → `{version}`
  * `dataset_config.task_type`
  * `dataset_config.valid_ratio` (없으면 기본값 사용)
  * `dataset_config.split_strategy` (없으면 `random`)
  * `dataset_config.stratify_key` (없으면 `category`)
  * `dataset_config.group_key` (없으면 `group_id`)
  * `dataset_config.max_train_samples` (선택)
* `task_format.yaml` (필수)
* `dataset/rawdata/profiling.md` (필수)
* `dataset/rawdata/*` (profiling.md에 포함된 source 전부)

작업 (코드 작성):

#### 프로파일링 문서 규칙 적용

* 반드시 `dataset/rawdata/profiling.md`를 기준으로 rawdata를 파싱한다.
* profiling.md와 상충하는 “추정 파싱 로직”을 새로 만들지 않는다.
* 필요한 경우 profiling.md를 먼저 갱신한 후 파이프라인을 수정한다.

#### Meta table 생성: `df_meta`

* profiling.md의 `Extraction rules`에 따라 `df_meta`를 생성한다.

* `df_meta`는 여러 source를 함께 관리 가능해야 하므로 `source` 컬럼을 포함한다.

* `dataset_config.max_train_samples`가 존재하면, 전처리/검증을 통과한 학습 가능 샘플 풀을 먼저 확정한 뒤, 그 풀에서 seed 기반으로 재현 가능하게 최대 max_train_samples개를 샘플링하여 df_meta를 구성한다.

* 별다른 사용자 지시가 없으면 source 기준 균등 분포 랜덤 샘플링을 적용한다.

* 최소 컬럼:

  * `sample_id`, `source`, `category`(없으면 null), `asset_paths`

* 권장 컬럼:

  * `group_id`, `lang`, `length`, `width`, `height`, `timestamp`, `label_summary`, `hash`

* 저장(권장):

  * `dataset/prepdata/{version}/common/df_meta.parquet`

#### Validation split

* split은 `seed` 기반으로 재현 가능해야 한다.
* `split_strategy`에 따라 아래를 지원한다.

  * `random`
  * `stratified` (기본 `category`)
  * `group` (기본 `group_id`)
* `stratified`의 경우 분포가 완전히 일치할 필요는 없으며, **크게 어긋나지 않는 수준이면 허용**한다.
* split 결과는 매니페스트로 저장(권장):

  * `dataset/prepdata/{version}/common/train_manifest.parquet`
  * `dataset/prepdata/{version}/common/valid_manifest.parquet`

#### Prepdata 생성: `task_format.yaml` 준수

* `task_format.yaml.supported_tasks[{task_type}]`의 `split_layout`과 `index_schema`를 준수하여 생성한다.

  * `dataset/prepdata/{version}/stats.json`
  * `dataset/prepdata/{version}/train/`
  * `dataset/prepdata/{version}/valid/`
  * `dataset/prepdata/{version}/common/`

작업 (테스트):

* `task_format.yaml.validation.checks` 기반 정합성 검사
* split 전략별 추가 검사:

  * `group`: 그룹 누수 없음
  * `stratified`: PASS/FAIL은 강제하지 않으며, 분포 요약만 리포트에 기록한다.
* 필수 파일/폴더 검사:

  * `stats.json`, `train/`, `valid/`, `common/`
  * `train.jsonl|train.parquet`, `valid.jsonl|valid.parquet`

결과(필수):

* 데이터 파이프라인 코드: `01_data_pipeline.py`
* `dataset/prepdata/{version}/stats.json`
* `dataset/prepdata/{version}/train/`
* `dataset/prepdata/{version}/valid/`
* `dataset/prepdata/{version}/common/`
* (권장) `dataset/prepdata/{version}/common/df_meta.parquet`
* (권장) `dataset/prepdata/{version}/_tests/pipeline_test_report.md`

---

### Step 2) 베이스라인 학습 코드 작성 및 테스트 (Baseline)

역할:

* `02_train_baseline.py`를 작성하되, 디버그/스모크 테스트 목적으로 최소 샘플(train/valid 각 100개)만으로 학습/검증이 정상 동작하는지 확인한다.
* 이 단계는 코드 작성과 스모크 테스트 통과가 완료되어야 다음 단계로 진행한다.

입력:

* `user_defined_rules.json` (최우선)

  * `seed` (없으면 42)
  * `dataset_config.dataset_id`  → `{version}` 및 `DATASET_ID`
  * `dataset_config.task_type`
  * `baseline_smoke_test.epoch` (선택, 없으면 1)
  * `baseline_smoke_test.train_samples` (없으면 100)
  * `baseline_smoke_test.valid_samples` (없으면 100)
  * `baseline_smoke_test.min_batch_size` (선택)
  * `baseline_smoke_test.max_batch_size` (선택)
  * (선택) `training_config.*`, `model_config.*`
* `task_format.yaml` (필수)
* `dataset/prepdata/{version}/train/`
* `dataset/prepdata/{version}/valid/`
* `dataset/prepdata/{version}/common/` (태스크에 따라 선택)

작업 (코드 작성):

#### 베이스라인 학습 스크립트 생성

* `02_train_baseline.py`를 작성한다.
* `02_train_baseline.py`에는 전역 변수 `DATASET_ID: str`를 반드시 포함하고, `{version}`(= `dataset_config.dataset_id`)과 동일해야 한다.
* 데이터 로딩은 `dataset/prepdata/{version}` 구조를 단일 진실 소스로 사용하며, `task_format.yaml.supported_tasks[{task_type}]`의 구조/스키마를 준수한다.
* 스모크 테스트 샘플 선택(Subset)은 `seed` 기반으로 재현 가능해야 한다.
* 최소 아티팩트/로그 구조를 갖춘다.

  * `Models/Artifacts/baseline/spec.json`
  * `Models/Artifacts/baseline/additional_assets/`
  * `Models/Artifacts/baseline/logs/version_0/{metrics.csv,hparams.yaml}`
* 스모크 테스트 모드를 지원한다.

  * train/valid에서 각각 N/M개(기본 100개)만 샘플링
  * `baseline_smoke_test.epoch`(기본 1)로 정상 동작만 검증

작업 (최적 배치 사이즈 탐색):

* `baseline_smoke_test.min_batch_size ~ baseline_smoke_test.max_batch_size` 범위에서 2^N 배치 사이즈 후보를 구성하여 탐색 작업을 수행한다.

  * 배치 탐색은 10 step으로만 수행한다.
  * 후보 생성 규칙: bs ∈ { 2^k | min_batch_size ≤ 2^k ≤ max_batch_size }
  * 가장 큰 batch_size부터 탐색을 시작하고, 가장 큰 batch_size가 성공 시 바로 탐색을 종료한다.
  * 성공한 후보 중 가장 큰 batch_size를 `optimal_batch_size`로 선택한다.
  * 성공 기준:

    * 10 step 동안 OOM/RuntimeError 없이 forward+backward가 완료되면 PASS로 간주한다.
  * 종료 시 탐색 결과를 반드시 `_tests/baseline_smoke_test_report.md`에 포함하여 기록한다.

작업 (expected_* 검증):

* `user_defined_rules.json.model_config`에서 `expected_` 접두사를 가진 모든 키를 검증 대상(contract)으로 취급한다.
  * 예: `expected_stride`, `expected_channels`, `expected_out_shape`, `expected_dtype` …
* 관측은 (가능하면) dummy forward 1회로 산출한다. (batch=1, minimal input)
* minimal input은 model_config의 입력 제약(H_TARGET, W_BUCKETS 등)을 만족하도록 구성한다.

* 검증 방식(MUST):

  1. `model_config`에서 `expected_`로 시작하는 항목들을 모두 수집해 `expected_map`을 만든다.
  2. `expected_map`이 비어있으면 본 검증 단계는 **SKIP** 한다(FAIL 아님).
  3. 현재 학습 코드가 생성/로딩한 “검증 가능한 실제 값”을 `observed_map`으로 만든다.
     * `observed_map`은 `expected_*`의 suffix와 동일한 이름을 사용한다.
       * 예: `expected_stride` ↔ `observed_stride`, `expected_channels` ↔ `observed_channels`
  4. 각 `expected_*`에 대해 PASS/FAIL을 결정한다(아래 타입 규칙 적용).

* 타입별 비교 규칙(MUST):

  * 스칼라(숫자/문자열/불리언): `observed == expected` 아니면 FAIL
  * 리스트/튜플: 길이/원소/순서 모두 동일해야 PASS
  * 딕셔너리: 모든 key가 존재하고 value가 동일해야 PASS
  * float: `model_config.expected_tolerance`(선택) 없으면 완전 일치, 있으면 `abs(observed-expected) <= tolerance`

* 관측 불가능 처리(MUST):

  * `observed_*` 값을 만들 수 없으면 FAIL 처리한다.
  * 단, `model_config.expected_allow_missing_keys: true`가 있으면 `SKIP(allow_missing)`로 기록하고 전체 FAIL로 만들지 않는다.

* 실패 시 동작(MUST):

  * FAIL이 1개라도 발생하면 학습(배치 탐색/스모크 러닝)을 진행하지 않고 즉시 중단한다.
  * 설정(`expected_*` 또는 관련 config) 또는 코드(관측 로직) 수정 후 재실행해야 한다.

* 리포트 기록(MUST):

  * 아래 내용을 `Models/Artifacts/baseline/_tests/baseline_smoke_test_report.md`에 포함한다.

    * `expected_map`
    * `observed_map`
    * `validation_table` (key, expected, observed, status(PASS/FAIL/SKIP), reason)
    * `overall_result: PASS|FAIL`

작업 (스모크 러닝):

* 스모크 테스트의 목적은 “학습 스크립트가 에러 없이 실행되고, 아티팩트 경로에 결과물이 생성되는지” 확인하는 것이다.
* 위 배치 탐색에서 선택한 `optimal_batch_size`로만 스모크 러닝을 수행한다.
* train N개 + valid M개로 실행한다.
* 테스트 통과 조건(최소):

  * 학습이 에러 없이 시작/종료
  * `best.ckpt`, `last.ckpt` 생성
  * `spec.json`, `additional_assets/`, 로그 생성

#### 학습 실패(에러/예외) 처리 규칙

  * `02_train_baseline.py` 실행 중 예외가 발생하면, 해당 버전의 아티팩트 디렉토리를 **초기화(삭제 후 재생성)** 한다.
    * 대상: `Models/Artifacts/baseline/` 전체
  * 초기화 후, 에러 로그/스택트레이스를 근거로 원인을 요약하고, 해결을 위한 **코드 수정**을 수행한다.
    * 수정 대상: `02_train_baseline.py` (필요 시 해당 스크립트가 import하는 로컬 모듈 포함)
    * 사용자 정의 제약(`user_defined_rules.json`, `user_instruction.md`)은 항상 우선하며, 제약을 우회하는 수정은 금지한다.
  * 수정이 완료되면 **동일 버전(baseline)** 으로 학습을 다시 실행한다.
  * (권장) 재시도 이력을 아래 파일에 포함시킨다.
    * `Models/Artifacts/baseline/_tests/baseline_smoke_test_report.md`
      * 최소 포함: `attempt`, `error_summary`, `root_cause_hypothesis`, `code_change_summary`, `status(PASS/FAIL)`

#### 아티팩트 저장

* 스모크 러닝이라도 **반드시** 아래 파일을 생성해야 한다.

  * `Models/Artifacts/baseline/best.ckpt`
  * `Models/Artifacts/baseline/last.ckpt`
* `best.ckpt` 선정 기준(권장): valid_loss 기준으로 best 저장. (loss 기반이어도 무방하나 `spec.json`에 기준을 명시)
* 체크포인트 저장이 불가능한 태스크/프레임워크라면, 실행을 중단하고 대체 규칙을 문서화해야 한다(임의로 생략 금지).

결과(필수):

* `02_train_baseline.py`
* `Models/Artifacts/baseline/spec.json`
* `Models/Artifacts/baseline/best.ckpt`
* `Models/Artifacts/baseline/last.ckpt`
* `Models/Artifacts/baseline/additional_assets/`
* `Models/Artifacts/baseline/logs/version_0/{metrics.csv,hparams.yaml}`
* `Models/Artifacts/baseline/_tests/baseline_smoke_test_report.md`

---

### Step 3) 산출물 검사 (Final check)

역할:

* Step 0~2 산출물이 모두 존재하고 테스트가 통과했는지 최종 점검한다.
* 최종 점검이 통과되면 다음 단계로 **`progressive-training`** 스킬을 트리거한다.

입력:

* Step 0~2 산출물 전체

작업:

* 체크리스트로 최종 검증한다.

  * `dataset/rawdata/profiling.md` 존재 및 source별 PASS 기록
  * `dataset/prepdata/{version}/` 내 `stats.json, train/, valid/, common/` 존재
  * Step 1 테스트 통과(리포트가 있으면 PASS 확인)
  * `02_train_baseline.py` 존재
  * `Models/Artifacts/baseline/` 내 `best.ckpt, last.ckpt, spec.json, logs/, additional_assets/, _tests/baseline_smoke_test_report.md` 존재
  * Step 2 스모크 테스트 통과(리포트가 있으면 PASS 확인)

결과:

* 최종 점검 통과 시:
  * **`.claude/skills/progressive-training/SKILL.md`** 파일을 읽고 작업한 dataset_id를 전달하여 후속 작업을 이어서 진행
* 최종 점검 실패 시:
  * 실패 원인을 상세히 작성하고 작업을 마무리

---

## 디렉토리 구조 (예시)

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

## 규칙

### MUST

* 실행 시 `./user_defined_rules.json`을 최우선으로 읽고 적용한다.
* 랜덤성이 개입되는 모든 작업은 루트 `seed` 값을 사용해야 하며, 없으면 42를 사용한다.
* 프로파일링 단계의 스캔/로드 상한은 `user_defined_rules.dataset_config.max_profile_samples`로 제어한다.
* 학습 데이터 구성 시 샘플 수 상한은 `user_defined_rules.dataset_config.max_train_samples`로 제어하며, 이는 전처리/검증을 통과한 학습 가능 샘플 풀에서 적용한다.
* `task_format.yaml`이 없으면 실행하지 않는다.
* Step 0는 `dataset/rawdata/`의 모든 source 폴더를 자동 탐색하여 `dataset/rawdata/profiling.md`를 작성한다.
* Step 1은 반드시 `dataset/rawdata/profiling.md`를 읽고 그 규칙에 따라 rawdata를 파싱한다.
* Step 2는 `02_train_baseline.py`를 생성하고 스모크 테스트를 통과해야 한다.
* Step 2는 **`best.ckpt`, `last.ckpt`를 반드시 생성**해야 한다.
* Step 3 최종 검사에서 모든 조건이 충족되어야 ready로 간주한다.

### SHOULD

* `df_meta.parquet`와 split 매니페스트(train/valid)를 `common/`에 저장해 데이터 관리/재현성을 높인다.
* `stratified` split은 분포 유지 검증과 허용 오차 기준을 문서화한다.
* Step 0~2 테스트 리포트를 `_tests/` 하위에 남겨 검증 기록으로 유지한다.
