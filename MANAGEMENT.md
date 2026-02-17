Deep Agents의 핵심 3요소 예시를 만들어드리겠습니다.

## 1. System Prompt 예시

```python
from deepagents import create_deep_agent

# 전문적인 리서치 에이전트를 위한 상세 System Prompt
research_system_prompt = """
당신은 전문 리서치 및 기술 문서 작성 에이전트입니다.

## 작업 방식
1. 주어진 주제를 먼저 작은 서브토픽으로 분해합니다
2. 각 서브토픽에 대해 깊이 있는 조사를 수행합니다
3. 수집한 정보를 구조화된 마크다운 문서로 작성합니다
4. 최종 검토 후 보고서를 완성합니다

## 품질 기준
- 모든 주장은 출처를 명시해야 합니다
- 기술 용어는 처음 등장할 때 설명합니다
- 코드 예시는 실행 가능하고 잘 주석 처리되어야 합니다
- 최종 문서는 목차, 본문, 참고자료 순으로 구성합니다

## 파일 관리
- research/ 디렉토리에 raw 데이터를 저장합니다
- drafts/ 디렉토리에 초안을 작성합니다
- final/ 디렉토리에 최종 보고서를 저장합니다
"""

agent = create_deep_agent(
    model="anthropic:claude-sonnet-4-20250514",
    system_prompt=research_system_prompt,
    tools=[internet_search, read_file, write_file]
)
```

## 2. Planning (write_todos) 예시

```python
# 에이전트가 자동으로 write_todos 도구를 사용하여 계획 수립
# 사용자 요청: "FastAPI와 PostgreSQL로 RESTful API 만들기"

# 에이전트가 내부적으로 생성하는 TODO 리스트:
"""
TODOS:
[ ] 1. 프로젝트 구조 설계
    - 디렉토리 구조 생성 (app/, models/, routes/, tests/)
    - requirements.txt 작성
    
[ ] 2. 데이터베이스 설정
    - SQLAlchemy 모델 정의
    - 데이터베이스 연결 설정
    - Alembic 마이그레이션 설정
    
[ ] 3. API 엔드포인트 구현
    - CRUD operations for User model
    - Request/Response 스키마 정의
    - 에러 핸들링 추가
    
[ ] 4. 테스트 작성
    - pytest fixtures 설정
    - 각 엔드포인트에 대한 테스트 작성
    
[ ] 5. 문서화
    - README.md 작성
    - API 문서 자동 생성 확인
"""

# 작업 진행 중 TODO 업데이트
"""
TODOS:
[x] 1. 프로젝트 구조 설계 ✓
[x] 2. 데이터베이스 설정 ✓
[->] 3. API 엔드포인트 구현 (진행중)
    - [x] GET /users
    - [x] POST /users
    - [ ] PUT /users/{id}
    - [ ] DELETE /users/{id}
[ ] 4. 테스트 작성
[ ] 5. 문서화
"""
```

## 3. Context Management 예시

```python
from deepagents import create_deep_agent
from langchain_core.tools import tool

# 파일 시스템을 활용한 Context Management
agent = create_deep_agent(
    model="anthropic:claude-sonnet-4-20250514",
    system_prompt=research_system_prompt,
    # 기본적으로 ls, read_file, write_file, edit_file 도구 제공됨
)

# 에이전트가 작업하는 방식 예시:

# Step 1: 초기 리서치 데이터 저장 (컨텍스트 윈도우 오버플로우 방지)
"""
agent: write_file("research/langchain_overview.md", content)
agent: write_file("research/langgraph_features.md", content)
agent: write_file("research/deep_agents_architecture.md", content)
"""

# Step 2: 필요할 때만 특정 파일 읽기
"""
agent: read_file("research/langchain_overview.md")
agent: 이 내용을 바탕으로 초안 작성...
agent: write_file("drafts/section1.md", draft_content)
"""

# Step 3: 여러 파일을 조합하여 최종 문서 생성
"""
agent: ls("drafts/")  # 모든 초안 확인
agent: read_file("drafts/section1.md")
agent: read_file("drafts/section2.md")
agent: edit_file("drafts/section1.md", edits)  # 부분 수정
agent: write_file("final/complete_report.md", final_content)
"""
```

## 실전 통합 예시

```python
from deepagents import create_deep_agent
from langchain_anthropic import ChatAnthropic

# 한국 정부 조달 시스템 분석 에이전트
procurement_agent_prompt = """
당신은 한국 정부 조달 시스템 전문 분석 에이전트입니다.

## 작업 흐름
1. **계획 수립**: write_todos를 사용해 분석 작업을 단계별로 분해
2. **데이터 수집**: 나라장터, 조달청 API 등에서 데이터 수집
3. **파일 관리**: 
   - raw_data/ : 원본 데이터 저장
   - analysis/ : 분석 중간 결과
   - reports/ : 최종 보고서
4. **서브에이전트 활용**: 복잡한 데이터 분석은 analysis-agent에 위임

## 출력 형식
- 엑셀 데이터는 CSV로 변환하여 저장
- 분석 결과는 마크다운 테이블로 정리
- 최종 보고서는 한글로 작성
"""

# Subagent 정의
analysis_subagent = {
    "name": "analysis-agent",
    "description": "입찰 데이터 통계 분석 전문 에이전트",
    "prompt": """통계 분석 전문가입니다. 
    주어진 CSV 데이터를 분석하여:
    - 기술통계 (평균, 중앙값, 표준편차)
    - 시계열 트렌드 분석
    - 상관관계 분석
    결과를 markdown 테이블로 반환하세요.""",
    "tools": ["read_file", "write_file"]
}

agent = create_deep_agent(
    model="anthropic:claude-sonnet-4-20250514",
    system_prompt=procurement_agent_prompt,
    tools=[internet_search, read_file, write_file, edit_file],
    subagents=[analysis_subagent]
)

# 사용
result = agent.invoke({
    "messages": [{
        "role": "user", 
        "content": "2025년 AI 관련 정부 발주 현황을 분석하고 보고서 작성해줘"
    }]
})
```

이렇게 **System Prompt로 전반적인 행동 지침을 제공**하고, **Planning으로 작업을 분해 추적**하며, **Context Management로 대용량 데이터를 효율적으로 처리**하는 것이 Deep Agents의 핵심입니다. [datacamp](https://www.datacamp.com/tutorial/deep-agents)