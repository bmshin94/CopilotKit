# CopilotKit 전수조사 분석 노트 (한국어)

> 작성: Claude Code 세션 / 대상 레포: `bmshin94/CopilotKit`
> 목적: 이 레포가 무엇이고, 언제 쓰며, 어떻게 활용·수익화할 수 있는지 정리

## 🔗 관련 링크

| 구분 | URL |
| --- | --- |
| 이 레포 (포크) | https://github.com/bmshin94/CopilotKit |
| 원본 레포 (upstream) | https://github.com/CopilotKit/CopilotKit |
| AG-UI 프로토콜 | https://github.com/ag-ui-protocol/ag-ui |
| 공식 문서 | https://docs.copilotkit.ai |
| Intelligence 대시보드 | https://dashboard.operations.copilotkit.ai |
| npm (react-core) | https://www.npmjs.com/package/@copilotkit/react-core |
| Discord | https://discord.gg/6dffbvGU3D |
| 예제 모음 | https://github.com/CopilotKit/CopilotKit/tree/main/examples |
| Generative UI 교육 레포 | https://github.com/CopilotKit/CopilotKit/tree/main/examples/showcases/generative-ui |
| AG-UI 앱 생성 CLI | `npx create-ag-ui-app my-agent-app` |

---

## 1. 한 줄 정의

> **CopilotKit = LLM 에이전트와 사용자 화면(UI) 사이를 연결하는 표준 계층(SDK)**

단순 챗봇 라이브러리가 아니라, **에이전트가 내 앱을 조작하고 내 앱이 에이전트 상태를 실시간으로 렌더링하는** 애플리케이션(agent-native app)을 만드는 도구.

### 레포 기본 정보

```
version : 1.72.0
license : MIT (거의 전 패키지, Copyright (c) Atai Barkai)
구조     : Nx + pnpm 모노레포 / packages 42개
문서     : showcase/shell-docs/src/content/  (루트 docs/ 는 심볼릭 링크)
릴리즈   : 컨벤셔널 커밋 기반 (changeset 사용 금지 — .changeset/* 만들면 CI 실패)
```

---

## 2. 아키텍처

```
[Frontend]                 [Runtime]                [Agent]
React/Angular/Vue/RN   →  Express/Hono/Next   →   LangGraph/CrewAI/Mastra/BuiltIn
        ↑                        ↑                        ↑
        └─────── AG-UI 프로토콜 (SSE 이벤트 스트림) ───────┘
```

- **AG-UI 프로토콜**: `RUN_STARTED → STEP_STARTED → 메시지/툴콜 이벤트 → STEP_FINISHED → RUN_FINISHED`
  이벤트를 SSE로 스트리밍하고 Zod로 검증. CopilotKit이 직접 만든 오픈 스펙이며 README에 따르면
  Google, LangChain, AWS, Microsoft, Mastra, PydanticAI가 채택.
- **ProxiedAgent**: 프론트엔드에서 원격 에이전트를 로컬 객체처럼 다루는 프록시 (내부적으로 HTTP+SSE).
- **AgentRunner**: 서버에서 스레드(대화 이력·상태) 관리. 기본 `InMemoryAgentRunner`, 영속화는 `SQLiteAgentRunner`.
- **Middleware**: `beforeRequestMiddleware` / `afterRequestMiddleware` 로 인증·로깅 삽입.

### 요청 1회 라이프사이클

1. 프론트에서 `CopilotKitCore` 가 런타임에 에이전트 목록 조회 → `ProxiedAgent` 생성
2. 사용자가 메시지 전송 → `runAgent()`
3. POST `/api/copilotkit` (payload: 메시지 + 등록 툴 목록 + 화면 컨텍스트 + threadId + state)
4. 런타임: 미들웨어 → 에이전트 resolve/clone → `AgentRunner` 실행
5. SSE로 이벤트 스트리밍 (텍스트는 글자 단위 스트리밍, 툴콜 이벤트 포함)
6. **프론트엔드 툴 호출 시 브라우저에서 핸들러 실행 → 결과를 에이전트에 반환 → 계속 진행**
7. Core가 메시지 스토어 갱신 → React/Angular 리렌더

---

## 3. 폴더 전수조사 결과

### `packages/` — 42개

