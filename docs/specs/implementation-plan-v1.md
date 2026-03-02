# AI Company 구현 계획서 v1

> 최초 작성: 2026-03-03
> 최종 업데이트: 2026-03-03 (세션 1 개선 작업 반영)
> 목적: 현재 시스템의 근본 문제를 진단하고, 단계별 개선 계획을 수립한다.

---

## 변경 이력

| 버전 | 날짜 | 주요 변경 |
|------|------|----------|
| v1.0 | 2026-03-03 | 최초 작성 — 진단 + 즉시/단기/중기 계획 |
| v1.1 | 2026-03-03 | 세션 1 개선 반영: QA 에이전트 추가, 시스템 일반화, 완료 항목 표시, Rube 연결 상태 오진단 수정 |

---

## 0. 현재 상태 정확한 진단: "실제 실행 vs 시뮬레이션"

### 실제로 동작한 것과 아닌 것 (초기 MBTI 인스타그램 데모 기준)

| 태스크 | 실제 실행 여부 | 근거 | 산출물 품질 |
|--------|--------------|------|------------|
| task-001~004 (시장조사) | ✅ 실제 실행 | WebSearch/WebFetch 사용, 실제 웹 데이터 수집 | 양호 |
| task-005 (브랜드 아이덴티티) | ⚠️ 문서만 생성 | tools_used: Read, Write, WebSearch만 사용. 실제 이미지 없음 | 텍스트 가이드만 존재 |
| task-006 (계정 설정) | ⚠️ 가이드만 생성 | "Rube MCP 자동 업로드 파이프라인 **설계**" → 설계만 함, 실행 안 함 | 문서만 존재 |
| task-007 (캐릭터 디자인) | ⚠️ 텍스트 묘사만 | tools_used: Read, Write, WebSearch. 실제 캐릭터 이미지 없음 | AI 프롬프트 텍스트만 존재 |
| task-008 (스크립트) | ✅ 실제 실행 | 텍스트 산출물이 목표. 실제 스크립트 작성됨 | 양호 |
| task-009 (만화 제작 1차) | ⚠️ PIL만으로 시도 | 이모지 깨짐 문제 발생. Bash + PIL로 기본 이미지 생성 | 품질 낮음 |
| task-009-v2 (카드 제작 CEO 피드백 반영) | ✅ **실제로 동작함** | `GEMINI_GENERATE_IMAGE` + PIL로 실제 PNG 파일 생성됨 | **실제 이미지 4개 존재** |
| task-010 (캡션/해시태그) | ✅ 실제 실행 | 텍스트 산출물이 목표 | 양호 |
| task-011~013 (업로드/스케줄링) | ❌ 미실행 | briefing_approved=false, 외부 서비스 연결 없음 | 없음 |

### 핵심 발견

**task-009-v2가 실제로 동작한 유일한 이미지 생성 사례다.**
이 태스크에서 `GEMINI_GENERATE_IMAGE`가 실제로 호출됐고, 결과물이 존재한다.
그러나 `tool_inventory.json`에는 gemini가 `not_connected`로 기록되어 있었다.

→ **`tool_inventory.json`이 과거 시점의 스냅샷이었기 때문이다.** 실제로 Rube MCP를 통해 `RUBE_MANAGE_CONNECTIONS`를 호출하면 instagram, gemini, canva 모두 `ACTIVE` 상태였다. 즉, 도구는 연결되어 있었지만 시스템이 이를 모르고 있었다.

---

## 1. 현재 구조의 문제와 한계

### 문제 1: ~~Rube MCP 연결이 없다~~ → **스냅샷이 오래됐다** (수정된 진단)

> ⚠️ **v1.0 오진단 수정**: 최초 진단 시 `tool_inventory.json`을 신뢰하여 Rube가 연결 안 됐다고 판단했으나, 실제로는 instagram/gemini/canva 모두 ACTIVE 상태였다.

**실제 문제:**

