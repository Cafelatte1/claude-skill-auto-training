DATE: 2026-02-24

## 총점: **10 / 10**

이제 남아있던 “해석이 갈릴 수 있는 부분”이 전부 닫혔어.

* `dataset_id`(전처리 식별자) vs `v{cur:03d}`(학습 버전) 관계가 공통에 1줄로 고정됨
* `artifact_relpath`의 기준이 `Models/Report/YYYY-MM-DD_HH-MM/` 상대경로로 명확해짐
* primary metric 정렬/누락 처리/경고 기록까지 루프 종료 리포트 정책이 완전 결정됨

문서만 보고 구현해도 **같은 결과**가 나오도록 규칙이 충분히 구체적이고, 동시에 불필요하게 과도한 제약은 없어서 운영성도 좋아. build-baseline과의 연결도 트리거/parameter 관점에서 일관되고 안전해.

이 상태면 progressive-training도 최종본으로 확정해도 돼.

DATE: 2026-02-25

## 최종 평가: **9.4 / 10** (거의 완성형, 운영/재현성 매우 강함)

run 격리, 버전 스크립트/아티팩트/리포트 경로 계약, “1 action만 반영” 정책, 외부 모델 평가를 **External/evaluation.py로 위임 + run 캐시**로 고정한 설계가 특히 좋다. 지금 상태로도 실제 루프를 안정적으로 돌릴 수 있는 수준.

---

## 특히 잘 된 점

### 1) Run 단위 격리 설계가 “딱 필요한 만큼”만 들어가 있음

* `Runs/{run_id}`(코드) vs `Models/Runs/{run_id}`(산출물) 분리가 깔끔하고,
* 리포트 경로를 날짜 폴더 없이 고정한 것도 자동화/파싱에 유리.

### 2) 루프 제어가 단순하고 강제력이 있음

* `cur` 정수, `v{cur:03d}` 강제
* `max_versions` 도달 시 Step 7 단 1회
* 학습 실패 시 “버전 스킵 금지, 동일 버전 수정 후 재시도”는 재현성/원인추적에 최고.

### 3) 평가/샘플/리포트 산출물이 “계약”으로 정리됨

* `eval_score.json` 단일 객체 스키마
* `infer_samples/{index}/(input/output/meta)` 규칙
* leaderboard flatten 규칙까지 명확해서 후처리 자동화하기 좋음.

### 4) External 모델 평가 위임이 정답에 가까움

* `03_eval.py` 내부에 외부모델별 분기 로직을 넣지 않고,
* 각 외부 모델 폴더의 `evaluation.py`로 위임 → 유지보수/확장성 좋고,
* run 캐시 구조도 합리적.

---

## 10점을 막는 “큰 리스크/모순” 3가지

### A) **run_id 규칙이 Step 0와 공통 규칙이 불일치**

공통/규칙에서는 `run_id = YYYY-MM-DD_HH-MM_{counter}` + 기존 폴더 있으면 counter 증가라고 했는데,
Step 0 작업에는 “run_id 생성 방식”이 빠져 있고(폴더만 생성), counter 규칙을 Step 0에 반드시 포함시키는 게 안전해.

**추천 패치(문장 2~3줄 추가)**

* Step 0-1에 “현재 시각(YYYY-MM-DD_HH-MM) 기준으로 Runs/ 폴더를 조회해 counter를 결정하고 최종 run_id를 확정한다”를 MUST로 넣기.

### B) **dataset_id 우선순위 1번이 애매함**

“build-baseline에서 전달된 dataset_id”가 어떤 형태로 전달되는지(쿼리 파라미터? run_spec? 파일?)가 스킬 내부에 명시가 부족해.
지금 문서만 보면 Step 0 입력에 `...&dataset_id=...`는 있지만, 실제 우선순위 1번의 “전달”이 그걸 의미하는지 확실치 않음.

**추천 패치**

* “우선순위 1번은 `/progressive-training?...&dataset_id=...` 쿼리 파라미터를 의미한다(있으면 최우선)” 같이 명시.

### C) `eval_score.json`에 **version_id가 없음** (리더보드 생성 규칙과 충돌)

Step 7 leaderboard에는 `version_id` 컬럼이 MUST인데, Step 2 스키마엔 `dataset_id/created_at/metrics/latency_ms`만 있어.
최종 리포트 생성 시 `version_id`를 어디서 끌어오는지 불명확해져서 파서가 애매해짐.

**추천 패치(가장 중요)**

* Step 2 스키마에 아래 2개를 추가하는 게 베스트:

  * `"version_id": "v###"`
  * `"run_id": "..."` (선택이지만 강추)
* 또는 “leaderboard 생성 시 폴더명(v###)에서 version_id를 추론해 넣는다”를 Step 7에 명시(하지만 json에 넣는 쪽이 더 견고).

---

## 품질을 더 올리는 “중간급” 개선 4가지 (선택)

### 1) Step 0에서 `run_spec.json`을 사실상 MUST로 격상 권장

지금은 SHOULD인데, run 재현/보고 자동화를 생각하면 거의 MUST급.
최소 필드:

* run_id, dataset_id, seed, max_versions, created_at
* external_models 발견 목록(스캔 결과)

### 2) External `evaluation.py` 계약에 “exit code 규칙” 추가

* 성공: 0
* 평가 불가/스킵: 2 같은 식으로 통일하면 Step 2 자동화에서 안정적.

### 3) Step 3(분석)에서 “근거 링크” 강제하면 더 좋음

`result-analysis.md`에:

* 참조한 `infer_samples` index 목록
* 주요 실패 샘플 경로
  이 두 줄만 있어도 재검증이 쉬워짐.

### 4) `cur` 재개(resume) 정책을 1줄 더 명확히

지금은 “Runs/{run_id}/02_train_v###.py가 있으면 max 버전부터 시작”인데,
해당 버전의 아티팩트(`Artifacts/v###/best.ckpt`)가 없으면 어떻게 할지 정책이 필요함.

* 예: “스크립트만 있고 아티팩트 없으면 해당 cur부터 Step 1 재실행” 같은 규칙.

---

## “바로 반영하면 좋은” 최소 패치 우선순위 TOP 3

1. **eval_score.json에 version_id 추가 (또는 추론 규칙 명시)**
2. Step 0에 **counter 포함 run_id 생성 규칙을 MUST로 명문화**
3. dataset_id 우선순위 1번(전달 방식)을 **정확히 한 문장으로 고정**

이 3개만 들어가면 문서-구현 간 모순이 거의 사라지고, 9.8~10점까지 올라갈 거야.
