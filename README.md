# ai_company

> **AI 에이전트로 구성된 회사**
> Spec-Driven, Agent-Oriented AI Organization Prototype

---

## 1. 프로젝트 개요

**ai_company** 프로젝트는
AI를 단순한 도구가 아닌 **조직의 구성원(직원)**으로 정의하고,
AI 에이전트들이 역할을 나누어 협업하며 하나의 회사를 운영하는 모델을 실험하는 프로젝트입니다.

본 프로젝트의 핵심 가설:

> CEO(사용자)가 회사의 목표를 정의하고 **레퍼런스(참고 자료)를 제공**하면,
> **RM 에이전트가 프로젝트를 분해**하고,
> **Tool Agent가 레퍼런스를 분석하여 최적 도구를 추천**하며,
> **HR 에이전트가 전문가를 채용**하고,
> **Expert 에이전트들이 레퍼런스를 참고하며 실제 업무를 수행**하여 목표를 달성할 수 있다.

---

## 2. 핵심 에이전트 구조 (v3)

```
                        CEO (사용자)
                            │
                            │ ① 목표 입력
                            ▼
                    ┌───────────────┐
                    │  RM 에이전트    │ ② 백로그 생성
                    └───────┬───────┘
                            │
                            ▼
                    ┌───────────────┐
                    │  CEO          │ ③ 레퍼런스 첨부 (URL, 이미지, 파일, 메모)
                    └───────┬───────┘
                            │
                            ▼
                    ┌───────────────┐
                    │ Tool 에이전트   │ ④ 레퍼런스 분석 → Tool 추천
                    └───────┬───────┘   (Analyze Once, Use Everywhere)
                            │
                            ▼
                    ┌───────────────┐
                    │  CEO          │ ⑤ Tool 추천 승인/수정
                    └───────┬───────┘
                            │
                            ▼
                    ┌───────────────┐
                    │  HR 에이전트    │ ⑥ 에이전트 고용 + Tool 할당
                    └───────┬───────┘   (분석 결과 읽기만, 재분석 안함)
                            │
                            ▼
                    ┌───────────────┐
                    │  RM 에이전트    │ ⑦ 태스크 할당
                    └───────┬───────┘
                            │
        ┌───────────────────┼───────────────────┐
        │                   │                   │
        ▼                   ▼                   ▼
┌───────────────┐   ┌───────────────┐   ┌───────────────┐
│ Expert Agent 1│   │ Expert Agent 2│   │ Expert Agent N│
│ (트렌드 리서처) │   │(미니멀 UI 디자이너)│   │(이커머스 전문가) │
└───────────────┘   └───────────────┘   └───────────────┘
  분석 결과 읽기 →     분석 결과 읽기 →     분석 결과 읽기 →
  바로 작업 수행       바로 작업 수행       바로 작업 수행
        │                   │                   │
        └───────────────────┴───────────────────┘
                            │
                            ▼
                    ┌───────────────┐
                    │  CEO          │ 결과 검토 및 다음 프로젝트
                    └───────────────┘
```

### 역할 분담

| 에이전트 | 역할 | 책임 | 모델 |
|----------|------|------|------|
| **CEO** | 의사결정자 | 목표 정의, 레퍼런스 제공, Tool 추천 승인, 인터럽트 응답 | sonnet |
| **RM** | 프로젝트 매니저 | 백로그 생성, 태스크 분해, 에이전트에 태스크 할당 | sonnet |
| **Tool Agent** | 도구 컨설턴트 | 레퍼런스 분석, Tool 추천/검증, `analyzed_content` 저장 | **sonnet** |
| **HR** | 에이전트 팩토리 | 분석 결과 기반 전문가 고용, Tool 배정 | sonnet |
| **Expert** | 실무자 | 분석 결과 참고하며 태스크 수행, 결과 보고 | sonnet |

---

## 3. 워크플로우 (v3 - Reference-Driven)

### 3.1 전체 워크플로우

