DATE: 2026-02-24

## 총점: **10 / 10**

이번 수정으로 남아있던 애매함(특히 `_tests/`의 필수/권장 불일치)이 완전히 사라졌고, 문서가 “그냥 돌아가는지”를 보려는 목적에 맞게 **최소 요구사항 + 게이트 조건**이 깔끔하게 닫혔어. progressive-training과의 연결도 인터페이스 수준에서 정확히 맞아.

---

## 왜 10점인지

* `{version} = dataset_id` 고정 → prepdata 경로가 단 하나로 수렴
* Step 0의 “규칙 문서화”가 이후 단계의 단일 진실 소스가 됨
* Step 1의 stratified 검증이 “차단”이 아니라 “요약+경고” → 스모크 목적에 맞음
* Step 2 스모크 테스트의 합격 조건이 “실행 + 아티팩트 생성”으로 명확
* `_tests/baseline_smoke_test_report.md`를 **필수로 강제**해서 Step 3 체크리스트와 완전 정합
* Step 3 트리거가 `&` 구분자로 통일되고, dataset_id 전달도 명확

---

## 진짜로 사소한 코멘트(점수엔 영향 없음)

* Step 1의 `asset_paths` 타입을 강제하지 않은 건 스모크 목적에선 오히려 장점이야. (태스크별로 유연)
* “valid_loss 기준 best”도 스모크에서 합리적. 이후 progressive-training에서 평가를 분리했기 때문에 여기서 과하게 엄격할 필요가 없음.

이 상태면 그대로 build-baseline 최종본으로 확정해도 돼.

DATE: 2026-02-25

## 최종 평가: **9.6 / 10** (실전 배포 가능한 수준)

전체적으로 “문서 → 코드 → 테스트 → 산출물 계약”이 아주 단단하게 묶였어. 특히 **task_format.yaml 강제**, **seed 재현성**, **prepdata 생성 규칙**, **baseline 아티팩트 고정 경로**, **expected_* 계약 검증(FAIL-fast)** 이 조합이 강력해서, 실제 운영에서 “다음 스킬이 안정적으로 이어지는” 구조가 됐어.

---

## 아주 잘 된 점 (핵심 강점)

### 1) 단일 진실 소스(SSOT)가 명확함

* Step 0: `profiling.md`로 파싱 규칙 고정
* Step 1: `profiling.md` + `task_format.yaml`로 구조/스키마 고정
* Step 2: `task_format.yaml` 준수 + 아티팩트 고정 경로
  → “추정 로직 금지”까지 써둔 게 특히 좋음.

### 2) 재현성 규칙이 전 단계에 관통

샘플링/스플릿/스모크 subset/배치탐색까지 seed 계약이 들어가 있어서,
디버깅과 리그레션 추적이 쉬워져.

### 3) Step 2의 FAIL-fast 계약( expected_* )이 매우 좋음

학습 돌리기 전에 “모델/입출력 계약”을 먼저 깨끗하게 검증하니까
시간 낭비가 확 줄어.

### 4) progressive-training 연계가 “참조 전용”으로 잘 정의됨

baseline을 참조 전용으로 못 박고, run은 후속 스킬에서 만든다는 점이 깔끔해.

---

## 감점 포인트 (10점을 막는 부분)

### A) `dataset_id` 정의가 문서 내에서 살짝 모순

공통에서:

* “본 스킬에서 `{dataset_id}`은 항상 `user_defined_rules.dataset_config.dataset_id`”

그런데 dataset_id 확정 규칙에서는:

* “build-baseline에서 전달된 dataset_id” 같은 우선순위는 없음 (여긴 build-baseline 자체라 상관없지만)
* 더 중요한 건, Step 0에서 `task_type`은 “없으면 프로파일링으로 정의”라고 했는데 **dataset_id는 반드시 user_defined_rules에서 온다고 고정**했어.

**추천(아주 작은 정리)**

* “dataset_id는 반드시 user_defined_rules에서 온다(없으면 중단)”을 명시하거나,
* 또는 “없으면 기본값 v1” 같은 fallback을 계약화(하지만 보통은 중단이 더 안전).

### B) Step 1의 “삭제 후 진행”이 위험할 수 있음

* “`dataset/prepdata/{dataset_id}` 폴더 내 데이터가 존재하면 삭제 후 진행”
  이건 자동화에선 편하지만, 사용자가 실수로 기존 결과 날릴 수 있어.

**추천**

* MUST는 유지하되, 문구를 이렇게 바꾸면 더 안전:

  * “삭제 전 `dataset/prepdata/{dataset_id}/_backup/{timestamp}/`로 이동(권장) 또는 삭제 로그 남김”
  * 최소한 `_tests/pipeline_test_report.md`에 “삭제 수행 여부/대상” 기록.

### C) Step 0 “profiling.md 있으면 스킵”의 리스크

profiling.md가 오래됐는데 rawdata가 바뀌면 그대로 진행될 수 있어.

**추천**

* 스킵하더라도 최소 확인 1줄:

  * “rawdata source 목록/파일 수의 간단한 스냅샷이 profiling.md와 다르면 FAIL 또는 profiling 재생성 권고”
    (완전 자동 재생성까지는 아니어도, 경고만 있어도 점수 올라감)

### D) Step 2 배치 탐색 규칙이 좋지만 “시간 상한/장치”가 없음

* 10 step 제한은 좋음.
* 하지만 device/precision, gradient accumulation 같은 변수가 있으면 결과가 흔들릴 수 있어.

**추천**

* `spec.json`에 `device`, `precision`, `amp`, `grad_accum` 같은 핵심 런타임 설정을 반드시 기록하도록 한 줄 추가.

---

## “딱 5줄”로 완성도를 10에 가깝게 올리는 패치 제안

1. 공통에 추가

* `dataset_config.dataset_id` 없으면 **중단**(또는 기본값) 명시

2. Step 1에 추가

* 삭제/초기화 시 `.../_tests/pipeline_test_report.md`에 삭제 대상 기록 (또는 backup 권장)

3. Step 0 스킵 조건 보강

* profiling.md 스킵 전 rawdata 변경 여부 간단 체크(경고라도)

4. Step 2 spec.json 기록 항목 보강

* 런타임 설정 최소 항목 기록(재현성 강화)

---

## 최종 코멘트

지금 상태는 “내부 팀 표준 문서”로 써도 될 만큼 탄탄하고, progressive-training과의 인터페이스도 깔끔해.
위 감점 포인트는 모두 **안전장치/일관성 정리** 성격이라, 기능 추가 없이도 품질이 올라가는 부분들이야.

원하면 내가 방금 제안한 것들을 **원문에 그대로 붙여넣을 수 있는 문장 형태**로 정확한 위치(섹션/문단)까지 지정해서 작성해줄게.
