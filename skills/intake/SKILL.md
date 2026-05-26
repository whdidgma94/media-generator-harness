---
name: intake
description: 사용자의 자연어 요청을 분석하여 이미지 생성 스펙(media-spec.json)을 확정한다. 생성 파이프라인의 첫 단계로, 비용 발생 전 사용자 확인 체크포인트 #1을 포함한다.
---

## 목적

자연어 요청(예: "고양이가 우주를 떠다니는 장면 3장")을 받아 미디어 타입, 수량, 스타일, 종횡비 등 생성 스펙을 구조화한 `media-spec.json`을 산출하고, 생성 시작 전 사용자 확인을 받는다.

## 입력

- `session-id`: orchestrator가 생성한 세션 식별자 (`YYYYMMDD-HHMMSS-{short-slug}` 형식)
- `request`: 사용자의 자연어 요청 문자열

## 출력

- `.harness-artifacts/media-generator-harness/output/{session-id}/media-spec.json`

---

## 절차

### Step 1 — 요청 수신 및 세션 디렉토리 확인

`session-id`와 `request`를 인자로 받는다.

`.harness-artifacts/media-generator-harness/output/{session-id}/` 디렉토리가 존재하는지 확인한다. 없으면 orchestrator에 디렉토리 생성을 요청하거나 직접 생성한다.

### Step 2 — sub-agent를 호출하여 intake-agent를 실행한다.

**전달 인자:**
- `session-id`
- `request` (사용자의 원문 자연어 요청)

intake-agent가 요청을 분석하여 `.harness-artifacts/media-generator-harness/output/{session-id}/media-spec.json`을 Write한다. 세부 분석 절차는 intake-agent.md에 정의되어 있다.

### Step 3 — 생성 스펙 검토

intake-agent가 Write한 `.harness-artifacts/media-generator-harness/output/{session-id}/media-spec.json`을 Read한다.

스펙의 논리적 일관성을 확인한다:
- `type`이 `"image"`인지
- `count`가 1 이상 20 이하인지 (초과 시 사용자에게 경고)
- `backend`가 `"gemini"`인지
- 필수 필드(`type`, `count`, `aspect_ratio`, `subject`, `backend`)가 모두 존재하는지

### Step 4 — **사용자 확인 체크포인트 #1**

사용자에게 다음 정보를 제시하고 생성 시작 여부를 확인한다:

```
=== 생성 스펙 확인 (체크포인트 #1) ===

요청: {request_original}

확정된 생성 스펙:
  - 미디어 타입: 이미지
  - 수량: {count}장
  - 스타일: {style}
  - 종횡비: {aspect_ratio}
  - 주제: {subject}
  - 분위기: {mood}
  - 백엔드: Gemini CLI (Imagen)

비용 안내:
  - Gemini Imagen 기준 이미지 {count}장 생성 예상 비용:
    약 $0.04 × {count} = ${ 0.04 * count } (Imagen 3 평균 단가 기준, 실제 과금은 모델/해상도에 따라 상이)
  - 상세: https://ai.google.dev/pricing

계속 진행하시겠습니까? (yes/no)
```

사용자가 **no** 또는 수정을 요청하면:
- 수정이 필요한 항목을 다시 받아 Step 2로 돌아가 intake-agent를 재실행한다.

사용자가 **yes**를 입력하면:
- `media-spec.json`에 `"confirmed": true` 필드를 추가하고 파일을 업데이트한다.
- 다음 단계(promptify)로 이행할 준비가 완료되었음을 orchestrator에 반환한다.

## 오류 처리

| 상황 | 처리 방식 |
|------|-----------|
| `count` > 20 | 사용자에게 경고 후 20으로 조정하거나 분할 실행 제안 |
| 요청이 너무 짧거나 불명확 | intake-agent가 `notes`에 가정 기록 후 진행, 체크포인트에서 사용자 확인 |
| `media-spec.json` Write 실패 | 오류 메시지를 출력하고 실행 중단 |
| 사용자가 체크포인트에서 취소 | 파이프라인 종료, `media-spec.json` 삭제하지 않고 보존 |
