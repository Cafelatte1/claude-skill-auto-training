---
name: progressive-training
description: 베이스라인 이후 모델을 run 단위로 격리하여 버전 단위로 반복 학습하고, 각 버전의 평가·분석 결과를 누적한 뒤 루프 종료 시 동일 프로토콜로 전체 버전 및 외부 모델을 비교 평가하여 run 리포트를 생성하는 수동 루프 스킬입니다. 실행 과정 전반에서 사용자 정의 규칙을 최우선으로 적용하며, 각 반복에서는 분석으로 도출한 단 하나의 저위험 개선 행동만 다음 버전에 반영합니다.
---

## When to use

* 사용자가 **`/progressive-training`** 를 입력했을 경우 혹은 build-baseline 작업이 성공적으로 완료된 후 진행한다.
* 베이스라인 이후, 모델/파이프라인을 **버전 단위(v001, v002, …)** 로 점진 개선할 때 사용한다.
* 실행 단위를 run으로 격리하고, run 안에서만 버전 스크립트/아티팩트/리포트를 누적하고 싶을 때 사용한다.
* 루프 종료 후 “현재 run에 존재하는 모든 버전” 및 외부 모델을 동일 프로토콜로 비교 평가하여 **리더보드/최종 리포트**를 만들 때 사용한다.

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

#### dataset_id 확정 규칙

* dataset_id는 다음 우선순위에 따라 설정하며, **Step 0에서 1회 결정 후 루프 종료까지 고정**한다.

  1. build-baseline에서 작업 완료하여 전달된 dataset_id
  2. `user_defined_rules.json.dataset_config.dataset_id`

#### run_id 확정 규칙 (핵심)

* 본 스킬은 실행 시작 시 **run_id를 1회 생성**하고, 루프 종료까지 고정한다.
* `run_id = YYYY-MM-DD_HH-MM_{counter}`
* 같은 YYYY-MM-DD_HH-MM에 대해 기존 Runs/{run_id}가 있으면 counter를 01,02,03…로 증가
* run_id는 아래 두 영역에 사용된다.

  * **버전 스크립트 폴더(프로젝트 루트):** `./Runs/{run_id}/02_train_v###.py`
  * **버전 아티팩트/리포트(Models 하위):**

    * `Models/Runs/{run_id}/Artifacts/v###/...`
    * `Models/Runs/{run_id}/Report/...` (yyyy-mm-dd 폴더 없음)

#### 경로 규칙 (MUST)

* 베이스라인 스크립트: `./02_train_baseline.py` (참조/복사 원본)
* 베이스라인 아티팩트: `Models/Artifacts/Baseline/...` (참조 전용)
* 외부 모델 루트: `Models/External/{model_path}/...` (참조 전용)
* run 스크립트 폴더: `./Runs/{run_id}/`

  * `./Runs/{run_id}/02_train_v{cur:03d}.py`
  * `./Runs/{run_id}/03_eval.py`
  * `./Runs/{run_id}/04_final_report.py`
* run 아티팩트 루트:

  * `Models/Runs/{run_id}/Artifacts/v{cur:03d}/...`
* run 리포트 루트:

  * `Models/Runs/{run_id}/Report/...`

#### 외부 모델 평가 캐시 규칙 (권장 표준)

* 외부 모델 평가 결과는 **run 단위로 캐시**한다.

  * `Models/Runs/{run_id}/Artifacts/External/{external_id}/eval_score.json`
  * (선택) `Models/Runs/{run_id}/Artifacts/External/{external_id}/infer_samples/...`
* `external_id`는 기본적으로 `{model_path}`를 안전한 문자열로 정규화한 값으로 사용한다.

  * 예: `Models/External/glm-ocr` → `external_id=glm-ocr`

예시:

```json
{
  "seed": 42,
  "loop_config": { "max_versions": 10 },
  "dataset_config": { "dataset_id": "v1" },
  "training_config": { "epochs": 3, "batch_size": 32 },
  "eval_config": {
    "primary_metric": "loss",
    "metrics": [
      {"name": "loss", "higher_is_better": false},
      ...
    ],
    "infer_samples": 10
  }
}
```

---

## External 모델 평가 인터페이스(계약)

### External 모델 폴더 규칙

