# ai_company

> **AI 에이전트로 구성된 회사**
> Spec-Driven, Agent-Oriented AI Organization Prototype

---

## 1. 프로젝트 개요

**ai_company** 프로젝트는
AI를 단순한 도구가 아닌 **조직의 구성원(직원)**으로 정의하고,
AI 에이전트들이 역할을 나누어 협업하며 하나의 회사를 운영하는 모델을 실험하는 프로젝트입니다.

본 프로젝트의 핵심 가설:

> CEO(사용자)가 회사의 목표를 정의하면,
> **RM 에이전트가 프로젝트를 분해**하고,
> **Tool Agent가 필요한 도구를 검증**하며,
> **HR 에이전트가 전문가를 채용**하고,
> **Expert 에이전트들이 실제 업무를 수행**하여 목표를 달성할 수 있다.

---

## 2. 핵심 에이전트 구조 (v2)

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
                    │  CEO          │ ③ Tool 선택
                    └───────┬───────┘
                            │
                            ▼
                    ┌───────────────┐
                    │ Tool 에이전트   │ ④ Tool 검증 + 대안 제시
                    └───────┬───────┘
                            │
                            ▼
                    ┌───────────────┐
                    │  HR 에이전트    │ ⑤ 에이전트 고용 + Tool 할당
                    └───────┬───────┘
                            │
                            ▼
                    ┌───────────────┐
                    │  RM 에이전트    │ ⑥ 태스크 할당
                    └───────┬───────┘
                            │
        ┌───────────────────┼───────────────────┐
        │                   │                   │
        ▼                   ▼                   ▼
┌───────────────┐   ┌───────────────┐   ┌───────────────┐
│ Expert Agent 1│   │ Expert Agent 2│   │ Expert Agent N│
│  (트렌드 리서처) │   │  (디자인 전문가) │   │  (이커머스 전문가)│
└───────────────┘   └───────────────┘   └───────────────┘
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
| **CEO** | 의사결정자 | 목표 정의, Tool 선택, 프로젝트 선택, 인터럽트 응답 | sonnet |
| **RM** | 프로젝트 매니저 | 백로그 생성, 태스크 분해, 에이전트에 태스크 할당 | sonnet |
| **Tool Agent** | 도구 검증자 | Tool 가용성 검증, 대안 제시, 마켓플레이스 검색 | haiku |
| **HR** | 에이전트 팩토리 | 전문가 에이전트 고용, Tool 배정 | sonnet |
| **Expert** | 실무자 | 할당된 Tool로 태스크 수행, 결과 보고 | sonnet |

---

## 3. 워크플로우 (v2 - Tool-Aware)

### 3.1 전체 워크플로우

```
┌─────────────────────────────────────────────────────────────────┐
│                        Phase 1: 계획 수립                        │
├─────────────────────────────────────────────────────────────────┤
│  ① CEO 목표 입력                                                │
│       ↓                                                         │
│  ② RM: 백로그 생성 (프로젝트/태스크)                            │
│       ↓                                                         │
│  ③ CEO: 각 태스크에 Tool 지정                                   │
│       ↓                                                         │
│  ④ Tool Agent: Tool 검증 + 대안 제시                            │
│       ↓                                                         │
│  ⑤ HR: 에이전트 고용 + Tool 할당                                │
│       ↓                                                         │
│  ⑥ RM: 에이전트에 태스크 할당                                   │
├─────────────────────────────────────────────────────────────────┤
│                        Phase 2: 실행                            │
├─────────────────────────────────────────────────────────────────┤
│  ⑦ CEO: 실행할 프로젝트 선택                                    │
│       ↓                                                         │
│  ⑧ Expert: 태스크 수행                                          │
│       ↓                                                         │
│  ⑨ 결과 보고                                                    │
└─────────────────────────────────────────────────────────────────┘
```

### 3.2 각 단계 상세

