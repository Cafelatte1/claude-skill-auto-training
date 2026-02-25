---
name: progressive-training
description: 베이스라인 이후 모델을 버전 단위로 반복 학습하면서 각 버전의 결과를 평가·요약하고, 분석을 통해 도출한 단 하나의 저위험 개선 행동만 반영해 다음 버전 학습 코드를 생성하는 수동 루프 스킬입니다. 실행 과정 전반에서 사용자 정의 규칙을 최우선으로 적용하며, 루프 종료 시 전체 버전을 동일한 평가 프로토콜로 재평가해 리더보드와 최적 아티팩트를 표준 리포트 형태로 저장합니다.
---

## When to use

* 사용자가 **`/progressive-training`** 를 입력했을 경우 혹은 build-baseline 작업이 성공적으로 완료된 후 진행한다.
* 베이스라인 이후, 모델/파이프라인을 **버전 단위(v001, v002, …)** 로 점진 개선할 때 사용한다.
* 각 버전의 학습 산출물과 평가 결과를 표준 구조로 누적하고, **분석 → 개선 행동 → 다음 버전 코드 생성**까지 연결해 반복할 때 사용한다.
* 루프 종료 후 “현재 존재하는 모든 버전”을 동일 프로토콜로 재평가하여 **리더보드 리포트**를 생성할 때 사용한다.

---

## Workflow

### 공통

* 본 스킬은 실행 시 **`./user_defined_rules.json`** 및 **`./user_instruction.md`**을 최우선으로 참조한다.
* 두 파일에 명시된 항목은 **사용자 정의 제약**으로 간주하며, 학습/평가/분석/다음 버전 작성 전 과정에서 임의로 변경하지 않는다.
* 두 파일은 본 SKILL.md와 동일 디렉토리에 있어야 한다. 없으면 **사용자 정의 제약이 없는 것**으로 간주한다.
* action-extract에 포함된 변경 사항이 `user_defined_rules.json`과 충돌하면, **항상 `user_defined_rules.json`이 우선**이다.
* 루프 반복 횟수/종료 조건은 `user_defined_rules.json.loop_config`로 제어한다. `max_versions`에 도달하면 루프를 종료하고 **Step 7(종합 리포트 생성)** 을 1회 실행한다.
* `cur`는 **정수**로 관리한다. 버전 표기는 항상 `version_id = v{cur:03d}` 를 사용한다.
  * 예: `cur=1 → v001`, `cur=12 → v012`
* 파일/폴더 경로는 항상 `v{cur:03d}` 형식을 따른다.
  * 예: `02_train_v001.py`, `Models/Artifacts/v001/`
* dataset_id는 다음 우선순위에 따라 설정하며, **Step 0에서 1회 결정 후 루프 종료까지 고정**한다.
  * 1순위: build-baseline에서 작업 완료하여 전달된 dataset_id
  * 2순위: `user_defined_rules.json.dataset_config.dataset_id`
* 각 버전의 `Models/Artifacts/v{cur:03d}/eval_score.json.dataset_id`에 기록한다.
* dataset_id는 데이터셋(전처리) 식별자이고, v{cur:03d}는 학습 버전이다.

예시:

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

## Step 0) 루프 초기화 (Initialize loop state)

목적:

* 루프 시작 전에 **dataset_id 확정**, **현재 버전(cur) 결정**, **v001 초기 생성**, **baseline→loop 전환 준비**를 수행한다.

입력:

* `./user_defined_rules.json`, `./user_instruction.md` (최우선)
* `dataset/prepdata/{dataset_id}/stats.json`
* 기존에 존재하는 `02_train_v###.py` 목록
* `02_train_baseline.py`
* `Models/Artifacts/baseline/spec.json` (존재 시)
* `Models/Artifacts/baseline/_tests/baseline_smoke_test_report.md` (존재 시)
* (선택) `/progressive-training?...&dataset_id=...` 쿼리 파라미터

작업:

1. **dataset_id 확정**

  * 공통의 dataset_id 결정 규칙에 따라 1회 확정하고 이후 루프 종료까지 고정한다.

2. **cur 결정 규칙**

  * `02_train_v###.py`가 하나 이상 존재하면:

    * 가장 큰 버전을 `cur`로 설정하고, **Step 1(학습)** 은 그 `cur`부터 시작한다.
  * `02_train_v###.py`가 하나도 없으면:

    * `cur = 1`로 설정한다. (`version_id = v001`)
    * 이때 아래 3) 초기화를 수행한다.

