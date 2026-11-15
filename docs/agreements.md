# 팀이 맞춘 것

개인 레포는 각자 자유롭게 운영합니다. 이 문서에는 결과를 서로 비교하려고 **똑같이 맞추기로 확정한 것만** 적습니다. 프롬프트, 양자화, 학습 방법처럼 여기에 없는 것은 각자 정합니다. 아직 안 정한 것은 [4절](#4-아직-안-정한-것)에 모으고, 회의에서 정해지면 위로 옮기며 "정한 때"를 적습니다.

| 항목 | 확정 내용 | 정한 때 |
|---|---|---|
| 백본 | `Qwen/Qwen2.5-VL-7B-Instruct` | 8차 회의 (11/10) |
| 데이터 | Hugging Face `CluVi-prometheus` 비공개 데이터셋 | v1 11/4 · v2 11/8 |
| 분할 | train 881 · val 100 · test 185 (건수 확정, val 목록은 미정) | 11/11 |
| 출력 | 분석 카드 JSON ([2절](#2-출력-분석-카드)) | 8차 회의 (11/10) |
| 평가지표 | 13개 ([3절](#3-평가지표-13개)) | 8차 회의 (11/10) |

## 1. 데이터와 분할

- `CluVi-prometheus/cesamard-defect` (v1): CESAMARD-Aspect 불만 리뷰에 부위 · 결함 · 보임 정도 · 박스 라벨을 붙인 것. 라벨 정의(가이드 v1.3)가 이 데이터셋 README에 있습니다.
- `CluVi-prometheus/cesamard-defect-v2`: v1 + 텍스트 충분성 라벨 (silver)
- 둘 다 비공개입니다. 아마존 고객 사진이 들어 있어 공개하지 않습니다.

| 분할 | 건수 | 정답 |
|---|---|---|
| train | 881 | silver (Claude 자동 라벨) |
| val | 100 | silver. train 981건에서 떼어 냄 |
| test | 185 | 사람 정답 = test 122 + test_blind 63 |

- v2의 `validation` 88건은 라벨이 없는 dev 세트라 팀 val과 다릅니다.

## 2. 출력: 분석 카드

모델이 어떤 방식으로 답을 만들든, 최종 결과는 리뷰 1건마다 아래 카드 하나로 냅니다.

```json
{
  "part_scope": "product",
  "part": "sleeve",
  "group6": "damage",
  "defect": "tear_hole",
  "visibility": "localized",

  "boxes": [
    {
      "image_index": 0,
      "bbox": [0.35, 0.20, 0.60, 0.55],
      "target": "defect"
    }
  ],

  "evidence": {
    "customer_claim": "고객은 소매가 찢어진 상태로 배송되었다고 주장함",
    "review_span": "The sleeve arrived torn.",
    "visual_findings": [
      {
        "image_index": 0,
        "description": "소매 봉제선 근처에 찢어진 부분이 보임",
        "box_ref": 0
      }
    ],
    "claim_support": "supported",
    "limitations": [
      "손상이 발생한 시점과 원인은 확인할 수 없음"
    ]
  }
}
```

라벨과 같은 필드는 정답 라벨과 같은 기준을 따라야 채점이 됩니다. 출처: 데이터셋 README의 라벨 정의 (가이드 v1.3)

| 필드 | 값 |
|---|---|
| `part_scope` | `product` · `packaging` · `accessory` · `label_doc` · `none` · `other` |
| `part` | 도메인별 부위 코드 (데이터셋 README의 부위 목록) |
| `group6` | `damage` · `contamination` · `manufacturing` · `not_as_described` · `function` · `non_visual` |
| `defect` | 세부 결함 27종 (각 `group6` 아래 목록) |
| `visibility` | `localized` · `diffuse` · `not_visible` |
| `boxes` | `localized`일 때만. `image_index`는 0부터, `bbox`는 `[x1, y1, x2, y2]` 0~1 정규화 · 왼쪽 위 원점 · EXIF 회전을 적용한 사진 기준, `target`은 `defect`(결함 자체) · `evidence`(문제를 증명하는 표시) |

출력에만 있는 필드 (정답 라벨 없음):

| 필드 | 뜻 |
|---|---|
| `evidence.customer_claim` | 고객이 무엇을 문제 삼는지 |
| `evidence.review_span` | 그 주장이 나온 리뷰 원문 그대로 |
| `evidence.visual_findings` | 사진마다 실제로 보이는 것 |
| `evidence.claim_support` | 사진이 고객 주장을 받쳐 주는가 |
| `evidence.limitations` | 사진으로 확인할 수 없는 것 |

- `group5`는 `group6`에서 `damage` + `manufacturing`을 `physical_defect`로 합친 값이라 평가할 때 계산합니다.

## 3. 평가지표 13개

지표 이름 13개가 확정입니다. "뜻"은 일반적인 정의이고, 측정 조건은 [4절](#4-아직-안-정한-것)에서 정합니다.

| 항목 | 지표 | 뜻 |
|---|---|---|
| Part | Scope Macro-F1 | `part_scope`의 Macro-F1 |
| Part | Part Macro-F1 | `part` 코드의 Macro-F1 |
| Defect | Group6 Macro-F1 | 대분류 6개의 Macro-F1 |
| Defect | Group5 Macro-F1 | `damage` · `manufacturing`을 합친 5개의 Macro-F1 |
| Defect | Fine-grained Macro-F1 | 세부 결함 27종의 Macro-F1 |
| Visibility | Macro-F1 | 보임 정도 3단계의 Macro-F1 |
| Grounding | Mean IoU | 예측 박스와 정답 박스의 IoU 평균 (IoU = 겹친 넓이 ÷ 합친 넓이) |
| Grounding | IoU@0.5 | IoU가 0.5 이상인 비율 |
| Grounding | Pointing Accuracy | 예측 박스의 중심이 정답 박스 안에 있는 비율 (정의 확정 필요) |
| Token | Retention Ratio (%) | LLM에 들어간 시각 토큰 ÷ 원래 시각 토큰 |
| Efficiency | Prefill Latency (ms) | 첫 출력 토큰이 나올 때까지 걸린 시간 (정의 확정 필요) |
| Efficiency | End-to-End Latency (ms) | 입력부터 마지막 출력 토큰까지 걸린 시간 |
| Efficiency | Peak VRAM (GB) | 추론 중 GPU 메모리 최댓값 |

- Macro-F1 = 클래스마다 F1을 따로 구해 단순 평균한 값 (작은 클래스도 같은 비중)

## 4. 아직 안 정한 것

| 정할 것 | 왜 필요한가 |
|---|---|
| val 100건 목록과 그 목록을 둘 곳 (예: HF 데이터셋에 split으로) | 셋이 각자 100건을 뽑으면 같은 분할이 아님 |
| 비교에 쓸 데이터셋 버전 (v1 / v2, revision) | 버전이 다르면 숫자를 나란히 놓을 수 없음 |
| 결과표를 채울 평가셋 (val silver / test 122 / test 185) | val은 Claude 라벨이라 사람 정답 대비 점수와 다름 |
| test를 쓰는 원칙 (설정 · 체크포인트는 val로 고르고 test는 마지막에만 볼지) | test로 고르면 숫자가 test에 맞춰짐 |
| 박스 채점 규칙 (박스 여러 개일 때 어느 것을, 사진이 다르면 어떻게) | 같은 예측도 규칙에 따라 IoU가 달라짐 |
| Prefill · Pointing 정의, 지연 측정 조건 (GPU, batch, warm-up) | 사람마다 GPU · 조건이 달라 그대로는 비교 불가 |
| `claim_support` 값 목록, `box_ref` 규칙, 박스가 없을 때 빈 배열인지 생략인지 | 출력 전용 필드라 정의가 없음 |
| 모델 · 체크포인트 업로드 방식 | 베이스라인 뒤에 정하기로 함 |
