# pi-dynamic-workflows 전수조사 & 활용 전략 정리 📊

> 이 문서는 `pi-dynamic-workflows` 저장소를 파일 단위로 전수조사한 결과와,
> 설치·사용법 / 정체 / 수익화 / 구현 가능성까지 논의한 내용을 정리한 기록입니다.

- 📅 작성일: 2026-09-29
- 🖋️ 작성: 카리나 (Claude Code 세션)

## 🔗 관련 GitHub 주소

| 대상 | 주소 | 비고 |
|---|---|---|
| 이 저장소 (작업 대상) | https://github.com/bmshin94/pi-dynamic-workflows | 분석·정리 진행한 곳 |
| 원본 저장소 | https://github.com/Michaelliv/pi-dynamic-workflows | ⭐ 약 1,228 / 🍴 71 |
| 호스트 프로젝트 Pi | https://github.com/earendil-works/pi | ⭐ 약 110,000 |
| npm 패키지 | https://www.npmjs.com/package/pi-dynamic-workflows | v1.0.1, MIT |
| 아이디어 원조 (Anthropic) | https://claude.com/blog/introducing-dynamic-workflows-in-claude-code | Claude Code Dynamic Workflows |

### 참고: 같은 생태계의 유사 프로젝트

| 저장소 | ⭐ | 특징 |
|---|---|---|
| https://github.com/tintinweb/pi-subagents | ~1,230 | 서브에이전트 + 플릿뷰 + 중간 조종 |
| https://github.com/QuintinShaw/pi-dynamic-workflows | ~548 | 재개(journaled resume), git worktree 격리, 비용 집계, `/workflows` TUI |
| https://github.com/six-ddc/codex-dynamic-workflows | ~22 | Codex / Gemini 백엔드로 이식 |
| https://github.com/kelvinschen/acpus | ~40 | ACP 에이전트 오케스트레이션 (durable workflows) |

---

## 1. 한 줄 정의

> **Pi(오픈소스 코딩 에이전트 CLI)에 `workflow` 툴 하나를 추가하는 확장(extension).**
> 메인 AI가 그 자리에서 자바스크립트를 작성해, 여러 서브에이전트에게 작업을 팬아웃하고
> 결과를 다시 합성(fan-in)하게 만든다.

핵심은 **"Dynamic"** — 사람이 YAML로 미리 파이프라인을 짜두는 게 아니라,
**LLM이 실행 시점에 코드로 워크플로우를 설계**한다.

---

## 2. 파일 전수조사

총 21개 파일 / 소스 약 1,586줄. 의존성은 런타임 `acorn` 단 1개.

```
pi-dynamic-workflows/
├── extensions/workflow.ts      # 🔌 진입점 (14줄)  - registerTool + session_start 훅
├── src/
│   ├── index.ts                # 📦 export 창구 (30줄)
│   ├── workflow.ts             # 🧠 심장 (455줄) - AST 파서 + vm 샌드박스 런타임
│   ├── workflow-tool.ts        # 🛠️ Pi 툴 정의 (209줄) - 스키마 + 프롬프트 가이드 + 렌더
│   ├── agent.ts                # 🤖 서브에이전트 러너 (131줄) - in-memory Pi 세션
│   ├── structured-output.ts    # 📋 종료형 구조화 출력 툴 (47줄)
│   └── display.ts              # 🖼️ 스냅샷 + 진행 렌더러 (235줄)
├── types/workflow.d.ts         # ⌨️ 워크플로우 전역 타입 (95줄, IntelliSense용)
├── tests/                      # ✅ parser/runtime/display/tool 4개 파일 414줄
├── .github/workflows/
│   ├── ci.yml                  # push/PR 시 npm test
│   └── release.yml             # v* 태그 → GitHub Release + npm publish
├── package.json                # v1.0.1, MIT, "pi": { "extensions": [...] }
├── biome.json / tsconfig.json  # 린터·포매터 / TS 설정
├── CLAUDE.md                   # (이 저장소에 추가된 페르소나 가이드)
└── README.md                   # 문서 (예제·표·원리 포함, 완성도 높음)
```

