# media-generator-harness

자연어 요청으로 이미지를 생성하는 하네스. Google Gemini CLI(Imagen)를 백엔드로 사용한다.

## 사전 준비

### Gemini CLI 설치 및 인증

```bash
# Gemini CLI 설치
npm install -g @google/gemini-cli

# 인증 (Google 계정 로그인)
gemini auth login
```

미설치 또는 미인증 상태에서 `/auto` 실행 시 하네스가 즉시 중단되고 설치·인증 가이드를 출력한다.

## 산출물 경로

모든 산출물은 `.harness-artifacts/media-generator-harness/output/{session-id}/` 하위에 저장한다.
`{session-id}`는 `YYYYMMDD-HHMMSS-{short-slug}` 형식으로 orchestrator가 생성한다.

산출물 목록:
- `media-spec.json` — intake 결과 (미디어 타입·수량·스타일·종횡비 등)
- `prompts/{item-id}.json` — 아이템별 positive/negative prompt + 파라미터
- `media/{item-id}.{png|jpg}` — 생성된 이미지 파일
- `metadata/{item-id}.json` — 사이드카 메타데이터 (prompt, seed, model, backend, timestamp, 파일 크기, 해상도)
- `curation-report.md` — 자동 검수 결과 (전체 아이템 PASS/FAIL/RETRY)
- `README.md` — 갤러리 인덱스 (썸네일 링크 + spec 요약)
- `generation-log.md` — 외부 API 호출 로그 (실패·재시도 포함)

> 생성된 `media/` 디렉토리는 `.gitignore`에 추가를 권장한다 (대용량 바이너리 파일).

## 파이프라인 규칙

1. **모델 배정**: orchestrator·intake·curator는 `claude-opus-4-7`, prompt-engineer·media-generator는 `claude-sonnet-4-6`, packager는 `claude-haiku-4-5-20251001`을 사용한다.
2. **쓰기 범위**: 모든 agent의 파일 쓰기는 `.harness-artifacts/media-generator-harness/output/{session-id}/` 하위로 제한한다. 그 외 경로 쓰기는 금지.
3. **사용자 확인 체크포인트 #1**: intake 직후 `media-spec.json`을 사용자에게 보여주고 생성 시작 여부를 확인한다. 이 단계에서 "이미지 N장 × Gemini Imagen 평균 단가" 수준의 예상 비용을 함께 안내한다. 사용자 승인 없이 외부 API를 호출하지 않는다.
4. **사용자 확인 체크포인트 #2**: package 직후 갤러리 `README.md`와 `curation-report.md`를 보여주고 최종 승인을 받는다.
5. **아이템 단위 병렬**: generate·curate 단계는 item-id별로 sub-agent를 병렬 호출한다. 동시 실행 상한은 `MEDIA_GEN_CONCURRENCY`(기본 3)로 orchestrator가 관리한다.
6. **디렉토리 사전 생성**: orchestrator가 병렬 단계 진입 전에 `media/`, `metadata/`, `prompts/` 디렉토리를 미리 생성한다 (경쟁 조건 회피).
7. **외부 도구 폴백**: 시작 시 `which gemini` 또는 `command -v gemini`로 설치 여부를 확인한다. 미설치·미인증 시 즉시 중단하고 설치·인증 가이드를 출력한다. API 호출 실패(429/5xx) 시 지수 백오프로 1회 재시도, 그래도 실패하면 해당 item을 `curation-report.md`의 FAIL 큐로 이동한다.
8. **agent 격리**: sub-agent 호출 시 `session-id`와 `item-id`(병렬 단계만)만 인자로 전달한다. spec·prompt 본문은 절대 프롬프트에 포함하지 않고 agent가 파일을 직접 Read한다.
9. **메타데이터 필수**: 모든 생성 이미지에 `metadata/{item-id}.json`이 1:1로 존재해야 한다 (재현성·라이선스 추적 목적).
10. **slug 일관성**: `item-id`는 `prompts/`, `media/`, `metadata/` 전 디렉토리에서 동일하게 사용한다.
11. **검수 수준**: 파일 존재·해상도·포맷 검사 + Claude vision으로 prompt 부합도 점수화. 낮은 점수는 사용자에게 보고하며 자동 재생성은 하지 않는다.
12. **v1 범위**: 이미지 생성만 지원한다. `media-spec.json`의 `type` 필드는 항상 `image`로 처리한다. 동영상(Veo) 지원은 v2로 연기한다.

## 커맨드

```
/auto "요청"       ← 전체 파이프라인 실행 (intake → promptify → generate → curate → package)
/intake "요청"     ← 요청 분석 → media-spec.json 생성만 실행
/promptify         ← media-spec.json → 아이템별 prompt JSON 생성만 실행
/generate          ← 승인된 prompt 기반으로 이미지 생성만 실행
/curate            ← 생성된 이미지 자동 검수만 실행
/package           ← 메타데이터 사이드카 + 갤러리 README.md 생성만 실행
```

## 알려진 제한

- Claude Code 자체는 이미지 생성 능력이 없다. 하네스는 외부 Gemini CLI를 호출하는 오케스트레이션 레이어로만 동작한다.
- Gemini Imagen API 사용 시 비용이 발생한다. 체크포인트 #1에서 예상 비용을 확인 후 진행한다.
- 생성 이미지 품질과 prompt 부합도는 Gemini Imagen 모델 성능에 의존하며 보장되지 않는다.
- 동영상(Veo) 생성은 v1에서 지원하지 않는다.
