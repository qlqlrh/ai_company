# Tool-Aware Workflow v2 설계서

## 1. 개요

### 1.1 문제점 (현재 시스템)

- 에이전트들이 사용 가능한 Tool을 모른 채 작업 계획 수립
- CEO가 기대하는 Tool(Figma, Canva, 웹검색 등)과 실제 사용 가능한 Tool 불일치
- Tool 없이 작업 진행 → 기대와 다른 결과물

### 1.2 해결 방향

- **"뭘 할지" → "어떻게 할지" → "누가 할지"** 순서로 변경
- CEO가 각 작업에 사용할 Tool 직접 지정
- Tool Agent가 사용 가능 여부 검증 및 대안 제시
- Tool-aware 에이전트 고용 및 할당

---

## 2. 새로운 워크플로우

### 2.1 전체 흐름도

```
┌─────────────────────────────────────────────────────────────────┐
│                        Phase 1: 계획 수립                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ① CEO 목표 입력                                                │
│       ↓                                                         │
│  ② RM: 백로그 생성 (프로젝트/태스크 목록)                   ~   ~  │
│       ↓                                                         │
│  ③ CEO: 각 태스크에 사용할 Tool 지정                            │
│       ↓                                                         │
│  ④ Tool Agent: Tool 검증 및 대안 제시                           │
│       ↓                                                         │
│  ⑤ HR: 에이전트 고용 + Tool 할당                                │
│       ↓                                                         │
│  ⑥ RM: 에이전트에 태스크 할당                                   │
│                                                                 │
├─────────────────────────────────────────────────────────────────┤
│                        Phase 2: 실행                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ⑦ CEO: 실행할 백로그(프로젝트) 선택                            │
│       ↓                                                         │
│  ⑧ Expert들: 할당된 Tool로 태스크 수행                          │
│       ↓                                                         │
│  ⑨ 결과 보고 → CEO 검토                                         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 2.2 단계별 상세

#### Phase 1-①: CEO 목표 입력

```python
CEORequest:
    goal: str                    # "쿠팡 입점 및 월 매출 1000만원"
    constraints: dict            # 예산, 기한 등
    context: str                 # 추가 배경 정보
```

#### Phase 1-②: RM 백로그 생성

- CEO 목표만 보고 필요한 프로젝트/태스크 분해
- 아직 Tool, 에이전트 할당 없음 (순수 백로그)

```python
BacklogItem:
    id: str
    type: "project" | "task"
    name: str
    description: str
    parent_id: Optional[str]     # task의 경우 프로젝트 ID
    dependencies: list[str]
    suggested_tools: list[str]   # RM이 제안하는 Tool 카테고리
    status: "draft"
```

**예시 출력:**

```
프로젝트 1: 쿠팡 입점 준비
  ├─ Task 1.1: 시장 트렌드 조사 [suggested: web_search]
  ├─ Task 1.2: 경쟁사 분석 [suggested: web_search, spreadsheet]
  └─ Task 1.3: 제품 카테고리 선정 [suggested: analysis]

프로젝트 2: 제품 소싱
  ├─ Task 2.1: 1688/타오바오 제품 검색 [suggested: web_search]
  └─ Task 2.2: 공급업체 리스트업 [suggested: spreadsheet]

프로젝트 3: 상품 등록
  ├─ Task 3.1: 제품 상세 페이지 디자인 [suggested: design_tool]
  ├─ Task 3.2: 상품 사진 편집 [suggested: image_editor]
  └─ Task 3.3: 쿠팡 상품 등록 [suggested: coupang_api]
```

#### Phase 1-③: CEO Tool 지정

CEO가 백로그를 보고 각 태스크에 사용할 Tool 선택/입력

```python
# CEO 인터럽트
ToolSelectionInterrupt:
    type: "TOOL_SELECTION"
    backlog: list[BacklogItem]
    available_tools: list[ToolInfo]  # 현재 연결 가능한 Tool 목록

# CEO 응답
ToolSelectionResponse:
    task_tool_mapping: dict[task_id, list[str]]
    # 예: {
    #   "task_1.1": ["tavily_search"],
    #   "task_3.1": ["figma", "canva"],  # CEO가 원하는 Tool
    # }
```

#### Phase 1-④: Tool Agent 검증

Tool Agent가 CEO 선택을 검증하고 피드백

```python
ToolValidationResult:
    task_id: str
    requested_tools: list[str]
    status: "available" | "unavailable" | "partially_available"
    available_tools: list[str]      # 사용 가능한 것
    unavailable_tools: list[str]    # 사용 불가능한 것
    alternatives: list[ToolAlternative]  # 대안 제시
    message: str

ToolAlternative:
    original_tool: str
    suggested_tool: str
    reason: str
    capability_match: float  # 0.0 ~ 1.0
```

**예시 응답:**

```
Task 3.1 (제품 상세 페이지 디자인):
  - 요청: figma, canva
  - figma: ✅ 사용 가능 (MCP 연결됨)
  - canva: ❌ 현재 미지원
    → 대안 제시:
      1. Figma (capability_match: 0.95)
      2. 미리캔버스 MCP (capability_match: 0.85)
      3. HTML/CSS 직접 생성 (capability_match: 0.60)

  CEO 선택 필요: [Figma로 진행] [미리캔버스 연결] [다른 Tool 입력]