3. **Baseline 사전 점검 및 1회 수정 (Baseline audit & one-time patch)**

  * `dataset/prepdata/{dataset_id}/stats.json`을 읽어 학습 데이터의 샘플 수와 task_type을 확인한다.
  * `Models/Artifacts/baseline/_tests/baseline_smoke_test_report.md`을 읽어 최적 배치 사이즈(`optimal_batch_size`)를 확인한다.
  * 데이터 경로가 `dataset/prepdata/{dataset_id}/...` 규칙을 따르는지
  * `DATASET_ID`/`VERSION` 등 전역 상수가 규칙과 일치하는지
  * 아티팩트 저장 경로가 `Models/Artifacts/baseline/` 규칙을 따르는지
  * 로그 경로가 `Models/Artifacts/baseline/logs/version_#/` 형태인지
  * 스모크 테스트에서 확인된 `optimal_batch_size`를 사용할 수 있도록 반영 가능한지(가능하면 반영)
  * 문제를 발견하면 **최소 수정만 1회 적용**한다.

    * 수정 대상: `02_train_baseline.py` (필요 시 import하는 로컬 모듈 포함)
    * 사용자 정의 제약(`user_defined_rules.json`, `user_instruction.md`)은 항상 우선
    * “성능 개선” 목적의 변경은 금지(루프 준비를 위한 안정화/정합성 목적만 허용)
  * 수정 내역을 아래 파일에 기록한다(필수):

    * `Models/Artifacts/baseline/_tests/baseline_patch_report.md`
      * 최소 포함: `issues_found`, `patch_summary`, `files_changed`, `compatibility_notes`

4. **baseline → v001 파일 준비**

  * `02_train_baseline.py`가 존재하면 **3 에서 점검/수정된 baseline을 기준으로** 복사하여 `02_train_v001.py`를 만든다.
  * `02_train_baseline.py`가 없고 루프용 학습 스크립트도 없다면 오류로 간주하여 루프를 종료한다.

---

## Step 1) 학습 (Train)

목적:

* 훈련을 수행하고 `v{cur:03d}` 버전에 대한 아티팩트를 생성한다.
* 훈련 프레임워크로 인한 로깅은 허용하되, **평가 산출물은 생성하지 않는다.**

입력:

* `user_defined_rules.json` (최우선)
* `02_train_v{cur:03d}.py`

작업:

* `user_defined_rules.json.training_config` 및 사용자 정의 제약을 **학습 설정에 우선 적용**한다.
* 학습을 실행하고 `Models/Artifacts/v{cur:03d}/` 아래에 아티팩트를 저장한다.
* `spec.json`은 재현성을 위해 반드시 생성한다.
* 추론/평가 재현에 필요한 전/후처리 자산(pre/post-processing assets)을 저장한다.
* 체크포인트는 `best.ckpt`, `last.ckpt`를 남긴다.

#### **학습 실패(에러/예외) 처리 규칙**

  * `02_train_v{cur:03d}.py` 실행 중 예외가 발생하면, 해당 버전의 아티팩트 디렉토리를 **초기화(삭제 후 재생성)** 한다.
    * 대상: `Models/Artifacts/v{cur:03d}/` 전체
  * 초기화 후, 에러 로그/스택트레이스를 근거로 원인을 요약하고, 해결을 위한 **코드 수정**을 수행한다.
    * 수정 대상: `02_train_v{cur:03d}.py` (필요 시 해당 스크립트가 import하는 로컬 모듈 포함)
    * 사용자 정의 제약(`user_defined_rules.json`, `user_instruction.md`)은 항상 우선하며, 제약을 우회하는 수정은 금지한다.
  * 수정이 완료되면 **동일 버전(v{cur:03d})** 으로 학습을 다시 실행한다.
    * 실패로 인해 다음 버전으로 넘어가면 안 된다(버전 스킵 금지).
  * (권장) 재시도 이력을 아래 파일에 기록한다.
    * `Models/Artifacts/v{cur:03d}/_tests/train_retry_report.md`
      * 최소 포함: `attempt`, `error_summary`, `root_cause_hypothesis`, `code_change_summary`, `status(PASS/FAIL)`