```
┌─────────────────────────────────────────────────────────────────┐
│                        Phase 1: 계획 수립                        │
├─────────────────────────────────────────────────────────────────┤
│  ① CEO 목표 입력                                                │
│       ↓                                                         │
│  ② RM: 백로그 생성 (프로젝트/태스크)                            │
│       ↓                                                         │
│  ③ CEO: 레퍼런스 첨부 (URL, 이미지, 파일, 메모)                 │
│       ↓                                                         │
│  ④ Tool Agent: 레퍼런스 분석 → Tool 추천 + 검증                 │
│       ↓                                                         │
│  ⑤ CEO: Tool 추천 승인/수정                                     │
│       ↓                                                         │
│  ⑥ HR: 에이전트 고용 + Tool 할당                                │
│       ↓                                                         │
│  ⑦ RM: 에이전트에 태스크 할당                                   │
├─────────────────────────────────────────────────────────────────┤
│                        Phase 2: 실행                            │
├─────────────────────────────────────────────────────────────────┤
│  ⑧ CEO: 실행할 프로젝트 선택                                    │
│       ↓                                                         │
│  ⑨ Expert: 레퍼런스 분석 결과 참조하며 태스크 수행               │
│       ↓                                                         │
│  ⑩ 결과 보고                                                    │
└─────────────────────────────────────────────────────────────────┘
```

### 3.2 각 단계 상세

| 단계 | 주체 | 설명 | 출력 |
|------|------|------|------|
| ① | CEO | 회사 목표/KPI/제약조건 입력 | `ceo_goal.json` |
| ② | RM | 목표를 프로젝트/태스크로 분해, Tool 카테고리 제안 | `backlog.json` |
| ③ | CEO | 각 태스크에 레퍼런스 첨부 (URL, 이미지, 파일, 메모) | `task_references.json` |
| ④ | Tool Agent | 레퍼런스 깊은 분석 → `analyzed_content` 저장 → Tool 추천 | `tool_recommendations.json` |
| ⑤ | CEO | Tool Agent 추천 승인 또는 수정 | `tool_selections.json` |
| ⑥ | HR | `analyzed_content` 읽고 전문가 고용 + Tool 배정 | `hired_agents.json` |
| ⑦ | RM | 각 태스크에 적합한 에이전트 할당 | `task_assignments.json` |
| ⑧ | CEO | 실행할 프로젝트 선택 | - |
| ⑨ | Expert | `analyzed_content` 읽고 바로 태스크 실행 | `execution_log.json` |
| ⑩ | RM | 최종 결과 보고 | 결과 파일들 |

### 3.3 v2 → v3 핵심 변경점

| 항목 | v2 (Tool-Aware) | v3 (Reference-Driven) |
|------|-----------------|----------------------|
| 패러다임 | CEO가 Tool 선택 | **CEO가 레퍼런스 제공** |
| Tool Agent 역할 | 검증자 (Validator) | **컨설턴트 (Consultant)** |
| Tool Agent 모델 | haiku | **sonnet** |
| Expert 입력 | 태스크 + Tool만 | 태스크 + Tool + **analyzed_content** |
| CEO 인터랙션 | 2~3회 (Tool 선택 + 재선택) | **1~2회** (레퍼런스 + 승인) |
| 분석 효율 | 각 에이전트가 레퍼런스 반복 분석 | **Tool Agent가 1번 분석, 나머지는 읽기만** |

---

## 4. 아키텍처: Skill vs Subagent

본 프로젝트는 Claude Code의 **Subagent** 아키텍처를 사용합니다.

| 구분 | Skill | Subagent |
|------|-------|----------|
| **위치** | `.claude/skills/` | `.claude/agents/` |
| **형태** | 지침서/프롬프트 | 독립 AI 엔티티 |
| **컨텍스트** | 메인 세션과 공유 | **별도 컨텍스트** |
| **실행** | Claude가 역할 연기 | 독립적으로 작동 |
| **용도** | 도구/작업방식 지침 | 복잡한 작업 위임 |

### 왜 Subagent인가?

- **독립 컨텍스트**: 각 에이전트가 자신만의 메모리와 상태를 가짐
- **Tool 격리**: 에이전트별로 사용 가능한 Tool을 명시적으로 제한
- **모델 최적화**: 역할에 따라 적합한 모델 사용
- **확장성**: 새 에이전트 추가가 용이

### Spec-Driven Development (SDD)

코드는 다음이 준비되기 전까지 작성되지 않습니다:

- 요구사항 명세
- 유저 스토리
- 기술 스펙
- 테스트 케이스

---

## 5. 핵심 개념: Analyze Once, Use Everywhere

v3의 가장 중요한 설계 원칙입니다.

### 문제 (v2)

같은 레퍼런스를 3번 분석하는 비효율:

