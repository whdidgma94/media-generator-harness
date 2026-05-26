---
name: auto
description: 전체 미디어 생성 파이프라인 오케스트레이터. "/auto '요청'" 형식으로 호출되며 intake → promptify → generate → curate → package 순서로 단계를 실행하고, 비용 발생 전 체크포인트 #1과 산출물 확인 체크포인트 #2를 포함한다.
---

# auto skill

사용자의 자연어 요청을 받아 이미지 미디어 생성 전체 파이프라인을 실행한다.
내부 절차는 orchestrator-agent에 위임한다.

## 호출 방법

```
/auto "요청 문자열"
```

예시:
- `/auto "고양이가 우주를 떠다니는 이미지 3장"`
- `/auto "감성적인 카페 인테리어 사진 4장, 따뜻한 색감"`

## 실행 절차

### Step 1 — orchestrator-agent 호출

**sub-agent를 호출**하여 orchestrator-agent를 실행한다.

전달 인자:
- `session-id`: `YYYYMMDD-HHMMSS-{short-slug}` 형식으로 orchestrator가 자체 생성
- `user-request`: 사용자가 `/auto` 에 전달한 원문 문자열

orchestrator-agent는 아래 내부 파이프라인을 자율적으로 실행한다:
- intake-agent 호출 → `media-spec.json` 산출 → **사용자 확인 체크포인트 #1** (비용 안내 포함)
- prompt-engineer-agent 호출 → `prompts/{item-id}.json` 산출
- media-generator-agent 아이템 단위 병렬 호출 → `media/{item-id}.png` 산출
- media-curator-agent 아이템 단위 병렬 호출 → `curation-report.md` 산출
- packager-agent 호출 → `metadata/{item-id}.json` + `README.md` 산출 → **사용자 확인 체크포인트 #2**

> orchestrator-agent의 내부 절차 상세는 `agents/orchestrator-agent.md` 에 정의한다.

## 입력

| 항목 | 설명 |
|------|------|
| 사용자 요청 문자열 | `/auto` 에 전달된 자연어 요청 (예: "고양이가 우주를 떠다니는 이미지 3장") |

## 출력

산출물은 `.harness-artifacts/media-generator-harness/output/{session-id}/` 하위에 저장된다.

| 파일 | 설명 |
|------|------|
| `media-spec.json` | 미디어 타입 / 수량 / 스타일 / 종횡비 등 확정된 생성 스펙 |
| `prompts/{item-id}.json` | 아이템별 positive/negative prompt + 파라미터 |
| `media/{item-id}.png` | 생성된 이미지 파일 |
| `metadata/{item-id}.json` | 사이드카 메타데이터 (prompt 원문, seed, model, backend, timestamp, file size, resolution) |
| `curation-report.md` | 자동 검수 결과 (PASS / WARN / FAIL 판정 포함) |
| `generation-log.md` | 외부 API 호출 로그 (실패 / 재시도 포함) |
| `README.md` | 갤러리 인덱스 (썸네일 링크 + spec 요약) |

## 사용자 확인 체크포인트

orchestrator-agent가 두 번의 체크포인트를 관리한다:

- **체크포인트 #1** (intake 직후): `media-spec.json` 내용과 "이미지 N장 × Gemini Imagen 예상 비용" 안내를 보여주고 생성 진행 여부를 확인한다. 사용자가 취소하면 파이프라인을 중단한다.
- **체크포인트 #2** (package 직후): 갤러리 `README.md` 와 `curation-report.md` 를 보여주고 최종 승인을 받는다. 미승인 시 조치 방법(수동 편집, 재생성 요청)을 안내한다.