결과(필수):

* `Models/Artifacts/v{cur:03d}/best.ckpt`
* `Models/Artifacts/v{cur:03d}/last.ckpt`
* `Models/Artifacts/v{cur:03d}/spec.json`
* `Models/Artifacts/v{cur:03d}/additional_assets/` (task에 필요한 tokenizer/processor/label_map/vocab/normalizer/config 등 포함)
* `Models/Artifacts/v{cur:03d}/logs/version_#/metrics.csv`
* `Models/Artifacts/v{cur:03d}/logs/version_#/hparams.yaml`

---

## Step 2) 평가 (Evaluation)

목적:

* 학습이 끝난 모델(`best.ckpt`)로 **validation set만** 사용해 평가를 수행하고,
* 현재 버전의 최종 메트릭 점수(`eval_score.json`) 및 실제 추론을 확인할 수 있는 자료(`infer_samples/`)를 생성한다.

입력:

* `user_defined_rules.json` (최우선)
* `02_train_v{cur:03d}.py`
* `Models/Artifacts/v{cur:03d}/best.ckpt`

작업:

1. **평가 스크립트 작성**
  * `03_eval.py` 평가 스크립트가 없다면 `02_train_v{cur:03d}.py` 및 `user_defined_rules.eval_config.metrics`를 참고해 loss를 포함한 메트릭 계산을 할 수 있도록 구현한다.
  * 평가 코드의 추론 부분은 실제 이 모델을 서빙하는 상황을 가정하고 구현한다.

2. **eval_score.json 생성**
  * validation set 전체를 활용하여 메트릭을 계산한다.
  * `eval_config.metrics`이 없으면 기존 학습 시 사용된 loss 값만 사용한다.
  * 버전당 **단일 JSON 객체**로 저장한다(확장 가능한 `metrics: list[dict]`).
  * `eval_config.primary_metric`은 반드시 `eval_score.json.metrics[].name`에 포함되어야 한다. 없으면 Step 7에서 최하위로 처리한다.
  * 다음 스키마를 따라 작성한다. (`created_at`는 ISO-8601)
  ```json
  {
    "dataset_id": ...,
    "created_at": ...,
    "num_samples": ...,
    "metrics": [
      {"name": ..., "value": ..., "higher_is_better": ...}
    ],
    "latency_ms": ...
  }
  ```

3. **infer_samples 생성**
  * validation set에서 최대 `infer_samples` 만큼 랜덤 샘플링하여 추론 후 `infer_samples/`를 생성한다.
    * infer_samples 수는 `user_defined_rules.json.eval_config.infer_samples`를 우선 사용하고, 없으면 10을 사용한다. 
    * 시드는 `user_defined_rules.json.seed` 우선, 없으면 42 사용한다.
  * 각 `{index}` 폴더에는:
    * `input.*`, `output.*`, `meta.json`은 MUST
    * `gt.*`는 optional
    * task 유형에 따라 optional 산출물 생성 가능: `viz.*`, `summary.*`

결과(필수):

* `Models/Artifacts/v{cur:03d}/eval_score.json`
* `Models/Artifacts/v{cur:03d}/infer_samples/{index}/...` (최대 `infer_samples`, 기본 10)

---

### Step 3) 결과 분석 (Result analysis)

입력:

* `user_defined_rules.json` (최우선)
* `Models/Artifacts/v{cur:03d}/eval_score.json`
* `Models/Artifacts/v{cur:03d}/infer_samples/{index}/...`
* (선택) `Models/Artifacts/v{cur:03d}/logs/version_0/metrics.csv`

  * `version_0`가 없으면 가장 최신 `logs/version_*/metrics.csv`를 사용한다.

작업:

* 직전 버전 대비 metric 변화량(delta)을 요약한다.
* 정체/회귀 구간과 trade-off를 식별한다.
* `infer_samples`를 근거로 실패 패턴을 정리하고, 원인 가설을 도출한다.

결과(필수):

* `Models/Artifacts/v{cur:03d}/result-analysis.md`

`result-analysis.md` 최소 구성(MUST):

* `Summary`
* `Metric deltas` (v{cur-1:03d} → v{cur:03d})
* `Evidence` (infer_samples 근거 요약)
* `Hypotheses`
* `Next focus candidates`

---

### Step 4) 개선 행동 도출 (Action extract)

입력:

* `user_defined_rules.json` (최우선)
* `Models/Artifacts/v{cur:03d}/result-analysis.md`

작업:

* 다음 버전에 반영할 개선 행동을 **오직 1개만** 선정한다.
* 선정 기준:

  * 구현 리스크가 낮고
  * 성능 개선 가능성이 높으며
  * 변경 범위가 좁아 원인-결과 추적이 가능한 것
* action이 `user_defined_rules.training_config`의 제약과 충돌하면, **제약을 만족하도록 action을 수정**하거나 해당 action을 폐기하고 차선책을 선택한다.

결과(필수):

* `Models/Artifacts/v{cur:03d}/action-extract.md`

`action-extract.md` 최소 구성(MUST):

* `Selected Action (1개)`

  * `Goal`
  * `Change`
  * `Where`
  * `Expected impact`
  * `Risk`
  * `Verification`
* `Deferred Candidates` (선택)

---

### Step 5) 종료 조건 검사 (Loop)

입력:

* 현재 `cur`

작업:

* `cur >= user_defined_rules.json.loop_config.max_versions` 이면 루프를 종료하고 Step 7을 1회 실행한다.
* 아니면 Step 6으로 진행한다.

---

### Step 6) 다음 버전 작성 (Next version authoring)

입력:

* `user_defined_rules.json` (최우선)
* `02_train_v{cur:03d}.py`
* `Models/Artifacts/v{cur:03d}/action-extract.md`

작업:

* `02_train_v{cur:03d}.py`를 복사하여 `02_train_v{cur+1:03d}.py`를 만든다.
* `Selected Action`에 명시된 변경만 반영한다.
* 적용 시, `user_defined_rules.json`의 사용자 정의 제약을 **항상 우선 적용**한다.
* 파일 상단에 변경 요약을 남긴다.

결과:

* `02_train_v{cur+1:03d}.py`
* `cur += 1` 후 Step 1로 돌아가 반복

---

### Step 7) 최종 종합 리포트 생성 (Final report after loop ends)

입력:

* `user_defined_rules.json` (최우선)
* `Models/Artifacts/` 아래 존재하는 모든 `v###`
* `02_train_v###.py`

작업:

* 모든 루프 종료 후 1회 실행한다.
* 리포트 저장 위치는 **`Models/Report/YYYY-MM-DD_HH-MM/`** 를 사용한다.
* `04_final_report.py` 스크립트를 작성하여 `leaderboard.csv`, `final-analysis.md` 파일을 생성한다.
* `leaderboard.csv` 파일은 각 버전에 저장된 `eval_score.json` 을 파싱해 생성한다.
* `eval_score.json.metrics`의 모든 요소를 **컬럼으로 펼쳐(flatten)** `leaderboard.csv`에 기록한다.

  * 컬럼명 규칙: `metric__{name}` (예: `metric__loss`, `metric__wer`)
  * 동일한 `{name}`이 중복될 수 있으면 `metric__{name}__{k}` 처럼 suffix로 유일화한다.
  * 어떤 버전에 특정 metric이 없으면 해당 셀은 비워둔다(빈 값).
* `Models/external/` 폴더 안에 비교 평가를 위한 외부 모델이 있으면 해당 모델도 동일한 metric으로 평가를 진행한 후, `leaderboard.csv`에 추가한다. (inference_sample.py 스크립트를 참고해 해당 모델의 추론 방식을 파악)
  * 단, 외부 모델은 `version_id`를 폴더명으로 간주한다. (예: Models/external/glm-ocr -> verion_id=glm-ocr)
* `leaderboard.csv`는 아래 컬럼만 포함한다(그 외 컬럼 생성 금지).

  * `version_id`, `dataset_id`, `created_at`, `latency_ms`, `artifact_relpath`, `metric__{name}...`
* artifact_relpath는 `Models/Report/YYYY-MM-DD_HH-MM/` 기준 상대경로로 기록한다.
* **primary metric 기반 정렬/선정 규칙**

  * primary metric 이름은 `user_defined_rules.json.eval_config.primary_metric`를 사용한다.
  * `eval_score.json.metrics`에서 `name == primary_metric`인 항목을 찾아 그 `value`를 사용한다.
  * 정렬 방향은 해당 metric의 `higher_is_better`를 따른다.

    * `higher_is_better=true` → 내림차순(클수록 상위)
    * `higher_is_better=false` → 오름차순(작을수록 상위)
  * primary metric이 누락된 버전은 leaderboard의 최하위로 보낸다(정렬 시 뒤로 밀기) 그리고 `final-analysis.md`에 경고로 기록한다.