| 그룹 | 패키지 | 역할 |
| --- | --- | --- |
| 두뇌 | `core`, `shared` | `CopilotKitCore` 오케스트레이터 (에이전트/툴/컨텍스트 레지스트리, 이벤트 구독) |
| 프론트엔드 | `react-core`, `react-ui`, `react-textarea`, `react-native`, `angular`, `vue`, `web-components` | 프레임워크별 바인딩. 전부 `core` 래퍼 |
| 런타임 | `runtime`, `runtime-python`, `runtime-go`, `runtime-ruby`, `runtime-dotnet`, `runtime-client-gql` | 서버 계층 — **5개 언어 지원 (PHP는 없음)** |
| 채널 | `channels`, `channels-core`, `channels-ui`, `channels-slack`, `channels-teams`, `channels-discord`, `channels-whatsapp`, `channels-telegram`, `channels-intelligence` | 같은 에이전트를 Slack/Teams/Discord/WhatsApp/Telegram에 배포 |
| Intelligence (상용) | `intelligence-langgraph`, `intelligence-langgraph-python`, `intelligence-mastra`, `intelligence-adk-python`, `intelligence-agent-framework-dotnet`, `intelligence-delivery-core`, `intelligence-delivery-python-core` | 영속 스레드·메모리·자동학습·분석 |
| Generative UI | `a2ui-renderer`, `mcp-apps-renderer` | 에이전트가 JSON으로 UI를 선언/렌더 |
| 개발 도구 | `web-inspector`, `sqlite-runner`, `demo-agents`, `agentcore-runner`, `voice`, `sdk-js` | 디버그 오버레이, 로컬 영속화, 음성, LangGraph 헬퍼 |
| 빌드 설정 | `tailwind-config`, `tsconfig`, `typescript-config` | 공용 설정 |

### `packages/react-core/src/v2/hooks/` — 실제 공개 API

| 훅 | 하는 일 |
| --- | --- |
| `useAgent` | 에이전트 직접 제어 + 상태 읽기/쓰기 (`agent.state`, `agent.setState`) |
| `useFrontendTool` | 브라우저에서 실행되는 툴을 LLM에 등록 |
| `useHumanInTheLoop` | 에이전트 실행을 멈추고 사용자 승인/수정을 받음 ⭐ |
| `useAgentContext` | 현재 화면 상태를 매 실행마다 에이전트에 주입 |
| `useRenderToolCall` / `useRenderTool` / `useDefaultRenderTool` | 툴콜을 React 컴포넌트로 렌더 (Generative UI) |
| `useRenderCustomMessages` / `useRenderActivityMessage` | 커스텀 메시지·액티비티 렌더 |
| `useComponent` | 컴포넌트 등록 |
| `useThreads` | 대화 목록/이력 |
| `useMemories` | 사용자 장기기억 (Intelligence) |
| `useSuggestions` / `useConfigureSuggestions` | 추천 질문 |
| `useInterrupt` | LangGraph interrupt 대응 |
| `useAttachments` | 파일 첨부 |
| `useCapabilities` | 런타임 기능 탐지 |
| `useLearnFromUserAction` / `useLearnFromUserActionInCurrentThread` | 사용자 행동 학습 (Intelligence) |
| `useLearningContainers` / `useLearningContainersInCurrentThread` | 학습 컨테이너 |

UI 컴포넌트: `CopilotChat`, `CopilotPopup`, `CopilotSidebar`, `CopilotPanel`, `CopilotTextarea`

### `examples/`

- **`integrations/` (23종)**: `langgraph-js`, `langgraph-python`, `langgraph-fastapi`, `crewai-crews`, `crewai-flows`,
  `mastra`, `adk`, `adk-angular`, `agno`, `llamaindex`, `pydantic-ai`, `strands-python`, `strands-typescript`,
  `ms-agent-framework-dotnet`, `ms-agent-framework-python`, **`claude-sdk-python`, `claude-sdk-typescript`**,
  `a2a-middleware`, `a2a-a2ui`, `mcp-apps`, `agentcore`, `agent-spec`, `_parity`
- **`showcases/` (35종)**: `research-canvas`, `multi-agent-canvas`, `spreadsheet`, `presentation`, `todo`,
  `deep-agents`, `deep-agents-finance-erp`, `deep-agents-job-search`, `enterprise-brex`, `microsoft-kanban`,
  `strands-crm`, `strands-file-analyzer`, `oracle-agent-memory`, `scene-creator`, `generative-ui`,
  `generative-ui-playground`, `grok-generative-ui`, `a2a-travel`, `a2ui-pdf-analyst`, `adk-dashboard`,
  `arcade-tools`, `chatkit-studio`, `claude-managed-agents`, `daytona-runcode`, `langgraph-js-support-agents`,
  `mcp-apps`, `mcp-demo`, `multi-page`, `open-mcp-client`, `orca`, `pydantic-ai-todos`, `reskinnable-demo`
