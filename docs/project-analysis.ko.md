# OpenCodeReview 프로젝트 분석 및 활용 방안

이 문서는 OpenCodeReview(`ocr`) 저장소를 분석한 결과와, 다른 프로젝트에 연계했을 때의
장점을 정리한 것이다.

## 1. 프로젝트 개요

**한 줄 요약**: Alibaba가 2년간 사내에서 수만 명 개발자에게 쓰던 AI 코드리뷰 도구를
오픈소스화한 Go CLI. Git diff를 읽어 LLM 에이전트로 라인 단위 리뷰 코멘트를 생성한다.
(Apache-2.0, `github.com/alibaba/open-code-review`)

### 핵심 설계 철학

"**결정론적 엔지니어링 × 에이전트 하이브리드**" — 반드시 정확해야 하는 부분은
엔지니어링 로직이, 동적 판단은 LLM이 담당한다.

- **결정론적 부분**: 파일 선택·필터링, 관련 파일 번들링(예: `message_en.properties`와
  `message_zh.properties`를 한 단위로 묶음), 번들별 서브에이전트 격리·병렬 처리,
  템플릿 기반 룰 매칭, 코멘트 위치 보정·반성(reflection) 모듈
- **에이전트 부분**: 리뷰 특화 프롬프트 + 프로덕션 트레이스 분석으로 증류한 전용 툴셋

### 주요 기능

| 영역 | 내용 |
|------|------|
| 커맨드 | `ocr review`(diff), `ocr scan`(전체 파일), `ocr delegate`(호스트 에이전트가 직접 리뷰), `ocr session`, `ocr viewer` |
| LLM | OpenAI/Anthropic/Gemini/Bedrock/Azure 등 멀티 프로바이더 (`internal/llm`) |
| 통합 | Claude Code·Codex·Cursor 플러그인/스킬, VSCode 확장, MCP 서버, GitHub Actions·GitLab·Gerrit·Bitbucket CI |
| 출력 | JSON, SARIF, 브라우저 세션 뷰어 |

### 코드 구조 (~105k LOC Go, 테스트 비중 매우 높음)

- `cmd/opencodereview/` — cobra 기반 CLI 진입점 (review/scan/config/delegate/session/viewer)
- `internal/agent/` — 파일 선택·번들링·예산·preview (결정론 엔진)
- `internal/llmloop/` — 에이전트 루프, 컨텍스트 압축, 재시도, 동시성 풀
- `internal/tool/` — 리뷰 전용 툴 (file_read, code_search, comment collector 등)
- `internal/session/` — 세션 persist/resume/manifest
- `internal/viewer/` — 세션 리뷰 웹 뷰어
- `pages/`(Astro 문서 사이트), `extensions/vscode/`, `plugins/`, `examples/`(각 CI 설정)

### 벤치마크 (AACR-Bench)

50개 레포·200개 PR·10개 언어·1,505개 정답 이슈로 검증. 일반 에이전트(Claude Code)
대비 **Precision/F1 우수, 토큰 ~1/9**, 다만 Recall은 의도적으로 낮춤(노이즈보다
정밀도 중시).

**미계획**: 자동 수정, 범용 코딩 어시스턴트, 자체 LLM 호스팅. OCR은 철저히 "리뷰 전용"
툴이다.

## 2. 이걸로 뭘 할 수 있나

OCR은 "Git 저장소에 붙이는 리뷰 엔진"이다. 붙이는 방식이 네 갈래로 나뉘고, 용도가
각각 다르다.

### 2.1 로컬 CLI — 개발자 손에서

```bash
ocr config provider && ocr config model   # LLM 1회 설정
ocr review                                 # 워킹트리 변경 리뷰
ocr review --from main --to feature        # 브랜치 범위
ocr scan --path internal/agent             # diff 없는 파일 전체 감사
ocr session list / ocr viewer              # 세션 재개·브라우저 리플레이
```

커밋 전 셀프 리뷰, 낯선 코드베이스 온보딩, 대규모 리팩터 검증에 쓸 수 있다.

