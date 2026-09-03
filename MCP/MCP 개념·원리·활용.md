---
tags: [mcp, ai-agent, 프로토콜, claude, 작동원리]
생성일: 2026-07-03
상태: 완료
---

> [!info] 관련 노트
> 🏠 [[🏠 홈]] · 📂 주제: MCP
> 🔗 함께 보기: [[6. 확장과 종합 — MCP·플러그인·전체 그림]] (Claude Code가 MCP를 코드로 어떻게 붙이는지)

# MCP 개념·원리·활용 — Model Context Protocol

> 맥락: AI 에이전트가 외부 도구·데이터에 연결되는 표준. Claude Code 작동원리 s19에서 살짝 봤던 걸 제대로 정리.
> 목표: "MCP가 뭐고, 어떻게 동작하고, 어디에 쓰는가"를 손에 쥐기.

---

## 0. 한 줄 요약
**MCP(Model Context Protocol)는 AI 앱을 외부 도구·데이터·워크플로우에 연결하는 "AI 세계의 USB-C" — 오픈 표준이다.**
- 2024년 11월 Anthropic이 오픈소스로 공개 → 18개월 만에 사실상 업계 표준이 됨.
- 2026년 3월 기준 월 9,700만+ SDK 다운로드, 서버 5,000개+, Anthropic·OpenAI·Google·Microsoft·AWS 전부 지원.

`★ 왜 USB-C 비유인가 ─────────────────────`
USB-C 하나면 노트북·폰·모니터·충전기가 다 꽂힌다. MCP도 마찬가지.
예전엔 "Claude에 GitHub 붙이기", "Cursor에 Slack 붙이기"를 **매번 따로** 만들어야 했다(M개 앱 × N개 도구 = M×N개 연결).
MCP라는 **공용 규격**이 생기니, 도구 제작자는 서버 1개만 만들면 **모든 AI 앱이** 그걸 쓴다(M+N).
**"한 번 만들면 어디서나 연결(build once, integrate everywhere)".**
`──────────────────────────────────────`

---

## 1. MCP가 뭔지 (개념)

MCP는 **"AI 앱 ↔ 외부 시스템"을 잇는 표준 규격**이다. 세 가지를 연결할 수 있다:
- **데이터(Data)**: 로컬 파일, 데이터베이스, API 응답 등 → AI가 **읽을** 정보
- **도구(Tools)**: 검색, 계산기, 배포 시스템 등 → AI가 **실행할** 행동
- **워크플로우(Prompts)**: 특정 작업용 프롬프트 템플릿

> ⚠️ 중요: MCP는 **"연결 규격"만** 정한다. AI가 그 정보를 어떻게 쓸지(어느 LLM, 어떻게 판단)는 관여 안 함. 순수하게 "컨텍스트를 주고받는 통로"만 표준화.

---

## 2. 아키텍처 — 3명의 등장인물

| 등장인물 | 정체 | 예시 |
|---|---|---|
| **Host (호스트)** | 여러 클라이언트를 관리하는 **AI 앱** | Claude Desktop, Claude Code, Cursor, VS Code |
| **Client (클라이언트)** | 서버 1개와 **전용 연결**을 유지하는 부품 | 호스트가 서버마다 하나씩 만듦 |
| **Server (서버)** | 도구·데이터를 **제공하는 프로그램** | GitHub 서버, 파일시스템 서버, Sentry 서버 |

**핵심 관계**: 호스트(AI앱) 1명이 서버마다 클라이언트를 **하나씩** 만든다.
- VS Code(호스트)가 Sentry 서버에 연결 → 클라이언트1 생성
- 같은 VS Code가 파일시스템 서버에도 연결 → 클라이언트2 생성
- → 클라이언트-서버는 항상 **1:1 전용 연결**

> 비유: **호스트=여러 나라와 거래하는 회사**, **클라이언트=각 나라 담당 주재원**, **서버=현지 공급업체**. 회사는 공급업체마다 주재원을 한 명씩 붙여 전담시킨다.

---

## 3. 2개의 층 (Layer) — 이게 동작 원리의 핵심

MCP는 양파처럼 두 겹이다:

### 🧅 안쪽 — 데이터 층 (Data Layer)
**JSON-RPC 2.0**이라는 표준 메시지 형식으로 대화. "무슨 말을 주고받나"를 정의.
- 생명주기 관리(연결 시작·능력 협상·종료)
- 서버 기능(도구·리소스·프롬프트)
- 클라이언트 기능(샘플링·질문·로깅)
- 알림(실시간 업데이트)

### 🧅 바깥쪽 — 전송 층 (Transport Layer)
"그 메시지를 어떤 통로로 나르나"를 정의. **통로가 2가지:**