* 외부 모델 폴더: `Models/External/{model_path}/`
* `inference_sample.py` (선택):

  * 외부 모델의 “샘플 단위 추론” 예시 구현 파일이다.
  * 존재한다면, **해당 구현을 참고해 `evaluation.py`를 작성/생성**한다.
* `evaluation.py` (권장/표준):

  * 외부 모델의 “validation 전체 평가 + infer_samples 생성 + eval_score.json 저장”을 수행하는 실행 엔트리포인트다.
  * 본 스킬은 외부 모델 평가를 `03_eval.py` 내부 분기가 아니라, **External의 `evaluation.py` 실행으로 위임**한다.

### `evaluation.py` 입력(표준)

`evaluation.py`는 아래 입력을 **CLI args 또는 환경변수** 중 하나로 받되, 스킬 구현에서는 CLI args를 표준으로 사용한다.

필수 입력(최소):

* `--dataset_id`
* `--run_id`
* `--external_id`
* `--output_dir`

  * 기본: `Models/Runs/{run_id}/Artifacts/External/{external_id}`
* `--seed`
* `--infer_samples` (정수, 0 가능)

선택 입력:

* `--metrics` (comma-separated)
* `--primary_metric`
* `--device` / `--precision` 등 외부 모델 자체 옵션(있을 경우)

### `evaluation.py` 출력(표준)

* `output_dir` 아래에 아래를 생성해야 한다.

  * `eval_score.json` (MUST, 스키마는 Step 2의 버전 평가와 동일)
  * (선택) `infer_samples/{index}/...`
  * (권장) `eval_notes.md` 또는 `spec.json` (외부 모델 로딩/추론/전처리 가정 기록)

---

## Step 0) 루프 초기화 (Initialize run + loop state)

목적:

* run_id 생성 및 run 폴더 구조 생성
* dataset_id 확정, cur 결정, v001 초기 스크립트 준비
* baseline을 run 구조에 맞게 “학습 시작 가능한 상태”로 전환한다.

입력:

* `./user_defined_rules.json`, `./user_instruction.md` (최우선)
* `dataset/prepdata/{dataset_id}/stats.json`
* `./02_train_baseline.py`
* `Models/Artifacts/Baseline/spec.json` (존재 시)
* `Models/Artifacts/Baseline/_tests/baseline_smoke_test_report.md` (존재 시)
* (선택) `/progressive-training?...&dataset_id=...` 쿼리 파라미터

작업:

1. **run_id 생성 및 폴더 준비 (MUST)**

   * 폴더 생성:

     * `./Runs/{run_id}/`
     * `Models/Runs/{run_id}/Artifacts/`
     * `Models/Runs/{run_id}/Report/`

2. **dataset_id 확정 (MUST)**

   * 공통 규칙에 따라 1회 확정하고 이후 루프 종료까지 고정한다.

3. **cur 결정 (MUST)**

   * `./Runs/{run_id}/02_train_v###.py`가 하나 이상 존재하면:

     * 가장 큰 버전을 `cur`로 설정하고, **Step 1(학습)** 은 그 `cur`부터 시작한다.
   * 하나도 없으면:

     * `cur = 1`로 설정하고(`v001`), 아래 4)~5)까지 수행한다.

4. **Baseline 사전 점검 (Baseline check)**

   * 목적: “run 기반 경로 정합성” 확보(성능 개선 목적 변경 금지)
   * 확인 항목(최소):

     * 데이터 경로가 `dataset/prepdata/{dataset_id}/...` 규칙을 따르는지
     * `DATASET_ID`/전역 상수가 규칙과 일치하는지
     * (중요) 아티팩트 저장 경로가 baseline 고정 경로(`Models/Artifacts/Baseline/`)를 사용하고 있는지
     * 스모크 테스트에서 확인된 `optimal_batch_size`를 반영 가능한지(가능하면 반영)
   * 학습 프로세스, 임포트 모듈, 사용자 정의 제약 등의 사항들이 적절한지 분석하고 문제를 발견하면 수정 사항을 기록한다.
   * 수정 사항 기록 파일(필수):

     * `Models/Artifacts/Baseline/_tests/baseline_patch_report.md`

