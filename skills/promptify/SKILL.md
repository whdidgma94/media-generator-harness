---
name: promptify
description: 생성 스펙(media-spec.json)을 읽어 Gemini Imagen 엔진 친화적인 영문 prompt + negative prompt + 파라미터를 아이템별로 산출한다. intake 완료 후, generate 실행 전에 호출된다.
---

# promptify skill

## 역할

`intake` skill이 확정한 `media-spec.json`을 읽고, 각 아이템에 대해 Gemini Imagen이 최적으로 해석할 수 있는 프롬프트 패키지를 작성한다.
출력은 `prompts/{item-id}.json` 파일이며, `generate` skill이 이 파일을 입력으로 사용한다.

## 입력

- `session-id` — orchestrator로부터 전달받는 세션 식별자 (`YYYYMMDD-HHMMSS-{short-slug}` 형식)
- `.harness-artifacts/media-generator-harness/output/{session-id}/media-spec.json` — intake 결과 파일

## 출력

- `.harness-artifacts/media-generator-harness/output/{session-id}/prompts/{item-id}.json` — 아이템별 프롬프트 패키지 (아이템 수만큼 생성)

## 절차

### Step 1 — media-spec.json 로드

1. `.harness-artifacts/media-generator-harness/output/{session-id}/media-spec.json`을 Read한다.
2. 다음 필드를 확인한다:
   - `type` — 항상 `image` (v1 범위)
   - `items` — 생성할 아이템 목록 (각 아이템에 `item-id`, `description`, `style`, `aspect_ratio`, `quantity` 포함)
   - `global_style` — 전체 공통 스타일 힌트 (있을 경우 각 아이템 prompt에 반영)

### Step 2 — prompt-engineer-agent 호출 (아이템별 순차)

각 아이템(`item-id`)에 대해 **sub-agent를 호출**하여 `prompt-engineer-agent`를 실행한다.

- 전달 인자: `session-id`, `item-id`
- agent 동작:
  - `media-spec.json`에서 해당 `item-id`의 설명·스타일·종횡비 등을 직접 Read한다.
  - **Positive prompt** — Gemini Imagen 권장 구조로 영문 작성:
    - `[subject]`, `[style descriptor]`, `[composition]`, `[lighting]`, `[quality modifier]` 순서로 구성
    - 예: `"A cat floating in outer space, surrealist digital art, wide-angle view, cosmic lighting, ultra-detailed, 8k"`
  - **Negative prompt** — 생성 품질을 저해하는 요소 영문 나열:
    - 예: `"blurry, low quality, watermark, text, distorted, overexposed"`
  - **Parameters** — Gemini Imagen 호환 파라미터:
    - `aspect_ratio`: media-spec의 값 그대로 매핑 (`1:1`, `16:9`, `4:3`, `9:16` 등)
    - `seed`: 재현성을 위한 정수 난수 (0–2147483647 범위)
    - `model`: `"imagen-3.0-generate-002"` (기본값)
    - `number_of_images`: 아이템의 `quantity` 값
  - 작성된 내용을 `prompts/{item-id}.json`으로 Write한다.

출력 파일 형식 예시:
```json
{
  "item-id": "item-001",
  "positive_prompt": "A cat floating in outer space, surrealist digital art, wide-angle composition, cosmic lighting, ultra-detailed, 8k",
  "negative_prompt": "blurry, low quality, watermark, text, distorted, overexposed, pixelated",
  "parameters": {
    "aspect_ratio": "16:9",
    "seed": 1847392650,
    "model": "imagen-3.0-generate-002",
    "number_of_images": 1
  }
}
```

### Step 3 — prompts/ 디렉토리 검증

1. 모든 아이템의 `prompts/{item-id}.json` 파일이 생성되었는지 확인한다.
2. 누락된 아이템이 있으면 해당 `item-id`에 대해 Step 2를 재실행한다.
3. 각 파일에 `positive_prompt`, `negative_prompt`, `parameters` 필드가 모두 존재하는지 검사한다.

### Step 4 — 완료 보고

`generate` skill에 전달할 준비가 완료되었음을 orchestrator에 보고한다.
- 생성된 prompt 파일 경로 목록
- 각 아이템의 `item-id`와 `positive_prompt` 첫 80자 미리보기

## 제약 사항

- **쓰기 범위**: `.harness-artifacts/media-generator-harness/output/{session-id}/prompts/` 하위로만 쓰기를 수행한다.
- **언어**: positive/negative prompt는 반드시 영문으로 작성한다 (Gemini Imagen 권장).
- **agent 격리**: prompt-engineer-agent 호출 시 `session-id`와 `item-id`만 인자로 전달하며, prompt 본문을 인자에 포함하지 않는다. agent가 `media-spec.json`을 직접 Read한다.
- **slug 일관성**: `item-id`는 `media-spec.json`의 값을 그대로 사용하며, `prompts/`, `media/`, `metadata/` 디렉토리 전체에서 동일한 `item-id`를 사용한다.