```

#### Phase 1-⑤: HR 에이전트 고용 + Tool 할당

- 확정된 백로그 + Tool 목록을 보고 필요한 에이전트 고용
- 각 에이전트에게 사용할 Tool 할당

```python
AgentDefinition:
    agent_id: str
    role_name: str
    description: str
    specialties: list[str]
    assigned_tools: list[str]      # 이 에이전트가 사용할 Tool
    tool_proficiency: dict[str, float]  # Tool별 숙련도
```

**예시:**

```
고용된 에이전트:
  1. 트렌드 리서처
     - Tools: [tavily_search, perplexity]

  2. 디자인 전문가
     - Tools: [figma, image_editor]

  3. 이커머스 전문가
     - Tools: [coupang_api, spreadsheet]
```

#### Phase 1-⑥: RM 태스크 할당

에이전트 목록을 보고 각 태스크에 적합한 에이전트 할당

```python
Task:
    # 기존 필드
    task_id: str
    name: str
    description: str

    # 할당 정보
    assigned_to: str              # 에이전트 ID
    assigned_tools: list[str]     # 이 태스크에서 사용할 Tool
    status: "ready"               # draft → ready
```

#### Phase 2-⑦: CEO 백로그 선택

CEO가 실행할 프로젝트 선택

```python
ExecutionRequest:
    project_id: str
    priority: str
    execution_mode: "sequential" | "parallel"
```

#### Phase 2-⑧~⑨: Expert 실행 및 보고

기존 흐름과 동일하나, 이제 Tool이 확정된 상태로 실행

---

## 3. Tool Agent 설계

### 3.1 역할

- 사용 가능한 Tool 목록 관리
- CEO 요청 Tool의 가용성 검증
- 대안 Tool 제시
- Tool 연결 상태 모니터링

### 3.2 핵심 기능

```python
class ToolAgent(BaseAgent):
    """Tool 검증 및 관리 전담 에이전트"""

    def __init__(self):
        self.tool_registry: ToolRegistry
        self.capability_mapping: dict  # 기능 → Tool 매핑

    def validate_tool_request(
        self,
        task_id: str,
        requested_tools: list[str]
    ) -> ToolValidationResult:
        """CEO가 요청한 Tool 검증"""
        pass

    def suggest_alternatives(
        self,
        unavailable_tool: str,
        required_capabilities: list[str]
    ) -> list[ToolAlternative]:
        """대안 Tool 제시"""
        pass

    def get_available_tools(
        self,
        category: Optional[str] = None
    ) -> list[ToolInfo]:
        """사용 가능한 Tool 목록 조회"""
        pass

    def check_tool_connection(
        self,
        tool_id: str
    ) -> ConnectionStatus:
        """Tool 연결 상태 확인"""
        pass
```

### 3.3 Tool 소스 타입

Tool Agent는 여러 소스에서 Tool을 탐색하고 추천할 수 있습니다:

```python
class ToolSourceType(Enum):
    MCP_SERVER = "mcp_server"       # Figma, GitHub, Slack, Rube 등
    SKILL = "skill"                  # frontend-design, commit 등
    BUILTIN = "builtin"              # WebSearch, WebFetch, Bash 등
    CUSTOM_API = "custom_api"        # Coupang API, Naver API 등
```

#### Tool 소스별 특성


| 소스             | 탐색 방법                     | 예시                           |
| -------------- | ------------------------- | ---------------------------- |
| **MCP Server** | MCP 설정 파일 스캔              | Figma, Rube, GitHub          |
| **Skills**     | `.claude/skills/` 디렉토리 스캔 | frontend-design, mcp-builder |
| **Built-in**   | 시스템 내장 목록                 | WebSearch, WebFetch          |
| **Custom API** | 수동 등록                     | Coupang API                  |


#### Skill 자동 탐색

```python
class SkillDiscovery:
    """Claude Code Skills 자동 탐색"""

    def scan_local_skills(self) -> list[SkillInfo]:
        """로컬 .claude/skills/ 디렉토리 스캔"""
        skills_dir = Path(".claude/skills")
        skills = []

        for skill_dir in skills_dir.iterdir():
            skill_md = skill_dir / "SKILL.md"
            if skill_md.exists():
                metadata = self._parse_skill_metadata(skill_md)
                skills.append(SkillInfo(
                    name=metadata["name"],
                    description=metadata["description"],
                    source_type=ToolSourceType.SKILL,
                    capabilities=self._extract_capabilities(metadata),
                    invoke_method=f"/{metadata['name']}"
                ))

        return skills

    def _extract_capabilities(self, metadata: dict) -> list[str]:
        """스킬 설명에서 capabilities 추출"""
        # LLM을 사용하여 description에서 capabilities 추론
        pass

class SkillInfo:
    name: str                    # "frontend-design"
    description: str             # "Create distinctive, production-grade..."
    source_type: ToolSourceType  # SKILL
    capabilities: list[str]      # ["web_design", "ui_development", "react", "html_css"]
    invoke_method: str           # "/frontend-design"