| | **stdio** (로컬) | **Streamable HTTP** (원격) |
|---|---|---|
| 방식 | AI앱이 서버를 **자식 프로세스**로 띄워 stdin/stdout로 대화 | 서버가 웹서비스, HTTP POST + SSE로 대화 |
| 속도 | 약 1ms (제일 빠름) | 10~100ms (네트워크 의존) |
| 위치 | 내 컴퓨터에서만 | 원격 배포, 여러 클라이언트 동시 |
| 인증 | 불필요 | [[OAuth 2.0과 OIDC — 소셜 로그인의 원리\|OAuth]]·API키·Bearer 토큰 (OAuth 권장) |
| 예 | 로컬 파일시스템 서버 | Sentry 같은 SaaS 제공 서버 |

> [!note] 층을 나눈 이유
> 데이터 층(JSON-RPC 메시지)은 전송 방식과 **무관하게 똑같다.** 그래서 로컬이든 원격이든 같은 메시지 형식을 쓴다. "무엇을 말하나(데이터)"와 "어떻게 나르나(전송)"를 분리한 깔끔한 설계.

---

## 4. 동작 흐름 (실제 대화 순서)

서버에 연결해서 도구를 쓰기까지 4단계:

### ① 초기화 — 능력 협상 (handshake)
클라이언트가 `initialize` 요청 → 서로 "난 이런 거 할 수 있어"를 선언(capability negotiation).
```json
// 클라이언트 → 서버
{"jsonrpc":"2.0", "id":1, "method":"initialize",
 "params":{"protocolVersion":"2025-06-18", "capabilities":{"elicitation":{}}, ...}}
// 서버 → 클라이언트 (응답)
{"result":{"capabilities":{"tools":{"listChanged":true}, "resources":{}}, ...}}
```
- 프로토콜 버전 맞추고, 서로 지원 기능을 확인. 안 맞으면 연결 종료.
- 끝나면 클라이언트가 `notifications/initialized` 로 "준비됐어" 신호.

### ② 도구 발견 (discovery)
```json
{"jsonrpc":"2.0", "id":2, "method":"tools/list"}   // "쓸 수 있는 도구 목록 줘"
```
서버가 도구 배열 반환 — 각 도구는 `name`·`description`·`inputSchema`(JSON Schema)를 가짐. AI앱은 이 목록을 모아 LLM에게 "네가 쓸 수 있는 도구들"로 등록.

### ③ 도구 실행 (execution)
```json
{"jsonrpc":"2.0", "id":3, "method":"tools/call",
 "params":{"name":"weather_current", "arguments":{"location":"Seoul", "units":"metric"}}}
```
서버가 실행하고 결과를 `content` 배열(텍스트·이미지 등)로 반환 → LLM 대화에 삽입.

### ④ 실시간 알림 (notifications)
서버 도구가 바뀌면 `notifications/tools/list_changed` 를 보냄(응답 불필요, `id` 없음) → 클라이언트가 `tools/list`를 다시 불러 최신화. **폴링 없이** 변경을 알 수 있음.

> 비유: **식당에서 밥 먹기**. ①입장하며 "영어 되세요?" 확인(초기화) → ②메뉴판 받기(tools/list) → ③주문·서빙(tools/call) → ④"오늘 특선 추가됐어요" 알림(notifications).

---

## 5. Primitive(기본 요소) — MCP가 주고받는 것들

### 서버가 제공하는 3가지 (제일 중요)
| Primitive | 뭔가 | 성격 | 예시 |
|---|---|---|---|
| **Tools (도구)** | AI가 **호출해 실행**하는 함수 | 행동 | DB 쿼리, 파일 쓰기, API 호출 |
| **Resources (리소스)** | AI가 **읽는** 데이터 소스 | 정보 | 파일 내용, DB 레코드, 스키마 |
| **Prompts (프롬프트)** | 재사용 **템플릿** | 지침 | 시스템 프롬프트, few-shot 예제 |

> 각 primitive는 발견(`*/list`)·조회(`*/get`)·실행(`tools/call`) 메서드를 가짐.

### 클라이언트가 제공하는 것들 (서버가 역으로 요청)
- **Sampling (샘플링)**: 서버가 클라이언트에게 "LLM 한 번 돌려줘"(`sampling/createMessage`). 서버가 자체 LLM SDK 없이도 **모델 독립적**으로 지능을 빌림.
- **Elicitation (추가 정보 요청)**: 서버가 사용자에게 "이 값 알려줘 / 이 작업 승인?"(`elicitation/create`).
- **Logging (로깅)**: 서버가 디버깅용 로그를 클라이언트에 전송.