5. **baseline → v001 스크립트 준비 (MUST)**

   * `./02_train_baseline.py`를 복사하여 `./Runs/{run_id}/02_train_v001.py`를 만든다.
   * `Models/Artifacts/Baseline/_tests/baseline_patch_report.md` 파일을 참고해 수정 사항이 있으면 `02_train_v001.py`에 적용한다.
   * v001 스크립트에는 아래 run 기반 경로를 반영해야 한다(필수):

     * `ARTIFACT_DIR = Models/Runs/{run_id}/Artifacts/v001`
     * logs, ckpt, additional_assets, spec.json, _tests 등도 모두 v001 아래로 생성되도록 한다.

결과(필수):

* `./Runs/{run_id}/02_train_v001.py`
* `Models/Runs/{run_id}/run_spec.json`

---

## Step 1) 학습 (Train)

목적:

* 훈련을 수행하고 `v{cur:03d}` 버전에 대한 아티팩트를 생성한다.
* 훈련 프레임워크 로깅은 허용하되, **평가 산출물은 생성하지 않는다.**

입력:

* `user_defined_rules.json` (최우선)
* `./Runs/{run_id}/02_train_v{cur:03d}.py`

작업:

* `user_defined_rules.json.training_config` 및 사용자 정의 제약을 **학습 설정에 우선 적용**한다.
* 학습을 실행하고 아래 경로에 아티팩트를 저장한다.

  * `Models/Runs/{run_id}/Artifacts/v{cur:03d}/...`
* `spec.json`은 재현성을 위해 반드시 생성한다.
* 전/후처리 자산(pre/post-processing assets)을 저장한다.
* 체크포인트는 `best.ckpt`, `last.ckpt`를 남긴다.

#### 학습 실패(에러/예외) 처리 규칙

* 예외 발생 시 해당 버전 아티팩트 디렉토리를 **초기화(삭제 후 재생성)** 한다.
* 로그/스택트레이스로 원인 요약 후, **동일 버전(v{cur:03d})** 스크립트만 수정하여 재시도한다(버전 스킵 금지).
* (권장) 재시도 이력:

  * `Models/Runs/{run_id}/Artifacts/v{cur:03d}/_tests/train_retry_report.md`

결과(필수):

* `Models/Runs/{run_id}/Artifacts/v{cur:03d}/best.ckpt`
* `Models/Runs/{run_id}/Artifacts/v{cur:03d}/last.ckpt`
* `Models/Runs/{run_id}/Artifacts/v{cur:03d}/spec.json`
* `Models/Runs/{run_id}/Artifacts/v{cur:03d}/additional_assets/`
* `Models/Runs/{run_id}/Artifacts/v{cur:03d}/logs/version_#/metrics.csv`
* `Models/Runs/{run_id}/Artifacts/v{cur:03d}/logs/version_#/hparams.yaml`

---

## Step 2) 평가 (Evaluation)

목적:

* 학습이 끝난 모델(`best.ckpt`)로 **validation set만** 사용해 평가를 수행하고,
* 현재 버전의 최종 메트릭 점수(`eval_score.json`) 및 실제 추론을 확인할 수 있는 자료(`infer_samples/`)를 생성한다.
* **`03_eval.py` 최초 생성 시 외부 모델을 선평가**하여 run 내 비교 기준을 확보한다.
* 외부 모델 평가는 `03_eval.py` 내부 분기가 아니라, **External 폴더의 `evaluation.py` 실행으로 위임**한다.

입력:

* `user_defined_rules.json` (최우선)
* `./Runs/{run_id}/02_train_v{cur:03d}.py`
* `Models/Runs/{run_id}/Artifacts/v{cur:03d}/best.ckpt`

작업:

### 1. 평가 스크립트 준비 (MUST)

* `./Runs/{run_id}/03_eval.py`가 없다면 생성한다.
* `03_eval.py`는 최소 아래를 지원해야 한다.

  * run_id, dataset_id, version_id를 입력받아 평가 실행
  * 현재 버전 모델 평가 + `eval_score.json` 저장
  * `infer_samples/` 생성
  * (최초 생성 시 또는 캐시 미존재 시) 외부 모델 평가를 수행하되, **External의 `evaluation.py`를 호출**하여 캐시 저장

### 2. 현재 버전 eval_score.json 생성 (MUST)

* validation set 전체를 활용하여 메트릭을 계산한다.
* `eval_config.metrics`이 없으면 기존 학습 시 사용된 loss 값만 사용한다.
* 버전당 **단일 JSON 객체**로 저장한다. (`created_at`는 ISO-8601)
* 저장 경로:

  * `Models/Runs/{run_id}/Artifacts/v{cur:03d}/eval_score.json`
