# AI Company 구현 계획서 v1

> 작성일: 2026-03-03
> 목적: 현재 시스템의 근본 문제를 진단하고, 단계별 개선 계획을 수립한다.

---

## 0. 현재 상태 정확한 진단: "실제 실행 vs 시뮬레이션"

### 실제로 동작한 것과 아닌 것

| 태스크 | 실제 실행 여부 | 근거 | 산출물 품질 |
|--------|--------------|------|------------|
| task-001~004 (시장조사) | ✅ 실제 실행 | WebSearch/WebFetch 사용, 실제 웹 데이터 수집 | 양호 |
| task-005 (브랜드 아이덴티티) | ⚠️ 문서만 생성 | tools_used: Read, Write, WebSearch만 사용. 실제 이미지 없음 | 텍스트 가이드만 존재 |
| task-006 (인스타 계정 설정) | ⚠️ 가이드만 생성 | "Rube MCP 자동 업로드 파이프라인 **설계**" → 설계만 함, 실행 안 함 | 문서만 존재 |
| task-007 (캐릭터 디자인) | ⚠️ 텍스트 묘사만 | tools_used: Read, Write, WebSearch. 실제 캐릭터 이미지 없음 | AI 프롬프트 텍스트만 존재 |
| task-008 (스크립트) | ✅ 실제 실행 | 텍스트 산출물이 목표. 실제 스크립트 작성됨 | 양호 |
| task-009 (만화 제작 1차) | ⚠️ PIL만으로 시도 | 이모지 깨짐 문제 발생. Bash + PIL로 기본 이미지 생성 | 품질 낮음 |
| task-009-v2 (카드 제작 CEO 피드백 반영) | ✅ **실제로 동작함** | `GEMINI_GENERATE_IMAGE` + PIL로 실제 PNG 파일 생성됨. CEO "대박!" 반응 | **실제 이미지 4개 존재** |
| task-010 (캡션/해시태그) | ✅ 실제 실행 | 텍스트 산출물이 목표 | 양호 |
| task-011~013 (업로드/스케줄링) | ❌ 미실행 | briefing_approved=false, Instagram 연결 없음 | 없음 |

### 핵심 발견

**task-009-v2가 실제로 동작한 유일한 이미지 생성 사례다.**
이 태스크에서 `GEMINI_GENERATE_IMAGE`가 실제로 호출됐고, 결과물이 존재한다.
그러나 이후 `tool_inventory.json`에는 gemini가 `not_connected`로 기록되어 있다.
→ **Rube MCP Gemini 연결이 일시적이었거나, 인벤토리 생성 시점이 연결 이전이었음을 의미한다.**

---

## 1. 현재 구조의 문제와 한계

### 문제 1: Rube MCP 연결이 실제로 없다 (가장 근본적인 문제)

```
tool_inventory.json 상태:
  instagram    → not_connected   ← Instagram 업로드 불가
  gemini       → not_connected   ← AI 이미지 생성 불가 (Rube 경유 시)
  canva        → not_connected   ← 디자인 도구 불가
  google_sheets → not_connected  ← 데이터 관리 불가
```

Expert 에이전트는 `feasible_tools`에 이 도구들이 나열되어 있지만,
실제로 도구가 없기 때문에 호출 시 실패하거나 다른 방식으로 대체한다.

**왜 이런 상태가 됐나:**
- `.mcp.json`에 Rube 서버 URL만 있고, OAuth 인증(rube.app/marketplace)이 완료되지 않음
- Tool Agent가 인벤토리 생성 시 `RUBE_MANAGE_CONNECTIONS`를 통해 상태를 확인했고, 모두 `not_connected`로 기록
- 이 상태 그대로 진행됨 → HR, RM, Expert 모두 연결 안 된 도구를 "쓸 수 있는 것처럼" 정의함

---

### 문제 2: Expert 에이전트가 "계획"을 실행하지 않고 "문서"를 생성한다

action 유형 태스크에서 기대했던 것:
```
task-006 (계정 설정): 실제로 Instagram 계정을 Business 계정으로 전환하고, 프로필 설정
task-007 (캐릭터 디자인): 실제 PNG 이미지 파일 생성
task-005 (브랜드 아이덴티티): 실제 로고, 색상 팔레트 이미지 생성
```

