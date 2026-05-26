---
name: curate
description: 생성된 미디어 파일을 아이템 단위로 병렬 검수한다. 파일 존재·크기·해상도·포맷 검증과 Claude vision을 이용한 prompt 부합도 점수화를 수행하고, 결과를 curation-report.md로 산출한다. /generate 완료 후 자동 호출되거나 독립 실행 가능.
---

# curate skill

생성된 미디어 파일 전체를 자동 검수하여 `curation-report.md`를 산출한다.
검수 결과는 PASS / FAIL / WARN(점수 미달) 세 가지로 분류된다.
자동 재생성은 수행하지 않으며, FAIL·WARN 아이템은 사용자 보고로 처리한다.

## 입력

- `session-id`: orchestrator로부터 전달받는 세션 식별자 (`YYYYMMDD-HHMMSS-{short-slug}` 형식)
- `.harness-artifacts/media-generator-harness/output/{session-id}/media-spec.json` — 검수 대상 아이템 목록 및 스펙
- `.harness-artifacts/media-generator-harness/output/{session-id}/prompts/{item-id}.json` — 아이템별 prompt (부합도 평가 기준)
- `.harness-artifacts/media-generator-harness/output/{session-id}/media/{item-id}.{png|jpg}` — 검수 대상 미디어 파일

## 출력

- `.harness-artifacts/media-generator-harness/output/{session-id}/curation-report.md` — 전체 아이템 검수 결과

---

## Step 1 — 검수 대상 아이템 목록 확인

`media-spec.json`을 Read하여 생성 아이템 목록과 스펙을 확인한다.

- 입력: `.harness-artifacts/media-generator-harness/output/{session-id}/media-spec.json`
- 확인 항목:
  - `items` — 아이템 ID 문자열 배열 (예: `["item-001", "item-002"]`)
  - `aspect_ratio` — 최상위 필드 (예: `"1:1"`)
  - `count` — 총 아이템 수

## Step 2 — 아이템 단위 병렬 검수 (media-curator-agent)

`media-spec.json`의 `items` 배열을 순회하며, 각 아이템에 대해 **sub-agent를 호출**하여 media-curator-agent를 실행한다.

- 호출 인자: `session-id`, `item-id`
- 동시 실행 상한: `MEDIA_GEN_CONCURRENCY`(기본 3)를 초과하지 않도록 배치 단위로 병렬 호출한다.
- 각 media-curator-agent는 파일 검증(존재·크기·포맷·해상도)과 Claude vision 부합도 평가(0~100점)를 독립적으로 수행하고 결과를 `curation-report.md`에 append한다.
- 판정 결과: **PASS**(≥70) / **WARN**(50~69) / **FAIL**(<50 또는 파일 오류). 세부 절차는 media-curator-agent.md 참조.

## Step 3 — curation-report.md 작성

모든 아이템의 검수 결과를 취합하여 `curation-report.md`를 작성한다.

- 출력 경로: `.harness-artifacts/media-generator-harness/output/{session-id}/curation-report.md`
- 포함 내용:
  - 검수 일시 및 session-id
  - 전체 요약: 총 아이템 수 / PASS 수 / WARN 수 / FAIL 수
  - 아이템별 상세 결과 테이블: `item_id`, `status`, `vision_score`, `fail_reasons`
  - WARN·FAIL 아이템 목록 (사용자 수동 확인 또는 재생성 참고용)
  - 재생성 권장 아이템 목록 (FAIL만)

보고서 예시 형식:

```markdown
# Curation Report

- Session: {session-id}
- Date: {timestamp}
- Total: {N} | PASS: {P} | WARN: {W} | FAIL: {F}

## 아이템별 결과

| item_id | status | vision_score | fail_reasons |
|---------|--------|--------------|--------------|
| item-001 | PASS | 87 | — |
| item-002 | WARN | 63 | 배경색이 요청과 다름 |
| item-003 | FAIL | 42 | 주요 피사체 없음, 포맷 불일치 |

## FAIL 아이템 (재생성 권장)

- item-003: 주요 피사체 없음, 포맷 불일치
```

## Step 4 — 결과 반환

curate skill은 `curation-report.md` 파일 경로와 함께 PASS/WARN/FAIL 아이템 수를 orchestrator에게 반환한다.
orchestrator는 이 결과를 package 단계(Step 5)와 사용자 확인 체크포인트 #2에서 활용한다.