* 스키마(MUST):

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

### 3. infer_samples 생성 (MUST)

* validation set에서 최대 `infer_samples` 만큼 seed 기반 샘플링하여 추론 후 저장한다.
* 저장 경로:

  * `Models/Runs/{run_id}/Artifacts/v{cur:03d}/infer_samples/{index}/...`
* 각 `{index}` 폴더 MUST:

  * `input.*`, `output.*`, `meta.json`
* Optional:

  * `gt.*`, `viz.*`, `summary.*`

### 4. 외부 모델 선평가 (MUST: 최초 생성 시 / SHOULD: 캐시 미존재 시)

#### 트리거 조건

* `./Runs/{run_id}/03_eval.py`를 **이번 Step에서 새로 생성한 경우** 반드시 수행한다.
* 또는 `Models/Runs/{run_id}/Artifacts/External/*/eval_score.json` 캐시가 없다면 수행한다(권장).

#### 대상

* `Models/External/*` 하위의 각 외부 모델 폴더

#### 실행 규칙(핵심 변경)

각 외부 모델 폴더에 대해 아래 우선순위를 적용한다.

1. **캐시 존재 시 스킵 (MUST)**

* `Models/Runs/{run_id}/Artifacts/External/{external_id}/eval_score.json` 가 존재하면 평가를 생략한다.

2. **`evaluation.py` 존재 시 실행 (MUST)**

* `Models/External/{model_path}/evaluation.py` 가 존재하면 이를 실행하여 캐시를 생성한다.

3. **`evaluation.py` 없고 `inference_sample.py` 존재 시: evaluation.py 생성 후 실행 (MUST)**

* `Models/External/{model_path}/inference_sample.py` 가 존재하면,

  * 이를 참고하여 `Models/External/{model_path}/evaluation.py` 를 생성한다(또는 기존 템플릿을 보강한다).
  * 생성된 `evaluation.py` 를 실행하여 캐시를 만든다.

4. **둘 다 없으면: 평가 불가 처리 (MUST)**

* `eval_notes.md`에 “평가 불가 사유(엔트리포인트 부재)”를 기록하고 해당 외부 모델은 리더보드에서 제외될 수 있음을 Step 7에 경고로 남긴다.

#### 캐시 출력(권장 표준)

* `Models/Runs/{run_id}/Artifacts/External/{external_id}/eval_score.json`
* (선택) `Models/Runs/{run_id}/Artifacts/External/{external_id}/infer_samples/...`
* 외부 모델별 로딩/추론 가정은 아래 중 하나로 기록(권장):

  * `Models/Runs/{run_id}/Artifacts/External/{external_id}/eval_notes.md`
  * 또는 `.../spec.json`

결과(필수):

* `./Runs/{run_id}/03_eval.py`
* `Models/Runs/{run_id}/Artifacts/v{cur:03d}/eval_score.json`
* `Models/Runs/{run_id}/Artifacts/v{cur:03d}/infer_samples/{index}/...` (최대 infer_samples)

---

## Step 3) 결과 분석 (Result analysis)

입력:

* `Models/Runs/{run_id}/Artifacts/v{cur:03d}/eval_score.json`
* `Models/Runs/{run_id}/Artifacts/v{cur:03d}/infer_samples/{index}/...`
* (선택) `Models/Runs/{run_id}/Artifacts/v{cur:03d}/logs/version_*/metrics.csv`

작업:

* 직전 버전 대비 metric 변화량(delta)을 요약한다.
* 정체/회귀 구간과 trade-off를 식별한다.
* `infer_samples`를 근거로 실패 패턴을 정리하고, 원인 가설을 도출한다.

결과(필수):

* `Models/Runs/{run_id}/Artifacts/v{cur:03d}/result-analysis.md`
* 최소 구성(MUST):

  * `Summary`
  * `Metric deltas (v{cur-1:03d} → v{cur:03d})`
  * `Evidence`
  * `Hypotheses`
  * `Next focus candidates`

---

## Step 4) 개선 행동 도출 (Action extract)

입력:

* `Models/Runs/{run_id}/Artifacts/v{cur:03d}/result-analysis.md`

작업:

* 다음 버전에 반영할 개선 행동을 **오직 1개만** 선정한다.
* 선정 기준:

  * 구현 리스크가 낮고
  * 성능 개선 가능성이 높으며
  * 변경 범위가 좁아 원인-결과 추적이 가능한 것
* action이 `user_defined_rules.training_config`의 제약과 충돌하면,

  * 제약을 만족하도록 action을 수정하거나 action을 폐기하고 차선책을 선택한다.

결과(필수):

* `Models/Runs/{run_id}/Artifacts/v{cur:03d}/action-extract.md`
* 최소 구성(MUST):

  * `Selected Action (1개)`: `Goal`, `Change`, `Where`, `Expected impact`, `Risk`, `Verification`
  * `Deferred Candidates` (선택)

---

## Step 5) 종료 조건 검사 (Loop)

작업:

* `cur >= user_defined_rules.loop_config.max_versions` 이면 루프를 종료하고 Step 7을 1회 실행한다.
* 아니면 Step 6으로 진행한다.

---

## Step 6) 다음 버전 작성 (Next version authoring)

입력:

* `./Runs/{run_id}/02_train_v{cur:03d}.py`
* `Models/Runs/{run_id}/Artifacts/v{cur:03d}/action-extract.md`

작업:

* `./Runs/{run_id}/02_train_v{cur:03d}.py`를 복사하여 `./Runs/{run_id}/02_train_v{cur+1:03d}.py`를 만든다.
* `Selected Action`에 명시된 변경만 반영한다.
* 반드시 v{cur+1:03d}에 맞게 아티팩트 저장 경로를 갱신한다.

  * `Models/Runs/{run_id}/Artifacts/v{cur+1:03d}/...`
* 파일 상단에 변경 요약을 남긴다.

결과:

* `./Runs/{run_id}/02_train_v{cur+1:03d}.py`
* `cur += 1` 후 Step 1로 돌아가 반복

---

## Step 7) 최종 종합 리포트 생성 (Final report after loop ends)

입력:

* `user_defined_rules.json` (최우선)
* `Models/Runs/{run_id}/Artifacts/` 아래 존재하는 모든 `v###`
* `./Runs/{run_id}/02_train_v###.py`
* (선택) `Models/Runs/{run_id}/Artifacts/External/*/eval_score.json` (외부 모델 캐시)

작업:

1. **리포트 저장 위치 고정 (MUST)**

* `Models/Runs/{run_id}/Report/` 를 사용한다. (yyyy-mm-dd 폴더 생성 금지)

2. **04_final_report.py 생성 및 실행 (MUST)**

* 스크립트 위치: `./Runs/{run_id}/04_final_report.py`
* 산출물:

  * `Models/Runs/{run_id}/Report/leaderboard.csv`
  * `Models/Runs/{run_id}/Report/final-analysis.md`

3. **leaderboard.csv 생성 규칙 (MUST)**

* 버전 eval 수집:

  * `Models/Runs/{run_id}/Artifacts/v###/eval_score.json`

* 외부 모델 eval 수집:

  * 원칙: `Models/Runs/{run_id}/Artifacts/External/*/eval_score.json` 캐시를 사용한다.
  * 캐시가 없다면, `./Runs/{run_id}/03_eval.py`를 호출해 보충 평가를 수행할 수 있다(권장).

    * 이 보충 평가는 `03_eval.py`가 External의 `evaluation.py`를 호출하는 방식으로 수행해야 한다.

* `eval_score.json.metrics`의 모든 요소를 **컬럼으로 펼쳐(flatten)** `leaderboard.csv`에 기록한다.

  * 컬럼명 규칙: `metric__{name}`
  * name 중복 시 `metric__{name}__{k}`로 유일화

* 포함 컬럼 제한(MUST):

  * `version_id`, `dataset_id`, `created_at`, `latency_ms`, `artifact_relpath`, `metric__{name}...`

* `artifact_relpath` 기준:

  * run 리포트 폴더(`Models/Runs/{run_id}/Report/`) 기준 상대경로를 기록한다.
  * 예:

    * 학습 버전: `../Artifacts/v010/best.ckpt`
    * 외부 모델: `../Artifacts/External/glm-ocr/` (폴더 기준 허용)

4. **primary metric 기반 정렬/선정 (MUST)**

