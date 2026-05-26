---
name: generate
description: prompts/ 디렉토리에 준비된 아이템별 prompt JSON을 읽어 Gemini CLI로 이미지를 병렬 생성한다. /generate 커맨드로 직접 호출하거나 auto skill의 파이프라인 Step 3에서 orchestrator가 호출한다.
---

# generate skill

`prompts/{item-id}.json`을 기반으로 Gemini CLI(Imagen)를 아이템 단위로 병렬 호출하여
이미지 파일을 생성하고, 호출 로그를 `generation-log.md`에 기록한다.

## 입력

- `session-id`: orchestrator가 생성한 세션 식별자 (예: `20240526-143000-cat-space`)
- 세션 디렉토리 내 `prompts/{item-id}.json` 파일 전체

## 출력

- `media/{item-id}.png` (또는 `.jpg`) — 아이템별 생성 이미지 파일
- `generation-log.md` — 외부 API 호출 기록 (성공/실패/재시도 포함)

## 절차

### Step 1 — 사전 환경 점검

1. `session-id`를 인자로 받아 세션 루트 경로를 확인한다:
   - `.harness-artifacts/media-generator-harness/output/{session-id}/`
2. `prompts/` 디렉토리를 읽어 생성할 item-id 목록을 수집한다.
3. `Bash(which gemini)` 또는 `Bash(command -v gemini)`로 Gemini CLI 설치 여부를 확인한다.
   - 미설치 시 즉시 중단하고 아래 안내를 출력한 뒤 종료한다:
     ```
     [오류] Gemini CLI가 설치되어 있지 않습니다.
     설치 방법: https://ai.google.dev/gemini-api/docs/gemini-cli
     인증: `gemini auth login` 실행 후 재시도하세요.
     ```
4. `media/`, `metadata/`, `prompts/` 디렉토리가 존재하는지 확인한다 (`Bash(ls ...)`).
   - 없으면 `Bash(mkdir -p ...)` 로 미리 생성한다 (병렬 실행 전 경쟁 조건 회피).

### Step 2 — 아이템별 병렬 생성

각 `item-id`에 대해 **sub-agent를 호출**하여 media-generator-agent를 실행한다.

- 전달 인자: `session-id`, `item-id`
- 동시 실행 상한: `MEDIA_GEN_CONCURRENCY` 환경변수 값 (기본 3)
  - 동시 실행 중인 agent 수가 상한에 도달하면 완료된 agent가 생길 때까지 대기한다.
- media-generator-agent는 다음을 독립적으로 수행한다:
  1. `prompts/{item-id}.json` Read → `positive_prompt`, `negative_prompt`, `params` 추출
  2. Gemini CLI 호출: `gemini image generate --prompt "..." --output media/{item-id}.png [params]`
  3. 실패(비정상 종료 / 429 / 5xx) 시 지수 백오프 후 1회 재시도
  4. 재시도 후에도 실패하면 `generation-log.md`에 FAIL 항목 기록 후 종료
  5. 성공 시 `media/{item-id}.png` 저장 확인 후 `generation-log.md`에 SUCCESS 기록

### Step 3 — 로그 취합 및 결과 요약

1. 모든 병렬 agent 완료 후 `generation-log.md`를 Read하여 전체 결과를 집계한다.
2. 성공/실패/재시도 횟수를 요약하여 터미널에 출력한다:
   ```
   [generate] 완료: 성공 N건 / 실패 M건 / 총 K건
   실패 항목: [item-id 목록] → curation 단계에서 FAIL 처리됩니다.
   ```
3. 실패 아이템이 있더라도 성공한 아이템이 1건 이상이면 다음 단계(curate)로 진행한다.
   - 전체 실패 시 즉시 중단하고 `generation-log.md` 경로를 사용자에게 안내한다.

## 파일 경로 규칙

| 파일 | 경로 |
|------|------|
| 입력 prompt | `.harness-artifacts/media-generator-harness/output/{session-id}/prompts/{item-id}.json` |
| 생성 이미지 | `.harness-artifacts/media-generator-harness/output/{session-id}/media/{item-id}.png` |
| 생성 로그 | `.harness-artifacts/media-generator-harness/output/{session-id}/generation-log.md` |

## 오류 처리 요약

| 상황 | 동작 |
|------|------|
| Gemini CLI 미설치 | 설치 안내 출력 후 즉시 중단 |
| API 호출 실패 (429/5xx) | 지수 백오프 후 1회 재시도 |
| 재시도 후에도 실패 | FAIL 로그 기록, 해당 item 건너뜀 |
| 전체 아이템 실패 | 즉시 중단, generation-log.md 경로 안내 |
| 부분 실패 | 성공 아이템으로 다음 단계 진행 |