```
tool_inventory.json (2026-01월 생성 스냅샷):
  instagram    → not_connected  ← 오래된 스냅샷
  gemini       → not_connected  ← 오래된 스냅샷
  canva        → not_connected  ← 오래된 스냅샷

RUBE_MANAGE_CONNECTIONS 실시간 확인:
  instagram    → ACTIVE ✅
  gemini       → ACTIVE ✅
  canva        → ACTIVE ✅
```

**왜 이런 상태가 됐나:**
- Tool Agent가 인벤토리 생성 당시 Rube가 연결되지 않은 상태였음
- 이후 Rube가 연결됐지만 인벤토리가 재생성되지 않음
- 오래된 스냅샷을 신뢰한 Expert/HR 에이전트들이 연결 안 된 도구를 기피

**해결책 (구현 완료 v1.1):**
- tool-agent.md에 "Tool Agent는 항상 RUBE_MANAGE_CONNECTIONS로 실시간 확인" 명시
- expert-agent.md에 Rube 3단계 호출 패턴 명시 (SEARCH → CONNECTIONS → EXECUTE)

---

### 문제 2: Expert 에이전트가 "계획"을 실행하지 않고 "문서"를 생성한다

ACTION 유형 태스크에서 기대했던 것:
```
외부 서비스 연동 태스크: 실제로 서비스를 호출하고 결과(ID, URL 등)를 반환
파일 생성 태스크: 실제 파일이 outputs/에 존재
```

실제로 일어난 것:
```
외부 서비스 태스크: "이렇게 연동하면 됩니다" 가이드 문서 작성
파일 생성 태스크: 파일 생성 방법을 설명하는 텍스트 문서 작성
```

**왜 이런 상태가 됐나:**
- Expert 에이전트 프롬프트가 "어떻게 해야 한다"는 가이드 중심
- 도구 호출 실패 시 문서 작성으로 조용히 우회
- `ACTION` 타입이어도 검증 없이 `completed` 처리
- "설명 문서를 작성했다"와 "실제로 실행했다"가 execution_log에서 구분되지 않음

**해결책 (구현 완료 v1.1):**
- expert-agent.md에 ACTION 태스크 완료 기준 명시
- 파일 없음/post_id 없음 → completed 불가
- 도구 연결 실패 → TOOL_ERROR 인터럽트 발생 (조용한 우회 금지)

---

### 문제 3: 결과물 품질 검증 루프가 없다

```
현재 흐름:
  Expert 실행 → execution_log에 "completed" 기록 → 종료

이상적인 흐름:
  Expert 실행 → QA 검증 → 70점 미만: 반려+피드백 → 재시도 → QA 재검증 → 승인
```

**왜 이런 상태가 됐나:**
- 결과물이 태스크 요구사항을 충족하는지 자동으로 확인하는 단계가 없었음
- CEO가 직접 결과물을 확인하고 수동으로 재시도를 요청해야 했음
- 3번 이상 품질이 안 좋으면 CEO가 직접 개입할 방법이 없었음

**해결책 (구현 완료 v1.1):**
- qa-agent.md 신규 생성: LLM-as-Judge 패턴, 0~100점 채점, 70점 이상 승인
- SKILL.md 워크플로우에 QA 루프 추가 (Expert → QA → approve/reject → 최대 3회)
- task_assignments.json에 qa_status, qa_score, qa_rejection_feedback 필드 추가
- 3회 초과 시 CEO 에스컬레이션 (qa_escalation 인터럽트)

---

### 문제 4: Expert 실행이 결정론적이지 않다

n8n의 파이프라인은 명확한 단계로 분해된다:
```
[데이터 수집] → [처리] → [변환] → [저장/전송] → [확인]
```

현재 Expert 에이전트 방식:
```
"태스크를 실행해라" → (AI가 알아서 판단) → 결과 저장
```

- 같은 태스크를 다시 실행해도 동일한 결과 보장 없음
- 실패 지점이 불명확 → 어디서 틀렸는지 파악 어려움
- QA가 반려해도 Expert가 무엇을 고쳐야 할지 불명확

