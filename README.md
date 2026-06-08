# aah-rag-agent-only — AWS Native RAG Agent Sample

AAH Code Deploy 시연용 **단일 Agent 서비스** 샘플. AgentCore Runtime 위에
ReAct loop + Anthropic native `tool_use` 패턴으로 동작.

## 특징 (이전 보험 RAG 샘플과의 차이)

기존 샘플은 RDS / ElastiCache / OpenSearch / Databricks 5개 리소스를 한 번에
프로비저닝하려다 매번 비용/타임아웃 위험에 부딪힘. 이 샘플은 **운영 단순화** 우선:

| 항목 | 이전 (`aah-aws-rag-agent`) | 이 샘플 |
|---|---|---|
| 컨테이너 | api + agent | **agent 1개만** |
| 의존 인프라 | RDS + Redis + OpenSearch + Memory + Databricks | **Memory + KB 2종 (모두 선택)** |
| 매니페스트 리소스 | 5개 (전부 필수) | **1개 (memory, 자동 프로비저닝)** |
| 외부 자원 | RDS/Redis/OpenSearch 매번 만들어야 | **Bedrock KB / Databricks KB 둘 다 ARN 박기만** |
| 빈 환경 | 컨테이너 시작 시 KeyError | **`enabled()` 체크로 자동 graceful** |
| Stack | Flask + LangChain + LangGraph | **boto3 only (의존성 1개)** |

## aah.yaml

```yaml
services:
  - id: agent              # AgentCore Runtime
    target: agentcore_runtime
    dockerfile: ./services/agent/Dockerfile
resources:
  - id: conversation_memory
    kind: agentcore_memory  # AAH 가 자동 프로비저닝
env_by_service:
  agent:
    - LLM_MODEL_ID         # 기본: claude-sonnet-4-6
    - BEDROCK_KB_ID        # 비우면 Bedrock KB 도구 비활성
    - DBX_HOST, DBX_TOKEN, DBX_INDEX_NAME, DBX_EMBED_ENDPOINT  # 비우면 Databricks 도구 비활성
```

## 동작 흐름

```
POST /invocations { "input": "약관에서 암 진단 후 보험금 지급일은?" }
    ↓
[1] AgentCore Memory 에서 이전 대화 가져옴
[2] LLM 도구 선택 — retrieve_databricks_kb 호출
[3] Databricks Vector Search (bge-m3) → 검색 결과
[4] LLM 결과 합성 → 인용 포함 답변
[5] USER + ASSISTANT 이벤트 저장
    ↓
{ "output": "...30일 이내...", "citations": [...] }
```

`Accept: text/event-stream` 헤더를 보내면 token 단위 SSE 응답:

```
event: iter_start    data: {"iter": 1}
event: tool_use_start data: {"name": "retrieve_databricks_kb"}
event: tool_use_end   data: {"name": "retrieve_databricks_kb", "input": {...}}
event: tool_result    data: {"name": "retrieve_databricks_kb", "count": 5}
event: iter_end       data: {"iter": 1, "had_tool_use": true}
event: iter_start     data: {"iter": 2}
event: token          data: {"text": "암 "}
event: token          data: {"text": "진단 "}
...
event: final          data: {"text": "..."}
```

## Code Deploy 등록 절차 (AAH)

1. `/develop/code-deploy` → **새 배포**
2. repo URL: `https://github.com/myzzix4/aah-rag-agent-only`
3. **aah.yaml 분석** — 1 service + 1 resource (memory) 인식
4. Resource Plan — `conversation_memory` = `create` (자동 프로비저닝)
5. 환경변수 (선택):
   - `BEDROCK_KB_ID`: 사용할 Bedrock KB ID (예: `GZBQHKQ2OO`)
   - `DBX_HOST`: `https://dbc-xxx.cloud.databricks.com`
   - `DBX_TOKEN`: PAT
   - `DBX_INDEX_NAME`: `workspace.default.aah_kb_bgem3_idx`
   - `DBX_EMBED_ENDPOINT`: `aah-bge-m3`
6. 배포 → 약 4~6분 (CodeBuild + ECR + AgentCore Runtime + Memory 자동)

## 디렉토리

```
services/agent/
  Dockerfile              ARM64 python:3.12-slim, boto3 only
  requirements.txt        boto3>=1.35.0 — 그게 다
  main.py                 HTTPServer + /ping + /invocations (JSON + SSE)
  src/
    agent.py              ReAct loop + Anthropic native tool_use streaming
    adapters/
      bedrock_kb.py       BEDROCK_KB_ID 있으면 활성. retrieve API.
      databricks_kb.py    DBX_* 있으면 활성. urllib + embed + vector query.
```

## 빠른 검증

배포 완료 후 ARN 으로 직접 invoke:

```bash
aws bedrock-agentcore invoke-agent-runtime \
  --agent-runtime-arn 'arn:aws:bedrock-agentcore:us-east-1:ACCT:runtime/aah_code_aah_rag_age_agent-XXXX' \
  --payload '{"input":"안녕"}' \
  --content-type application/json \
  --accept application/json \
  /dev/stdout
```

또는 AAH `/develop/code-deploy` 행 펼치기 → **💬 채팅 테스트** 버튼.