* best 버전은 위 primary metric 정렬 기준의 **1위**로 결정한다.
* best 버전의 아티팩트를 아래로 복사한다.

  * `Models/Report/YYYY-MM-DD_HH-MM/Artifacts/{best_version_id}/{best.ckpt, spec.json, additional_assets/...}`
* `final-analysis.md` 파일에 leaderboard 상위 3개(best 포함)의 학습 코드를 참조해 어떤 식으로 구현이 되어 높은 순위에 랭크한 것인지 분석한 내용을 담는다.

  * 참조 대상(권장): 상위 3개 버전의 `02_train_v{version_id}.py` + 각 버전의 `action-extract.md` + `result-analysis.md`

결과(필수):

* `04_final_report.py`
* `Models/Report/YYYY-MM-DD_HH-MM/final-analysis.md`
* `Models/Report/YYYY-MM-DD_HH-MM/leaderboard.csv`
* `Models/Report/YYYY-MM-DD_HH-MM/Artifacts/{best_version_id}/{best.ckpt, spec.json, additional_assets/...}`

`leaderboard.csv` 스키마(MUST):

* `version_id`
* `dataset_id`
* `created_at`
* `latency_ms`
* `artifact_relpath` (예: `Artifacts/v010/best.ckpt`)
* `metric__{name}` (metrics 전체 요소를 펼친 값 컬럼; name별 1개 이상)

예시(`leaderboard.csv`, 3줄):

```csv
version_id,dataset_id,created_at,latency_ms,artifact_relpath,metric__loss,metric__wer
v008,v1,2026-02-23T22:01:10,87,Artifacts/v008/best.ckpt,0.142,0.091
v009,v1,2026-02-23T22:01:10,86,Artifacts/v009/best.ckpt,0.137,0.095
v010,v1,2026-02-23T22:01:10,85,Artifacts/v010/best.ckpt,0.131,0.089
```

---

## 디렉토리 구조

### 버전 아티팩트

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

### 종합 리포트

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

## 규칙

### MUST

* 사용자가 `/progressive-training` 를 입력했거나 build-baseline Step 3에서 자동 트리거된 경우에만 실행한다.
* 실행 시 **`./user_defined_rules.json`을 최우선으로 읽고 적용**한다.
* `user_defined_rules.json`의 사용자 정의 제약과 충돌하는 변경은 적용하지 않는다.
* 버전 표기는 항상 `v{cur:03d}`를 사용한다.
* Step 0에서 dataset_id를 결정하고, 루프 종료까지 단일 값으로 고정한다.
* 모든 버전은 `Models/Artifacts/v###/` 구조를 따른다.
* `02_train_v###.py`는 `eval_score.json` 및 `infer_samples/`를 생성/수정하면 안 된다.
* `eval_score.json`은 **단일 JSON 객체**로 저장한다.
* `infer_samples`는 validation set에서 **최대 `infer_samples`(기본 10) 랜덤 샘플링**하며, **고정 시드**를 사용한다. 시드는 `user_defined_rules.json.seed`에서 참조하며, 없으면 **42**를 사용한다.
* 다음 버전 변경은 반드시 `result-analysis.md` → `action-extract.md`에서 도출된 **Selected Action 1개**만 반영한다.
* 루프 반복 횟수는 `user_defined_rules.json.loop_config.max_versions`로 제어하며, 도달 시 루프를 종료하고 Step 7을 1회 실행한다.
* 종합 리포트는 **`Models/Report/YYYY-MM-DD_HH-MM/`** 에 저장한다.
* 각 버전 아티팩트에는 추론/평가 재현에 필요한 전처리/후처리 자산(pre/post-processing assets)을 함께 저장한다. 필요 시 `additional_assets/`에 tokenizer/processor/label_map/vocab/normalizer/config 등을 포함하며, 해당 task에서 불필요한 경우 생략 가능하다(대신 `spec.json`에 `not_required`로 명시).

### SHOULD

* `infer_samples/{index}/meta.json`에 샘플 식별자/조건/설정 요약/사용 시드를 포함한다.
