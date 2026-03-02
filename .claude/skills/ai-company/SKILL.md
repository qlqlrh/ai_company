---
name: ai-company
description: "AI Company 멀티 에이전트 시스템 시뮬레이션. CEO가 목표를 입력하면, Tool Agent가 인벤토리를 생성하고 HR이 역량 기반으로 Expert를 고용합니다. 프로젝트 실행 전 태스크별 Task Briefing으로 CEO가 Tool/레퍼런스/지시사항을 검토·승인한 뒤, Expert가 자동 병렬로 실행합니다."
---

# AI Company v4 - Interactive Execution Multi-Agent System

CEO(사용자)가 목표를 입력하면, 에이전트들이 협업하여 목표를 달성하는 시스템입니다.
v4의 핵심은 **Task Briefing** — 태스크 실행 전에 CEO가 "누가, 어떤 Tool로, 어떻게"를 검토하고 승인합니다.

## 워크플로우 (v4)

```
┌─────────────────────────────────────────────────────────────────┐
│                    Phase 1: 계획 수립                            │
├─────────────────────────────────────────────────────────────────┤
│  ① CEO 목표 입력                                                │
│       ↓                                                         │
│  ② RM: 백로그 생성 (dependency_graph 포함)                      │
│       ↓                                                         │
│  ③ Tool Agent: Tool 인벤토리 생성 (3-tier 검증)                 │
│       ↓                                                         │
│  ④ HR: 인벤토리 참고하여 역량 기반 Expert 생성                  │
│       ↓                                                         │
│  ⑤ RM: Expert 할당 + 기본 Tool 배정                             │
├─────────────────────────────────────────────────────────────────┤
│                    Phase 2: 프로젝트 실행 (반복)                 │
├─────────────────────────────────────────────────────────────────┤
│  ⑥ CEO: 실행할 프로젝트 선택                                    │
│       ↓                                                         │
│  ⑦ 태스크별 실행 계획 보고 (Task Briefing)                      │
│       ↓                                                         │
│  ⑧ CEO: 태스크별 승인 / Tool 변경 / 레퍼런스 추가 / 지시사항    │
│       ↓                                                         │
│  ⑨ (필요시) Tool Agent 레퍼런스 분석 + Tool 조정                │
│       ↓                                                         │
│  ⑩ 태스크 실행 + QA 검증 (dependency_graph 기반 자동 병렬)      │
│      expert-agent → qa-agent → 승인 또는 반려+재시도 (최대 3회) │
│       ↓                                                         │
│  ⑪ 결과 보고 (QA 통과한 결과물만) → 다음 프로젝트 ⑥으로        │
└─────────────────────────────────────────────────────────────────┘
```

## 실행 방법

### 명령어
- `/ai-company` - 현재 상태 확인 및 계속 진행
- `/ai-company init` - 새 세션 시작
- `/ai-company status` - 현재 상태 요약
- `/ai-company reset` - 모든 상태 초기화

## Step-by-Step 가이드

### Phase 0: 환경 확인 (시작 전 필수)

**`/ai-company init` 실행 시 가장 먼저 수행합니다.**

```
1. company/state/ 폴더 존재 여부 확인 → 없으면 생성
2. .mcp.json 확인 → Rube 설정 여부 확인
   - Rube 설정 있음 → "Rube MCP가 설정되어 있습니다. 연결 상태는 Tool Agent가 확인합니다."
   - Rube 설정 없음 → "외부 서비스(이미지 생성, SNS 등)가 필요하면 .mcp.json에 Rube를 추가하세요."
3. 기존 session.json 확인
   - 있음 → "진행 중인 세션이 있습니다. 이어서 진행하시겠습니까? (Y/N)"
   - 없음 → 새 세션 시작
```

### Step 1: CEO 목표 입력

사용자에게 목표를 입력받습니다:

```
══════════════════════════════════════════════════════════════
   AI Company v4 - Interactive Execution Multi-Agent System
══════════════════════════════════════════════════════════════

안녕하세요, CEO님! 회사의 목표를 입력해주세요.

예시:
- "신규 SaaS 제품의 랜딩 페이지와 마케팅 자료 제작"
- "경쟁사 분석 후 블로그 콘텐츠 20개 작성 및 발행"
- "고객 응대 자동화 시스템 구축"
- "유튜브 채널 기획부터 첫 영상 10개 업로드까지"

목표:
```

**ceo-agent 호출**: 목표를 JSON으로 저장

```bash
# company/state/session.json
{
  "session_id": "ses_YYYYMMDD_NNN",
  "phase": "goal_input",
  "created_at": "ISO-8601"
}

# company/state/ceo_goal.json
{
  "goal": "입력받은 목표",
  "created_at": "ISO-8601"
}
```

### Step 2: RM 백로그 생성

**rm-agent 호출**: 목표를 프로젝트/태스크로 분해 + **dependency_graph 생성**

분석 시 고려사항:
1. 목표 달성에 필요한 프로젝트 식별
2. 각 프로젝트를 구체적 태스크로 분해
3. **태스크 간 의존성을 명시적 dependency_graph로 표현**
4. 각 태스크에 필요한 Tool 카테고리 제안

```
══════════════════════════════════════════════════════════════
   RM: 백로그 생성 완료
══════════════════════════════════════════════════════════════

프로젝트 1: 시장 조사 및 경쟁사 분석
   ├─ Task 001: 경쟁사 리서치        [RESEARCH] 의존: 없음
   ├─ Task 002: 타겟 페르소나 정의   [DOCUMENT] 의존: task-001
   └─ Task 003: 차별화 전략 수립     [DOCUMENT] 의존: task-001, task-002

프로젝트 2: 콘텐츠 전략
   ├─ Task 004: 콘텐츠 소재 수집     [RESEARCH] 의존: 없음
   ├─ Task 005: 콘텐츠 캘린더 수립   [DOCUMENT] 의존: task-004
   └─ Task 006: 해시태그 전략        [RESEARCH] 의존: task-001

의존성 그래프:
  task-001: []  ←── 독립 (병렬 가능)
  task-004: []  ←── 독립 (병렬 가능)
  task-002: [task-001]
  task-006: [task-001]
  task-003: [task-001, task-002]
  task-005: [task-004]
```

저장: `company/state/backlog.json` (dependency_graph 포함)

### Step 3: Tool Agent 인벤토리 생성

**tool-agent 호출**: 사용 가능한 Tool 전체를 인벤토리로 정리

검증 프로세스 (3-tier):
1. Built-in 체크 (WebSearch, WebFetch, Read, Write, Edit, Bash, Glob, Grep)
2. Local Skills 스캔 (.claude/skills/)
3. MCP 서버 + Rube 마켓플레이스 검색 (RUBE_SEARCH_TOOLS + RUBE_MANAGE_CONNECTIONS)
4. 외부 API 웹 검색 (WebSearch)
5. 마켓플레이스 검색 (MCP.so 등)

```
══════════════════════════════════════════════════════════════
   Tool Agent: Tool 인벤토리 생성 완료
══════════════════════════════════════════════════════════════

Built-in (8): WebSearch, WebFetch, Read, Write, Edit, Bash, Glob, Grep
Skills (2): /frontend-design, /mcp-builder
MCP/Rube: 백로그의 태스크 유형을 분석하여 필요한 도구 카테고리를 검색한 결과
  - [목표에 맞는 Rube 앱 목록과 연결 상태 표시]
  예) gemini: ACTIVE / slack: NOT_CONNECTED / github: ACTIVE

태스크별 기본 Tool 추천:
  [각 태스크 ID]: [백로그 suggested_tool_categories 기반 추천 도구]
```

저장: `company/state/tool_inventory.json`

### Step 4: HR 에이전트 고용

