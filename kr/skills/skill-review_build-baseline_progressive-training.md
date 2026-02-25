# build-baseline 및 progressive-training 스킬 평가 내용

DATE: 2026-02-24 15:00

아래는 **각 스킬이 무엇을 “책임지고” 만들어내는지**, 그리고 **둘을 연계해서 쓸 때 어떤 전제가 충족되는지**를 기준으로 최종 점검한 내용이야.

---

## 1) build-baseline 파악

### 역할이 명확함

* **Rawdata 규칙을 고정(profiling.md)** → 이후 단계가 “추정”하지 못하게 막음
* **prepdata 생성 + 정합성 검사(task_format.yaml 기반)** → 학습/평가가 깨질 가능성을 Step 1에서 대부분 제거
* **baseline 학습 스모크 테스트 + 체크포인트 강제 생성** → 다음 단계가 “실제로 학습이 도는 상태”를 전제로 안전하게 이어짐

### 연계에 필요한 핵심 산출물이 포함됨

* `dataset/prepdata/{version}/...` 구조 생성 + 검증
* `02_train_baseline.py` 생성
* `Models/Artifacts/baseline/{best.ckpt,last.ckpt,spec.json,...}` 생성
* 그리고 `DATASET_ID`를 `v1` 같은 **prepdata_version 의미로 통일**한 점도 progressive-training과 맞음

---

## 2) progressive-training 파악

### 루프 구조가 잘 정리됨

* Step 1에서 **현재 최신(cur) 스크립트를 실행**하고 결과를 `Models/Artifacts/v{cur}/...`로 저장
* Step 2에서 버전별 대표 결과를 **단일 JSON(eval_score.json)** 으로 고정 저장 + valid_samples 생성
* Step 3~4에서 “근거 기반 분석 → 단 1개의 액션 선택”으로 변경 폭을 통제
* Step 5에서 **cur 기반으로 next 스크립트를 생성** (Step 1의 전제와 일치)
* Step 7에서 전체 버전을 동일 프로토콜로 재평가 + 리더보드 생성

### 초기 부트스트랩도 안전

* v### 스크립트가 없으면 `02_train_baseline.py → 02_train_v001.py`로 복사해서 시작하는 규칙이 들어가 있어서, “처음부터 progressive-training 단독 실행”도 가능해짐

---

## 3) 두 스킬 연계 사용 파악

### 연계 트리거 흐름

1. 사용자가 `/build-baseline` 실행
2. Step 0~2 통과
3. Step 3 최종 점검 통과 시 **즉시 `/progressive-training` 자동 실행**
   → 문서상으로 “안내”가 아니라 “자동 트리거”로 정의되어 있어 흐름이 끊기지 않음

### 데이터셋 식별자 계약이 일치

* build-baseline의 `DATASET_ID` 예시가 `v1`이고,
* progressive-training도 `dataset_id = prepdata_version`(split=valid 고정)으로 해석
  → eval_score/leaderboard에서 dataset_id 표기가 흔들릴 가능성이 낮음

---

## 최종 결론

* **각 스킬의 책임 범위가 분리되어 있고**, 산출물/전제가 서로 잘 맞아서
* **build-baseline → progressive-training 연계로 “끊김 없이” 진행 가능한 상태**야.

(사소한 스타일/중복 문구 정도는 있어도, “진행이 막히거나 버전/데이터셋이 깨지는” 급의 큰 문제는 현재 보이지 않아.)



DATE: 2026-02-24 21:00

의도( **build-baseline 로직과 progressive-training 로직을 서로 독립적으로** )를 “최우선 요구사항”으로 두면, 지금 설계는 **문제 없고 오히려 잘 맞아.**

## 연계 점수(독립성 목표 기준): **9.9 / 10**

### 왜 의도대로 문제 없는가