```

#### 예시: frontend-design 스킬 등록

```python
{
    "tool_id": "skill_frontend_design",
    "name": "Frontend Design Skill",
    "source_type": "skill",
    "description": "Create distinctive, production-grade frontend interfaces",
    "capabilities": [
        "web_design",
        "ui_development",
        "react_components",
        "html_css",
        "landing_page",
        "dashboard"
    ],
    "invoke_method": "/frontend-design",
    "status": "available",
    "recommended_for": [
        "제품 상세 페이지 제작",
        "랜딩 페이지 디자인",
        "웹 컴포넌트 개발",
        "UI 스타일링"
    ]
}
```

### 3.4 Tool Capability Mapping

```python
TOOL_CAPABILITIES = {
    # 웹 검색
    "tavily_search": {
        "category": "web_search",
        "capabilities": ["search", "news", "research"],
        "status": "available",
        "type": "mcp"
    },
    "perplexity": {
        "category": "web_search",
        "capabilities": ["search", "analysis", "summary"],
        "status": "available",
        "type": "api"
    },

    # 디자인
    "figma": {
        "category": "design",
        "capabilities": ["ui_design", "prototyping", "collaboration"],
        "status": "available",
        "type": "mcp"
    },
    "canva": {
        "category": "design",
        "capabilities": ["graphic_design", "templates", "social_media"],
        "status": "not_supported",
        "type": "api"
    },
    "miricanvas": {
        "category": "design",
        "capabilities": ["graphic_design", "templates", "korean_templates"],
        "status": "available",
        "type": "mcp"
    },

    # 스프레드시트
    "google_sheets": {
        "category": "spreadsheet",
        "capabilities": ["data_entry", "analysis", "collaboration"],
        "status": "available",
        "type": "mcp"
    },

    # 이커머스
    "coupang_api": {
        "category": "ecommerce",
        "capabilities": ["product_listing", "order_management"],
        "status": "requires_auth",
        "type": "api"
    }
}

# 카테고리 → Tool 매핑 (대안 제시용)
CATEGORY_TOOLS = {
    "web_search": ["tavily_search", "perplexity", "serper", "exa"],
    "design": ["figma", "miricanvas", "canva"],
    "spreadsheet": ["google_sheets", "excel", "notion"],
    "ecommerce": ["coupang_api", "naver_api", "cafe24_api"],
    "image_editor": ["remove_bg", "clipdrop", "photoshop_api"],
}
```

### 3.5 External Marketplace Discovery

Tool Agent는 로컬에 없는 Tool을 외부 마켓플레이스에서 검색하여 설치를 추천할 수 있습니다.

#### 지원하는 외부 마켓플레이스

```python
EXTERNAL_MARKETPLACES = {
    # MCP 서버 레지스트리
    "mcp_so": {
        "name": "MCP.so",
        "url": "https://mcp.so/",
        "type": "mcp_server",
        "api": "https://mcp.so/api/search",  # 검색 API
        "count": "17,000+ servers"
    },
    "mcp_market": {
        "name": "MCP Market",
        "url": "https://mcpmarket.com/",
        "type": "mcp_server",
        "api": "https://mcpmarket.com/api/search"
    },
    "lobehub_mcp": {
        "name": "LobeHub MCP",
        "url": "https://lobehub.com/mcp",
        "type": "mcp_server"
    },

    # Skills 저장소
    "anthropic_skills": {
        "name": "Anthropic Official Skills",
        "url": "https://github.com/anthropics/skills",
        "type": "skill",
        "method": "github_api"  # GitHub API로 디렉토리 목록 조회
    },
    "claude_market": {
        "name": "Claude Market",
        "url": "https://github.com/claude-market/marketplace",
        "type": "mixed",  # skills + mcp + plugins
        "method": "github_api"
    }
}
```

#### 계층적 검색 전략

마켓플레이스에 등록되지 않은 Tool도 많으므로, 3단계 계층적 검색을 수행합니다:

```
┌─────────────────────────────────────────────────────────────────┐
│                    Tool 검색 전략 (3단계)                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  [1차] 마켓플레이스 API 검색                                    │
│    - MCP.so API, MCP Market API                                │
│    - GitHub API (anthropics/skills, claude-market)             │
│    - 장점: 구조화된 데이터, 빠름, 신뢰성 높음                   │
│    - 단점: 등록된 Tool만 검색 가능                              │
│                           ↓ 못 찾으면                           │
│                                                                 │
│  [2차] 웹 검색 (WebSearch/Tavily)                               │
│    - GitHub 검색: "{keyword} mcp server github"                │
│    - npm 검색: "npm mcp server {keyword}"                      │
│    - PyPI 검색: "pypi mcp {keyword}"                           │
│    - 블로그/문서: "claude code {keyword} integration"          │
│    - 장점: 범위 넓음, 최신 정보                                 │
│    - 단점: 노이즈 있음, 검증 필요                               │
│                           ↓ 못 찾으면                           │
│                                                                 │
│  [3차] LLM 지식 기반 추천                                       │
│    - Claude의 학습 데이터 기반 Tool 추천                        │
│    - "이런 Tool이 존재할 수 있음" 형태로 제안                   │
│    - 장점: 폭넓은 지식                                          │
│    - 단점: 최신 정보 아닐 수 있음, 검증 필수                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

#### 검색 전략별 구현