**해결 방향 (단기 목표):**
- 태스크 유형별 실행 단계를 명시적으로 분해
- 각 단계 실패 시 TOOL_ERROR 즉시 발생 (다음 단계 진행 불가)

---

### 문제 5: HR이 만든 에이전트가 실제로 독립 실행되지 않는다

설계 의도:
```
HR이 expert-001, 002, 003, 004를 "고용" → 각자 독립적으로 실행
```

실제 동작:
```
hired_agents.json에 4명의 에이전트 정의가 저장됨
→ expert-agent.md 하나로 모든 expert를 처리
→ 역할 구분이 프롬프트 내 설명으로만 존재 (실제 다른 컨텍스트/능력이 아님)
```

**왜 이런 상태가 됐나:**
- Claude Code의 Agent tool은 미리 정의된 subagent 파일만 호출할 수 있음
- `hired_agents.json`이 생성됐다고 해서 `.claude/agents/expert-XXX.md` 파일이 자동 생성되지 않음
- HR 에이전트가 JSON 파일만 만들고 실제 agent 정의 파일 생성까지 하지 않음

**해결책 (구현 완료 v1.1):**
- hr-agent.md에 agent 파일 생성 책임 추가
- `hired_agents.json` 생성 직후 `.claude/agents/expert-{id}.md`도 생성
- 각 파일에 역할 특화 YAML frontmatter + 시스템 프롬프트 작성

---

### 문제 6: 상태 파일과 실제 실행 결과 사이의 검증이 없다 (문제 3과 연관)

```
execution_log.json:
  status: "completed"
  summary: "파이프라인 설계 완료"

→ "완료"라고 기록되어 있지만, 실제로는 실행이 아닌 설계서만 존재
→ 이후 의존 태스크가 이 "완료"를 신뢰하고 진행하려 하면 실패
```

→ QA 에이전트 추가로 부분 해결. qa_status 필드로 실제 검증 상태 추적 가능.

---

### 문제 7: 시스템이 특정 도메인(인스타그램)에 종속되어 있었다

초기 설계에서 에이전트 프롬프트, 예시, 역할 정의가 모두 "MBTI 인스타그램 계정 운영"에 특화되어 있었다. 다른 목표(예: 이커머스 분석, 이메일 마케팅, 코드 리뷰 자동화)를 CEO가 입력하면 시스템이 엉뚱하게 동작했다.

**해결책 (구현 완료 v1.1):**
- SKILL.md, hr-agent.md, tool-agent.md, expert-agent.md의 모든 도메인 특화 예시를 일반화
- "인스타그램", "밈", "@cs_student_tips" 같은 고정 예시 제거
- CEO 목표에 따라 백로그/에이전트/도구가 동적으로 결정되도록 수정

---

## 2. 의도와 다르게 동작한 이유 (근본 원인)

### 근본 원인 A: 스냅샷을 실시간 상태로 오인

```
tool_inventory.json 생성 시점의 Rube 연결 상태 → 이후 변경된 상태를 반영 못함
→ Expert/HR이 연결 안 된 도구라고 판단 → 우회 행동 발생
```

→ 해결: Tool Agent는 항상 RUBE_MANAGE_CONNECTIONS 실시간 호출

---

### 근본 원인 B: ACTION 태스크 완료 기준이 모호했다

```
"실행"의 의미가 에이전트마다 달랐음:
  Expert 해석: "outputs/에 무언가를 저장했으면 완료"
  설계 의도:   "실제 파일/API 결과가 존재해야 완료"
```

→ 해결: 완료 기준을 타입별로 명시 (파일 생성 / 외부 서비스 post_id / 코드 실행 로그)

---

### 근본 원인 C: 실패 시 조용한 우회가 허용됐다

```
n8n: [API 호출] → 실패 → [Error Branch] → 명시적 에러 알림
기존: [API 호출] → 실패 → "대신 문서 만들었습니다" → completed
```