```
Tool Agent: URL 접속 → 분석 → Tool 추천
HR Agent:   URL 접속 → 분석 → 전문가 구체화    ← 중복
Expert:     URL 접속 → 분석 → 작업 수행         ← 중복
```

### 해결 (v3)

Tool Agent가 1번만 깊게 분석하고, `analyzed_content`에 결과 저장:

```
Tool Agent: URL 접속 → 깊은 분석 → analyzed_content에 저장 (1번만)
HR Agent:   analyzed_content 읽기 → 전문가 구체화
Expert:     analyzed_content 읽기 → 바로 작업 수행
```

### `analyzed_content` 구조

| 필드 | 설명 | 주 소비자 |
|------|------|----------|
| `direction` | 모든 레퍼런스를 종합한 태스크 방향성 | HR, Expert |
| `fetched_summary` | URL/파일/이미지 분석 요약 | HR, Expert |
| `key_findings` | 핵심 발견 사항 | HR (역할 도출) |
| `style_elements` | 색상, 레이아웃, 타이포그래피 | Expert (디자인 작업) |
| `structure_elements` | 페이지/문서 구조 분석 | Expert (구조 재현) |
| `actionable_insights` | 구체적 활용 지침 | Expert (실행 계획) |

---

## 6. 레퍼런스 시스템

### 레퍼런스 타입

CEO가 각 태스크에 첨부하는 참고 자료입니다.

| 타입 | 형태 | 예시 | 용도 |
|------|------|------|------|
| `url` | 웹 주소 | 경쟁사 사이트, 참고 디자인 | "이런 느낌으로" |
| `image` | 이미지 파일 경로 | 목업, 스크린샷, 로고 | "이렇게 생긴 것" |
| `file` | 로컬 파일 경로 | 기존 보고서, 템플릿, 데이터셋 | "이 포맷 따라서" |
| `note` | 텍스트 메모 | 스타일 가이드, 톤앤매너 | "이런 방향으로" |

### 레퍼런스 의도 (intent)

| Intent | 의미 |
|--------|------|
| `style` | "이런 스타일/느낌으로 만들어줘" |
| `content` | "이 내용을 참고해서 작성해줘" |
| `template` | "이 형식/구조를 따라줘" |
| `data` | "이 데이터를 활용해줘" |
| `competitor` | "이 경쟁사를 분석해줘" |

---

## 7. Tool 시스템

### 7.1 Tool 카테고리

| 타입 | 설명 | 예시 |
|------|------|------|
| **Built-in** | Claude Code 내장 도구 | WebSearch, WebFetch, Read, Write, Edit, Bash |
| **Skills** | 설치된 스킬 | /frontend-design, /mcp-builder |
| **MCP** | 외부 서비스 연동 | figma, **rube**, github |

### 7.2 Rube MCP - 외부 서비스 통합의 핵심

> **Rube**는 본 프로젝트에서 **외부 서비스 연동의 중심축** 역할을 합니다.

#### Rube란?