```python
class ToolSearchStrategy(Enum):
    MARKETPLACE_API = "marketplace_api"   # 1차: 마켓플레이스
    WEB_SEARCH = "web_search"             # 2차: 웹 검색
    LLM_KNOWLEDGE = "llm_knowledge"       # 3차: LLM 지식

class HierarchicalToolSearch:
    """계층적 Tool 검색"""

    async def search(self, query: str, task: Task) -> ToolSearchResult:
        """3단계 계층적 검색 수행"""

        all_results = []

        # 1차: 마켓플레이스 API 검색
        marketplace_results = await self._search_marketplaces(query)
        all_results.extend(marketplace_results)

        if self._has_good_matches(marketplace_results):
            return ToolSearchResult(
                results=all_results,
                search_strategy_used=[ToolSearchStrategy.MARKETPLACE_API],
                confidence="high"
            )

        # 2차: 웹 검색 (마켓플레이스에서 못 찾으면)
        web_results = await self._search_web(query, task)
        all_results.extend(web_results)

        if self._has_good_matches(web_results):
            return ToolSearchResult(
                results=all_results,
                search_strategy_used=[
                    ToolSearchStrategy.MARKETPLACE_API,
                    ToolSearchStrategy.WEB_SEARCH
                ],
                confidence="medium"
            )

        # 3차: LLM 지식 기반 (웹에서도 못 찾으면)
        llm_suggestions = await self._get_llm_suggestions(query, task)
        all_results.extend(llm_suggestions)

        return ToolSearchResult(
            results=all_results,
            search_strategy_used=[
                ToolSearchStrategy.MARKETPLACE_API,
                ToolSearchStrategy.WEB_SEARCH,
                ToolSearchStrategy.LLM_KNOWLEDGE
            ],
            confidence="low"  # 검증 필요 명시
        )

    async def _search_web(self, query: str, task: Task) -> list[ExternalToolInfo]:
        """웹 검색으로 Tool 찾기"""

        search_queries = [
            f"{query} mcp server github",
            f"{query} claude code integration",
            f"npm mcp server {query}",
            f"{query} api automation tool",
        ]

        results = []
        for search_query in search_queries:
            # WebSearch 도구 사용
            web_results = await self.web_search.search(search_query)
            parsed = self._parse_web_results(web_results)
            results.extend(parsed)

        return self._deduplicate_and_rank(results)

    async def _get_llm_suggestions(self, query: str, task: Task) -> list[ExternalToolInfo]:
        """LLM 지식 기반 Tool 추천"""

        prompt = f"""
        Task: {task.description}
        필요 기능: {query}

        이 Task를 수행할 수 있는 Tool, MCP 서버, 또는 API가 있다면 알려주세요.
        - Tool 이름
        - 공식 웹사이트 또는 GitHub URL (알고 있다면)
        - 주요 기능

        주의: 확실하지 않은 정보는 "검증 필요"라고 표시해주세요.
        """

        response = await self.llm.invoke(prompt)
        return self._parse_llm_suggestions(response, verified=False)
```

#### 웹 검색 쿼리 템플릿

```python
WEB_SEARCH_TEMPLATES = {
    "mcp_server": [
        "{keyword} mcp server",
        "{keyword} mcp server github",
        "{keyword} model context protocol",
        "claude {keyword} integration mcp",
    ],
    "npm_package": [
        "npm {keyword} mcp",
        "npmjs {keyword} claude",
    ],
    "github_repo": [
        "github {keyword} mcp server",
        "github {keyword} claude integration",
        "github {keyword} api wrapper",
    ],
    "general_tool": [
        "{keyword} automation tool api",
        "{keyword} no-code integration",
        "zapier {keyword} alternative api",
    ]
}
```

#### 검색 결과 신뢰도 표시

```python
class SearchConfidence(Enum):
    HIGH = "high"       # 마켓플레이스에서 발견 (검증됨)
    MEDIUM = "medium"   # 웹 검색에서 발견 (GitHub 등)
    LOW = "low"         # LLM 추천 (검증 필요)

class ExternalToolInfo:
    # ... 기존 필드들
    confidence: SearchConfidence
    source_strategy: ToolSearchStrategy
    verification_needed: bool
    verification_url: Optional[str]  # 검증할 수 있는 URL
```

#### 실제 검색 예시

```
[Tool Agent] Task 분석: "네이버 스마트스토어 상품 자동 등록"

[1차] 마켓플레이스 검색
  - MCP.so: "naver" 검색 → 결과 없음
  - claude-market: "smartstore" 검색 → 결과 없음

[2차] 웹 검색 수행
  검색어: "naver smartstore mcp server github"
  검색어: "naver commerce api integration"
  검색어: "스마트스토어 자동화 api"

  발견:
  ✅ GitHub: naver-commerce-api (Python wrapper)
     URL: github.com/example/naver-commerce-api
     신뢰도: MEDIUM (GitHub에서 발견, 별 50개)

  ✅ npm: @naver/commerce-sdk
     URL: npmjs.com/package/@naver/commerce-sdk
     신뢰도: MEDIUM (공식 SDK로 보임)

[3차] LLM 지식 (참고용)
  💡 "네이버 커머스 API가 있으며, OAuth 인증 필요.
      공식 문서: developers.naver.com/products/commerce
      (검증 필요)"

[CEO에게 제시]
┌─────────────────────────────────────────────────────────────┐
│ 🔍 검색 결과 (마켓플레이스에 없어 웹 검색 수행)              │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ 1. naver-commerce-api (GitHub) ⭐ 신뢰도: 중간             │
│    - Python 기반 네이버 커머스 API 래퍼                    │
│    - GitHub 별: 50개                                       │
│    - 설치: pip install naver-commerce-api                  │
│    → MCP 서버로 래핑 필요할 수 있음                        │
│                                                             │
│ 2. @naver/commerce-sdk (npm) ⭐ 신뢰도: 중간               │
│    - 네이버 공식 SDK로 추정                                │
│    - 설치: npm install @naver/commerce-sdk                 │
│                                                             │
│ 3. 네이버 커머스 API 직접 연동 💡 LLM 추천                 │
│    - 공식 API 문서: developers.naver.com                   │
│    - (검증 필요 - 직접 확인 권장)                          │
│                                                             │
│ [1번 설치] [2번 설치] [직접 URL 입력] [건너뛰기]           │
└─────────────────────────────────────────────────────────────┘
```