실제로 일어난 것:
```
task-006: Instagram Graph API 연동 방법 문서 작성 (설계서)
task-007: MBTI 캐릭터의 외형 텍스트 묘사 + AI 프롬프트 작성
task-005: 색상 코드와 폰트 이름이 담긴 브랜드 가이드 문서 작성
```

**왜 이런 상태가 됐나:**
- Expert 에이전트의 프롬프트가 "어떻게 해야 한다"는 가이드 중심이며, 실제 실행을 강제하지 않음
- 도구가 없으면 (Rube not_connected), 에이전트는 합리적으로 "할 수 있는 것"인 문서 생성으로 우회
- "실행했다"와 "계획을 문서로 만들었다"가 execution_log에서 구분되지 않음
- 태스크 타입이 `ACTION`이어도 검증 없이 `completed`로 처리됨

---

### 문제 3: 이미지 생성 파이프라인이 결정론적이지 않다

n8n의 이미지→Instagram 파이프라인은 명확한 단계로 분해된다:
```
[이미지 생성] → [공개 URL 호스팅] → [미디어 컨테이너 생성] → [처리 대기] → [게시]
```

현재 Expert 에이전트 방식:
```
"만화 이미지를 만들어라" → (AI가 알아서 판단) → 결과 파일 저장
```

- 이미지가 생성돼도 공개 URL로 호스팅하는 단계가 없음
- Instagram Graph API는 로컬 파일 직접 업로드 불가 (공개 URL 필수)
- 생성 실패 시 폴백 경로 없음
- 같은 태스크를 다시 실행해도 동일한 결과 보장 없음

---

### 문제 4: Instagram 업로드가 구조적으로 불가능하다

Instagram Graph API의 필수 요구사항:

| 요구사항 | 현재 상태 |
|---------|----------|
| Instagram Business/Creator 계정 | 미확인 (task-006은 가이드만 작성) |
| Meta Developer App 등록 | 미완료 |
| Access Token (Page Access Token) | 없음 |
| 이미지 공개 URL (CDN/호스팅) | 없음 |
| 2-step 업로드 프로세스 구현 | 없음 |

→ task-011 (초기 10개 업로드)은 현재 구조에서는 실행 자체가 불가능하다.

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
- `hired_agents.json`이 생성됐다고 해서 새로운 `.claude/agents/expert-XXX.md` 파일이 자동 생성되지 않음
- HR 에이전트가 JSON 파일만 만들고, 실제 agent 정의 파일 생성까지 하지 않음

---

### 문제 6: 상태 파일과 실제 실행 결과 사이의 검증이 없다

```
execution_log.json의 task-006:
  status: "completed"
  summary: "Rube MCP 자동 업로드 파이프라인 설계 완료"

→ "완료"라고 기록되어 있지만, 실제로는 업로드 파이프라인이 없음
→ 이후 단계(task-011 업로드)가 이 "완료"를 신뢰하고 진행하려 하면 실패
```

현재 아키텍처에는 "이 태스크가 실제로 ACTION을 수행했는가?" 검증 단계가 없다.

---

## 2. 의도와 다르게 동작한 이유 (근본 원인)

### 근본 원인 A: "도구 연결"과 "도구 정의"를 혼동

```
tool_inventory.json에 gemini가 listed → Expert가 gemini를 쓸 수 있다고 인식
그러나 actually → Rube OAuth 연결 없으면 gemini 호출 = 에러
```

해결 전략: 도구 가용성을 런타임에 검증하거나, 연결된 도구만 사용

---

### 근본 원인 B: Action 태스크가 Document 태스크처럼 처리됨

Expert 에이전트 프롬프트 구조:
```
"task_assignments.json에서 태스크를 읽어라 → enriched_description 따라 실행 → 결과를 outputs/에 저장"
```

"실행"의 의미가 모호함:
- 이미지 생성 태스크 → 이미지를 만들어야 함
- 하지만 도구가 없으면 → 이미지를 어떻게 만들어야 하는지 설명하는 문서를 만듦
- 에이전트 입장에서는 "산출물을 outputs/에 저장했으니 완료"

해결 전략: 태스크 타입(ACTION vs DOCUMENT)에 따른 산출물 검증 기준 명시

---

### 근본 원인 C: Rube MCP가 프로젝트의 단일 실패 지점