> ⚠️ 현재 체크아웃엔 `node_modules`가 없음 → 테스트 돌리려면 `npm install` 먼저.

---

## 3. 동작 원리

```text
사용자: "워크플로우로 이 저장소 감사해줘"
  → Pi 메인 모델이 JavaScript 스크립트를 작성
  → workflow 툴이 파싱(AST 검증) + Node vm 샌드박스에서 실행
  → 스크립트의 agent() / parallel() / pipeline() 호출
  → 각 agent()가 in-memory Pi 서브세션을 생성 (표준 코딩 툴 보유)
  → 스냅샷이 실시간 진행 표시로 스트리밍
  → 최종 JSON 결과를 부모 어시스턴트에 반환
```

### 워크플로우 스크립트 형태

```js
export const meta = {                 // 반드시 첫 문장
  name: 'inspect_project',            // 필수
  description: 'Inspect a repository',// 필수
  phases: [{ title: 'Scan' }],        // 선택 (문서용 개요)
}

phase('Scan')
const inventory = await agent('저장소 구조 파악', { label: 'repo inventory' })

phase('Analyze')
const summary = await agent('요약: ' + inventory, { label: 'module summary' })

return { inventory, summary }
```

### 사용 가능한 전역 (`src/workflow.ts:186`)

| 전역 | 설명 |
|---|---|
| `agent(prompt, opts)` | 서브에이전트 1개 실행. 텍스트 또는 `opts.schema`로 검증된 객체 반환 |
| `parallel(thunks)` | `() => agent(...)` 배열을 동시 실행. 결과는 **입력 순서** 유지 |
| `pipeline(items, ...stages)` | 항목별로 단계 순차 통과, 항목끼리는 병렬. 각 단계 `(prev, original, index)` |
| `phase(title)` | 현재 단계 표시 → 진행 화면 그룹핑 |
| `log(message)` | 워크플로우 로그 |
| `args` | 툴의 `args` 파라미터로 들어온 JSON |
| `cwd`, `process.cwd()` | 서브에이전트 작업 경로 |
| `budget` | `{ total, spent(), remaining() }` 토큰 예산 추적 |

### 설계 하이라이트

| # | 내용 | 위치 |
|---|---|---|
| 1 | **결정론 강제** — AST를 재귀 순회해 `Date.now()`/`new Date()`/`Math.random()` 차단 | `workflow.ts:312` |
| 2 | **meta 리터럴 강제** — 스프레드·계산된 키·템플릿 보간·함수호출 금지, `__proto__`/`constructor`/`prototype` 키 차단(프로토타입 오염 방어) | `workflow.ts:267` |
| 3 | **허용목록 샌드박스** — `fs`/네트워크/`require`/`import`/진짜 `process` 미제공. `console`은 로그 함수로 교체 | `workflow.ts:186` |
| 4 | **부분 실패 허용** — 서브에이전트 실패 시 전체 중단 대신 `null` 반환 + 로그 | `workflow.ts:124` |
| 5 | **동시성 자동 조절** — `min(hardwareConcurrency - 2, 16)`, 17줄짜리 자체 limiter | `workflow.ts:71`, `:388` |
| 6 | **취소 전파** — `AbortSignal`이 최하위 `session.abort()`까지 전달, 리스너 정리까지 | `workflow-tool.ts:136`, `agent.ts:70` |
| 7 | **구조화 출력 + 조기 종료** — `terminate: true`로 마지막 인사 턴 토큰 절약 | `structured-output.ts:37` |
| 8 | **결과 직렬화 검증** — `structuredClone` 실패 시 "await 빠뜨렸나?" 안내 | `workflow.ts:429` |

### 진행 표시 아이콘 (`src/display.ts:207`)

`○` 대기 · `●` 실행중 · `✓` 성공 · `✗` 실패 · `-` 건너뜀 / 단계 진행중은 `▶`

---

## 4. 언제 쓰고, 언제 쓰지 말아야 하나