**hr-agent 호출**: Tool 인벤토리를 참고하여 **역량 기반** Expert 생성

```
══════════════════════════════════════════════════════════════
   HR: 에이전트 고용 완료
══════════════════════════════════════════════════════════════

┌──────────────────────────────────────────────────────────────┐
│ Expert 1: 시장 조사 전문가                                    │
├──────────────────────────────────────────────────────────────┤
│ 역량: research, data_analysis                                │
│ 실현 가능 Tool: WebSearch, WebFetch, INSTAGRAM_GET_USER_INFO │
│ 기본 Tool: Read, Write, WebSearch                            │
└──────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────┐
│ Expert 4: 밈 콘텐츠 제작 전문가                               │
├──────────────────────────────────────────────────────────────┤
│ 역량: image_creation, meme_design, copywriting               │
│ 실현 가능 Tool: GEMINI_GENERATE_IMAGE, imgflip_api, PIL      │
│ 기본 Tool: Read, Write, WebSearch, Bash                      │
└──────────────────────────────────────────────────────────────┘
```

저장: `company/state/hired_agents.json`

### Step 5: RM 태스크 할당

**rm-agent 호출**: Expert에 태스크 할당 + 인벤토리의 task_tool_defaults 기반 기본 Tool 배정

저장: `company/state/task_assignments.json`

session.json → phase: `ready_to_execute`

### Step 6: CEO 프로젝트 선택

```
══════════════════════════════════════════════════════════════
   실행 준비 완료!
══════════════════════════════════════════════════════════════

실행할 프로젝트를 선택해주세요:

  [1] 시장 조사 및 경쟁사 분석 (3 tasks)
  [2] 콘텐츠 전략 및 소재 준비 (3 tasks)
  [3] 계정 설정 및 브랜딩 (3 tasks)
  [4] 초기 콘텐츠 제작 및 론칭 (3 tasks)
  [5] 성장 전략 및 커뮤니티 빌딩 (3 tasks)
  [6] 운영 자동화 및 분석 시스템 (3 tasks)

선택:
```

### Step 7: Task Briefing (v4 핵심)

CEO가 프로젝트를 선택하면, **각 태스크별로 실행 계획을 보고**합니다.

```
══════════════════════════════════════════════════════════════
  프로젝트 N: [프로젝트명]
  실행 계획 (K tasks)
══════════════════════════════════════════════════════════════

  [1] Task XXX: [태스크명]
      담당: [역할명] (expert-00N)
      Tool: [task_assignments.json의 default_tools 기반]
      출력: [expected_outputs]
      의존: [선행 태스크]

  [2] Task XXX: [태스크명]
      ...

──────────────────────────────────────────────────────────────
  수정하실 태스크가 있으면 번호를 말씀해주세요.
  레퍼런스를 첨부하시려면 "task-XXX에 [URL/파일/메모] 추가"
  Tool을 변경하시려면 "task-XXX는 [다른 Tool] 써줘"
  전체 OK면 "승인" 또는 "실행"
══════════════════════════════════════════════════════════════
```

### Step 8: CEO 태스크별 승인/수정

CEO의 개입 유형:

| 유형 | 예시 | 결과 |
|------|------|------|
| **승인** | "OK", "전부 승인" | briefing_approved = true → 실행 |
| **Tool 변경** | "task-010은 Imgflip API 써줘" | ceo_tools 업데이트 |
| **레퍼런스 추가** | "이 이미지 참고해" (파일/URL/메모) | → Step 9 분석 |
| **지시사항** | "PNG 출력, AI 티 내지 마" | ceo_instructions 업데이트 |
| **일괄 승인** | "전부 OK" | 모든 태스크 한번에 승인 |