→ 해결: 실행 불가 상황에서 TOOL_ERROR 인터럽트 의무화, QA가 이를 0점 처리

---

### 근본 원인 D: 결과물을 검증하는 독립적인 관찰자가 없었다

Expert 에이전트가 스스로 "완료"를 선언하는 구조였다. 자기 평가는 편향될 수밖에 없다.

```
기존: Expert → "완료" 선언 → 다음 태스크
개선: Expert → QA(독립 평가) → 승인/반려 → 재시도 또는 다음 태스크
```

→ 해결: QA 에이전트를 별도 컨텍스트에서 실행하는 독립 검증자로 추가

---

### 근본 원인 E: 시스템이 특정 데모 시나리오에 과적합되어 있었다

에이전트 프롬프트가 인스타그램 MBTI 계정 운영이라는 단일 시나리오에 맞춰 설계됐다. 일반화 부재로 인해 다른 목표를 입력하면 에이전트들이 맥락에 맞지 않는 행동을 했다.

---

## 3. 구현 작업 현황

### 세션 1 완료 항목 (2026-03-03)

| 항목 | 파일 | 내용 |
|------|------|------|
| ✅ ACTION 태스크 완료 기준 명시 | expert-agent.md | 파일 없음/post_id 없음 → completed 불가, TOOL_ERROR 의무화 |
| ✅ Rube MCP 3단계 호출 패턴 | expert-agent.md | RUBE_SEARCH_TOOLS → RUBE_MANAGE_CONNECTIONS → RUBE_MULTI_EXECUTE_TOOL |
| ✅ 재시도 시 qa_rejection_feedback 우선 확인 | expert-agent.md | QA 반려 후 재실행 시 피드백 반드시 읽도록 강제 |
| ✅ QA 에이전트 추가 | qa-agent.md (신규) | 0~100점 채점, 70점 기준 승인/반려, 최대 3회 재시도, CEO 에스컬레이션 |
| ✅ QA 루프 워크플로우 반영 | SKILL.md | Step 10에 Expert → QA → approve/reject 루프 추가 |
| ✅ HR 에이전트 agent 파일 생성 책임 추가 | hr-agent.md | hired_agents.json + .claude/agents/expert-{id}.md 동시 생성 |
| ✅ 시스템 일반화 (인스타그램 종속 제거) | SKILL.md, hr-agent.md, tool-agent.md | 도메인 특화 예시 → 범용 예시로 대체 |
| ✅ Phase 0 환경 체크 추가 | SKILL.md | company/state/ 폴더, .mcp.json, 기존 세션 여부 확인 단계 |
| ✅ tool_inventory.json 실시간 재확인 지침 | tool-agent.md | RUBE_MANAGE_CONNECTIONS 항상 실시간 호출하도록 명시 |

---

## 4. 단기 목표 (Short-term — 2주 내)

### [단기-1] Expert 실행을 결정론적 단계로 분해

현재 Expert 실행 방식:
```
expert-agent.md → "task_assignments.json 읽고 실행해라" (비결정론적)
```

n8n 방식으로 변환:
```
파일 생성 태스크 예시:
  Step 1: 필요한 도구 가용성 확인 (RUBE_MANAGE_CONNECTIONS)
  Step 2: 데이터/콘텐츠 생성 (API 호출)
  Step 3: 후처리 (합성, 변환, 검증)
  Step 4: outputs/{task_id}_{name}.{ext}로 저장
  Step 5: execution_log.json에 결과 기록 (output_files, metadata)
```

각 단계가 실패하면 즉시 TOOL_ERROR를 발생시키고 다음 단계로 넘어가지 않음

**수정 대상**: `expert-agent.md` — 태스크 유형별 실행 단계 추가

---

### [단기-2] 에러 핸들링 + 폴백 체인

n8n의 Error Branch 개념을 적용:

```
[Rube를 통한 도구 호출]
    │
    ├── 성공 → 다음 단계
    │
    └── 실패 (API 에러/연결 불가)
         ↓
    [폴백: 직접 API 호출 (Rube 우회)]
         │
         ├── 성공 → 다음 단계
         │
         └── 실패
              ↓
         [폴백: Built-in 도구(Bash+Python)로 대체]
              │
              └── 실패 → TOOL_ERROR 인터럽트 발생
```

---

### [단기-3] task_assignments.json에 completion_criteria 필드 추가

```json
{
  "task_id": "task-XXX",
  "type": "ACTION",
  "completion_criteria": {
    "required_outputs": [
      {"type": "file", "pattern": "outputs/task-XXX_*", "min_count": 1},
      {"type": "external_id", "field": "post_id"}
    ]
  }
}
```

QA 에이전트가 이 기준을 읽고 자동으로 검증

---

### [단기-4] dry_run 모드 도입

외부 서비스 연동이 필요한 태스크를 테스트할 때 실제 호출 없이 흐름만 검증:

```json
// session.json에 추가
{
  "execution_mode": "dry_run" | "production",
  "dry_run_behavior": {
    "external_api_calls": "mock",
    "file_generation": "real",
    "web_search": "real"
  }
}
```

`dry_run` 모드에서 외부 API 호출은 mock 응답을 반환하고 실제 요청은 발생하지 않음
`production` 모드에서만 실제 외부 서비스 호출

---

## 5. 중기 목표 (Mid-term — 1달 내)

### [중기-1] HR이 생성한 agent 파일로 실제 병렬 독립 실행

현재: 단일 expert-agent.md 파일이 모든 역할을 처리
목표: HR이 고용한 에이전트들이 실제로 독립 컨텍스트에서 병렬 실행

Claude Code Agent tool로 각 expert-{id}.md를 독립 호출:
```
의존성 없는 태스크들 → 동시에 각자 다른 agent 파일로 병렬 실행
  expert-001 (리서치 전문가) → task-001 수행
  expert-002 (디자인 전문가) → task-004 수행
  expert-003 (콘텐츠 작가)  → task-005 수행
```

**조건**: HR이 세션 1에서 추가한 agent 파일 생성 기능이 정상 동작해야 함

---

### [중기-2] QA 에이전트 고도화

현재 QA: 공통 기준(요구사항/CEO지시/목표정합성/완성도)으로 채점
목표: 태스크 유형별 전문 검증

| 태스크 유형 | 추가 검증 기준 |
|------------|--------------|
| RESEARCH | hallucination 감지, 출처 검증, 날짜 신선도 |
| ACTION (파일 생성) | 파일 크기, 포맷 정합성, 이미지 품질 |
| ACTION (외부 서비스) | API 응답 코드, ID 유효성, 실제 서비스 반영 확인 |
| DOCUMENT | 분량 적절성, 템플릿 준수, 내용 충실도 |

---

### [중기-3] 시뮬레이션 → 실제 실행 이관 체계

전체 파이프라인을 dry_run으로 먼저 검증한 뒤 production으로 전환하는 프로세스:

```
1. CEO가 목표 입력 → 백로그/에이전트/태스크 생성
2. dry_run 실행 → 모든 태스크의 흐름/에러 확인
3. CEO가 dry_run 결과 검토 → production 승인
4. production 실행 → 실제 외부 서비스 호출
```

---

### [중기-4] Rube MCP 연결 모니터링

현재 문제: 인벤토리가 스냅샷이어서 실제 연결 상태와 불일치
목표: 실행 시마다 연결 상태를 자동 갱신

```
SKILL.md Phase 0에 추가:
  - RUBE_MANAGE_CONNECTIONS 전체 앱 상태 확인
  - tool_inventory.json과 불일치 시 자동 업데이트
  - 연결 끊긴 앱이 있으면 CEO에게 알림
```

---

### [중기-5] 성과 기반 피드백 루프 (도메인 종속 없이)