### ✅ 적합
- 코드베이스 전체 감사 (모듈별 병렬 분석)
- 다관점 리뷰 (보안 / 성능 / 가독성 동시 심사)
- 대규모 리팩터링 (파일 단위 팬아웃)
- 팬아웃 리서치 (후보 N개 조사 → 비교표 합성)

### ❌ 부적합
- 파일 하나 읽기/한 줄 수정 (일반 툴이 빠름)
- 앞 결과가 반드시 필요한 완전 순차 작업 (병렬 이득 없음)
- 프롬프트 가이드에 **"사용자가 workflow / fan-out / multi-agent를 명시적으로 요청할 때만 사용"** 이 명시돼 있음 (`workflow-tool.ts:57`)

### 실익
1. ⏱️ 속도 — 순차 대비 N배 단축
2. 🎯 품질 — 서브에이전트마다 깨끗한 새 세션 → 컨텍스트 오염 없음
3. 💰 **메인 컨텍스트 절약** — 서브가 1만 줄 읽어도 메인엔 요약만 돌아옴 (체감상 최대 이득)
4. 🎓 학습 교재 — "LLM이 쓴 코드를 안전하게 실행하는 법" 레퍼런스

---

## 5. 현재 한계 (README "Status"에도 명시)

- 🚧 프로토타입 단계
- ❌ 저장/재개(resumable runs) 미지원 → 중단 시 처음부터
- ❌ `/workflows` 관리 UI 없음
- ⚠️ `opts.model` / `opts.isolation: 'worktree'` / `opts.agentType`은 **실제 동작 X** — 서브에이전트 프롬프트에 텍스트 힌트로만 삽입됨 (`workflow.ts:444`)
- ⚠️ `budget.total` 기본값이 `null`(무제한)이고, 툴 레벨에서 `tokenBudget`을 넘기지 않음 → 실제 상한을 걸려면 코드 수정 필요
- ⚠️ 토큰 추정이 `JSON.stringify(결과).length / 4` 뿐 (`workflow.ts:453`) — 입력 토큰·모델별 단가·캐시 할인 미반영
- ⚠️ Node `vm`은 완전한 보안 샌드박스가 아님 → "실수 방지 + 결정론 보장" 용도로 봐야 함. 진짜 격리는 별도 프로세스/컨테이너 필요
- 🔗 Pi 전용 (Claude Code / Cursor에 그대로 못 붙임)

---

## 6. 설치 및 사용법

