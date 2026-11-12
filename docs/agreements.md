# 팀이 맞춘 것

개인 레포는 각자 자유롭게 운영합니다. 이 문서에는 결과를 서로 비교하려고 **똑같이 맞추기로 확정한 것만** 적습니다. 프롬프트, 양자화, 학습 방법처럼 여기에 없는 것은 각자 정합니다. 더 맞출 규칙이 생기면 이 문서에 PR로 추가합니다.

| 항목 | 확정 내용 | 정한 때 |
|---|---|---|
| 백본 | `Qwen/Qwen2.5-VL-7B-Instruct` | 8차 회의 (11/10) |
| 데이터 | Hugging Face `CluVi-prometheus` 비공개 데이터셋 | v1 11/4 · v2 11/8 |
| 분할 | train 881 · val 100 · test 185 | 11/11 |
| 출력 | 분석 카드 JSON ([2절](#2-출력-분석-카드)) | 8차 회의 (11/10) |
| 평가지표 | 13개 ([3절](#3-평가지표-13개)) | 8차 회의 (11/10) |

## 1. 데이터와 분할

- `CluVi-prometheus/cesamard-defect` (v1): CESAMARD-Aspect 불만 리뷰에 부위 · 결함 · 보임 정도 · 박스 라벨을 붙인 것. 라벨 정의(가이드 v1.3)는 이 데이터셋 README에 있습니다.
- `CluVi-prometheus/cesamard-defect-v2`: v1 + 텍스트 충분성 라벨 (silver)
- 둘 다 비공개입니다. 아마존 고객 사진이 들어 있어 공개하지 않습니다.

| 분할 | 건수 | 정답 |
|---|---|---|
| train | 881 | silver (Claude 자동 라벨) |
| val | 100 | silver. train 981건에서 떼어 냄 |
| test | 185 | 사람 정답 = test 122 + test_blind 63 |

- test는 최종 평가에만 씁니다. 설정 · 임계값 · 체크포인트는 val로 고릅니다.
- v2의 `validation` 88건은 라벨이 없는 dev 세트라 팀 val과 다릅니다.
- val 100건 목록은 확정되면 여기에 붙입니다.

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

| 필드 | 값 |
|---|---|
| `part_scope` | `product` · `packaging` · `accessory` · `label_doc` · `none` · `other` |
| `part` | 도메인별 부위 코드 (데이터셋 README의 부위 목록) |
| `group6` | `damage` · `contamination` · `manufacturing` · `not_as_described` · `function` · `non_visual` |
| `defect` | 세부 결함 27종 (각 `group6` 아래 목록) |
| `visibility` | `localized` · `diffuse` · `not_visible` |
| `boxes` | `localized`일 때만. `image_index`는 0부터, `bbox`는 `[x1, y1, x2, y2]` 0~1 정규화 · 왼쪽 위 원점 · EXIF 회전을 적용한 사진 기준, `target`은 `defect` · `evidence` |
| `evidence` | `customer_claim` 고객 주장 · `review_span` 리뷰 원문 그대로 · `visual_findings` 사진별 관찰 · `claim_support` 사진이 주장을 받쳐 주는가 · `limitations` 사진으로 확인할 수 없는 것 |

- `group5`는 `group6`에서 `damage` + `manufacturing`을 `physical_defect`로 합친 값이라 평가할 때 계산합니다.

## 3. 평가지표 13개

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
| Grounding | Pointing Accuracy | 예측 박스의 중심이 정답 박스 안에 있는 비율 |
| Token | Retention Ratio (%) | LLM에 들어간 시각 토큰 ÷ 원래 시각 토큰 |
| Efficiency | Prefill Latency (ms) | 첫 출력 토큰이 나올 때까지 걸린 시간 |
| Efficiency | End-to-End Latency (ms) | 입력부터 마지막 출력 토큰까지 걸린 시간 |
| Efficiency | Peak VRAM (GB) | 추론 중 GPU 메모리 최댓값 |

- Macro-F1 = 클래스마다 F1을 따로 구해 단순 평균한 값 (작은 클래스도 같은 비중)
- 박스를 몇 개 채점할지, 지연을 어떤 조건에서 잴지 같은 세부 규칙은 결과를 비교할 때 맞추고 여기에 추가합니다.