- 그 외: `canvas`, `shadcn`, `slack`, `teams`, `e2e`, `v1`, `v2`

### `skills/` — Claude Code Agent Skills 8개

| 스킬 | 용도 |
| --- | --- |
| `copilotkit` | 모든 CopilotKit 질문의 진입점. "기억으로 답하지 말고 문서/소스를 검색하라"가 핵심 원칙 |
| `copilotkit-cli` | CLI 사용법. 디버깅 전에 `verify` 먼저 |
| `copilotkit-channels` | Slack/Teams 채널의 **코드** 절반 |
| `channels-setup` | 채널 처음 셋업 (워크플로는 CLI가 출력) |
| `setup-slack-channel` | Slack 앱/토큰 생성 등 **프로바이더** 절반 (5개 시스템 정렬) |
| `inspector-workbench` | Inspector UI 작업 (스탠드얼론 워크벤치 실행 + 스크린샷 필수) |
| `inspector-docs` | Inspector 페인 ↔ 문서 동기화 |
| `intelligence-docs` | Intelligence 랜딩 페이지 동기화 |

### `.claude-plugin/` — Claude Code 플러그인 정의

```
marketplace.json : copilotkit-plugins (owner: CopilotKit, v1.72.0)
plugin.json      : name "copilotkit", category "ai-frameworks", MIT
```

### `.mcp.json` — MCP 서버 2개

```json
{
  "nx-mcp":          { "type": "stdio", "command": "npx", "args": ["nx", "mcp"] },
  "copilotkit-docs": { "type": "http",  "url": "https://mcp.copilotkit.ai/mcp" }
}
```

`copilotkit-docs` 는 4개 코퍼스 대상 검색 툴 4개 + 탐색 툴 2개를 제공.
(분석 세션에서는 프록시 403으로 연결 실패 — 설정 부재가 아니라 네트워크 차단)

### 기타 폴더

| 폴더 | 내용 |
| --- | --- |
| `showcase/` | 문서 사이트(`shell-docs`), `shell-dashboard`, `shell-dojo`, Playwright E2E, docker-compose 6종, `eval-tiers.json`, `eval-webhook`, `pocketbase`, `aimock` |
| `sdk-python/` | LangGraph용 Python SDK (poetry/uv) |
| `codemods/` | 버전 마이그레이션 자동 변환 (`migrate-attachments.ts`) |
| `tools/` | `runtime-conformance`, `learned-skill-conformance` — 다국어 런타임 스펙 준수 검증 |
| `community/` | 2024/2025 커뮤니티 데모 + 기여 가이드 |
| `.claude/docs/` | `architecture.md`, `hooks.md`, `workflow.md`, `git.md`, `documentation.md`, `aeo-synthetics.md` |
| `dev-docs/`, `prds/` | 내부 개발 문서·PRD |

---

## 4. 3대 핵심 기능 (코드)

### ① 프론트엔드 툴 — AI가 내 앱을 조작

```tsx
useFrontendTool({
  name: "setBackgroundColor",
  description: "페이지 배경색을 바꾼다",
  parameters: z.object({ color: z.string() }),
  handler: async ({ color }) => {
    document.body.style.background = color; // 브라우저에서 실행
    return `배경을 ${color}로 바꿨어요`;
  },
});
```

### ② Generative UI — 답변이 텍스트가 아니라 컴포넌트

```tsx
useRenderToolCall({
  name: "showFlights",
  render: ({ args, status }) =>
    status === "inProgress" ? <Skeleton /> : <FlightCards flights={args.flights} />,
});
```

Generative UI 3분류: **Static (AG-UI)** / **Declarative (A2UI)** / **Open-Ended (MCP Apps & Open JSON)**

### ③ Human-in-the-Loop — AI가 멈춰서 승인 요청

