# 로컬 Claude 세션 대시보드 — 설계안

> 다른 Claude Code 세션이 이 문서만 보고 바로 착수할 수 있도록 정리한 설계안입니다.
> **사용자 컴퓨터에서 로컬로 실행하는 Claude Code 세션에서 작업하세요.** 실제 `~/.claude` 데이터와 로그인된 `claude` CLI가 필요합니다.

## 1. 목표와 제약

**목표:** 내 컴퓨터에서 돌아가는 Claude Code 세션을 브라우저 한 화면에서 보고(목록·진행 상황·사용량), 제어(새 작업 실행, 이어서 지시, 중단)한다.

**제약 (반드시 지킬 것)**
- 로컬 전용. 서버는 `127.0.0.1`에만 바인딩한다. 외부 배포나 클라우드 동기화는 하지 않는다.
- 외부 API를 연동하지 않는다. Anthropic API 키도 쓰지 않는다.
- Claude 호출은 **구독 계정으로 로그인된 로컬 `claude` CLI를 자식 프로세스로 실행**하는 방식으로만 한다. 그래서 사용량이 요금제에서 차감된다.
  - Agent SDK에 구독 계정의 OAuth 토큰을 넣는 방식은 쓰지 않는다.
- 데이터베이스는 필수가 아니다. 진실의 원천은 `~/.claude/projects/` 아래의 `.jsonl` 파일이다.

**범위 밖 (v1)**
- claude.ai 클라우드 세션(Claude Code on the web, 코워크 원격): 서버에만 있는 데이터라 외부 연동 없이는 볼 수 없다.
- 코워크 로컬 세션: 저장 위치와 형식이 확인되지 않았다. 조사 과제로 남긴다(§9).

## 2. 기술 스택

| 영역 | 선택 | 이유 |
|---|---|---|
| 런타임 | Node.js 20+ / TypeScript | `claude` CLI와 같은 생태계, 프로세스 제어 용이 |
| 백엔드 | Fastify + `@fastify/websocket` | 가벼움, REST와 WS를 한 서버에서 처리 |
| 파일 감시 | chokidar | `.jsonl` 추가·변경 실시간 감지 |
| 프론트엔드 | React + Vite + TypeScript | 빠른 개발. 빌드 결과물은 백엔드가 정적 서빙 |
| UI | Tailwind CSS | 별도 디자인 시스템 없이 빠르게 |
| 상태 관리 | TanStack Query (REST) + WS 이벤트로 캐시 갱신 | |
| 테스트 | Vitest | 파서·프로세스 매니저 단위 테스트 |

실행: `npm run dev`(개발), `npm start`(빌드 후 단일 포트 실행). 기본 포트는 `4317`이고 바꿀 수 있다.

## 3. 아키텍처

```
브라우저 (React)
   │  REST + WebSocket (127.0.0.1:4317, 접근 토큰 필요)
   ▼
로컬 서버 (Fastify)
   ├─ SessionIndexer  : ~/.claude/projects/**/*.jsonl 스캔·감시·파싱 → 메모리 인덱스
   ├─ RunManager      : `claude` 자식 프로세스 실행/입력/중단, stream-json 출력 중계
   ├─ UsageAggregator : 메시지의 usage 필드 합산 (세션별·일별·모델별)
   └─ EventBus        : 위 모듈의 변경을 WS로 브로드캐스트
   │
   ├─ 파일 읽기 → ~/.claude/projects/
   └─ 프로세스 실행 → claude CLI (구독 계정 로그인 상태)
```

### 세션 출처는 두 종류

1. **관리 세션**: 대시보드가 직접 실행한 `claude` 프로세스. 출력을 실시간으로 받고, 입력과 중단도 할 수 있다.
2. **외부 세션**: 사용자가 터미널이나 IDE에서 직접 띄운 세션. `.jsonl` 파일 변화로 **읽기 전용 모니터링**만 한다.
   - 실행 중에는 중단하지 않는다(다른 프로세스를 강제 종료하지 않음).
   - 끝난 뒤에는 `--resume`으로 이어서 지시할 수 있다.

