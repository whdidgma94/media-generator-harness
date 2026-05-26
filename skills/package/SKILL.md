---
name: package
description: 검수 완료된 미디어 파일에 사이드카 메타데이터 JSON을 첨부하고 갤러리 README.md를 생성한다. curate 단계 이후 호출되며, 사용자 확인 체크포인트 #2(최종 산출물 승인)를 포함한다.
---

# package skill

## 역할

검수(curate)가 완료된 미디어 아이템 전체에 대해 사이드카 메타데이터(`metadata/{item-id}.json`)를 생성하고,
갤러리 인덱스(`README.md`)를 산출한다. 파이프라인의 최종 단계이며 **사용자 확인 체크포인트 #2**를 포함한다.

## 입력

- `session-id` — orchestrator로부터 전달받은 세션 식별자 (`YYYYMMDD-HHMMSS-{short-slug}` 형식)
- `.harness-artifacts/media-generator-harness/output/{session-id}/media-spec.json` — 원래 생성 스펙
- `.harness-artifacts/media-generator-harness/output/{session-id}/prompts/{item-id}.json` — 아이템별 프롬프트 및 파라미터
- `.harness-artifacts/media-generator-harness/output/{session-id}/media/{item-id}.{png|jpg}` — 생성된 미디어 파일
- `.harness-artifacts/media-generator-harness/output/{session-id}/curation-report.md` — 검수 결과 (PASS/FAIL/RETRY 목록)
- `.harness-artifacts/media-generator-harness/output/{session-id}/generation-log.md` — 외부 API 호출 로그

## 출력

- `.harness-artifacts/media-generator-harness/output/{session-id}/metadata/{item-id}.json` — 아이템별 사이드카 메타데이터 (PASS 아이템 전체)
- `.harness-artifacts/media-generator-harness/output/{session-id}/README.md` — 갤러리 인덱스

## 절차

### Step 1 — packager-agent 호출 (메타데이터 생성)

**sub-agent를 호출**하여 packager-agent를 실행한다.

- 전달 인자: `session-id`
- agent가 직접 Read하는 파일:
  - `curation-report.md` — PASS 판정 아이템 목록 확인
  - `prompts/{item-id}.json` — positive/negative prompt, seed, params
  - `media/{item-id}.{png|jpg}` — 파일 존재 여부 및 크기 (curation-report.md에 기록된 값 활용)
- agent가 Write하는 파일:
  - `metadata/{item-id}.json` (PASS 아이템별 1개씩)
    ```json
    {
      "item_id": "{item-id}",
      "prompt": "...",
      "negative_prompt": "...",
      "seed": 12345,
      "model": "imagen-3",
      "backend": "gemini-cli",
      "aspect_ratio": "1:1",
      "timestamp": "2026-05-26T12:34:56Z",
      "file_size_bytes": 204800,
      "resolution": "1024x1024",
      "session_id": "{session-id}"
    }
    ```
  - FAIL 아이템은 메타데이터를 생성하지 않는다.

### Step 2 — packager-agent 호출 (README.md 생성)

packager-agent를 동일 호출 내에서 계속 실행하여 `README.md`를 작성한다.

- agent가 Read하는 파일:
  - `media-spec.json` — 원본 요청 스펙 (스타일, 수량, aspect 등)
  - `metadata/{item-id}.json` (전체) — 생성 파라미터 요약
  - `curation-report.md` — PASS/FAIL 집계
- `README.md` 포함 내용:
  1. 요청 원문 요약 및 생성 일시
  2. 전체 아이템 PASS/FAIL 집계 (표 형식)
  3. PASS 아이템별 이미지 링크 + 프롬프트 요약 (갤러리 인덱스)
  4. 사용한 백엔드 / 모델 정보
  5. FAIL 아이템 목록 (있을 경우) — `curation-report.md` 링크 포함

### Step 3 — 사용자 확인 체크포인트 #2

**사용자 확인 체크포인트**

아래 내용을 사용자에게 제시하고 최종 승인 여부를 묻는다:

1. `README.md` 전문 (갤러리 인덱스)
2. `curation-report.md` 요약 (PASS N건 / FAIL N건 / RETRY N건)
3. 생성된 `metadata/` 파일 목록 (아이템 수 확인)
4. FAIL 아이템이 있을 경우 재생성 여부 안내 (`/generate` 재실행 가능)

사용자 선택지:
- **승인(완료)** — 산출물을 완성으로 처리한다.
- **FAIL 재생성 요청** — orchestrator가 해당 item-id만 `/generate` → `/curate` → `/package` 순서로 재실행한다.
- **취소** — 현재 산출물을 보존하되 미완료 상태로 표시한다.

## 사전 조건

- `curate` 단계가 완료되어 `curation-report.md`가 존재해야 한다.
- PASS 아이템의 미디어 파일(`media/{item-id}.{png|jpg}`)이 실제로 존재해야 한다.
- `metadata/` 디렉토리가 사전에 생성되어 있어야 한다 (orchestrator가 병렬 단계 진입 전 생성).

## 오류 처리

- `metadata/` 디렉토리는 orchestrator-agent가 병렬 단계 진입 전에 사전 생성한다 (plan.md 규칙 6). packager-agent는 Bash 권한이 없으므로 디렉토리 생성을 수행하지 않는다.
- 미디어 파일이 없는 PASS 아이템이 발견되면 해당 아이템을 FAIL로 재분류하고 `README.md`에 명시한다.
- `README.md` 생성 실패 시 오류를 사용자에게 즉시 보고하고 수동 확인을 안내한다.

## 주의사항

- 모든 PASS 아이템에 대해 `metadata/{item-id}.json`이 1:1로 존재해야 한다 (재현성·라이선스 추적).
- `item-id`는 `prompts/`, `media/`, `metadata/` 전 디렉토리에서 동일하게 사용한다 (slug 불일치 회피).
- packager-agent의 쓰기 범위는 `.harness-artifacts/media-generator-harness/output/{session-id}/` 하위로 엄격히 제한한다.