#### 외부 검색 흐름

```python
class MarketplaceDiscovery:
    """외부 마켓플레이스에서 Tool 검색"""

    async def search_external(
        self,
        query: str,
        tool_types: list[ToolSourceType] = None
    ) -> list[ExternalToolInfo]:
        """
        외부 마켓플레이스 검색

        Args:
            query: 검색어 (예: "gif creator", "3d rendering")
            tool_types: 검색할 Tool 타입 필터

        Returns:
            발견된 외부 Tool 목록
        """
        results = []

        # 1. MCP 레지스트리 검색
        if not tool_types or ToolSourceType.MCP_SERVER in tool_types:
            mcp_results = await self._search_mcp_registries(query)
            results.extend(mcp_results)

        # 2. Skills 저장소 검색
        if not tool_types or ToolSourceType.SKILL in tool_types:
            skill_results = await self._search_skill_repos(query)
            results.extend(skill_results)

        return results

    async def _search_mcp_registries(self, query: str) -> list[ExternalToolInfo]:
        """MCP.so, MCP Market 등에서 검색"""
        # WebSearch 또는 API 호출
        pass

    async def _search_skill_repos(self, query: str) -> list[ExternalToolInfo]:
        """anthropics/skills, claude-market 등에서 검색"""
        # GitHub API로 디렉토리/README 검색
        pass

class ExternalToolInfo:
    """외부에서 발견된 Tool 정보"""
    name: str                    # "slack-gif-creator"
    description: str             # "슬랙용 GIF 생성 스킬"
    source_type: ToolSourceType  # SKILL
    marketplace: str             # "anthropic_skills"
    repository_url: str          # "https://github.com/anthropics/skills"
    install_path: str            # "skills/slack-gif-creator"
    install_method: InstallMethod
    compatibility_score: float   # Task와의 적합도

class InstallMethod(Enum):
    GIT_CLONE = "git_clone"           # Skills: git clone → .claude/skills/
    MCP_CONFIG = "mcp_config"         # MCP: mcp.json에 설정 추가
    NPM_INSTALL = "npm_install"       # npm 기반 MCP 서버
    PIP_INSTALL = "pip_install"       # Python 기반 MCP 서버
```

#### 자동 설치 기능

```python
class ToolInstaller:
    """외부 Tool 자동 설치"""

    async def install_skill(self, tool: ExternalToolInfo) -> InstallResult:
        """Skill 설치 (git clone)"""
        # 1. 저장소 클론
        repo_url = tool.repository_url
        skill_path = f".claude/skills/{tool.name}"

        # 2. sparse checkout으로 특정 디렉토리만
        commands = [
            f"git clone --depth 1 --filter=blob:none --sparse {repo_url} /tmp/skills_temp",
            f"cd /tmp/skills_temp && git sparse-checkout set {tool.install_path}",
            f"cp -r /tmp/skills_temp/{tool.install_path} {skill_path}",
            f"rm -rf /tmp/skills_temp"
        ]

        return InstallResult(success=True, path=skill_path)

    async def install_mcp_server(self, tool: ExternalToolInfo) -> InstallResult:
        """MCP 서버 설치 (설정 추가)"""
        # 1. mcp.json 또는 .claude/settings.json 수정
        # 2. npm install 또는 pip install (필요시)
        pass
```

#### 실제 사용 예시

```
[Tool Agent] Task 분석: "3D 제품 렌더링 이미지 생성"

[1단계] 로컬 Tool 탐색
  - 로컬 Skills: 해당 기능 없음
  - 설정된 MCP: 해당 기능 없음
  - Built-in: 해당 기능 없음

[2단계] 외부 마켓플레이스 검색
  검색어: "3d rendering", "blender", "product visualization"

  MCP.so 검색 결과:
    ✅ blender-mcp (score: 0.92)
       - Blender 3D 소프트웨어 연동
       - 설치: npm install blender-mcp

  anthropics/skills 검색 결과:
    ❌ 관련 스킬 없음

  claude-market 검색 결과:
    ✅ product-3d-renderer (score: 0.85)
       - AI 기반 제품 3D 렌더링
       - 설치: git clone

[3단계] CEO에게 추천
  ┌──────────────────────────────────────────────────────┐
  │ 💡 이 Task를 수행하려면 추가 Tool 설치가 필요합니다  │
  ├──────────────────────────────────────────────────────┤
  │                                                      │
  │ 1. blender-mcp (MCP Server) ⭐ 추천                  │
  │    출처: mcp.so                                      │
  │    설명: Blender 3D 소프트웨어와 연동               │
  │    요구사항: Blender 설치 필요                       │
  │    설치 명령: npm install blender-mcp               │
  │                                                      │
  │ 2. product-3d-renderer (Plugin)                     │
  │    출처: claude-market                              │
  │    설명: AI 기반 제품 이미지 3D 변환                │
  │    요구사항: 없음                                    │
  │    설치: git clone                                   │
  │                                                      │
  │ [1번 설치] [2번 설치] [직접 Tool 입력] [건너뛰기]    │
  └──────────────────────────────────────────────────────┘

[CEO] 2번 설치해줘

[Tool Agent]
  ✅ product-3d-renderer 설치 완료
  위치: .claude/plugins/product-3d-renderer/

  이제 해당 Tool을 사용하여 Task를 수행할 수 있습니다.
```