```tsx
useHumanInTheLoop({
  name: "confirmRefund",
  parameters: z.object({ amount: z.number() }),
  render: ({ args, respond }) => (
    <div>
      {args.amount}원 환불할까요?
      <button onClick={() => respond("approved")}>승인</button>
      <button onClick={() => respond("rejected")}>거절</button>
    </div>
  ),
});
```

### ④ Shared State — AI와 UI가 같은 상태를 봄

```tsx
const { agent } = useAgent({ agentId: "research_agent" });
<h1>{agent.state.city}</h1>
<button onClick={() => agent.setState({ city: "NYC" })}>Set City</button>
```

---

## 5. 언제 쓰는가

| 상황 | 적합도 |
| --- | --- |
| 단순 Q&A 챗봇 | ❌ 과함 (그냥 API 호출) |
| 앱을 조작하는 AI 사이드바 | ✅✅ 최적 |
| 에이전트 작업 상태 실시간 시각화 | ✅✅ 최적 |
| 승인 워크플로 (AI 초안 → 사람 승인 → 실행) | ✅✅ 최적 (HITL) |
| LangGraph/CrewAI 이미 있고 UI만 필요 | ✅✅ 존재 이유 |
| 웹 + 모바일 + Slack에 같은 에이전트 배포 | ✅✅ Channels |

### 도움 되는 지점

1. "에이전트 → UI" 배관 공사(SSE 파싱, 툴콜 라운드트립, 스트리밍 상태관리, 스레드 영속화)를 직접 안 해도 됨
2. 프레임워크 락인 없음 — 에이전트를 바꿔도 프론트 코드 유지
3. MIT 라이선스 → 상업적 이용/재배포 자유 (Intelligence만 상용)
4. 레퍼런스 코드 58개(통합 23 + 쇼케이스 35) 무료
5. Claude Code 스킬·MCP 내장 → 코딩 에이전트가 최신 문법으로 코드 작성

---

## 6. 설치 및 사용법

### CLI (권장)

```bash
npx copilotkit@latest create             # 새 프로젝트 스캐폴딩 (기존 앱 미변경)
npx --yes copilotkit@latest onboard start # 기존 앱에 붙이기 (코딩 에이전트 가이드)
npx copilotkit@latest verify --json      # 문제 생기면 손 디버깅 전에 이걸 먼저
npx copilotkit@latest skills install     # Claude Code용 스킬 설치
npx copilotkit@latest login              # Intelligence 로그인
npx copilotkit@latest project select     # .env에 CPK_INTELLIGENCE_API_KEY 기록
```

### 수동 설치 (Next.js)

```bash
npx create-next-app@latest my-copilot-app && cd my-copilot-app
npm install @copilotkit/react-core @copilotkit/runtime
# .env
OPENAI_API_KEY=sk-...
```

```ts
// app/api/copilotkit/[[...slug]]/route.ts
import { CopilotRuntime, createCopilotRuntimeHandler, BuiltInAgent }
  from "@copilotkit/runtime/v2";

const runtime = new CopilotRuntime({
  agents: { default: new BuiltInAgent({ model: "openai:gpt-5.4-mini" }) },
});
const handler = createCopilotRuntimeHandler({ runtime, basePath: "/api/copilotkit" });
export const GET = handler;
export const POST = handler;
export const PATCH = handler;
export const DELETE = handler;
```

```tsx
// app/providers.tsx
"use client";
import { CopilotKitProvider } from "@copilotkit/react-core/v2";

export function Providers({ children }: { children: React.ReactNode }) {
  return <CopilotKitProvider runtimeUrl="/api/copilotkit">{children}</CopilotKitProvider>;
}
```

> ⚠️ 이미 LangGraph/CrewAI/Mastra 에이전트가 있으면 `BuiltInAgent`를 쓰면 안 된다.
> 그것은 CopilotKit 자체 에이전트로, 기존 에이전트를 **대체**한다.
> 이 경우 프론트 단계만 가져오고 런타임 배선은 해당 프레임워크 quickstart를 따를 것.

### 이 모노레포 자체 개발

```bash
pnpm install
npx nx run-many -t build
npx nx affected -t test lint   # 반드시 nx 경유 (CLAUDE.md 규칙)
```

레포 규칙: `.changeset/*` 생성 금지 / 문서는 `showcase/shell-docs/src/content/` 에만 작성 /
워크트리에서 작업 / 버전·CHANGELOG는 릴리즈 툴링이 관리

---

## 7. 플러그인? 스킬? MCP? → 전부 포함

