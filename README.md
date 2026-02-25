# Claude Skill: 자동 학습 파이프라인

Claude를 자율 코딩 에이전트로 사용하여 원천 데이터 탐색부터 반복적 모델 개선까지, ML 학습 전 과정을 자동화하는 두 개의 연계 Claude 스킬 모음입니다.

---

## 개요

이 레포지토리는 순서대로 실행되도록 설계된 두 개의 Claude 스킬을 제공합니다.

| 스킬 | 트리거 | 역할 |
|---|---|---|
| **build-baseline** | `/build-baseline` | 원천 데이터 탐색 → 전처리 → 베이스라인 학습 코드 작성 및 스모크 테스트 통과 |
| **progressive-training** | `/progressive-training` 또는 자동 트리거 | 버전 단위 반복 개선 루프 실행 및 최종 리더보드 리포트 생성 |

`build-baseline`이 성공적으로 완료되면 **`progressive-training`을 자동으로 트리거**합니다. 별도의 수동 전환이 필요 없습니다.

---

## 전체 워크플로우

```
사용자: /build-baseline
         │
         ▼
┌─────────────────────────────────┐
│         build-baseline          │
│                                 │
│  Step 0: 원천 데이터 프로파일링    │  ─── dataset/rawdata/* 자동 탐색
│  Step 1: 데이터 파이프라인        │  ─── 01_data_pipeline.py 작성 + 테스트
│  Step 2: 베이스라인 학습          │  ─── 02_train_baseline.py 작성 + 스모크 테스트
│  Step 3: 최종 산출물 검사         │  ─── 전체 아티팩트 검증
└────────────────┬────────────────┘
                 │ 자동 트리거
                 ▼
┌─────────────────────────────────┐
│       progressive-training      │
│                                 │
│  Step 0: 루프 초기화              │  ─── dataset_id 확정, cur=1 설정
│  Step 1: v{cur} 학습             │  ─── 02_train_v001.py 실행 → 아티팩트
│  Step 2: 평가                    │  ─── eval_score.json + infer_samples/
│  Step 3: 결과 분석               │  ─── result-analysis.md
│  Step 4: 개선 행동 도출           │  ─── 저위험 개선 행동 1개 선정
│  Step 5: 종료 조건 검사           │  ─── cur < max_versions?
│  Step 6: 다음 버전 작성           │  ─── 02_train_v{cur+1}.py 생성 → Step 1
│            ...                  │
│  Step 7: 최종 종합 리포트          │  ─── leaderboard.csv + final-analysis.md
└─────────────────────────────────┘
```

---

## 설치 및 사용

### 1. Claude 프로젝트에 스킬 추가

스킬 파일을 Claude 프로젝트의 스킬 디렉토리에 복사합니다. 권장 경로는 `.claude/skills/`입니다.

```
.claude/
└── skills/
    ├── build-baseline/
    │   ├── SKILL.md
    │   ├── task_format.yaml          ← 필수
    │   ├── user_defined_rules.json   ← 선택
    │   └── user_instruction.md       ← 선택
    └── progressive-training/
        ├── SKILL.md
        ├── user_defined_rules.json   ← 선택
        └── user_instruction.md       ← 선택
```

> **참고:** `task_format.yaml`은 `build-baseline`에만 필요합니다. 파일이 없으면 스킬이 즉시 중단됩니다.

### 2-1. `build-baseline`용 `user_defined_rules.json` 설정

```json
{
  "seed": 42,
  "dataset_config": {
    "prepdata_version": "v1",
    "task_type": "ocr-recognition",
    "valid_ratio": 0.1,
    "split_strategy": "uniform",
    "stratify_key": "category",
    "max_profile_samples": 100,
    "max_train_samples": 1000
  },
  "model_config": {
    "H_TARGET": 64,
    "W_BUCKETS": [256, 384, 512],
    "MAX_TOKENS": 64,
    "vision_backbone": "tf_efficientnetv2_s.in21k_ft_in1k",
    "vision_pretrained": true,
    "vision_backbone_out_indices": [1],
    "expected_stride": 4,
    "expected_channels": 48,
    "expected_allow_missing_keys": false
  },
  "baseline_smoke_test": {
    "num_gpus": 1,
    "epoch": 1,
    "train_samples": 100,
    "valid_samples": 100,
    "min_batch_size": 4,
    "max_batch_size": 32
  }
}
```