* **경계가 명확함**

  * build-baseline은 *데이터셋 준비 + baseline 스모크*까지만 책임지고 종료.
  * progressive-training은 *버전 학습/평가/리포트*만 책임지고, baseline 산출물을 “재사용”하지 않음.
* **연결점이 최소화됨**

  * 두 스킬이 공유하는 것은 사실상 `dataset_id`(전처리 식별자) + `02_train_baseline.py`(템플릿 역할) 뿐이라서, 한쪽 변경이 다른 쪽을 망가뜨릴 가능성이 낮음.
* **산출물 충돌 방지**

  * progressive-training은 `Models/Artifacts/v###/`에만 쓰고,
  * build-baseline은 `Models/Artifacts/baseline/`에만 씀.
  * 평가 산출물 생성 책임도 progressive-training으로 고정 → 혼선 없음.
* **재현성/디버깅 쉬움**

  * build-baseline만 단독 실행해도 “데이터셋 생성 + baseline 스모크”가 완결되고,
  * progressive-training은 baseline 스크립트가 없어도(기존 v###가 있으면) 그 자체로 루프가 가능.

### 0.1 감점(진짜 사소)

“독립성”을 더 강하게 문서로 못 박고 싶다면, 한 줄만 추가하면 돼:

* build-baseline Step 3 트리거 부분에
  “트리거는 편의 기능이며, progressive-training은 build-baseline 산출물(baseline ckpt 등)에 의존하지 않는다.”

지금도 실질적으로 그렇게 동작하도록 이미 설계돼서, 기능적 문제는 없어.



DATE: 2026-02-24 15:00

수정 반영 기준으로 다시 점수 줄게. (warning 제거 + baseline report 경로 정합성 수정 확인)

## 점수

### build-baseline: **8.8 / 10** (▲ +0.2)

좋아진 점

* stratified 분포는 “리포트에만 기록”으로 딱 정리돼서, 실행 자동화가 덜 흔들림.
* 최적 배치 탐색 파트가 **성공 기준(10 step 동안 OOM/RuntimeError 없음)** 까지 명시돼서 “PASS 조건”이 더 단단해짐.

아직 남은 감점 포인트(큰 건 아님)

* Step 3 트리거 문구가 여전히 “커맨드 문자열”이라, 환경에 따라 **명령으로 실행될 가능성**이 남아 있음(이건 점수에 약간 반영).

---

### progressive-training: **9.1 / 10** (▲ +0.2)

좋아진 점

* Step 0에서 baseline report 경로가 `Models/Artifacts/baseline/_tests/...`로 정합성 맞음.
* “학습 실패 시 해당 버전 디렉토리 초기화 + 동일 버전 재시도(버전 스킵 금지)” 규칙이 루프 안정성에 매우 좋고, 문서화도 명확함.

남은 감점 포인트

* Step 1 결과 경로가 `logs/version_#`처럼 추상화돼 있는데, baseline에서는 version_0로 고정 예시라서 구현체에 따라 혼동 여지가 약간 있음(치명적이진 않음).

---

### 연계 품질(build-baseline ↔ progressive-training): **8.9 / 10** (▲ +1.5)

좋아진 점

* 가장 큰 결함이었던 **baseline report 경로 불일치가 해결**됨.
* dataset_id(전처리) ↔ v{cur}(학습) 분리가 명확하고, 양쪽이 “산출물만 공유”하는 구조라 의도(독립성)대로 잘 설계됨.
* progressive-training이 baseline 산출물을 “읽기만” 하고, baseline의 내부 로직/상태에 의존하지 않아서 결합도가 낮음.

남은 감점 포인트(연계 관점)

* 트리거 문자열이 여전히 커맨드로 오인될 수 있음 → 이거만 “명령 실행이 아닌 메시지 출력”으로 완전 봉인하면 연계 점수도 **9.5+** 가능.

원하면, Step 3 트리거 문구를 “절대 커맨드로 실행되지 않게” 만드는 안전한 문장 2~3가지 버전으로 바로 써줄게.