## 4. 데이터 소스 상세

### 4.1 세션 기록 파일
- 위치: `~/.claude/projects/<인코딩된 작업 경로>/<sessionId>.jsonl`
  - 환경변수 `CLAUDE_CONFIG_DIR`가 있으면 그 경로를 우선한다.
- 한 줄에 이벤트 하나(JSON)가 있다. 흔히 보이는 필드는 `type`(user/assistant/system/summary 등), `sessionId`, `uuid`, `parentUuid`, `timestamp`, `cwd`, `gitBranch`, `message`(role, content, model, usage)이다.
- ⚠️ **공식 문서화된 형식이 아니다.** 버전에 따라 바뀔 수 있다.
  - 파서는 모르는 필드와 타입을 무시하고, 깨진 줄은 건너뛰는 방어적 구조로 만든다.
  - 착수 첫 단계에서 실제 파일 몇 개를 열어 필드를 확인하고, `src/server/parser/README.md`에 기록한다.
- 큰 파일 대비: 처음에는 전체를 스트림으로 읽고, 이후에는 마지막으로 읽은 바이트 오프셋부터 추가분만 읽는다.

### 4.2 세션 요약 정보 (인덱스에 보관)
```ts
type SessionSummary = {
  id: string;            // sessionId
  project: string;       // cwd
  gitBranch?: string;
  title: string;         // summary 이벤트 또는 첫 user 메시지 앞부분
  startedAt: string;
  lastActivityAt: string;
  messageCount: number;
  lastMessagePreview: string;
  models: string[];
  usage: { input: number; output: number; cacheRead: number; cacheWrite: number };
  source: "managed" | "external";
  status: "running" | "waiting" | "idle" | "done" | "error";
};
```

### 4.3 상태 판정 규칙
- 관리 세션은 프로세스 상태로 정확히 안다: 실행 중이면 running, 정상 종료면 done, 비정상 종료면 error.
- 외부 세션은 추정한다.
  - 파일 수정이 최근 N초(기본 30초) 이내면 running이다.
  - 마지막 이벤트가 assistant의 tool_use이고 응답이 없으면 waiting(권한 승인 대기 추정)이다.
  - 그 외에는 idle이다.
  - 화면에는 "추정"이라는 표시를 함께 띄운다.

## 5. Claude CLI 제어 (RunManager)

> 플래그는 설치된 CLI 버전에서 `claude --help`로 반드시 먼저 확인할 것. 아래는 기준안이다.

| 동작 | 실행 방식 |
|---|---|
| 새 작업 | `claude -p --output-format stream-json --verbose --input-format stream-json` 을 지정 폴더(cwd)에서 실행하고, 첫 프롬프트를 stdin으로 전송 |
| 이어서 지시 | 실행 중이면 같은 프로세스 stdin에 user 메시지 JSON을 추가 전송. 끝난 세션은 `--resume <sessionId>`로 새로 실행 |
| 중단 | SIGINT → 5초 후에도 살아 있으면 SIGTERM |
| 옵션 | `--model`, `--permission-mode`(default / acceptEdits / plan), `--allowedTools`, `--max-turns` |

- stream-json 출력은 줄 단위로 파싱해서 WS로 중계한다. 첫 `system/init` 이벤트에서 sessionId를 얻는다.
- 동시 실행 개수 제한은 기본 3개이고 설정에서 바꿀 수 있다. 요금제 한도를 빨리 소진하지 않기 위해서다.
- **권한 처리**: 헤드리스 모드에서는 대화형 권한 승인을 할 수 없다. v1에서는 실행할 때 permission-mode와 허용 도구를 미리 고르게 한다. 화면에서 승인 버튼을 누르는 방식(`--permission-prompt-tool` + 로컬 MCP 서버)은 v3 과제다.
- 대시보드 서버가 종료되면 관리 세션도 정리한다. 실행 중인 자식 프로세스에 SIGINT를 보낸다.

## 6. 화면 구성