시스템 전체가 Rube MCP 연결에 의존하도록 설계:
```
Instagram 업로드 → Rube → not_connected → 실패
Gemini 이미지 생성 → Rube → not_connected → 실패
```

Rube가 없으면 Expert가 할 수 있는 것: 텍스트 작성, 웹 검색, 기본 Python 스크립트

해결 전략: Rube 없이도 동작하는 직접 API 경로를 기본으로 구현

---

### 근본 원인 D: n8n과 달리 실패 감지/복구 메커니즘이 없다

n8n 노드는 실패하면 즉시 명시적 에러를 발생시키고 Error Branch로 분기
현재 Expert 에이전트는 실패하면 할 수 있는 방향으로 조용히 우회하고 "completed" 처리

```
n8n: [Gemini API] → 실패 → [Error Branch] → "Gemini 연결 실패: Access Denied" 알림
현재: [Gemini API] → 실패 → "대신 PIL로 간단한 이미지 만들었습니다" → completed
```

---

### 근본 원인 E: HR 에이전트의 에이전트 "고용"이 파일 생성에 그침

HR 에이전트가 `hired_agents.json`을 만드는 것은 "고용 기록"이지 "실제 에이전트 생성"이 아님
실제 독립 에이전트가 되려면 `.claude/agents/expert-001.md`와 같은 파일이 생성되어야 함
그리고 그 파일에 해당 전문가에 맞는 시스템 프롬프트가 작성되어야 함

---

## 3. 즉시 해결책 (Immediate — 이번 주 내)

### [즉시-1] Gemini API 직접 연결 (Rube 우회)

**문제**: Rube MCP의 Gemini가 not_connected
**해결**: 환경변수로 Gemini API 키 직접 설정 → Bash에서 Python으로 직접 호출

구현 방법:
```bash
# 환경변수 설정
export GEMINI_API_KEY="your-api-key"

# expert-agent.md에 추가할 Bash 실행 패턴
python3 - <<'EOF'
import google.generativeai as genai
import base64, os

genai.configure(api_key=os.environ["GEMINI_API_KEY"])
model = genai.GenerativeModel("gemini-2.0-flash-exp-image-generation")
response = model.generate_content(
    "kawaii owl character, INTJ, dark purple, cute anime style",
    generation_config={"response_modalities": ["IMAGE"]}
)
with open("output.png", "wb") as f:
    f.write(base64.b64decode(response.candidates[0].content.parts[0].inline_data.data))
EOF
```

**수정 대상**: `expert-agent.md` — 이미지 생성 시 Rube 대신 직접 API 호출 패턴 명시

---

### [즉시-2] Meta Graph API로 Instagram 업로드 파이프라인 구현

**문제**: Instagram 업로드 경로가 설계만 있고 구현이 없음
**해결**: Meta Graph API 2-step 프로세스를 Expert가 직접 실행하는 스크립트로 구현