| 단계 | 주체 | 설명 | 출력 |
|------|------|------|------|
| ① | CEO | 회사 목표/KPI/제약조건 입력 | `ceo_goal.json` |
| ② | RM | 목표를 프로젝트/태스크로 분해, Tool 카테고리 제안 | `backlog.json` |
| ③ | CEO | 각 태스크에 사용할 Tool 선택 | `tool_selections.json` |
| ④ | Tool Agent | Tool 가용성 검증, 대안 제시 | `tool_validations.json` |
| ⑤ | HR | 검증된 Tool 기반 Expert 에이전트 고용 | `hired_agents.json` |
| ⑥ | RM | 각 태스크에 적합한 에이전트 할당 | `task_assignments.json` |
| ⑦ | CEO | 실행할 프로젝트 선택 | - |
| ⑧ | Expert | 할당된 Tool로 태스크 실행 | `execution_log.json` |
| ⑨ | RM | 최종 결과 보고 | 결과 파일들 |

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
- **모델 최적화**: 단순 작업(Tool 검증)은 haiku, 복잡한 작업은 sonnet 사용
- **확장성**: 새 에이전트 추가가 용이

### Spec-Driven Development (SDD)

코드는 다음이 준비되기 전까지 작성되지 않습니다:

- 요구사항 명세
- 유저 스토리
- 기술 스펙
- 테스트 케이스

---

## 5. Tool 시스템

### 5.1 Tool 카테고리

| 타입 | 설명 | 예시 |
|------|------|------|
| **Built-in** | Claude Code 내장 도구 | WebSearch, WebFetch, Read, Write, Edit, Bash |
| **Skills** | 설치된 스킬 | /frontend-design, /mcp-builder |
| **MCP** | 외부 서비스 연동 | figma, rube, github |

### 5.2 Tool Agent의 검증 프로세스

```
CEO 요청 Tool
     │
     ▼
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│  Built-in   │ ──► │   Skills    │ ──► │    MCP      │
│   검증      │     │   검증       │     │   검증       │
└─────────────┘     └─────────────┘     └─────────────┘
     │                   │                   │
     └───────────────────┴───────────────────┘
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

## 6. 실행 방법 (Claude Code)

### 6.1 Subagent 호출 (Task tool)

```bash
# 명시적 호출
"ceo-agent를 사용해서 목표를 입력해줘"
"rm-agent로 백로그를 생성해줘"
"tool-agent로 Tool을 검증해줘"
"hr-agent로 에이전트를 고용해줘"
"expert-agent로 태스크를 실행해줘"