### 2-2. `progressive-training`용 `user_defined_rules.json` 설정

```json
{
  "seed": 42,
  "loop_config": {
    "max_versions": 5
  },
  "dataset_config": {
    "dataset_id": "v1"
  },
  "training_config": {
    "num_gpus": 1,
    "epochs": 3,
    "eval_step": "every_epoch",
    "loss_function": "CTC"
  },
  "eval_config": {
    "primary_metric": "loss",
    "metrics": [
      {"name": "loss", "higher_is_better": false},
      {"name": "wer", "higher_is_better": false},
      {"name": "cer", "higher_is_better": false},
      {"name": "exact_match", "higher_is_better": true}
    ],
    "infer_samples": 10
  }
}
```

### 3. `user_instruction.md` 설정

자유 형식의 마크다운 파일로, `user_defined_rules.json`으로 표현하기 어려운 자연어 지시사항을 작성합니다. 두 스킬 모두 실행 시 이 파일을 최우선으로 읽고, 명시된 내용을 사용자 정의 제약으로 간주하여 전 과정에서 준수합니다.

```markdown
## User Instruction
- 패키지 관리는 uv를 활용하세요.
- 이미지 처리에는 pillow 라이브러리를 활용하세요.
- 모델 저장 시 safetensors 포맷을 사용하세요.
- 로깅은 wandb 대신 tensorboard를 사용하세요.
```

`user_defined_rules.json`과 `user_instruction.md` 모두 `SKILL.md`와 동일한 디렉토리에 위치해야 합니다. 두 파일이 없으면 사용자 정의 제약이 없는 것으로 간주합니다.

---

### 4. 실행

원천 데이터를 `dataset/rawdata/` 아래에 배치한 뒤 Claude에서 입력합니다.

```
/build-baseline
```

Claude가 전체 파이프라인을 자율적으로 실행합니다. `build-baseline`이 완료되면 `progressive-training`이 자동으로 시작됩니다.

베이스라인이 이미 존재하는 경우 `progressive-training`을 단독으로 실행할 수도 있습니다.

```
/progressive-training
```

---

## 스킬 상세

### 1. `build-baseline`

원천 데이터에서 즉시 실행 가능한 스모크 테스트 통과 베이스라인까지의 과정을 자동화합니다.

**단계별 설명:**
- **Step 0 — 원천 데이터 프로파일링**: `dataset/rawdata/{source}/` 폴더 전체를 자동 탐색하여 샘플 단위, 라벨 스키마, 경로 규칙을 파악하고 `dataset/rawdata/profiling.md`에 문서화합니다.
- **Step 1 — 데이터 파이프라인**: `profiling.md`와 `task_format.yaml`을 기준으로 `01_data_pipeline.py`를 생성합니다. 검증 split 전략(random / stratified / group)을 적용해 `prepdata/{version}/train/`, `valid/`, `common/`을 생성합니다.
- **Step 2 — 베이스라인 학습**: `02_train_baseline.py`를 작성하고 `optimal_batch_size` 자동 탐색, `expected_*` 모델 계약 검증, 스모크 테스트를 수행합니다. `best.ckpt`, `last.ckpt`, `spec.json`, `baseline_smoke_test_report.md`를 생성합니다.
- **Step 3 — 최종 산출물 검사**: 모든 아티팩트 존재 여부와 테스트 통과 여부를 검증한 뒤 `progressive-training`으로 넘깁니다.

**필요 파일 (`SKILL.md`와 동일 디렉토리):**

| 파일 | 필수 여부 | 설명 |
|---|---|---|
| `task_format.yaml` | ✅ 필수 | 태스크별 데이터 구조 표준 정의 |
| `user_defined_rules.json` | 선택 | 시드, 데이터셋 설정, 모델 설정, 스모크 테스트 설정 |
| `user_instruction.md` | 선택 | 자유 형식 사용자 지시사항 (예: 선호 라이브러리 등) |