* primary metric 이름: `user_defined_rules.json.eval_config.primary_metric`
* 정렬 방향: 해당 metric의 `higher_is_better`를 따른다.
* primary metric 누락 버전은 최하위로 보낸다 + `final-analysis.md`에 경고 기록
* best 버전: 정렬 1위

5. **best 아티팩트 복사 (MUST)**

* best 버전의 아티팩트를 아래로 복사한다.

  * `Models/Runs/{run_id}/Report/Artifacts/{best_version_id}/{best.ckpt, spec.json, additional_assets/...}`

6. **final-analysis.md 내용 (MUST)**

* leaderboard 상위 3개(best 포함)에 대해:

  * 어떤 구현 변화가 순위 상승에 기여했는지 분석
* 참조 대상(권장):

  * `./Runs/{run_id}/02_train_v###.py`
  * `Models/Runs/{run_id}/Artifacts/v###/action-extract.md`
  * `Models/Runs/{run_id}/Artifacts/v###/result-analysis.md`

결과(필수):

* `./Runs/{run_id}/04_final_report.py`
* `Models/Runs/{run_id}/Report/final-analysis.md`
* `Models/Runs/{run_id}/Report/leaderboard.csv`
* `Models/Runs/{run_id}/Report/Artifacts/{best_version_id}/...`

예시(`leaderboard.csv`, 3줄):

```csv
version_id,dataset_id,created_at,latency_ms,artifact_relpath,metric__loss,metric__wer
v008,v1,2026-02-23T22:01:10,87,../Artifacts/v008/best.ckpt,0.142,0.091
v009,v1,2026-02-23T22:01:10,86,../Artifacts/v009/best.ckpt,0.137,0.095
v010,v1,2026-02-23T22:01:10,85,../Artifacts/v010/best.ckpt,0.131,0.089
```

---

## 디렉토리 구조

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

## 규칙

### MUST

* 사용자가 `/progressive-training` 를 입력했거나 build-baseline Step 3에서 자동 트리거된 경우에만 실행한다.
* 실행 시 **`./user_defined_rules.json`을 최우선으로 읽고 적용**한다.
* `run_id = YYYY-MM-DD_HH-MM_{counter}`를 Step 0에서 1회 생성하고 루프 종료까지 고정한다.
* 버전 스크립트는 반드시 `./Runs/{run_id}/02_train_v###.py`에 저장한다.
* 버전 아티팩트는 반드시 `Models/Runs/{run_id}/Artifacts/v###/`에 저장한다.
* 리포트는 반드시 `Models/Runs/{run_id}/Report/`에 저장한다(날짜 폴더 생성 금지).
* baseline은 참조 전용으로 `./02_train_baseline.py`와 `Models/Artifacts/Baseline/`를 사용한다.
* 외부 모델은 참조 전용으로 `Models/External/{model_path}/...`를 사용한다.
* `02_train_v###.py`는 `eval_score.json` 및 `infer_samples/`를 생성/수정하면 안 된다.
* `eval_score.json`은 **단일 JSON 객체**로 저장한다.
* `infer_samples`는 validation set에서 **최대 `infer_samples` 랜덤 샘플링**하며, 시드는 `user_defined_rules.json.seed`(없으면 42)를 사용한다.
* 다음 버전 변경은 반드시 `result-analysis.md` → `action-extract.md`에서 도출된 **Selected Action 1개**만 반영한다.
* 종합 리포트 생성 시 외부 모델 비교는 run 캐시(`Artifacts/External/...`)를 우선 사용한다.
* 외부 모델 평가는 `03_eval.py` 내부 분기가 아니라, **External 폴더의 `evaluation.py` 실행으로 수행**한다.
* 외부 모델 폴더에 `inference_sample.py`가 존재하고 `evaluation.py`가 없다면, `evaluation.py`를 생성(또는 보강)한 뒤 실행하여 `eval_score.json`을 생성해야 한다.

### SHOULD

* `Models/Runs/{run_id}/run_spec.json`에 run 메타(run_id, dataset_id, seed, max_versions, External_models 목록)를 저장한다.
* 외부 모델 평가 시 `Models/Runs/{run_id}/Artifacts/External/{external_id}/eval_notes.md`에 로딩/추론/전처리 가정 및 실행 커맨드를 기록한다.
* `infer_samples/{index}/meta.json`에 샘플 식별자/조건/설정 요약/사용 시드를 포함한다.
