# media-generator-harness

자연어 요청으로 이미지를 생성하는 Claude Code 하네스. Google Gemini CLI(Imagen)를 백엔드로 사용한다.

## 파이프라인 흐름

```
/auto "요청"
    │
    ▼
[Step 1] intake          자연어 → media-spec.json (미디어 타입·수량·스타일·종횡비)
    │
    ▼  ★ 체크포인트 #1   media-spec.json + 예상 비용 확인 → 사용자 승인
    │
    ▼
[Step 2] promptify       media-spec.json → prompts/{item-id}.json (N개 병렬 대상 확정)
    │
    ▼
[Step 3] generate        아이템별 gemini CLI 병렬 호출 → media/{item-id}.png (최대 3개 동시)
    │
    ▼
[Step 4] curate          파일 존재·해상도·포맷 검증 + Claude vision 부합도 평가 → curation-report.md
    │
    ▼
[Step 5] package         metadata/{item-id}.json + 갤러리 README.md 생성
    │
    ▼  ★ 체크포인트 #2   갤러리 README.md + curation-report.md 확인 → 최종 승인
```

## 시작하기

```bash
# 1. 레포 클론
git clone <repo-url>
cd media-generator-harness

# 2. Gemini CLI 설치 및 인증
npm install -g @google/gemini-cli
gemini auth login

# 3. Claude Code로 하네스 실행
claude
/auto "고양이가 우주를 떠다니는 감성적인 사진 4장"
```

## 커맨드 목록

| 커맨드 | 설명 |
|--------|------|
| `/auto "요청"` | 전체 파이프라인 실행. 체크포인트 2회 포함 |
| `/intake "요청"` | 요청 분석 → `media-spec.json` 생성만 실행 |
| `/promptify` | `media-spec.json` → 아이템별 prompt JSON 생성만 실행 |
| `/generate` | 승인된 prompt 기반으로 이미지 생성만 실행 |
| `/curate` | 생성된 이미지 자동 검수만 실행 |
| `/package` | 메타데이터 사이드카 + 갤러리 `README.md` 생성만 실행 |

## 지원 포맷

| 항목 | v1 지원 범위 |
|------|-------------|
| 미디어 타입 | 이미지 (PNG, JPG) |
| 생성 백엔드 | Google Gemini CLI (Imagen) |
| 검수 방식 | 파일 검증 + Claude vision 부합도 점수화 |
| 동시 생성 수 | 최대 3개 (MEDIA_GEN_CONCURRENCY 환경변수로 조정) |
| 동영상 | v2에서 지원 예정 (Veo) |

## 외부 의존성

| 도구 | 설치 방법 | 미설치 시 동작 |
|------|-----------|----------------|
| `gemini` CLI | `npm install -g @google/gemini-cli` | 하네스 즉시 중단 + 설치 가이드 출력 |
| Gemini 인증 | `gemini auth login` | 하네스 즉시 중단 + 인증 가이드 출력 |

> Gemini Imagen API는 사용량에 따라 비용이 발생한다. 체크포인트 #1에서 이미지 수량 기준 예상 비용을 확인할 수 있다.

## 산출물 경로

```
.harness-artifacts/media-generator-harness/output/
└── {YYYYMMDD-HHMMSS-short-slug}/       ← session-id (orchestrator 생성)
    ├── media-spec.json                  ← intake 결과
    ├── prompts/
    │   └── {item-id}.json              ← 아이템별 prompt + 파라미터
    ├── media/
    │   └── {item-id}.png               ← 생성된 이미지 (*.gitignore 권장)
    ├── metadata/
    │   └── {item-id}.json              ← 사이드카 메타데이터
    ├── curation-report.md              ← 검수 결과 (PASS/FAIL/RETRY)
    ├── generation-log.md               ← API 호출 로그
    └── README.md                       ← 갤러리 인덱스
```

## Agent 구성

| Agent | 모델 | 역할 |
|-------|------|------|
| orchestrator-agent | claude-opus-4-7 | 파이프라인 흐름 제어, skill 간 인자 전달, 체크포인트 관리 |
| intake-agent | claude-opus-4-7 | 자연어 요청 분석 → `media-spec.json` 생성 |
| prompt-engineer-agent | claude-sonnet-4-6 | `media-spec.json` → 아이템별 `prompts/{item-id}.json` 생성 |
| media-generator-agent | claude-sonnet-4-6 | 단일 아이템 생성 worker (병렬). Gemini CLI 호출 + 결과 저장 |
| media-curator-agent | claude-opus-4-7 | 단일 아이템 검수 worker (병렬). 파일 검증 + vision 부합도 평가 |
| packager-agent | claude-haiku-4-5-20251001 | 메타데이터 사이드카 + 갤러리 `README.md` 생성 |