### 2.2 CI 게이트 — 자동 리뷰 봇

`examples/` 에 GitHub Actions·GitLab CI·Gerrit·Bitbucket·GitFlic 설정이 이미 있다.
`--format json|sarif` 로 출력하면 SARIF를 GitHub Code Scanning에 바로 물릴 수 있어
**PR마다 리뷰 코멘트 + 보안 탭 알림**이 자동화된다.

### 2.3 Delegate 모드 — 호스트 에이전트에 위임 (가장 실용적)

```bash
ocr delegate preview          # 리뷰 대상 파일 목록 + 제외 사유만 뽑기
ocr delegate rule src/a.go    # 파일별 적용 규칙 JSON만 뽑기
```

OCR은 **LLM을 호출하지 않고** 결정론적 부분(파일 선택·필터·규칙 해석·diff 수집)만
담당하고, 실제 추론은 Claude Code/Codex/Cursor 같은 호스트의 구독 LLM이 한다.
→ **별도 API 키 불필요.**

```bash
npx skills add alibaba/open-code-review --skill open-code-review-delegate
```

한 줄로 붙는다.

### 2.4 파이프라인 부품 — 규칙·MCP 연동

- `.opencodereview/rule.json` 을 레포에 커밋 → 팀 규칙을 코드처럼 버전 관리
  (4계층 우선순위: CLI `--rule` > 프로젝트 > 전역 > 내장 시스템 기본값)
- OCR은 **MCP 클라이언트**로 동작 → Jira/GitHub 이슈, 사내 문서, 린터·스키마 검증기를
  리뷰어의 도구로 꽂을 수 있다 (`ocr config set mcp_servers.…`).
  참고: OCR *자신을* MCP 서버로 노출하는 기능은 아직 없다 (ROADMAP에만 언급).

## 3. 다른 프로젝트에 연계했을 때의 장점

| 장점 | 근거 |
|---|---|
| **커버리지가 안 새어나감** | 파일 선택·번들링이 LLM 판단이 아니라 엔지니어링 로직이라, 큰 changeset에서도 에이전트처럼 파일을 골라 건너뛰지 않는다 |
| **위치 정확도** | 독립된 코멘트 위치 보정 + 반성(reflection) 모듈로 "라인 드리프트"를 잡는다 |
| **비용·속도** | 동일 모델로 일반 에이전트 대비 토큰 **~1/9**, CI 지연 부담이 작다 |
| **정밀도 우선** | Precision/F1이 높아 false alarm 트리아지 비용이 적다 (Recall은 의도적으로 낮춤) |
| **재현성** | 자연어 스킬과 달리 템플릿 기반 규칙 매칭이라 리뷰 품질이 프롬프트 변형에 흔들리지 않는다 |
| **락인 없음** | 멀티 LLM 프로바이더 + Delegate 모드로 기존 구독·게이트웨이를 그대로 재사용 |
| **큰 리포에서 안정** | 번들별 서브에이전트 격리·병렬 실행 → 컨텍스트 폭발 없이 확장 |

## 4. 이 저장소 자체를 다룰 때

- 순수 Go(1.25), ~105k LOC, **테스트 파일이 소스만큼 많음**.
  `make test`(LC_ALL=C 고정), `make check`, `make coverage`(90% 게이트)로 검증.
- 소스는 **영어만** 허용(`make english-check`). 번역문은 `docs/i18n/*`,
  `pages/src/content/docs/<locale>/`, `pages/src/i18n/<locale>.ts` 에만 둔다.
- AGENTS.md가 엄격하다: **AI 사용을 이슈/PR에 공개**, AI 커밋 트레일러 금지,
  `AI생성 → 수정 → 수정` 반복 금지.

## 결론

기존 프로젝트에 붙이는 순간 "PR마다 자동으로 도는, 토큰 적게 먹고 위치가 정확한 리뷰
게이트"가 생기고, API 키를 새로 안 만들고 싶으면 Delegate 모드로 호스트 에이전트
구독에 얹으면 된다.