#### Anthropic 공식 Skills 목록 (자동 인덱싱)

```python
# GitHub API로 자동 수집 가능
ANTHROPIC_SKILLS_INDEX = {
    "algorithmic-art": {
        "description": "알고리즘 기반 아트 생성",
        "capabilities": ["generative_art", "creative_coding"]
    },
    "brand-guidelines": {
        "description": "브랜드 가이드라인 생성",
        "capabilities": ["branding", "design_system"]
    },
    "canvas-design": {
        "description": "캔버스 기반 디자인",
        "capabilities": ["visual_design", "graphics"]
    },
    "frontend-design": {
        "description": "프론트엔드 UI 코드 생성",
        "capabilities": ["web_design", "ui_development", "react", "html_css"]
    },
    "mcp-builder": {
        "description": "MCP 서버 생성 가이드",
        "capabilities": ["mcp_development", "tool_creation"]
    },
    "slack-gif-creator": {
        "description": "슬랙용 GIF 생성",
        "capabilities": ["gif_creation", "slack_integration"]
    },
    "web-artifacts-builder": {
        "description": "웹 아티팩트 빌더",
        "capabilities": ["web_components", "interactive_demos"]
    },
    # ... 더 많은 스킬
}
```

### 3.6 Smart Tool Recommendation

Tool Agent가 Task를 분석하여 최적의 Tool 조합을 추천합니다:

```python
class ToolRecommendation:
    """Task에 대한 Tool 추천 결과"""
    task_id: str
    task_description: str

    # 추천 Tool 목록 (우선순위 순)
    recommended_tools: list[RecommendedTool]

    # 추천 이유
    reasoning: str

    # 조합 사용 제안
    combination_suggestion: Optional[str]

class RecommendedTool:
    tool_id: str
    name: str
    source_type: ToolSourceType   # MCP, SKILL, BUILTIN, CUSTOM_API
    match_score: float            # 0.0 ~ 1.0
    capabilities_matched: list[str]
    recommendation_reason: str
    invoke_method: str            # MCP tool name 또는 /skill-name
```

#### 추천 로직 예시

```python
class ToolAgent:
    def recommend_tools(self, task: Task) -> ToolRecommendation:
        """Task에 적합한 Tool 추천"""

        # 1. Task 분석하여 필요 capabilities 추출
        required_caps = self._analyze_task_requirements(task)
        # 예: ["web_design", "ui_code", "responsive"]

        # 2. 모든 소스에서 Tool 탐색
        all_tools = (
            self.mcp_registry.get_all() +
            self.skill_discovery.scan_local_skills() +
            self.builtin_tools.get_all() +
            self.custom_apis.get_all()
        )

        # 3. Capability 매칭 및 점수 계산
        matches = []
        for tool in all_tools:
            score = self._calculate_match_score(tool, required_caps)
            if score > 0.3:  # threshold
                matches.append((tool, score))

        # 4. 점수순 정렬 및 조합 제안
        matches.sort(key=lambda x: x[1], reverse=True)

        return ToolRecommendation(
            task_id=task.task_id,
            recommended_tools=[...],
            combination_suggestion=self._suggest_combination(matches)
        )
```

#### 실제 추천 예시

**Task:** "쿠팡 제품 상세 페이지 제작"

```
[Tool Agent 분석]
필요 capabilities: web_design, product_page, ecommerce_ui, responsive

[추천 결과]
┌─────────────────────────────────────────────────────────────────┐
│ 순위 │ Tool                    │ 타입   │ 점수 │ 이유           │
├──────┼─────────────────────────┼────────┼──────┼────────────────┤
│  1   │ frontend-design         │ SKILL  │ 0.95 │ 웹 UI 코드 생성 │
│  2   │ Figma MCP               │ MCP    │ 0.85 │ 디자인 시안    │
│  3   │ miricanvas MCP          │ MCP    │ 0.75 │ 그래픽 에셋    │
│  4   │ WebSearch               │ BUILTIN│ 0.60 │ 레퍼런스 조사  │
└─────────────────────────────────────────────────────────────────┘

💡 조합 제안:
   1단계: WebSearch로 쿠팡 베스트셀러 페이지 레퍼런스 조사
   2단계: Figma MCP로 디자인 시안 제작 (또는 스킵 가능)
   3단계: frontend-design 스킬로 실제 HTML/CSS/JS 코드 생성

   권장: 빠른 결과물이 필요하면 frontend-design 스킬 단독 사용 추천
```

**Task:** "경쟁사 가격 모니터링 시스템 구축"