CEO 개입 예시:
```
CEO: "task-XXX는 [특정 Tool]로 해줘.
      이거 참고해" (URL/이미지/파일 첨부)
      "[구체적 지시사항]"

→ 업데이트:
  [N] Task XXX: [태스크명]  [수정됨]
      담당: [역할명]
      Tool: [변경된 Tool]    ← 변경
      레퍼런스: CEO 제공 자료  ← 추가
      지시: [CEO 지시사항]    ← 추가

CEO: "OK 승인"
→ 실행 시작
```

저장: `company/state/task_assignments.json` (ceo_tools, ceo_references, ceo_instructions 업데이트)

### Step 9: 레퍼런스 분석 + Tool 조정 (필요시만)

CEO가 레퍼런스를 첨부한 경우에만 **tool-agent 호출**.
"Analyze Once, Use Everywhere" 원칙에 따라 분석합니다.

분석 프로세스:
1. 레퍼런스 타입 + 의도(intent) 분석
2. URL → WebFetch / 이미지 → Read / 파일 → Read / 메모 → 키워드 추출
3. analyzed_content에 저장 (fetched_summary, key_findings, style_elements, actionable_insights)
4. direction(방향성) 도출
5. enriched_description(상세 태스크 설명) 생성
6. CEO 요청 Tool 가용성 확인 (인벤토리에서 조회)

저장: `company/state/task_assignments.json` 내 해당 태스크의 ceo_references.analyzed_content, enriched_description, direction

### Step 10: 태스크 실행 + QA 검증 (자동 병렬)

**expert-agent 호출 → qa-agent 검증** 사이클을 dependency_graph 기반으로 자동 병렬 실행합니다.

**단일 태스크 실행 사이클:**

```
[1] expert-agent 호출 (태스크 실행)
      ↓
[2] qa-agent 호출 (결과물 검증)
      ↓ score ≥ 70
[3] ✅ 승인 → task status = "qa_approved" → 다음 태스크
      ↓ score < 70 (최대 3회까지)
[4] ❌ 반려 → qa_rejection_feedback을 expert에게 전달 → [1]로 재시도
      ↓ 3회 초과
[5] 🚨 CEO 인터럽트 (qa_escalation)
```

**병렬 실행 시:**
- 독립 태스크는 동시에 실행 (각자 expert → QA 사이클 독립 진행)
- 의존 태스크는 선행 태스크가 **QA 승인까지 완료**된 후 시작

**오케스트레이터의 역할:**
1. dependency_graph에서 실행 가능한 태스크 그룹 추출
2. 각 태스크에 대해 expert-agent 호출
3. expert 완료 → qa-agent 호출하여 결과 검증
4. QA 반려 시 → retry_instructions와 함께 expert-agent 재호출
5. QA 승인 시 → 다음 의존 태스크 unlock

**오케스트레이터는 QA가 통과된 태스크만 최종 완료로 처리합니다.**

```
══════════════════════════════════════════════════════════════
   실행 중: 프로젝트 N - [프로젝트명]
══════════════════════════════════════════════════════════════

의존성 분석:
  task-XXX: 선행 완료 → 바로 실행
  task-YYY: task-XXX 의존 → 대기

[실행] Task XXX: [태스크명]
   담당: [역할명]
   Tool: [승인된 Tool 목록]
   레퍼런스: analyzed_content 참조 (있는 경우)
   지시: [ceo_instructions]

   결과: company/outputs/[산출물 파일]
   완료 ✅

[실행] Task YYY: [태스크명] (task-XXX 완료 → 시작)
   ...
```

독립 태스크가 여러 개인 경우 자동 병렬:
```
[병렬 실행]
  task-004 (독립) ──┐
  task-006 (독립) ──┤ 동시 실행!
                    ↓
  task-005 (task-004 의존) → 순차
```

저장: `company/state/execution_log.json`

### Step 11: 결과 보고