| 층 | 정체 | 근거 |
| --- | --- | --- |
| 1. 본체 | **npm 라이브러리/SDK** | `packages/*`, `@copilotkit/*` 42개 |
| 2. 프로토콜 | **AG-UI** 오픈 표준 | `@ag-ui/core`, SSE + Zod |
| 3. Claude Code 플러그인 | ✅ | `.claude-plugin/plugin.json` (`copilotkit`) |
| 4. Agent Skills | ✅ 8개 | `skills/*/SKILL.md` |
| 5. MCP 서버 | ✅ 1개 제공 | `copilotkit-docs` (https://mcp.copilotkit.ai/mcp) |

정리: **SDK가 본체고, 그 SDK를 AI가 잘 쓰도록 돕는 플러그인(스킬 + MCP)을 함께 배포한 것.**

반대 방향도 지원: CopilotKit 앱이 MCP **클라이언트**가 될 수 있음
(`examples/showcases/open-mcp-client`, `mcp-apps`, 문서 `webmcp.mdx`).

---

## 8. API 토큰이 필요한가

| 키 | 필요 시점 | 필수 | 비용 |
| --- | --- | --- | --- |
| `OPENAI_API_KEY` / `ANTHROPIC_API_KEY` / Google | 에이전트가 모델 호출 | ✅ 사실상 필수 | 모델 제공사에 직접 지불 |
| `CPK_INTELLIGENCE_API_KEY` | Intelligence(영속 스레드·메모리·분석) | ❌ 선택 | CopilotKit 유료 플랜 |
| Slack/Teams 앱 토큰 | Channels 사용 시 | ❌ 선택 | 플랫폼 무료 |

- **CopilotKit 자체 라이선스 키는 없음** (MIT)
- 로컬 모델(Ollama/vLLM)을 쓰면 외부 키 0개로도 동작
- 보안: Intelligence 키는 반드시 서버에만 보관. `identifyUser`는 인증 게이트가 아니므로
  핸들러의 `onRequest` 훅으로 인증을 강제하고, `threads/events`·`threads/state`·`agent/stop`에
  스레드 소유권 검사를 반드시 추가 (문서 `auth#thread-authorization`)

---

## 9. OSS vs Intelligence

| 기능 | OSS (MIT, 무료) | Intelligence (상용) |
| --- | --- | --- |
| 챗 UI / 툴 / Generative UI / HITL | ✅ | ✅ |
| 대화 저장 | 인메모리 or `sqlite-runner` 직접 구성 | ✅ 관리형 영속 스레드 |
| 사용자 장기기억 | ❌ | ✅ 시맨틱 메모리 |
| 자동 학습 (대화 → Insight → 재사용 Skill) | ❌ | ✅ |
| 제품 분석 대시보드 | ❌ | ✅ |
| Slack/Teams 관리형 채널 | 어댑터만 | ✅ 서명 인그레스·크레덴셜 관리 |
| 셀프호스팅 | — | ✅ (자체 K8s/VPC) |

**개발·MVP는 100% 무료로 가능. 프로덕션 운영 편의성이 유료.**

---

## 10. 왜 GitHub에서 유명한가

### 레포에서 확인된 근거

- Trendshift 뱃지 (repository #5730) — 트렌딩 상위 등재
- Product Hunt Top Post 뱃지 (post #428778)
- npm 버전 뱃지, Discord 커뮤니티 (서버 ID 1122926057641742418)
- (정확한 현재 스타 수는 GitHub에서 직접 확인 권장)

### 유명해진 이유 5가지

1. **빈 자리 선점** — LangChain은 "에이전트 두뇌", Vercel AI SDK는 "스트리밍 챗"을 잡았으나
   "에이전트가 앱 UI를 조작하는 층"은 비어 있었음
2. **프로토콜을 만들어 생태계 중심이 됨** — AG-UI를 오픈 스펙으로 공개하고 Google/LangChain/AWS/
   Microsoft/Mastra/PydanticAI가 채택 → 라이브러리가 아니라 **표준**이 됨 (결정타)
3. **데모의 시각적 임팩트** — research-canvas, multi-agent-canvas, spreadsheet, presentation 등
   스크린샷/GIF 한 장으로 설명이 끝나는 데모를 대량 배치 (쇼케이스 35 + 통합 23)
4. **"Generative UI" 용어와 분류(Static/Declarative/Open-Ended)를 선점**해 담론 주도
5. **MIT + 완전 셀프호스팅** → 도입 부담이 낮아 스타·채택이 빠름

---

## 11. 로컬 에이전트 구축에 도움 되는가 → 매우 그렇다

### 역할 구분

| CopilotKit이 해주는 것 | 해주지 않는 것 |
| --- | --- |
| 챗 UI, 스트리밍, 마크다운 | 에이전트 추론 로직 (LangGraph/CrewAI 담당) |
| 툴콜 라운드트립 (브라우저 ↔ 서버) | 모델 추론 (Ollama/vLLM 담당) |
| 실시간 상태 동기화 | 벡터 DB / RAG |
| HITL 일시정지·재개 | 파인튜닝 |
| 스레드 영속화 (`sqlite-runner`) | |
| 디버그 오버레이 (`web-inspector`) | |

### 추천 완전 로컬 스택

```
[UI]      Next.js + @copilotkit/react-core
             ↓ AG-UI / SSE
[Runtime] @copilotkit/runtime  (+ sqlite-runner → 로컬 영속화)
             ↓
[Agent]   LangGraph Python (examples/integrations/langgraph-fastapi 참고)
             ↓
[Model]   Ollama / vLLM  (외부 키 0개, 데이터 외부 전송 0)
```

- `sqlite-runner`로 클라우드 없이 대화 영속화
- `web-inspector` 로컬호스트 오버레이로 이벤트/상태 시각 디버깅
- `docker-compose.{dev,local,real-claude,record,replay}.yml` 등 컴포즈 파일 6종 제공
- **`examples/integrations/claude-sdk-python`, `claude-sdk-typescript`** — Claude Agent SDK 연동 예제 내장
- 주의: 로컬 소형 모델은 툴콜 정확도가 낮음 → 툴 수를 줄이고 description을 명확히

---

## 12. React / PHP로 만들 수 있는가

### React → 완전 지원 (1급 시민)

```
packages/react-core     : CopilotKitProvider + 훅 20여 개
packages/react-ui       : CopilotChat, CopilotPopup, CopilotSidebar, CopilotPanel
packages/react-textarea : CopilotTextarea (AI 자동완성)
packages/react-native   : 모바일
```

지원: Next.js(App/Pages), React Router, Remix, TanStack Start, Vite SPA, React Native.
README 지원 표에서 React/Next.js만 **GA**.

그 외 프론트엔드: Angular ✅, Vue ✅, Web Components(바닐라) ✅

### PHP → 공식 패키지 없음. 다만 구현 가능

공식 런타임은 TS/JS · Python · Go · Ruby · .NET 5종. **PHP는 없다.**
그러나 런타임의 본질은 **HTTP POST + SSE 응답**이고 AG-UI는 공개 스펙이므로 3가지 경로가 있다.

**경로 1 — 하이브리드 (가장 현실적, 권장)**

```
React 프론트 (CopilotKit)
      ↓ /api/copilotkit
Node 런타임 (@copilotkit/runtime)   ← 얇게, 배선만
      ↓ REST 호출
기존 PHP 백엔드 (Laravel 등)         ← 비즈니스 로직·DB 그대로 유지
```

툴 핸들러가 PHP API를 호출하게만 하면 기존 PHP 자산을 100% 재사용 가능.

**경로 2 — PHP로 AG-UI 런타임 직접 구현**

- Laravel/Symfony에서 `text/event-stream` 응답을 만들고
  `RUN_STARTED`, `TEXT_MESSAGE_*`, `TOOL_CALL_*`, `RUN_FINISHED` 이벤트를 순서대로 방출
- 검증: 레포의 `tools/runtime-conformance` 로 스펙 준수 테스트 가능
- PHP는 long-lived 스트리밍이 약하므로 **Laravel Octane / Swoole / RoadRunner** 권장
- 참조 구현: `packages/runtime-ruby`, `packages/runtime-go` 코드를 보고 이식
- 난이도: 중상

**경로 3 — PHP를 에이전트 층으로만** — 런타임은 Node, 에이전트 로직만 PHP HTTP 엔드포인트. 가장 쉽다.

---

## 13. 수익화 아이디어

> 대전제
> - ❌ CopilotKit 래퍼를 파는 것은 돈이 안 된다 (본체가 MIT 무료)
> - ✅ CopilotKit으로 **특정 업종의 반복 노동을 제거**하고 그 업종에 파는 것이 돈이 된다
>
> 차별 무기: **① Generative UI ② Shared State ③ HITL(승인)**

### 트랙 A — 버티컬 SaaS (추천 ⭐⭐⭐)

| # | 아이디어 | 왜 CopilotKit인가 | 가격 | 난이도 / 수익성 |
| --- | --- | --- | --- | --- |
| A-1 | **세무/회계 장부 정리 코파일럿** — 엑셀 거래내역의 계정과목 자동 분류 | `spreadsheet` 쇼케이스 + HITL로 애매한 건("접대비 vs 복리후생비") 승인 요청. 세무는 오류가 곧 리스크라 승인 UI가 필수 | 사무소당 월 30~100만원 | ⭐⭐⭐ / 💰💰💰💰 |
| A-2 | **부동산 매물 브리핑 에이전트** — 조건 듣고 매물 카드·지도·비교표 생성 | `research-canvas` + `multi-agent-canvas` | 월 10~30만원, 프랜차이즈 단위 계약 | ⭐⭐ / 💰💰💰 |
| A-3 | **병원 예약·문진 프론트데스크 에이전트** — 대화로 문진표 수집 → EMR 전송 | HITL로 간호사 최종 확인(의료 필수) + Channels 확장 | 병원당 월 20~50만원 | ⭐⭐⭐ / 💰💰💰💰 |
| A-4 | **이커머스 셀러 운영 코파일럿** — 부진 상품 분석 → 상세페이지 재작성 → 승인 후 API 반영 | `strands-crm`, `enterprise-brex` | 월 5~20만원 × 대량 | ⭐⭐ / 💰💰💰 |

### 트랙 B — 개발 서비스 (현금흐름 빠름 ⭐⭐⭐)

| # | 아이디어 | 내용 | 가격 | 난이도 / 수익성 |
| --- | --- | --- | --- | --- |
| B-1 | **"AI 코파일럿 심어드립니다" 구축 에이전시** | 기존 SaaS·사내 어드민에 2~4주 만에 AI 사이드바 이식. 58개 예제 덕에 구축 속도가 압도적 | 프로젝트 1,500~5,000만원 + 유지보수 월 200~500만원 | ⭐⭐ / 💰💰💰💰💰 |
| B-2 | **PHP/Laravel 특화 포지셔닝 (틈새 독점)** | 공식 PHP 런타임이 없음 = 빈 시장. `copilotkit-php`를 MIT로 공개하고 `tools/runtime-conformance`로 스펙 준수 인증 → 권위 확보 후 구축·컨설팅으로 수익화(오픈코어) | 패키지 무료 + 구축 수익 | ⭐⭐⭐⭐ / 💰💰💰 + 브랜드 |
| B-3 | **AG-UI 커넥터 전문점** | 국산 SaaS(네이버웍스, 잔디, 카카오워크 등)용 AG-UI 어댑터 유료 제작. `channels-*` 구조를 템플릿화해 2번째부터 원가 급감 | 건당 계약 | ⭐⭐⭐ / 💰💰💰 |

### 트랙 C — 제품/템플릿 판매 (수동소득 ⭐⭐)

| # | 아이디어 | 내용 | 가격 | 난이도 / 수익성 |
| --- | --- | --- | --- | --- |
| C-1 | **프리미엄 스타터킷** | CopilotKit + Next.js + 인증 + 결제 + 팀 + 스레드 영속화 올인원 보일러플레이트. Next.js 보일러플레이트 시장은 검증됐고 AI 에이전트 버전은 아직 비어 있음 | $99~$299 (팀 $499) | ⭐⭐ / 💰💰💰 |
| C-2 | **유료 Generative UI 컴포넌트 키트** | 차트·칸반·타임라인·승인카드·진행트래커. `a2ui-renderer`, `mcp-apps-renderer` 기반, shadcn 스타일 코드 판매 | $149~ | ⭐⭐ / 💰💰 |
| C-3 | **교육 상품** | 한국어 CopilotKit/AG-UI 강의가 거의 없음 = 선점 가능. 인프런/유데미 + 유튜브 + 멘토링 | 강의 ₩99,000 | ⭐ / 💰💰 + 브랜딩 |

### 트랙 D — 인프라/툴링 (고난도 고수익 ⭐)

| # | 아이디어 | 내용 | 가격 |
| --- | --- | --- | --- |
| D-1 | 관리형 AG-UI 런타임 호스팅 | Intelligence 셀프호스팅 대행 + 한국 리전(데이터 국내 보관) | 월 50~300만원 |
| D-2 | 에이전트 옵저버빌리티 | `web-inspector` 확장 → 프로덕션 툴콜 실패율/비용/레이턴시 대시보드 | 월 $99~$999 |
| D-3 | 에이전트 평가(Eval) 서비스 | `showcase/eval-tiers.json`, `eval-webhook` 구조 참고 → 회귀 테스트 SaaS | 월 $199~ |

### 트랙 E — 킬러 조합 (방어력 최고 ⭐⭐⭐⭐)

**E-1. "승인 워크플로 + 감사로그" 엔터프라이즈 에이전트**

```
기업이 AI를 도입하지 못하는 진짜 이유는 성능이 아니라 "책임 소재"다.
"AI가 잘못 송금하면 누가 책임지나?" → 도입 중단.

useHumanInTheLoop = 이 문제의 정답:
"AI가 제안 → 사람이 승인 → 실행 + 누가 언제 승인했는지 전부 기록"
```

- 타깃: 재무(송금/지출결의), 인사(채용/급여), 법무(계약검토), 물류(발주)
- 차별점: 경쟁 챗봇은 "말만" 하지만 이 제품은 **승인받고 실제로 실행**
- 가격: 엔터프라이즈 연 3,000만원~2억
- 참고 코드: `enterprise-brex`, `deep-agents-finance-erp`, `microsoft-kanban`
- 난이도 ⭐⭐⭐⭐ / 수익성 💰💰💰💰💰

### 실행 로드맵

| 기간 | 할 일 | 목표 |
| --- | --- | --- |
| 1~2주 | `npx copilotkit@latest create` → `spreadsheet`, `research-canvas` 실행 → HITL 데모 직접 제작 | 감각 익히기 |
| 3~6주 | **트랙 B-1** 착수: 지인/기존 고객사 1곳에 AI 사이드바 이식 | 첫 레퍼런스 + 현금 |
| 2~3개월 | 그 프로젝트를 **템플릿화**(트랙 C-1) + 블로그·영상 기록 | 자산화 + 인바운드 |
| 4~6개월 | 반복 수요 업종 하나를 **버티컬 SaaS**로(트랙 A) | MRR 전환 |
| 병행 | PHP 레거시 고객을 만나면 **B-2 `copilotkit-php`** 오픈소스 공개 | 브랜드 독점 |

### 리스크 체크

- **LLM 원가**: 토큰비가 마진을 잠식. 가격에 반영하거나 BYOK(고객 키) 모델로
- **버전 이동 속도**: 1.72.0, API가 빠르게 변함 → `codemods/` + 스킬/MCP 문서 검색으로 대응
  (스킬이 "기억으로 답하지 말라"고 명시한 이유)
- **Intelligence 락인**: 피하려면 `sqlite-runner` + 자체 영속화로 설계
- **MIT 준수**: 재배포 시 저작권 고지 포함

---

## 14. 요약 카드

| 질문 | 답 |
| --- | --- |
| 이게 뭐야? | 에이전트 ↔ UI 연결 계층 SDK. AG-UI 프로토콜의 레퍼런스 구현 |
| 언제 써? | 에이전트가 앱을 조작하거나, 작업 상태를 시각화하거나, 사람 승인이 필요할 때 |
| 플러그인/스킬/MCP? | SDK 본체 + Claude Code 플러그인 + 스킬 8개 + MCP 서버 1개, 전부 포함 |
| 토큰 필요? | LLM 키는 사실상 필수. CopilotKit 자체 키는 없음. Intelligence만 선택적 유료 |
| 왜 유명해? | 빈 자리 선점 + AG-UI 표준화 + 데모 임팩트 + 용어 선점 + MIT |
| 로컬 에이전트? | 매우 유용. `sqlite-runner` + Ollama로 완전 로컬 가능 |
| React? | 완전 지원 (GA). Angular/Vue/RN/Web Components도 지원 |
| PHP? | 공식 없음. 하이브리드(Node 런타임 + PHP 백엔드) 권장, 직접 구현도 가능 |
| 수익화? | 래퍼 판매 ❌ / 버티컬 SaaS·구축 에이전시·스타터킷·승인워크플로 엔터프라이즈 ✅ |