### 6.1 세션 목록 (메인, `/`)
- 상단: 요약 카드 4개(실행 중 / 대기 / 오늘 세션 수 / 오늘 토큰 합계), [+ 새 작업] 버튼
- 필터: 상태, 프로젝트(폴더), 출처(관리/외부), 기간, 검색어(제목·내용)
- 표(실시간 갱신) 열: 상태 배지 · 제목 · 프로젝트 · 브랜치 · 마지막 활동(상대 시간) · 메시지 수 · 토큰 · 출처
- 행 클릭 → 상세 화면. 행에서 바로 [중단] / [이어서] 가능

### 6.2 세션 상세 (`/sessions/:id`)
- 헤더: 제목, 상태, 프로젝트 경로, 브랜치, 모델, 누적 토큰, 버튼(중단 · 이어서 · 폴더 경로 복사 · 터미널용 `claude --resume <id>` 명령 복사)
- 본문: 대화 타임라인
  - user / assistant 메시지는 마크다운으로 렌더링
  - tool_use / tool_result는 접을 수 있는 카드(도구 이름, 입력 요약, 결과 앞부분)
  - 오류는 강조 표시
- 실행 중이면 새 이벤트가 추가될 때 자동 스크롤. 사용자가 위로 스크롤하면 자동 스크롤을 멈춘다.
- 하단 입력창: 관리 세션이나 종료된 세션에 후속 지시를 보낸다. 실행 중인 외부 세션에서는 비활성화하고 이유를 표시한다.

### 6.3 새 작업 실행 (모달)
- 작업 폴더 선택(최근 프로젝트 목록 + 직접 입력), 프롬프트, 모델, 권한 모드, 허용 도구(체크박스), 최대 턴 수
- 실행하면 바로 상세 화면으로 이동한다.

### 6.4 사용량 (`/usage`)
- 일별 토큰 막대 차트(입력/출력/캐시), 모델별 비율, 프로젝트별 상위 목록
- 안내 문구: "요금제 잔여 한도는 로컬에서 알 수 없어 표시하지 않습니다. 기록 파일 기준 토큰 합계입니다."

### 6.5 설정 (`/settings`)
- 포트, 데이터 경로(`~/.claude`), 동시 실행 제한, running 판정 시간(N초), 기본 모델·권한 모드, 접근 토큰 재발급

## 7. 서버 API

**REST** (모든 요청에 접근 토큰 필요)
| 메서드 | 경로 | 설명 |
|---|---|---|
| GET | `/api/sessions?status=&project=&q=&from=&to=` | 세션 요약 목록 |
| GET | `/api/sessions/:id` | 요약 + 이벤트 목록(페이지네이션 `?cursor=`) |
| POST | `/api/runs` | 새 작업 `{cwd, prompt, model?, permissionMode?, allowedTools?, maxTurns?}` → `{runId, sessionId}` |
| POST | `/api/sessions/:id/messages` | 후속 지시 `{prompt}` (실행 중이면 stdin, 아니면 resume) |
| POST | `/api/sessions/:id/interrupt` | 관리 세션 중단 |
| GET | `/api/usage?from=&to=&groupBy=day\|model\|project` | 사용량 집계 |
| GET | `/api/projects` | 알려진 작업 폴더 목록 |
| GET/PUT | `/api/settings` | 설정 |

**WebSocket** `/ws` — 서버에서 브라우저로 보내는 이벤트
- `session.upserted` (요약 변경)
- `session.event` (`{sessionId, event}` 새 타임라인 항목)
- `run.status` (`{runId, sessionId, status, exitCode?}`)
- `usage.updated`