# Claude가 description을 보고 자동 위임할 수도 있음
```

### 6.2 Skill 호출 (슬래시 명령어)

```bash
/ai-company           # 전체 시스템 오케스트레이션
/frontend-design      # 웹 UI 디자인
/mcp-builder          # MCP 서버 생성
```

---

## 7. 디렉토리 구조

```
ai_company/
├── .claude/
│   ├── agents/                  # Subagent 정의 (독립 컨텍스트)
│   │   ├── ceo-agent.md         # CEO 역할 에이전트
│   │   ├── rm-agent.md          # RM(Resource Manager) 에이전트
│   │   ├── tool-agent.md        # Tool 검증 에이전트
│   │   ├── hr-agent.md          # HR 에이전트
│   │   └── expert-agent.md      # Expert 에이전트
│   │
│   ├── skills/                  # 유틸리티 스킬
│   │   ├── ai-company/          # 메인 오케스트레이터
│   │   ├── frontend-design/     # 디자인 스킬
│   │   ├── mcp-builder/         # MCP 빌더
│   │   └── skill-creator/       # 스킬 생성기
│   │
│   └── CLAUDE.md                # Claude Code 가이드
│
├── company/
│   ├── state/                   # 상태 파일 (JSON)
│   │   ├── session.json         # 세션 정보
│   │   ├── ceo_goal.json        # CEO 목표
│   │   ├── backlog.json         # RM 백로그
│   │   ├── tool_selections.json # CEO Tool 선택
│   │   ├── tool_validations.json# Tool 검증 결과
│   │   ├── hired_agents.json    # 고용된 에이전트
│   │   ├── task_assignments.json# 태스크 할당
│   │   └── execution_log.json   # 실행 로그
│   │
│   ├── outputs/                 # 실행 결과물
│   └── assets/                  # 리소스 파일
│
├── docs/
│   ├── specs/                   # 기술 스펙
│   │   ├── tool-aware-workflow-v2.md
│   │   ├── agentic-ai-architecture.md
│   │   └── e2e-test-scenarios.md
│   └── reports/                 # 보고서
│
└── README.md
```

---

## 8. 상태 파일 스키마

### session.json
```json
{
  "session_id": "ses_20250119_001",
  "phase": "execution",
  "created_at": "2025-01-19T10:00:00Z",
  "version": "v2"
}
```

### backlog.json (일부)
```json
{
  "backlog_id": "bl_20250119_001",
  "goal": "쿠팡 입점 및 월 매출 1000만원",
  "projects": [
    {
      "id": "proj-001",
      "name": "쿠팡 입점 준비",
      "tasks": [
        {
          "id": "task-001",
          "name": "시장 트렌드 조사",
          "type": "RESEARCH",
          "suggested_tool_categories": ["web_search"]
        }
      ]
    }
  ]
}
```

### hired_agents.json (일부)
```json
{
  "agents": [
    {
      "agent_id": "expert-001",
      "role_name": "트렌드 리서처",
      "assigned_tools": ["WebSearch", "WebFetch"],
      "assigned_tasks": ["task-001", "task-002"]
    }
  ]
}
```

---

## 9. 핵심 개념

| 개념 | 설명 |
|------|------|
| **Tool-Aware** | CEO가 Tool을 직접 선택하고, 사전에 검증받는 방식 |
| **Subagent** | 독립 컨텍스트에서 실행되는 AI 에이전트 |
| **Backlog** | RM이 목표를 분해한 프로젝트/태스크 목록 |
| **Tool Validation** | Tool Agent가 수행하는 가용성 검증 |
| **Interrupt** | Expert가 CEO에게 정보/승인을 요청하는 메커니즘 |
| **Phase** | 워크플로우의 현재 단계 (goal_input → execution → completed) |

---

## 10. 차별점

- **Subagent 아키텍처**: 독립 컨텍스트로 실행되는 진정한 멀티 에이전트 시스템
- **Tool-Aware 워크플로우**: CEO가 직접 Tool을 선택하고 검증받음
- **사전 검증**: 실행 전에 모든 Tool 가용성 확인
- **마켓플레이스 연동**: 없는 Tool은 외부에서 검색 및 추천
- **동적 에이전트 생성**: 검증된 Tool 기반으로 전문가 자동 생성
- **Human-in-the-loop**: CEO의 의사결정이 핵심 지점에서 개입

---

## 11. 프로젝트 상태

**v2 Subagent 아키텍처 완료**

- [x] Subagent 구조로 마이그레이션
  - [x] ceo-agent.md
  - [x] rm-agent.md
  - [x] tool-agent.md
  - [x] hr-agent.md
  - [x] expert-agent.md
- [x] 유틸리티 스킬 유지
  - [x] ai-company (오케스트레이터)
  - [x] frontend-design
  - [x] mcp-builder
  - [x] skill-creator
- [x] E2E 테스트 시나리오 작성
- [x] 상태 파일 구조 정의
- [ ] 실제 E2E 테스트 수행

---

## 12. 참고 문서

| 문서 | 경로 | 설명 |
|------|------|------|
| CLAUDE.md | `.claude/CLAUDE.md` | Claude Code 가이드 |
| Tool-Aware 워크플로우 | `docs/specs/tool-aware-workflow-v2.md` | v2 워크플로우 상세 설계 |
| Agentic AI 아키텍처 | `docs/specs/agentic-ai-architecture.md` | 시스템 아키텍처 스펙 |
| E2E 테스트 시나리오 | `docs/specs/e2e-test-scenarios.md` | 통합 테스트 시나리오 |
| 구현 보고서 | `docs/reports/implementation-report.md` | 구현 결과 및 기술 스택 |

---

## 13. 안내

본 프로젝트는
AI가 인간을 대체하는 것이 아니라,
**인간과 AI의 협업 방식을 재정의하기 위한 실험적 프로젝트**입니다.