---

### 2. `progressive-training`

베이스라인 이후 최대 버전 수에 도달할 때까지 버전 관리 기반 개선 루프를 실행하고, 최종 리더보드를 생성합니다.

**루프 (Step 1~6, 반복):**
- **학습**: 현재 버전 학습 실행 → `Models/Artifacts/v{cur:03d}/`에 아티팩트 저장
- **평가**: validation set 평가 → `eval_score.json` + `infer_samples/`
- **결과 분석**: `result-analysis.md` 작성 (메트릭 변화량, 실패 패턴, 원인 가설)
- **개선 행동 도출**: 저위험 개선 행동 **정확히 1개**만 선정 → `action-extract.md`
- **다음 버전 작성**: 선정된 변경 1개만 반영하여 다음 버전 학습 스크립트 생성

**루프 종료 후 (Step 7):**
- 존재하는 모든 버전을 동일 프로토콜로 재평가
- `leaderboard.csv` 생성 (모든 메트릭을 컬럼으로 펼침)
- 최고 성능 버전의 아티팩트를 `Models/Report/YYYY-MM-DD_HH-MM/`에 저장

**핵심 원칙:**
- **버전당 변경은 1개** — 원인-결과 추적이 가능하도록 범위를 좁게 유지
- `user_defined_rules.json`의 제약은 루프 전 과정에서 **절대 무시되지 않음**
- 모든 랜덤성은 고정 `seed`(기본값: `42`)를 사용하여 **재현 가능**

---

## 전체 실행 후 디렉토리 구조

```
your-project/
├── dataset/
│   ├── rawdata/
│   │   ├── profiling.md              ← build-baseline Step 0에서 생성
│   │   ├── source_a/
│   │   └── source_b/
│   └── prepdata/
│       └── v1/
│           ├── stats.json
│           ├── train/
│           ├── valid/
│           └── common/
├── 01_data_pipeline.py
├── 02_train_baseline.py
├── 02_train_v001.py
├── 02_train_v002.py
│   ...
├── 03_eval.py
├── 04_final_report.py
└── Models/
    ├── Artifacts/
    │   ├── baseline/
    │   │   ├── best.ckpt
    │   │   ├── spec.json
    │   │   └── _tests/baseline_smoke_test_report.md
    │   └── v001/
    │       ├── best.ckpt
    │       ├── eval_score.json
    │       ├── result-analysis.md
    │       ├── action-extract.md
    │       └── infer_samples/
    └── Report/
        └── YYYY-MM-DD_HH-MM/
            ├── leaderboard.csv
            ├── final-analysis.md
            └── Artifacts/
                └── v{best}/
```

---

## 지원 태스크 타입

`task_format.yaml`은 다음 태스크 타입에 대한 기본 구조를 내장하고 있습니다.

**텍스트:** `text-generation`, `text-classification`, `token-classification`, `text-matching`, `text-ranking`  
**비전:** `image-classification`, `object-detection`, `image-segmentation`, `image-generation`, `image-captioning`  
**멀티모달:** `vqa`, `ocr-recognition`, `document-understanding`  
**오디오:** `speech-recognition`, `audio-classification`  
**테이블형:** `tabular-classification`, `tabular-regression`  
**검색 / 추천:** `dense-retrieval`, `recommendation`  
**기타:** `custom` (완전 사용자 정의 구조)

---

## 레포지토리 구조

```
v1/
├── en/
│   └── skills/
│       ├── build-baseline/SKILL.md
│       └── progressive-training/SKILL.md
└── kr/
    └── skills/
        ├── build-baseline/
        │   ├── SKILL.md
        │   ├── task_format.yaml
        │   ├── user_defined_rules.json   ← 예시
        │   └── user_instruction.md       ← 예시
        └── progressive-training/
            ├── SKILL.md
            ├── user_defined_rules.json   ← 예시
            └── user_instruction.md       ← 예시
```

`en/`과 `kr/`은 기능적으로 동일한 스킬을 담고 있습니다. 작업 환경에 맞는 언어 버전을 선택하세요.

---

## 라이선스

MIT