## 8. 보안 (필수)
localhost 서버가 `claude`를 실행할 수 있다는 것은, 막지 않으면 **아무 웹사이트나 내 컴퓨터에서 명령을 실행시킬 수 있다**는 뜻이다. 그래서 다음을 지킨다.
- `127.0.0.1`에만 바인딩한다(`0.0.0.0` 금지).
- 서버를 시작할 때 무작위 접근 토큰을 만들고 `http://127.0.0.1:4317/?token=...` 형태로 터미널에 출력한다. 이후에는 HttpOnly 쿠키로 유지한다. REST와 WS 모두 토큰을 검증한다.
- `Origin` / `Host` 헤더를 검사해서 자기 자신 외에는 거부한다. DNS 리바인딩과 CSRF를 막기 위해서다.
- CORS는 열지 않는다.
- `cwd`는 실제 존재하는 디렉터리인지 검증한다. 프롬프트와 인자는 셸을 거치지 않고 `spawn`의 인자 배열로 넘긴다(`shell: false`).
- 로그에 프롬프트나 파일 내용을 남기지 않는다(옵션으로만 허용).

## 9. 조사 과제 (착수 전·중에 확인)
1. 설치된 `claude` 버전과 §5 플래그 지원 여부 (`claude --version`, `claude --help`)
2. 실제 `.jsonl` 필드 구조 확인 (§4.1). 특히 summary 이벤트, usage 필드 위치, sidechain(서브에이전트) 표시 방식
3. 코워크 로컬 세션 저장 위치·형식 (macOS `~/Library/Application Support/Claude/` 등). 읽을 수 있고 형식이 안정적이면 v3에서 읽기 전용 소스로 추가
4. Windows 경로와 시그널 처리 차이 (SIGINT 대체 방법)

## 10. 디렉터리 구조
```
.
├─ package.json            # workspaces 또는 단일 패키지
├─ src/
│  ├─ server/
│  │  ├─ index.ts          # Fastify 부트스트랩, 정적 서빙, 토큰/Origin 미들웨어
│  │  ├─ config.ts
│  │  ├─ indexer/          # SessionIndexer (scan, watch, tail)
│  │  ├─ parser/           # jsonl → 정규화 이벤트 (+ README.md에 실측 형식 기록)
│  │  ├─ runs/             # RunManager (spawn, stdin, interrupt, stream-json)
│  │  ├─ usage/            # UsageAggregator
│  │  ├─ routes/           # REST 라우트
│  │  └─ ws.ts             # EventBus ↔ WebSocket
│  ├─ shared/types.ts      # 서버·클라이언트 공용 타입
│  └─ web/                 # React + Vite
│     ├─ pages/ (SessionList, SessionDetail, Usage, Settings)
│     ├─ components/ (StatusBadge, Timeline, ToolCard, NewRunModal, ...)
│     └─ lib/ (api.ts, ws.ts)
└─ tests/ (parser fixtures: 익명화한 샘플 jsonl)
```

## 11. 단계별 마일스톤

**M1 — 읽기 전용 대시보드**
- 프로젝트 골격, 보안 미들웨어(§8), SessionIndexer, 파서, 세션 목록과 상세 화면, WS 실시간 갱신
- 완료 기준: 터미널에서 `claude`로 대화하면 목록과 상세 화면에 몇 초 안에 반영된다.

**M2 — 제어**
- RunManager, 새 작업 모달, 후속 지시, 중단, 동시 실행 제한
- 완료 기준: 대시보드에서 실행한 작업의 출력이 실시간으로 보이고, 중단 버튼으로 멈추며, 끝난 세션에 이어서 지시할 수 있다. 사용량은 요금제 계정으로 차감된다(API 키 미사용 확인).

**M3 — 부가 기능**
- 사용량 화면, 설정 화면, 검색, 화면에서 권한 승인(`--permission-prompt-tool`), 코워크 로컬 세션(조사 결과에 따라), 데스크톱 알림(작업 완료·대기 시)

## 12. 작업 원칙 (구현 세션용)
- 파서는 실제 파일로 만든 픽스처 테스트부터 작성한다. 픽스처의 개인 정보와 경로는 익명화한다.
- `~/.claude` 아래 파일은 **절대 수정하거나 삭제하지 않는다**(읽기 전용).
- 외부 네트워크 호출을 추가하지 않는다(폰트·CDN 포함, 모두 로컬 번들).