외부 서비스 액션 후 결과를 추적하고 다음 실행에 반영:

```
외부 서비스 액션 → 결과 데이터 수집
→ 어떤 콘텐츠/방식이 목표에 기여했는지 분석
→ ceo_goal.json의 KPI 대비 진행률 업데이트
→ 다음 Task Briefing에 성과 데이터 포함
```

---

## 6. 구현 우선순위 요약

```
세션 1 완료 (2026-03-03):
  ✅ ACTION 태스크 완료 기준 명시 (expert-agent.md)
  ✅ Rube MCP 3단계 호출 패턴 (expert-agent.md)
  ✅ QA 에이전트 추가 (qa-agent.md 신규)
  ✅ QA 루프 워크플로우 반영 (SKILL.md)
  ✅ HR 에이전트 agent 파일 생성 책임 추가 (hr-agent.md)
  ✅ 시스템 일반화 (전체 에이전트 파일)
  ✅ Phase 0 환경 체크 (SKILL.md)

단기 (2주 내):
  ☐ [단기-1] Expert 실행 결정론적 단계 분해
  ☐ [단기-2] 에러 핸들링 + 폴백 체인
  ☐ [단기-3] completion_criteria 필드 추가
  ☐ [단기-4] dry_run 모드 도입

중기 (1달 내):
  ☐ [중기-1] HR agent 파일 → 실제 병렬 독립 실행
  ☐ [중기-2] QA 에이전트 고도화 (유형별 전문 검증)
  ☐ [중기-3] dry_run → production 이관 체계
  ☐ [중기-4] Rube MCP 연결 모니터링 자동화
```

---

## 7. 최종 목표 상태 (일반화)

```
CEO가 어떤 목표를 입력하든:

  Phase 1 (계획):
    ① CEO 목표 입력
    ② RM → 프로젝트/태스크/의존성 그래프 생성
    ③ Tool Agent → 실제 연결 상태 기반 인벤토리 생성
    ④ HR → 역량 기반 에이전트 고용 + agent 파일 생성
    ⑤ RM → 태스크 할당

  Phase 2 (실행):
    ⑥ Task Briefing → CEO 승인/수정
    ⑦ Expert → 실제 도구 사용하여 결과물 생성
    ⑧ QA → 품질 검증 (70점 기준 자동 판별)
    ⑨ 미달 시: 구체적 피드백 → Expert 재시도 (최대 3회)
    ⑩ CEO 개입 없이 자동 반복 → 목표 달성

  목표:
    - ACTION 태스크 성공률 > 95%
    - QA 1회 통과율 > 70%
    - CEO 개입: Task Briefing 1회 + 문제 발생 시만
    - 실행 흐름: 완전 자동화 (인간 루프 최소화)
```

---

## 부록: 시스템 아키텍처 변경 이력

### v1.0 → v1.1 변경 요약

| 구성요소 | v1.0 (이전) | v1.1 (현재) |
|---------|------------|------------|
| 에이전트 수 | 5개 (CEO, RM, Tool, HR, Expert) | **6개** (+ QA) |
| ACTION 완료 기준 | 없음 (암묵적) | 명시적 (파일 존재 / post_id 기록 필수) |
| 실패 처리 | 조용한 우회 → completed | TOOL_ERROR 인터럽트 의무화 |
| 결과물 검증 | CEO 수동 확인 | QA 자동 채점 (0~100, 70점 기준) |
| 재시도 | 수동 재실행 | 자동 반려+피드백+재시도 (최대 3회) |
| HR 산출물 | hired_agents.json | hired_agents.json + `.claude/agents/` 파일 |
| 도메인 종속성 | 인스타그램 특화 | **범용 (어떤 목표든 동작)** |
| 도구 상태 확인 | 스냅샷 신뢰 | RUBE_MANAGE_CONNECTIONS 실시간 확인 |
| 환경 체크 | 없음 | Phase 0 환경 검증 단계 추가 |