`★ 인사이트: 왜 클라이언트도 기능을 제공하나 ──────`
보통은 서버가 능력을 주지만, MCP는 **역방향도 허용**한다. 서버가 "번역이 필요한데 내 안엔 LLM이 없어" → **샘플링**으로 호스트의 LLM을 빌려 쓴다.
덕분에 서버 개발자는 무거운 LLM SDK를 안 넣고도 똑똑한 서버를 만든다.
`──────────────────────────────────────`

---

## 6. 어떻게 활용하나 (실전)

### 붙이는 법 (사용자 입장)
대부분의 AI 앱은 **설정 파일에 서버 한 줄** 추가하면 끝. 예를 들어 Claude Desktop이면 설정에 서버 명령을 등록:
```json
{ "mcpServers": {
    "filesystem": { "command": "npx", "args": ["-y", "@modelcontextprotocol/server-filesystem", "/내폴더"] }
}}
```
→ 앱 재시작하면 그 서버의 도구가 AI에게 자동으로 생김. (Claude Code에선 `claude mcp add ...` 같은 명령)

### 주로 이렇게 쓰인다 (대표 활용)
- **AI 코딩 도구**: Cursor·VS Code·Claude Code·Replit·Sourcegraph가 MCP로 **프로젝트 컨텍스트·외부 서비스**에 실시간 접근.
- **개인 비서**: Google Calendar·Notion 서버를 붙여 일정 관리·메모 정리.
- **디자인→코드**: Figma 서버를 붙여 디자인에서 웹앱 생성.
- **기업 챗봇**: 여러 사내 DB에 연결해 채팅으로 데이터 분석.
- **물리 세계**: Blender로 3D 디자인 → 3D 프린터 출력까지.

### 많이 쓰는 서버들
GitHub · Slack · Google Drive · PostgreSQL · Notion · Jira · Salesforce · 파일시스템 · Sentry 등 대부분의 SaaS가 공식 MCP 서버 제공.

---

## 7. 교육용 s19와 실제 MCP 차이 (복습 연결)
[[6. 확장과 종합 — MCP·플러그인·전체 그림|작동원리 s19]]는 MCP를 **Python 함수로 흉내낸 목업**이었음. 실제와 다른 점:
- 실제 MCP는 **서브프로세스 + stdin/stdout JSON-RPC**(또는 HTTP). s19는 함수 호출로 대체.
- 실제는 **OAuth 인증, 양방향 알림, 재연결** 로직 있음. s19엔 없음.
- 하지만 **핵심(도구 목록 받기 `tools/list` + 실행 `tools/call`, 이름 네임스페이싱 `mcp__server__tool`)** 은 s19가 정확히 잡았음.

---

## 8. 용어집
- **MCP (Model Context Protocol)**: AI 앱을 외부 도구·데이터에 잇는 오픈 표준.
- **Host / Client / Server**: AI 앱 / 서버당 1개 연결 부품 / 도구·데이터 제공 프로그램.
- **JSON-RPC 2.0**: 요청·응답·알림을 주고받는 표준 메시지 형식(데이터 층).
- **Transport(전송)**: stdio(로컬 프로세스) / Streamable HTTP(원격 웹).
- **Primitive**: MCP가 주고받는 기본 요소. 서버=도구·리소스·프롬프트, 클라이언트=샘플링·추가요청·로깅.
- **Capability negotiation(능력 협상)**: 초기화 때 서로 지원 기능을 선언·확인하는 handshake.
- **Notification(알림)**: 응답 없이 보내는 실시간 변경 통지(`id` 없음).
- **Sampling(샘플링)**: 서버가 클라이언트의 LLM을 빌려 쓰는 것.
- **SSE (Server-Sent Events)**: 서버가 클라이언트로 실시간 스트리밍하는 HTTP 방식.

---

## 9. 다음에 빠른 재현 체크리스트
1. MCP = "AI의 USB-C". 도구 제작자는 서버 1개, 모든 AI 앱이 사용(M+N).
2. 등장인물 3: **Host**(AI앱) → **Client**(서버당 1개) → **Server**(도구 제공).
3. 2층: **데이터 층**(JSON-RPC 무엇을) + **전송 층**(stdio/HTTP 어떻게).
4. 흐름: 초기화(능력협상) → `tools/list`(발견) → `tools/call`(실행) → 알림(변경).
5. 서버 primitive 3개: **도구(행동)·리소스(정보)·프롬프트(지침)**.
6. 붙이기 = 설정에 서버 한 줄 → 앱 재시작 → 도구 자동 등장.
7. 주 활용: 코딩도구(Cursor/CC)·개인비서(Notion/Calendar)·기업DB 챗봇.

---

## 10. 참고 링크
- 공식 문서: https://modelcontextprotocol.io
- 아키텍처: https://modelcontextprotocol.io/docs/learn/architecture
- 공식 서버 모음: https://github.com/modelcontextprotocol/servers
- Anthropic 발표: https://www.anthropic.com/news/model-context-protocol