```
[Tool Agent 분석]
필요 capabilities: web_scraping, data_collection, scheduling, spreadsheet

[추천 결과]
┌─────────────────────────────────────────────────────────────────┐
│ 순위 │ Tool                    │ 타입   │ 점수 │ 이유           │
├──────┼─────────────────────────┼────────┼──────┼────────────────┤
│  1   │ Rube MCP                │ MCP    │ 0.90 │ 워크플로우 자동화│
│  2   │ Google Sheets MCP       │ MCP    │ 0.85 │ 데이터 저장    │
│  3   │ WebFetch                │ BUILTIN│ 0.70 │ 웹 데이터 수집 │
│  4   │ Bash                    │ BUILTIN│ 0.50 │ 스크립트 실행  │
└─────────────────────────────────────────────────────────────────┘

💡 조합 제안:
   Rube MCP로 자동화 레시피 생성 → Google Sheets에 결과 저장
```

#### MCP vs Skill 선택 가이드

```python
TOOL_SELECTION_GUIDE = {
    "디자인 시안/목업 필요": {
        "primary": "figma_mcp",
        "reason": "시각적 디자인 편집 필요"
    },
    "실제 동작하는 코드 필요": {
        "primary": "skill_frontend_design",
        "reason": "프로덕션급 코드 생성"
    },
    "외부 서비스 연동": {
        "primary": "mcp_server",  # 해당 서비스의 MCP
        "reason": "API 직접 호출"
    },
    "데이터 조사/검색": {
        "primary": "builtin_websearch",
        "reason": "빠른 정보 수집"
    },
    "복잡한 워크플로우 자동화": {
        "primary": "rube_mcp",
        "reason": "멀티 앱 오케스트레이션"
    }
}
```

### 3.6 Tool 상태 정의

```python
class ToolStatus(Enum):
    AVAILABLE = "available"           # 바로 사용 가능
    REQUIRES_AUTH = "requires_auth"   # 인증 필요
    REQUIRES_SETUP = "requires_setup" # 설정 필요
    NOT_SUPPORTED = "not_supported"   # 현재 미지원
    ERROR = "error"                   # 오류 상태
    DISABLED = "disabled"             # 비활성화됨
```

---

## 4. 수정된 그래프 구조

### 4.1 새로운 Phase 구조

```python
class CompanyPhase(Enum):
    # Phase 1: 계획 수립
    INITIALIZATION = "initialization"
    BACKLOG_CREATION = "backlog_creation"      # RM 백로그 생성
    TOOL_SELECTION = "tool_selection"          # CEO Tool 선택
    TOOL_VALIDATION = "tool_validation"        # Tool Agent 검증
    AGENT_HIRING = "agent_hiring"              # HR 에이전트 고용
    TASK_ASSIGNMENT = "task_assignment"        # RM 태스크 할당
    PLANNING_COMPLETE = "planning_complete"

    # Phase 2: 실행
    AWAITING_EXECUTION = "awaiting_execution"  # CEO 백로그 선택 대기
    EXECUTING = "executing"                    # Expert 실행 중
    COMPLETED = "completed"
```

### 4.2 그래프 노드

```python
NODES = [
    "initialize",        # CEO 요청 확인
    "rm_backlog",        # RM: 백로그 생성
    "ceo_tool_select",   # CEO: Tool 선택 (interrupt)
    "tool_validate",     # Tool Agent: 검증
    "hr_hire",           # HR: 에이전트 고용
    "rm_assign",         # RM: 태스크 할당
    "ceo_execute",       # CEO: 실행할 프로젝트 선택 (interrupt)
    "executor",          # Expert: 태스크 실행
    "completion",        # 완료 보고
]
```

### 4.3 그래프 흐름

```
           ┌──────────────┐
           │  initialize  │
           └──────┬───────┘
                  ↓
           ┌──────────────┐
           │  rm_backlog  │ ← RM이 백로그 생성
           └──────┬───────┘
                  ↓
        ┌─────────────────────┐
        │  ceo_tool_select    │ ← CEO Tool 선택 (INTERRUPT)
        └─────────┬───────────┘
                  ↓
           ┌──────────────┐
           │ tool_validate│ ← Tool Agent 검증
           └──────┬───────┘
                  │
        ┌─────────┴─────────┐
        ↓                   ↓
   [모두 valid]      [일부 invalid]
        ↓                   ↓
        │         ┌─────────────────────┐
        │         │  ceo_tool_select    │ ← 재선택 요청
        │         └─────────────────────┘
        ↓
           ┌──────────────┐
           │   hr_hire    │ ← HR 에이전트 고용
           └──────┬───────┘
                  ↓
           ┌──────────────┐
           │  rm_assign   │ ← RM 태스크 할당
           └──────┬───────┘
                  ↓
        ┌─────────────────────┐
        │    ceo_execute      │ ← CEO 프로젝트 선택 (INTERRUPT)
        └─────────┬───────────┘
                  ↓
           ┌──────────────┐
           │   executor   │ ← Expert 실행
           └──────┬───────┘
                  │
        ┌─────────┴─────────┐
        ↓                   ↓
   [태스크 남음]      [프로젝트 완료]
        ↓                   ↓
   ← executor 반복    ┌──────────────┐
                      │ ceo_execute  │ ← 다음 프로젝트 선택
                      └──────────────┘
                            or
                      ┌──────────────┐
                      │  completion  │ ← 전체 완료
                      └──────────────┘
```

---

## 5. 상태 스키마 변경

### 5.1 CompanyState 확장