```
══════════════════════════════════════════════════════════════
   프로젝트 완료 보고서
══════════════════════════════════════════════════════════════

프로젝트: 초기 콘텐츠 제작 및 론칭
상태: 완료

결과물:
  1. 밈 이미지 30개 (PNG 1080x1080)
     → company/outputs/memes/
  2. 캡션 및 해시태그 목록
     → company/outputs/task011_captions.md
  3. 론칭 체크리스트
     → company/outputs/task012_checklist.md

다음 프로젝트:
  [5] 성장 전략 및 커뮤니티 빌딩 (3 tasks)
  [6] 운영 자동화 및 분석 시스템 (3 tasks)

계속하시겠습니까? [다음 프로젝트] [종료]
```

## 상태 파일 구조 (v4)

```
company/state/
├── session.json              # 세션 정보 (phase, current_project)
├── ceo_goal.json             # CEO 목표
├── backlog.json              # RM 백로그 + dependency_graph
├── tool_inventory.json       # Tool 인벤토리
├── hired_agents.json         # 역량 기반 에이전트
├── task_assignments.json     # 할당 + CEO 승인/레퍼런스/지시사항
└── execution_log.json        # 실행 로그
```

## 상태 전이 (Phase)

```
Phase 1:
  goal_input → backlog_creation → tool_inventory → hiring → assignment → ready_to_execute

Phase 2 (프로젝트 단위 반복):
  project_selection → task_briefing → ceo_review → executing → project_completed
  → project_selection (다음 프로젝트)

전체 완료:
  all_completed
```

## 사용 가능한 Tool 목록

### Built-in (항상 사용 가능)
- `WebSearch` - 웹 검색
- `WebFetch` - 웹 페이지 내용 가져오기
- `Read` - 파일 읽기
- `Write` - 파일 쓰기
- `Edit` - 파일 편집
- `Bash` - 명령어 실행
- `Glob` - 파일 검색
- `Grep` - 텍스트 검색

### Skills (설치된 경우)
- `/frontend-design` - 웹 UI 코드 생성
- `/mcp-builder` - MCP 서버 생성
- 기타 `.claude/skills/` 디렉토리의 스킬들

### MCP (설정된 경우)
- `rube` - 외부 서비스 통합 허브 (500+ 앱: Instagram, Gemini, Canva, Gmail, Slack 등)
- `figma` - Figma 디자인 연동
- `github` - GitHub 연동
- 기타 설정된 MCP 서버들

## 레퍼런스 시스템 (Just-In-Time)

v4에서 레퍼런스는 Phase 1이 아닌 **Phase 2 Task Briefing에서 태스크별로** 수집합니다.

**레퍼런스 타입:**
- URL: 웹 주소 (경쟁사, 참고 디자인 등)
- 이미지: 이미지 파일 경로 (목업, 스크린샷 등)
- 파일: 로컬 파일 경로 (보고서 템플릿, 데이터셋 등)
- 메모: 텍스트 메모 (스타일 가이드, 톤앤매너 등)

**레퍼런스 의도(intent):**
- `style`: "이런 스타일로 만들어줘"
- `content`: "이 내용을 참고해줘"
- `template`: "이 형식을 따라줘"
- `data`: "이 데이터를 활용해줘"
- `competitor`: "이 경쟁사를 분석해줘"

**분석 원칙 (Analyze Once, Use Everywhere):**
Tool Agent가 1번만 깊게 분석 → analyzed_content에 저장 → Expert가 읽기만

## 주의사항

1. 이 스킬은 **Claude Code 환경**에서만 동작합니다
2. MCP 서버 사용 시 사전 설정이 필요합니다 (`.mcp.json`)
3. 상태 파일은 `company/state/` 폴더에 JSON으로 저장됩니다
4. 세션 종료 후에도 상태가 유지되어 계속 진행 가능합니다
5. 레퍼런스는 **Task Briefing 시점에** 태스크별로 제공합니다 (Phase 1에서 일괄 수집하지 않음)
6. dependency_graph 기반 자동 병렬 실행을 위해 독립 태스크를 최대한 식별합니다