[Rube](https://rube.app)는 Composio 기반의 MCP 서버로, AI 에이전트가 **500개 이상의 외부 비즈니스 앱**에 자연어로 접근할 수 있게 해주는 통합 플랫폼입니다.

#### 왜 Rube가 핵심인가?

```
Without Rube                          With Rube
─────────────────                     ─────────────────
Expert Agent                          Expert Agent
    │                                     │
    ├── WebSearch ✅                      ├── WebSearch ✅
    ├── WebFetch ✅                       ├── WebFetch ✅
    ├── Read/Write ✅                     ├── Read/Write ✅
    │                                     │
    ├── Gmail 전송 ❌                     ├── Gmail 전송 ✅ (via Rube)
    ├── Slack 메시지 ❌                   ├── Slack 메시지 ✅ (via Rube)
    ├── Google Sheets ❌                  ├── Google Sheets ✅ (via Rube)
    ├── Notion 페이지 ❌                  ├── Notion 페이지 ✅ (via Rube)
    ├── Jira 이슈 ❌                      ├── Jira 이슈 ✅ (via Rube)
    └── DB 조작 ❌                        └── DB 조작 ✅ (via Rube)
```

#### Rube가 지원하는 주요 앱 카테고리

| 카테고리 | 대표 앱 | 활용 예시 |
|----------|---------|----------|
| **커뮤니케이션** | Gmail, Slack, Discord, Teams | 이메일 발송, 채널 메시지, 알림 |
| **프로젝트 관리** | Jira, Asana, Trello, Linear | 이슈 생성, 태스크 관리 |
| **문서/노트** | Notion, Google Docs, Confluence | 문서 작성, 위키 업데이트 |
| **데이터/스프레드시트** | Google Sheets, Airtable | 데이터 입력, 분석 결과 저장 |
| **CRM/세일즈** | HubSpot, Salesforce | 고객 관리, 리드 추적 |
| **저장소/코드** | GitHub, GitLab, Bitbucket | PR 생성, 이슈 관리 |
| **디자인** | Figma, Canva | 디자인 에셋 관리 |
| **결제/이커머스** | Stripe, Shopify | 결제 처리, 주문 관리 |

#### 워크플로우에서의 Rube 위치 (v3)

```
Phase 1 (계획)                           Phase 2 (실행)
─────────────────────────────────────    ─────────────────────────────────────
④ Tool Agent 레퍼런스 분석                ⑨ Expert 태스크 수행
   레퍼런스에서 외부 서비스 필요 감지           │
   "Sheets에 정리, Slack으로 알림"            ├── Built-in Tool 직접 사용
         │                                    │     (WebSearch, Bash, Read...)
   Rube 통해 가용성 검증                       │
   "Google Sheets → Rube 가능 ✅"             └── Rube MCP 경유 사용
   "Slack → Rube 가능 ✅"                          (Gmail, Sheets, Slack...)
         │                                          │
   analyzed_content에 기록                           ▼
                                              Rube가 OAuth/API Key로
                                              외부 서비스 인증 처리
```

#### 왜 Rube인가? (대안 비교)

| 도구 | 방식 | 앱 수 | 특징 | 오픈소스 |
|------|------|-------|------|----------|
| **Rube (Composio)** | MCP 서버 | 500+ | Claude Code 네이티브 지원, 노코드, 무료 | X |
| ACI.dev | MCP / Function Calling | 500+ | 오픈소스, 자체 호스팅 가능 | O |
| Pipedream | MCP + 워크플로우 빌더 | 10,000+ | 코드 작성 가능, 앱 수 최대 | X |
| n8n | 워크플로우 자동화 | 400+ | 오픈소스, 비주얼 빌더 | O |
| Arcade.dev | Function Calling | - | JIT 권한, 보안 특화 | X |

**Rube를 선택한 이유:**

1. **Claude Code 네이티브 호환**: `.mcp.json` 한 줄 설정으로 즉시 사용 가능
2. **노코드/자연어 기반**: Expert가 자연어로 "이메일 보내줘"라고 하면 Rube가 처리
3. **Composio 인프라**: SOC 2 Type II, 토큰 격리, OAuth 자동 관리
4. **무료 제공**: 실험 프로젝트 단계에서 비용 부담 없음
5. **MCP 표준**: 향후 다른 MCP 서버로 교체/병행 가능

#### Rube 설정

```json
// .mcp.json (프로젝트 루트에 포함됨)
{
  "mcpServers": {
    "rube": {
      "type": "http",
      "url": "https://rube.app/mcp"
    }
  }
}
```

**초기 설정 (최초 1회):**

1. 프로젝트 클론 후 Claude Code 실행
2. Rube MCP 서버 연결 승인
3. [rube.app/marketplace](https://rube.app/marketplace)에서 사용할 앱 연결 (OAuth 인증)
4. 연결 완료 후 Expert 에이전트가 해당 앱을 바로 사용 가능

### 7.3 Tool Agent의 분석/추천 프로세스 (v3)

```
CEO 레퍼런스
     │
     ▼
┌──────────────────────────┐
│  레퍼런스 깊은 분석        │
│  - URL → WebFetch        │
│  - 이미지 → Read          │
│  - 파일 → Read            │
│  - 메모 → 키워드 분석      │
└────────────┬─────────────┘
             │
             ▼
┌──────────────────────────┐
│  analyzed_content 저장    │  ← Analyze Once
│  - direction (방향성)     │
│  - style_elements        │
│  - structure_elements    │
│  - actionable_insights   │
└────────────┬─────────────┘
             │
             ▼
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│  Built-in   │ ──► │   Skills    │ ──► │    MCP      │
│   검증      │     │   검증       │     │   검증       │
└─────────────┘     └─────────────┘     └─────────────┘
             │
        사용 불가 시
             ▼
┌─────────────────────┐
│  대안 검색           │
│  - 유사 Tool 추천    │
│  - 마켓플레이스 검색  │
│  - 웹 검색          │
└─────────────────────┘
```

---

## 8. 실행 방법 (Claude Code)

### 8.1 Subagent 호출 (Task tool)

```bash
# Step-by-step
"ceo-agent로 목표를 입력해줘"
"rm-agent로 백로그를 생성해줘"
"ceo-agent로 레퍼런스를 첨부해줘"
"tool-agent로 분석하고 Tool 추천해줘"
"ceo-agent로 Tool 추천을 승인해줘"
"hr-agent로 에이전트를 고용해줘"
"rm-agent로 태스크를 할당해줘"
"expert-agent로 태스크를 실행해줘"
```

### 8.2 Skill 호출 (슬래시 명령어)

```bash
/ai-company           # 전체 시스템 오케스트레이션
/frontend-design      # 웹 UI 디자인
/mcp-builder          # MCP 서버 생성
```

---

## 9. 디렉토리 구조

```
ai_company/
├── .mcp.json                      # 프로젝트 레벨 MCP 설정 (Rube 포함)
├── .claude/
│   ├── agents/                    # Subagent 정의 (독립 컨텍스트)
│   │   ├── ceo-agent.md           # CEO: 목표 정의, 레퍼런스 제공, 승인
│   │   ├── rm-agent.md            # RM: 백로그 생성, 태스크 할당
│   │   ├── tool-agent.md          # Tool: 레퍼런스 분석, Tool 추천
│   │   ├── hr-agent.md            # HR: 전문가 고용
│   │   └── expert-agent.md        # Expert: 태스크 실행
│   │
│   ├── skills/                    # 유틸리티 스킬
│   │   ├── ai-company/            # 메인 오케스트레이터
│   │   ├── frontend-design/       # 디자인 스킬
│   │   ├── mcp-builder/           # MCP 빌더
│   │   └── skill-creator/         # 스킬 생성기
│   │
│   └── CLAUDE.md                  # Claude Code 가이드
│
├── company/
│   ├── state/                     # 상태 파일 (JSON)
│   │   ├── session.json           # 세션 정보
│   │   ├── ceo_goal.json          # CEO 목표
│   │   ├── backlog.json           # RM 백로그
│   │   ├── task_references.json   # CEO 레퍼런스 (v3)
│   │   ├── tool_recommendations.json # Tool 분석+추천 (v3)
│   │   ├── tool_selections.json   # CEO 승인된 최종 Tool
│   │   ├── hired_agents.json      # 고용된 에이전트
│   │   ├── task_assignments.json  # 태스크 할당
│   │   └── execution_log.json     # 실행 로그
│   │
│   ├── outputs/                   # 실행 결과물
│   └── assets/                    # 리소스 파일
│
├── docs/
│   ├── specs/                     # 기술 스펙
│   │   ├── reference-driven-workflow-v3.md  # v3 워크플로우 (최신)
│   │   ├── tool-aware-workflow-v2.md        # v2 워크플로우 (레거시)
│   │   ├── agentic-ai-architecture.md
│   │   └── e2e-test-scenarios.md
│   └── reports/                   # 보고서
│
└── README.md
```

---

## 10. 상태 파일 스키마

### session.json
```json
{
  "session_id": "ses_20260206_001",
  "phase": "reference_collection",
  "created_at": "2026-02-06T10:00:00Z",
  "version": "v3"
}
```

### task_references.json (v3 신규)
```json
{
  "references_id": "ref_20260206_001",
  "task_references": {
    "task-006": {
      "references": [
        {
          "ref_id": "ref-004",
          "type": "url",
          "value": "https://www.coupang.com/vp/products/123",
          "intent": "style",
          "note": "이 페이지의 레이아웃 참고"
        }
      ]
    }
  }
}
```

### tool_recommendations.json (v3 신규, analyzed_content 포함)
```json
{
  "task_recommendations": [
    {
      "task_id": "task-006",
      "reference_analysis": {
        "direction": "쿠팡 스타일 레이아웃 + 미니멀 + 브랜드 컬러(딥 그린)",
        "analyzed_content": [
          {
            "ref_id": "ref-004",
            "fetched_summary": "쿠팡 상품 상세 페이지. 2컬럼 레이아웃.",
            "style_elements": {"layout": "2컬럼", "colors": "#346aff 포인트"},
            "structure_elements": {"hero": "이미지 캐러셀", "sidebar": "가격/장바구니"},
            "actionable_insights": "2컬럼 유지하되 브랜드 컬러로 교체"
          }
        ]
      },
      "recommended_tools": [
        {"tool": "/frontend-design", "type": "skill", "match_score": 0.95}
      ]
    }
  ]
}
```

### hired_agents.json
```json
{
  "agents": [
    {
      "agent_id": "expert-002",
      "role_name": "미니멀 UI 디자이너",
      "assigned_tools": ["/frontend-design", "WebFetch"],
      "assigned_tasks": ["task-006"],
      "reference_context": "쿠팡 상품 페이지 레이아웃, 브랜드 컬러, 미니멀 스타일"
    }
  ]
}
```

---

## 11. 핵심 개념

| 개념 | 설명 |
|------|------|
| **Reference-Driven** | CEO가 레퍼런스를 제공하면, Tool Agent가 분석하여 최적 Tool 추천 |
| **Analyze Once, Use Everywhere** | Tool Agent가 1번 분석한 결과를 HR/Expert가 재활용 |
| **analyzed_content** | 레퍼런스 깊은 분석 결과 (스타일, 구조, 핵심 발견, 활용 지침) |
| **direction** | 모든 레퍼런스를 종합한 태스크 방향성 한 문장 |
| **Subagent** | 독립 컨텍스트에서 실행되는 AI 에이전트 |
| **Backlog** | RM이 목표를 분해한 프로젝트/태스크 목록 |
| **Interrupt** | Expert가 CEO에게 정보/승인을 요청하는 메커니즘 |

---

## 12. 차별점

- **Reference-Driven 워크플로우**: CEO는 Tool을 몰라도 됨. 레퍼런스만 주면 시스템이 최적 도구를 찾아줌
- **Analyze Once, Use Everywhere**: 레퍼런스 1회 분석으로 전체 파이프라인이 동작. 토큰 낭비 최소화
- **Subagent 아키텍처**: 독립 컨텍스트로 실행되는 진정한 멀티 에이전트 시스템
- **Rube MCP 기반 외부 서비스 통합**: 500+ 비즈니스 앱과 AI-Native 연동
- **동적 에이전트 생성**: 분석 결과 기반으로 전문가 자동 생성 (e.g., "미니멀 UI 디자이너")
- **Human-in-the-loop**: CEO의 의사결정이 핵심 지점에서 개입

---

## 13. 프로젝트 상태

**v3 Reference-Driven 아키텍처 완료**

- [x] v3 설계 스펙 작성 (`reference-driven-workflow-v3.md`)
- [x] Subagent 구조 v3 업데이트
  - [x] ceo-agent.md (레퍼런스 제공자 + 승인자)
  - [x] rm-agent.md
  - [x] tool-agent.md (컨설턴트, sonnet, analyzed_content)
  - [x] hr-agent.md (분석 결과 기반 고용)
  - [x] expert-agent.md (분석 결과 기반 실행)
- [x] 유틸리티 스킬 v3 반영
  - [x] ai-company (오케스트레이터)
  - [x] frontend-design
  - [x] mcp-builder
  - [x] skill-creator
- [x] 상태 파일 구조 v3 정의
- [ ] 실제 E2E 테스트 수행

---

## 14. 참고 문서

| 문서 | 경로 | 설명 |
|------|------|------|
| CLAUDE.md | `.claude/CLAUDE.md` | Claude Code 가이드 |
| Reference-Driven 워크플로우 | `docs/specs/reference-driven-workflow-v3.md` | **v3 워크플로우 상세 설계 (최신)** |
| Tool-Aware 워크플로우 | `docs/specs/tool-aware-workflow-v2.md` | v2 워크플로우 (레거시) |
| Agentic AI 아키텍처 | `docs/specs/agentic-ai-architecture.md` | 시스템 아키텍처 스펙 |
| E2E 테스트 시나리오 | `docs/specs/e2e-test-scenarios.md` | 통합 테스트 시나리오 |

---

## 15. 안내

본 프로젝트는
AI가 인간을 대체하는 것이 아니라,
**인간과 AI의 협업 방식을 재정의하기 위한 실험적 프로젝트**입니다.