```python
class CompanyState:
    # 기존 필드
    ceo_request: CEORequest
    agents: dict[str, AgentDefinition]
    projects: dict[str, Project]
    tasks: dict[str, Task]

    # 새로운 필드
    backlog: list[BacklogItem]                    # 백로그 (draft 상태)
    tool_selections: dict[str, list[str]]         # task_id → selected_tools
    tool_validations: dict[str, ToolValidationResult]  # 검증 결과
    available_tools: dict[str, ToolInfo]          # 사용 가능한 Tool 목록

    # Phase 관리
    current_phase: CompanyPhase
    planning_complete: bool
```

### 5.2 새로운 Interrupt 타입

```python
class InterruptType(Enum):
    # 기존
    INFO_REQUEST = "info_request"
    APPROVAL_REQUEST = "approval_request"
    TOOL_CONNECTION = "tool_connection"
    ERROR_REPORT = "error_report"
    PROGRESS_REPORT = "progress_report"

    # 새로 추가
    TOOL_SELECTION = "tool_selection"         # CEO Tool 선택 요청
    TOOL_RESELECTION = "tool_reselection"     # Tool 재선택 요청 (invalid 시)
    PROJECT_SELECTION = "project_selection"   # 실행할 프로젝트 선택
    BACKLOG_REVIEW = "backlog_review"         # 백로그 검토 요청
```

---

## 6. 기본 에이전트 구성

시스템 시작 시 기본 생성되는 에이전트:

```python
DEFAULT_AGENTS = [
    {
        "agent_id": "ceo-agent",
        "role_name": "CEO",
        "type": "system",
        "description": "회사 목표 설정 및 최종 의사결정"
    },
    {
        "agent_id": "hr-agent",
        "role_name": "HR Manager",
        "type": "system",
        "description": "에이전트 고용 및 Tool 할당"
    },
    {
        "agent_id": "rm-agent",
        "role_name": "Resource Manager",
        "type": "system",
        "description": "프로젝트/태스크 생성 및 에이전트 할당"
    },
    {
        "agent_id": "tool-agent",      # 새로 추가
        "role_name": "Tool Manager",
        "type": "system",
        "description": "Tool 가용성 검증 및 대안 제시"
    }
]
```

---

## 7. 구현 우선순위

### Phase 1: 핵심 구조 변경

1. Tool Agent 구현 (`src/agents/tool.py`)
2. 그래프 노드 순서 변경 (`src/graph/nodes.py`)
3. 새로운 Interrupt 타입 추가 (`src/schemas/interrupt.py`)
4. CompanyState 확장 (`src/context/state.py`)

### Phase 2: Tool 시스템 강화

1. Tool Capability Mapping 구현 (`src/tools/capabilities.py`)
2. Tool 대안 제시 로직 구현
3. Tool 상태 관리 강화

### Phase 3: UI/UX 개선

1. 백로그 뷰어 (프론트엔드)
2. Tool 선택 UI
3. 실행 프로젝트 선택 UI

---

## 8. 예상 사용자 경험

```
[CEO] 목표: 쿠팡 입점하고 월 매출 1000만원 달성

[RM] 백로그를 생성했습니다:
  📁 프로젝트 1: 쿠팡 입점 준비
    📋 Task 1.1: 시장 트렌드 조사
    📋 Task 1.2: 경쟁사 분석
    📋 Task 1.3: 제품 카테고리 선정
  📁 프로젝트 2: 제품 소싱
    📋 Task 2.1: 1688 제품 검색
    ...
  📁 프로젝트 3: 상품 등록
    📋 Task 3.1: 제품 상세 페이지 디자인
    ...

[System] 각 태스크에 사용할 Tool을 선택해주세요.

[CEO] Task 3.1에는 Figma 사용할게. Task 1.1은 웹검색.

[Tool Agent] Tool 검증 결과:
  ✅ Figma - 사용 가능
  ✅ 웹검색(Tavily) - 사용 가능
  ⚠️ Task 2.1 (1688 검색) - 전용 Tool 없음
    → 대안: 웹검색 + 수동 검색 조합 추천

[CEO] OK, 대안으로 진행

[HR] 에이전트를 고용했습니다:
  👤 트렌드 리서처 (Tools: tavily_search)
  👤 디자인 전문가 (Tools: figma)
  👤 이커머스 전문가 (Tools: google_sheets)

[RM] 태스크 할당 완료. 어떤 프로젝트부터 실행할까요?
  1. 쿠팡 입점 준비 (3 tasks)
  2. 제품 소싱 (2 tasks)
  3. 상품 등록 (3 tasks)

[CEO] 1번부터 시작

[Expert] Task 1.1 시작: 시장 트렌드 조사 (using Tavily)
...
```

---

## 9. 변경 요약


| 항목      | 기존               | 변경                                          |
| ------- | ---------------- | ------------------------------------------- |
| 순서      | HR → RM → Expert | RM(백로그) → Tool Agent → HR → RM(할당) → Expert |
| Tool 선택 | 시스템 자동           | CEO 직접 지정                                   |
| Tool 검증 | 실행 시점            | 계획 단계 (사전 검증)                               |
| 기본 에이전트 | CEO, HR, RM      | CEO, HR, RM, **Tool Agent**                 |
| 실행 방식   | 자동 진행            | CEO가 백로그 선택 후 실행                            |