필요 선행 작업 (사람이 1회 직접 해야 함):
1. [developers.facebook.com](https://developers.facebook.com)에서 App 생성
2. Instagram Basic Display API 또는 Instagram Graph API 권한 획득
3. Long-lived Access Token 발급 (60일 유효)
4. Instagram Business Account ID 확인

구현할 파이프라인:
```python
# company/assets/scripts/instagram_upload.py
import requests, time, sys

ACCESS_TOKEN = os.environ["INSTAGRAM_ACCESS_TOKEN"]
IG_USER_ID = os.environ["INSTAGRAM_USER_ID"]

def upload_to_instagram(image_url: str, caption: str) -> str:
    """
    Step 1: 미디어 컨테이너 생성
    Step 2: 처리 완료 대기 (최대 60초)
    Step 3: 게시
    """
    # Step 1
    res = requests.post(
        f"https://graph.facebook.com/v19.0/{IG_USER_ID}/media",
        data={"image_url": image_url, "caption": caption, "access_token": ACCESS_TOKEN}
    )
    container_id = res.json()["id"]

    # Step 2: 처리 대기
    for _ in range(12):
        status = requests.get(
            f"https://graph.facebook.com/v19.0/{container_id}",
            params={"fields": "status_code", "access_token": ACCESS_TOKEN}
        ).json().get("status_code")
        if status == "FINISHED":
            break
        time.sleep(5)

    # Step 3: 게시
    res = requests.post(
        f"https://graph.facebook.com/v19.0/{IG_USER_ID}/media_publish",
        data={"creation_id": container_id, "access_token": ACCESS_TOKEN}
    )
    return res.json().get("id")
```

이미지 공개 URL 문제 해결 - imgbb 무료 호스팅:
```python
# 로컬 이미지를 공개 URL로 임시 호스팅
def upload_image_to_imgbb(image_path: str) -> str:
    import base64
    IMGBB_API_KEY = os.environ["IMGBB_API_KEY"]  # 무료, 하루 32MB
    with open(image_path, "rb") as f:
        b64 = base64.b64encode(f.read()).decode()
    res = requests.post(
        "https://api.imgbb.com/1/upload",
        data={"key": IMGBB_API_KEY, "image": b64}
    )
    return res.json()["data"]["url"]
```

**수정 대상**: `company/assets/scripts/` 폴더에 스크립트 파일 생성

---

### [즉시-3] Expert 에이전트의 ACTION 태스크 검증 기준 명시

**문제**: Expert가 이미지 생성 대신 텍스트 문서를 만들어도 completed로 처리됨
**해결**: `expert-agent.md`에 태스크 타입별 산출물 검증 조건 추가

```markdown
## 태스크 완료 기준

### ACTION 태스크 (이미지/디자인 관련)
- ✅ 완료: outputs/ 폴더에 실제 .png, .jpg 파일이 존재
- ❌ 미완료: "이미지를 이렇게 만들면 됩니다" 형태의 설명서만 존재
- ❌ 미완료: AI 프롬프트만 작성되고 실제 이미지 생성 안 됨

### ACTION 태스크 (SNS 업로드 관련)
- ✅ 완료: Instagram post ID가 반환되고 execution_log에 기록됨
- ❌ 미완료: "업로드 방법" 가이드 문서만 존재

### ACTION 태스크를 수행할 수 없는 경우
필요한 도구/인증이 없어서 실제 ACTION이 불가능하면:
1. 즉시 TOOL_ERROR 인터럽트 발생
2. "어떤 도구/인증이 필요한가"를 명시
3. CEO가 해결 후 재실행
```

---

### [즉시-4] tool_inventory.json 실제 연결 상태 재확인

현재 `tool_inventory.json`이 과거 시점의 스냅샷이어서 부정확함
Tool Agent를 재실행하여 현재 실제 연결 상태를 반영한 인벤토리를 새로 생성해야 함

또한 task-009-v2에서 `GEMINI_GENERATE_IMAGE`가 실제로 동작했다면,
그 시점에 Rube Gemini 연결이 되어 있었던 것 → 현재도 연결 가능성 확인 필요

---

## 4. 단기 해결책 (Short-term — 2주 내)

### [단기-1] n8n 패턴 적용: Expert 실행을 결정론적 단계로 분해

현재 Expert 실행 방식:
```
expert-agent.md → "task_assignments.json 읽고 실행해라" (비결정론적)
```

n8n 방식으로 변환:
```
이미지 생성 태스크:
  Step 1: Gemini API로 캐릭터 이미지 생성 → output.png 저장
  Step 2: imgbb API로 공개 URL 획득
  Step 3: PIL로 카드 합성 (1080x1080, 텍스트 오버레이, 워터마크)
  Step 4: 최종 PNG 저장 → outputs/{task_id}_{mbti}.png
  Step 5: execution_log.json에 결과 기록 (output_files, image_urls)
```

각 단계가 실패하면 즉시 TOOL_ERROR를 발생시키고 다음 단계로 넘어가지 않음

**수정 대상**: `expert-agent.md` — 태스크 유형별 실행 단계 명시

---

### [단기-2] 이미지 생성 → 업로드 통합 파이프라인

```
[Gemini API] → [캐릭터 PNG]
      ↓
[PIL 합성] → [카드 PNG 1080x1080]
      ↓
[imgbb 업로드] → [공개 URL]
      ↓
[Instagram 2-step] → [포스트 ID] → [execution_log 기록]
```

이 파이프라인을 `company/assets/scripts/content_pipeline.py`로 구현
Expert 에이전트가 이 스크립트를 Bash로 호출하는 방식

---

### [단기-3] 에러 핸들링 + 폴백 체인

n8n의 Error Branch 개념을 적용:

```
[Gemini API 이미지 생성]
    │
    ├── 성공 → PIL 합성 → 계속
    │
    └── 실패 (API 에러/연결 불가)
         ↓
    [폴백: Gemini API 직접 호출 (Rube 우회)]
         │
         ├── 성공 → PIL 합성 → 계속
         │
         └── 실패
              ↓
         [폴백: PIL만으로 텍스트+색상 카드 생성]
              │
              └── TOOL_ERROR 인터럽트 + "Gemini 연결 필요" 메시지
```

---

### [단기-4] Access Token 자동 갱신 시스템

Instagram Access Token은 60일 만료
자동 갱신 스크립트 구현:

```python
# company/assets/scripts/refresh_token.py
# Long-lived token을 60일마다 갱신
# 만료 7일 전 CEO에게 알림 (실행 로그에 경고 기록)
```

---

### [단기-5] HR 에이전트 개선: JSON 파일 생성 → 실제 Agent 파일 생성

현재:
```
HR 에이전트 → hired_agents.json 생성 (끝)
```

개선:
```
HR 에이전트 → hired_agents.json 생성
           → .claude/agents/expert-001.md 생성 (역할 특화 프롬프트 포함)
           → .claude/agents/expert-002.md 생성
           → .claude/agents/expert-003.md 생성
           → .claude/agents/expert-004.md 생성
```

각 agent 파일은 해당 역할에 맞는 YAML frontmatter + 시스템 프롬프트:
```yaml
---
name: visual-designer-expert
description: MBTI 만화 캐릭터 디자인 및 이미지 생성 전문가
tools: Read, Write, Edit, Bash, WebSearch, WebFetch, Glob, Grep
model: claude-sonnet-4-6
---

당신은 비주얼 디자이너입니다. 전문 분야:
- Gemini API를 통한 카와이 스타일 캐릭터 생성
- PIL/Pillow를 통한 Instagram 카드 합성 (1080x1080)
- 브랜드 가이드라인 100% 준수

[구체적인 실행 패턴, API 호출 코드 템플릿 포함]
```

**수정 대상**: `hr-agent.md` — JSON 파일 생성 외 agent 파일 생성 책임 추가

---

### [단기-6] task_assignments.json에 ACTION 검증 필드 추가

```json
{
  "task_id": "task-011",
  "type": "ACTION",
  "completion_criteria": {
    "required_outputs": [
      {"type": "file", "pattern": "outputs/*.png", "count": 10},
      {"type": "instagram_post_id", "count": 10}
    ],
    "verification_method": "check_outputs_exist_and_instagram_ids"
  }
}
```

---

## 5. 중기 해결책 (Mid-term — 1달 내)

### [중기-1] Claude Agent Teams 활용

현재: 단일 expert-agent.md 파일이 모든 역할을 처리
목표: HR이 고용한 에이전트들이 실제로 독립 컨텍스트에서 병렬 실행

```
# settings.json에 추가
{
  "env": {
    "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1"
  }
}
```

Agent Teams 구조:
```
Team Lead (ai-company 스킬)
    ├── 공유 Task List (company/state/team_tasks.json)
    ├── visual-designer-expert (독립 컨텍스트)
    │   └── 이미지 생성, PIL 합성 전담
    ├── content-writer-expert (독립 컨텍스트)
    │   └── 스크립트, 캡션, 해시태그 전담
    └── sns-manager-expert (독립 컨텍스트)
        └── Instagram 업로드, 인게이지먼트 전담
```

**수정 대상**:
- `ai-company/SKILL.md` — Agent Teams 오케스트레이션 로직 추가
- HR 에이전트가 생성한 agent 파일들이 실제 Agent Teams member로 동작

---

### [중기-2] 콘텐츠 자동화 파이프라인 완성

목표 플로우:
```
[Google Sheets: 콘텐츠 아이디어]
         ↓ (Rube 연결 또는 API 직접)
[content-writer-expert: 스크립트 생성]
         ↓
[visual-designer-expert: 이미지 생성]
         ↓
[공개 URL 호스팅 (imgbb)]
         ↓
[sns-manager-expert: Instagram 업로드 (Meta Graph API)]
         ↓
[execution_log.json: 성과 기록]
         ↓
[분석 대시보드 업데이트]
```

---

### [중기-3] 성과 기반 피드백 루프

인스타그램 인사이트 API → 성과 데이터 수집
→ 어떤 콘텐츠 유형이 잘 됐는지 자동 분석
→ 다음 콘텐츠 계획에 반영

```python
# 주간 성과 자동 수집
def collect_weekly_insights(since_timestamp, until_timestamp):
    # Instagram Insights API 호출
    # likes, comments, saves, reach, impressions
    # → 가장 성과 좋은 포스트 유형, 시간대, 해시태그 분석
    # → content_strategy.json 업데이트
```

---

### [중기-4] 시뮬레이션 모드 vs 실제 실행 모드 분리

현재: 시뮬레이션과 실제 실행이 구분되지 않아서 execution_log가 부정확
목표: 명확한 모드 분리

```json
// session.json에 추가
{
  "execution_mode": "simulation" | "dry_run" | "production",
  "simulation": {
    "instagram_upload": false,    // 실제 업로드 안 함, 로그만 기록
    "image_generation": true,     // 실제 이미지 생성
    "web_search": true            // 실제 검색
  }
}
```

`simulation` 모드에서는 Instagram 업로드를 건너뛰고 로컬 파일로만 저장
`production` 모드에서만 실제 Instagram API 호출

---

### [중기-5] Rube MCP 연결 체계화

단기적으로 직접 API를 쓰더라도, 장기적으로는 Rube를 통한 500+ 앱 연동이 목표

Rube 연결 우선순위:
1. Gemini (이미지 생성 - 핵심)
2. Instagram (업로드 - 핵심)
3. Google Sheets (데이터 관리)
4. Canva (디자인 - 선택)

각 연결에 대해:
- rube.app/marketplace에서 OAuth 완료
- tool_inventory.json 재생성
- hired_agents.json의 feasible_tools 업데이트

---

## 6. 구현 우선순위 요약

```
즉시 (이번 주):
  ✅ [즉시-1] Gemini API 직접 연결 스크립트
  ✅ [즉시-2] Instagram 업로드 스크립트 (Meta Graph API)
  ✅ [즉시-3] expert-agent.md ACTION 검증 기준 추가
  ✅ [즉시-4] tool_inventory.json 재생성 (현재 실제 상태 반영)

단기 (2주 내):
  ☐ [단기-1] Expert 실행 단계 결정론적 분해
  ☐ [단기-2] 이미지→업로드 통합 파이프라인
  ☐ [단기-3] 에러 핸들링 + 폴백 체인
  ☐ [단기-5] HR 에이전트 → 실제 agent 파일 생성

중기 (1달 내):
  ☐ [중기-1] Claude Agent Teams 활용
  ☐ [중기-2] 콘텐츠 자동화 파이프라인 완성
  ☐ [중기-4] 시뮬레이션 vs 실제 실행 모드 분리
```

---

## 7. 최종 목표 상태

```
사용자가 "인스타 업로드해줘"라고 하면:
  1. content-writer-expert: 스크립트 + 캡션 생성 (30초)
  2. visual-designer-expert: Gemini로 이미지 생성 + PIL 합성 (2분)
  3. sns-manager-expert: imgbb 호스팅 + Instagram Graph API 2-step 업로드 (30초)
  4. execution_log에 post_id, URL, 성과 추적 시작

총 소요 시간: 약 3분
실패율: < 5% (폴백 체인 적용)
CEO 개입: Task Briefing에서만 (1회)
```

---

## 부록: 필요한 API 키/인증 목록

| 서비스 | 획득 방법 | 저장 위치 | 비용 |
|--------|---------|---------|------|
| Gemini API Key | [aistudio.google.com](https://aistudio.google.com) | 환경변수 `GEMINI_API_KEY` | 무료 (60 req/min) |
| Meta App (Instagram) | [developers.facebook.com](https://developers.facebook.com) | 환경변수 `INSTAGRAM_ACCESS_TOKEN`, `INSTAGRAM_USER_ID` | 무료 |
| imgbb API | [api.imgbb.com](https://api.imgbb.com) | 환경변수 `IMGBB_API_KEY` | 무료 (32MB/day) |
| Rube (선택) | [rube.app/marketplace](https://rube.app/marketplace) | `.mcp.json` + OAuth | 유료 또는 무료 플랜 |