### 전제
Pi CLI가 먼저 설치돼 있어야 함 (정확한 명령은 https://github.com/earendil-works/pi 참고).

```bash
npm install -g @earendil-works/pi-coding-agent
```

### 설치

```bash
# A) npm에서
pi install npm:pi-dynamic-workflows

# B) 로컬 체크아웃에서
pi install /path/to/pi-dynamic-workflows
```

Pi 안에서:

```text
/reload
```

`session_start` 이벤트에서 `workflow` 툴이 자동 활성화됨 (`extensions/workflow.ts:8`).

### 사용

```text
워크플로우로 이 저장소 조사해서 주요 모듈 요약해줘
Run a workflow to review src/ from security, performance, and readability angles.
```

> 💡 **"워크플로우"라는 말을 명시해야** 툴이 발동한다 (가이드라인에 그렇게 지시돼 있음).
> 실행 중 `Esc` → 전체 취소, 진행 중이던 서브에이전트는 `skipped` 처리.

### 개발

```bash
npm install
npm test        # biome check + tsc build + unit tests
npm run check   # 린트/포맷만
npm run build   # dist/ 생성
```

### 에디터 자동완성

```js
/// <reference types="pi-dynamic-workflows/workflow" />
```

---

## 7. 플러그인? 스킬? MCP?

### 결론: **플러그인(Pi Extension)**

`package.json` 근거:
```json
"keywords": ["pi-package", "pi", "workflow", "agents"],
"pi": { "extensions": ["extensions/workflow.ts"] }
```

| 구분 | 정체 | 실행 위치 | 해당 |
|---|---|---|---|
| 플러그인/확장 | 호스트 앱 안에 코드 주입, 툴 등록·이벤트 훅 | 호스트와 동일 프로세스 | ✅ |
| MCP 서버 | 별도 프로세스, stdio/HTTP 프로토콜 | 독립 프로세스 | ❌ |
| 스킬 | 프롬프트·문서 뭉치, 코드 실행 없음 | — | ❌ |

**MCP가 아닌 근거:** `pi.registerTool()` 직접 호출, `createAgentSession()`으로 같은 프로세스 내 in-memory 세션 생성, MCP 매니페스트/JSON-RPC 코드 전무.

**단, 라이브러리로는 쓸 수 있음** — `src/index.ts`가 전부 export하므로 직접 MCP 서버나 HTTP API로 감쌀 수 있다.

```ts
import { runWorkflow, parseWorkflowScript, WorkflowAgent } from 'pi-dynamic-workflows'
```

---

## 8. API 토큰

| 레이어 | 토큰 필요? | 설명 |
|---|---|---|
| 이 확장 자체 | ❌ | 코드에 API 키를 읽는 부분이 없음. 인증은 `authStorage`로 Pi에 위임 (`agent.ts:18`) |
| Pi CLI (호스트) | ✅ | Anthropic/OpenAI 키 또는 계정 로그인. 워크플로우가 `ctx.modelRegistry`/`ctx.model`을 상속 (`workflow-tool.ts:99`) |
| 배포용 | 선택 | 포크해서 npm 배포할 때만 `NPM_TOKEN` (GitHub Secrets, `release.yml:44`) |

### ⚠️ 비용 경고

```
일반 질문 1번  = LLM 호출 1회
워크플로우 1번 = LLM 호출 N+1회  (서브에이전트 N + 최종 합성)
```

기본 동시성이 **CPU 코어 - 2, 최대 16**이라 토큰 소모가 매우 빠르다.

방어책:
```js
log('남은 예산: ' + budget.remaining())   // 스크립트 안에서
```
```ts
createWorkflowTool({ concurrency: 3 })   // 확장 코드에서 동시성 제한
```
그리고 폭주 시 즉시 `Esc`.

---

## 9. GitHub에서 유명한 이유

1. 🌊 **거대한 생태계의 명확한 빈칸** — Pi 본체(⭐110k)에 서브에이전트 병렬 실행이 없었다
2. 🏆 **Claude Code 후광** — 검증된 아이디어를 빠르게 이식 (마케팅 비용 0)
3. ⚡ **타이밍** — Anthropic 발표 직후 생성 (2026-05-28), first mover
4. 💎 **실제 품질** — 런타임 의존성 1개, 테스트 414줄, CI + 자동 배포, README 완성도
5. 🎓 **읽을 만한 크기** — 1,586줄로 AST 검증 + vm 샌드박스 + 동시성 + 취소를 다 보여줌
6. 😌 **정직함** — 한계를 README에 명시

### 냉정한 평가
- 스타 상당수는 "나중에 써보려는 북마크" (Pi 안 쓰면 사용 불가)
- Issue 17개 open, 마지막 푸시 5월 말 → 활발한 유지보수는 아님
- 기능은 `QuintinShaw` 포크가 더 많음 (재개, 워크트리, 비용 집계, TUI)
- 결론: **"아이디어 검증 + 학습 교재"로 유명한 프로젝트.** 프로덕션 표준은 아직 아님

---

## 10. 로컬 에이전트 구축에 도움되는가 → 매우 그렇다

### 바로 재사용 가능한 패턴

| ⭐ | 패턴 | 위치 | 왜 |
|---|---|---|---|
| ⭐⭐⭐⭐⭐ | **AST 화이트리스트 검증** | `workflow.ts:267`, `:312` | LLM 생성 코드/설정을 안전하게 파싱하는 정석. `eval` ❌, 정규식 ❌ |
| ⭐⭐⭐⭐⭐ | **허용목록 vm 컨텍스트** | `workflow.ts:186` | "뭘 막을까"가 아니라 "뭘 허용할까" 사고법 |
| ⭐⭐⭐⭐⭐ | **동시성 제한기 17줄** | `workflow.ts:388` | `p-limit` 없이 레이트리밋 방어 |
| ⭐⭐⭐⭐⭐ | **구조화 출력 + terminate** | `structured-output.ts:37` | 마지막 턴 제거로 토큰 절약 |
| ⭐⭐⭐⭐ | **AbortSignal 전파 체인** | `workflow-tool.ts` → `workflow.ts` → `agent.ts:70` | 취소 처리 완성형 예제 |
| ⭐⭐⭐⭐ | **UI/로직 분리 진행 표시** | `display.ts` (특히 `:102`) | UI 있으면 위젯, 없으면 텍스트 자동 전환 |
| ⭐⭐⭐⭐⭐ | **프롬프트 가이드라인 16줄** | `workflow-tool.ts:56-71` | "AI가 실수할 지점을 미리 다 막은" 레퍼런스 |

### 읽는 순서 (약 1시간)

```
1. extensions/workflow.ts   (14줄  — 전체 그림)
2. src/structured-output.ts (47줄  — JSON 강제 기법)
3. src/agent.ts             (131줄 — 서브에이전트 생명주기)
4. src/display.ts           (235줄 — 진행 UI)
5. src/workflow-tool.ts     (209줄 — 프롬프트 엔지니어링)
6. src/workflow.ts          (455줄 — 파서 + 런타임)
```

### 🔑 핵심 발견: Pi 없이도 엔진만 쓸 수 있다

```ts
const result = await runWorkflow(script, {
  agent: 나만의에이전트,   // { run(prompt, opts) } 하나만 있으면 됨
  concurrency: 3,
  tokenBudget: 100_000,
})
```

`options.agent`는 `Pick<WorkflowAgent, "run">` 타입 (`workflow.ts:22`, `:70`).
`tests/workflow-runtime.test.ts:5`의 가짜 에이전트가 **3줄**뿐인 게 증거.
→ Ollama / vLLM / OpenAI 등 자체 에이전트를 꽂아 쓸 수 있다.

### 로컬 LLM 사용 시 주의
- 동시 16개는 로컬 GPU에 과함 → VRAM 기준으로 2~4개
- 작은 모델(7B급)은 JSON Schema 준수를 잘 못함
- 재개 기능이 없어 긴 작업 중단 시 손실 → journaling 직접 추가 시 원본보다 개선됨

---

## 11. React / PHP로 만들 수 있나

| 부분 | React | PHP | 비고 |
|---|---|---|---|
| 결과 보기 UI / 대시보드 | ✅ 최적 | ✅ 가능 | 실시간이면 React 압도적 |
| 실행 트리거 + 진행률 스트리밍 | ✅ 최적 | ⚠️ 제한적 | SSE/WebSocket 필요 |
| 엔진(파서 + 샌드박스 실행) | ❌ 비추 | ❌ 거의 불가 | 아래 참조 |
| 서브에이전트 세션 관리 | ❌ | ❌ | Pi의 Node 내부 기능 |

**React(브라우저) 불가 이유:** `node:vm` 없음 / 파일·셸 접근 차단 / API 키 프론트 노출 위험

**PHP 불가 이유:** JS AST 파서(acorn) 부재 / JS 샌드박스 부재(V8Js는 사실상 사장) / 요청-응답 모델이 장시간 16병렬 스트리밍에 부적합 / AbortSignal급 취소 전파 메커니즘 없음

### ✅ 정답 구조 — 엔진은 Node 그대로, 껍데기만 만든다

```
┌─────────────────────────────────┐
│  프론트엔드 (React)               │  실시간 진행바, 결과 트리,
│                                 │  비용 차트, 리포트 공유
└──────────────┬──────────────────┘
               │ WebSocket / SSE
┌──────────────▼──────────────────┐
│  API 서버 (PHP 또는 Node)         │  인증, 결제, 저장, 권한
└──────────────┬──────────────────┘
               │ 작업 큐 (Redis / DB)
┌──────────────▼──────────────────┐
│  워커 (Node.js)                  │  runWorkflow() 그대로 import
│  pi-dynamic-workflows 사용        │  ← 여기만 Node 필수
└─────────────────────────────────┘
```

### 최소 예제 (SSE)

백엔드 (Node):
```ts
import express from 'express'
import { runWorkflow } from 'pi-dynamic-workflows'

const app = express()

app.post('/api/run', express.json(), async (req, res) => {
  res.setHeader('Content-Type', 'text/event-stream')
  res.setHeader('Cache-Control', 'no-cache')
  const send = (type, data) => res.write(`data: ${JSON.stringify({ type, data })}\n\n`)

  try {
    const result = await runWorkflow(req.body.script, {
      concurrency: 3,
      tokenBudget: 200_000,
      onPhase:      (title) => send('phase', title),
      onAgentStart: (e) => send('start', { label: e.label, phase: e.phase }),
      onAgentEnd:   (e) => send('end',   { label: e.label, ok: e.result !== null }),
      onLog:        (m) => send('log', m),
    })
    send('done', result)
  } catch (e) {
    send('error', String(e))
  }
  res.end()
})

app.listen(3000)
```

프론트 (React):
```tsx
function WorkflowRunner({ script }: { script: string }) {
  const [phases, setPhases] = useState<string[]>([])
  const [agents, setAgents] = useState<Record<string, 'running' | 'done' | 'error'>>({})

  const run = async () => {
    const res = await fetch('/api/run', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ script }),
    })
    const reader = res.body!.getReader()
    const decoder = new TextDecoder()

    for (;;) {
      const { done, value } = await reader.read()
      if (done) break
      for (const line of decoder.decode(value).split('\n\n')) {
        if (!line.startsWith('data: ')) continue
        const { type, data } = JSON.parse(line.slice(6))
        if (type === 'phase') setPhases((p) => [...new Set([...p, data])])
        if (type === 'start') setAgents((a) => ({ ...a, [data.label]: 'running' }))
        if (type === 'end')   setAgents((a) => ({ ...a, [data.label]: data.ok ? 'done' : 'error' }))
      }
    }
  }

  return <button onClick={run}>워크플로우 실행 ▶</button>
}
```

`onPhase` / `onAgentStart` / `onAgentEnd` / `onLog` 콜백이 이미 준비돼 있어 연결이 쉽다 (`workflow.ts:26-29`).

### PHP로 갈 경우
PHP는 **API 서버 + 큐 조율자** 역할로 100% 가능. 실행만 Node 워커에 위임.

```php
$jobId = uniqid();
Redis::lpush('workflow_queue', json_encode([
    'id' => $jobId, 'script' => $script, 'userId' => $user->id,
]));
return response()->json(['jobId' => $jobId]);
```

### 🔐 보안 수칙 3가지
1. LLM API 키는 절대 프론트로 내리지 않는다 (서버 전용)
2. 워커는 컨테이너 격리 — `createCodingTools`가 파일·셸 접근 권한을 가짐. 남의 스크립트를 그냥 실행하면 대참사. Docker + 읽기전용 마운트 + 네트워크 차단
3. 사용자별 토큰 상한을 서버 레벨에서도 강제 (`tokenBudget`만 믿지 말 것)

---

## 12. 수익화 아이디어

### 전제 3가지
1. **엔진은 팔 수 없다** — MIT 라이선스, 누구나 무료 사용
2. **개발자에게 툴 파는 건 어렵다** — 개발자가 만든 *결과물*을 비개발자에게 파는 쪽이 유리
3. **팔 수 있는 것** — ① 좋은 워크플로우 스크립트(설계 노하우) ② 사람이 읽을 수 있는 리포트 ③ 정확한 비용 추적/팀 관리 ④ 도메인 전문성

라이선스는 **MIT** — 상업적 이용·수정·재배포·유료 판매 모두 자유 (조건: 저작권 표시 유지).

### 아이디어 비교

| 아이디어 | 난이도 | 기간 | 시장 | 마진 | 추천 |
|---|---|---|---|---|---|
| ① 자동 코드 감사 SaaS | 높음 | 2~3개월 | 큼 | 중 | ⭐⭐⭐⭐⭐ |
| ② 워크플로우 템플릿 마켓 | 중간 | 1~2개월 | 중 | 높음 | ⭐⭐⭐⭐ |
| ③ 비용 관측 대시보드 | 중간 | 2개월 | 중 | 높음 | ⭐⭐⭐⭐ |
| ④ 교육 콘텐츠 | **낮음** | **2~4주** | 큼 | **90%+** | ⭐⭐⭐⭐⭐ |
| ⑤ 특화 감사 서비스형 | 낮음 | 2주 | 작음 | 중 | ⭐⭐⭐ |
| ⑥ 멀티백엔드 포팅 | 중간 | 1개월 | — | 낮음 | ⭐⭐ |
| ⑦ 비개발자용 노코드 래퍼 | 높음 | 3개월+ | 매우 큼 | 중 | ⭐⭐⭐⭐ |

### ① 자동 코드 감사 SaaS
GitHub 저장소 연결 → 워크플로우가 관점별 동시 감사 → PDF/웹 리포트 발행.

감사 관점: 보안 취약점 / 성능 병목 / 테스트 커버리지 / 의존성 위험 / 문서화 / 아키텍처 스멜 / 접근성 / 시크릿 유출 → 최종 합성 에이전트가 우선순위 실행계획 작성.

| 플랜 | 가격 | 내용 |
|---|---|---|
| Free | ₩0 | 공개 저장소 1회, 요약만 |
| Pro | 월 ₩39,000 | 비공개 5개, 주간 자동 감사 |
| Team | 월 ₩199,000 | 무제한 + PR 자동 코멘트 + Slack |
| Enterprise | 협의 | 온프레미스, 커스텀 관점, SSO |

MVP 3주: (1주) GitHub OAuth + clone + 3관점 / (2주) React 리포트 + PDF / (3주) 결제 + 랜딩
리스크: CodeRabbit·Snyk·SonarQube 경쟁 → "AI 멀티관점 + 경영진용 리포트"로 차별화. **감사 1회 원가 실측 후 가격 설계 필수.**

### ② 워크플로우 템플릿 마켓플레이스
엔진은 쉽고 **좋은 스크립트 작성이 어렵다** → 그 노하우를 상품화.

예: 레거시 마이그레이션 감사 ₩29,000 / RN 출시 전 체크 ₩39,000 / OSS 라이선스 컴플라이언스 ₩49,000 / 다국어 번역 크로스체크 ₩19,000 / WCAG 2.2 접근성 감사 ₩59,000

수익: 판매 수수료 20~30% + 구독형 월 ₩29,000. 콜드스타트 대응으로 **직접 제작 10개 선판매** 후 외부 개방. 복제 방어는 "템플릿 + 지속 업데이트 + 지원" 묶음 판매.

### ③ 비용 관측 대시보드
현재 비용 추정이 `JSON.stringify(결과).length / 4` 뿐(`workflow.ts:453`) → 실제 지출을 아무도 모른다. 이게 지불 의사가 있는 고통.

기능: 실제 토큰 기반 정확한 원가(모델별 단가 + 캐시 할인) / 팀·프로젝트·개인별 분해 / 예산 초과 알림 및 자동 차단 / 가장 비싼 워크플로우 Top 10 / CI 통합.
가격: 팀당 월 ₩99,000(5명) ~ ₩299,000(20명). 세일즈 포인트는 "서비스 비용 < 절감액".
보너스: 비용 추적 개선을 원본에 PR 기여 → 생태계 전문가 포지셔닝 = 마케팅.

### ④ 교육 콘텐츠 (가장 빠른 수익화)
제작 2~4주, 초기비용 거의 0, 마진 90%+, 시장 확장 중.

커리큘럼:
```
1장. 왜 멀티에이전트인가 — 순차 vs 병렬 실측
2장. LLM이 쓴 코드를 안전하게 실행하기 — AST 화이트리스트
3장. vm 샌드박스 설계 — 허용목록 사고법
4장. 동시성 제한기 17줄로 만들기
5장. 취소(AbortSignal) 전파 체인
6장. 구조화 출력으로 토큰 절약
7장. 실시간 진행 UI (터미널 + 웹)
8장. 프롬프트 가이드라인 작성법 (실제 16줄 해부)
9장. 실전: 나만의 에이전트 오케스트레이터
10장. 비용 관리와 프로덕션 배포
```

| 형태 | 가격 | 제작 |
|---|---|---|
| 전자책/PDF | ₩29,000 | 2주 |
| 온라인 강의 | ₩99,000 | 4주 |
| 기업 워크샵 | 1회 ₩2,000,000+ | 마진 최고 |
| 유튜브/블로그 | 유입 채널 | 다른 상품의 마케팅 |

### ⑤~⑦ 요약
- ⑤ **특화 감사 서비스형** — 결과물 판매형 컨설팅(예: 접근성 감사 1건 ₩500,000). 90% 자동화 + 10% 사람 검수. 즉시 매출, 확장성 낮음
- ⑥ **멀티백엔드 포팅** — 이미 `six-ddc/codex-dynamic-workflows`가 진행 중. 직접 수익화는 어려우나 명성 쌓기용
- ⑦ **비개발자용 노코드 래퍼** — 마케터·기획자 대상(예: 경쟁사 웹사이트 5개 자동 비교). 시장 최대, 제작 난이도 최고

### 권장 실행 순서

```
0~1개월 : 블로그 기술 시리즈 무료 공개 (비용 0, 시장 검증 + 브랜딩)
1~3개월 : 반응 좋으면 전자책·강의로 수익화 + 코드 감사 SaaS MVP
3~6개월 : SaaS 유료 전환 + 기업 워크샵으로 스케일
```

이유: SaaS부터 만들면 "만들었는데 아무도 안 씀" 리스크가 크다. 콘텐츠 선행이면 ① 시장 반응 무료 측정 ② 초기 고객 확보 ③ 역량 축적을 동시에 얻는다.

---

## 13. 빠른 참조 — 코드 위치 색인

| 알고 싶은 것 | 파일:줄 |
|---|---|
| 확장 진입점 / 툴 등록 | `extensions/workflow.ts:4` |
| 워크플로우 툴 스키마 | `src/workflow-tool.ts:14` |
| AI용 프롬프트 가이드라인 16줄 | `src/workflow-tool.ts:56-71` |
| 취소(abort) 처리 | `src/workflow-tool.ts:135` |
| 동시성 계산 | `src/workflow.ts:71` |
| agent() 구현 | `src/workflow.ts:101` |
| parallel() 구현 | `src/workflow.ts:139` |
| pipeline() 구현 | `src/workflow.ts:158` |
| vm 컨텍스트 (허용 목록) | `src/workflow.ts:186` |
| 스크립트 파서 / meta 추출 | `src/workflow.ts:228` |
| 리터럴 전용 평가기 | `src/workflow.ts:267` |
| 결정론 AST 검사 | `src/workflow.ts:312` |
| 동시성 제한기 | `src/workflow.ts:388` |
| 토큰 추정 | `src/workflow.ts:453` |
| 서브에이전트 세션 생성 | `src/agent.ts:61` |
| abort 리스너 등록/정리 | `src/agent.ts:70-77` |
| 구조화 출력 계약 프롬프트 | `src/agent.ts:104` |
| terminate: true 조기 종료 | `src/structured-output.ts:40` |
| 진행 렌더러 | `src/display.ts:128` |
| 상태 아이콘 | `src/display.ts:207` |
| 워크플로우 전역 타입 | `types/workflow.d.ts:11` |

---

*이 문서는 저장소 전체(21개 파일, 소스 1,586줄)를 직접 읽고 작성했습니다.* ✨
