# Korean Law Datagen

## 개요

**Korean Law Datagen**은 한국 법률 도메인 특화 Tool-use SFT 데이터를 생성하는 파이프라인입니다.

`korean-law-mcp` MCP 서버가 노출하는 92개 법률 도구(법령 검색, 판례 검색, 법령 조문 조회, 행정심판 결정문 등)를 활용하여, 실제 한국 법률 질문에 Agent가 도구를 사용해 답변하는 고품질 궤적을 생성합니다.

**핵심 특징:**
- 92개 한국 법률 MCP 도구 (law.go.kr Open API 기반)
- FAISS 코사인 유사도 기반 의미론적 중복 제거
- 고장난 API 엔드포인트 12개 사전 차단 (BLOCKED_TOOLS)
- 다중 레이어 신뢰성(Faithfulness) 검증 (인-트래젝터리 + 사후 필터)
- JSON-RPC stdio 기반 MCP 클라이언트
- 6단계 파이프라인 (Stage 6은 별도 신뢰성 필터)

---

## 디렉토리 구조

```
korean-law-datagen/
├── datagen/
│   ├── stage1_onboarding.py           # Stage 1: MCP 도구 목록 등록
│   ├── stage2_task_synthesis.py       # Stage 2: 법률 질문 합성
│   ├── stage3_task_filtering.py       # Stage 3: 의미론적 중복제거 + 품질 평가
│   ├── stage4_trajectory_gen.py       # Stage 4: Agent 궤적 생성
│   ├── stage5_post_filtering.py       # Stage 5: 규칙 + LLM 품질 필터
│   ├── stage6_faithfulness_filter.py  # Stage 6: 신뢰성 후처리 검증
│   ├── free_trajectory_gen.py         # 대안: 모든 도구 제공 방식
│   ├── law_tools.py                   # Lazy-load 도구 레지스트리
│   ├── mcp_client.py                  # MCP JSON-RPC stdio 클라이언트
│   ├── llm_client.py                  # 다중 엔진 LLM 래퍼
│   ├── utils.py                       # 유틸리티 (중복제거, I/O, 체크포인팅)
│   └── run_pipeline.sh                # 전체 파이프라인 실행 스크립트
├── prompts/
│   ├── system_prompt_base.md          # Agent 시스템 프롬프트 (Task별 도구)
│   ├── free_system_prompt.md          # Agent 시스템 프롬프트 (전체 도구)
│   ├── task_synthesis.md              # 법률 질문 생성 프롬프트
│   ├── task_quality_check.md          # Task 품질 평가 프롬프트
│   ├── response_quality_check.md      # 응답 품질 평가 프롬프트
│   ├── faithfulness_judge.md          # 신뢰성 검증 프롬프트
│   └── context_to_summary.md         # 긴 도구 결과 요약 프롬프트
├── data/
│   ├── {timestamp}/
│   │   ├── tool_registry.json
│   │   ├── tasks_raw.jsonl
│   │   ├── tasks_filtered_{judge}.jsonl
│   │   └── trajectories_raw_{agent}.jsonl
│   ├── completed/
│   │   └── law_sft_{agent}_{judge}_{ts}.jsonl
│   └── rejected/
├── tests/
│   ├── test_stage1.py
│   ├── test_stage3.py
│   ├── test_stage4.py
│   ├── test_stage5.py
│   ├── test_utils.py
│   ├── test_llm_client.py
│   └── test_law_tools.py
└── pyproject.toml
```

---

## 파이프라인 전체 흐름

```
Stage 1: MCP 도구 온보딩
    korean-law-mcp 서브프로세스 → 92개 도구 스키마 등록
         ↓
Stage 2: 법률 질문 합성
    페르소나 + 1~3개 도구 샘플링 → LLM으로 한국어 법률 질문 생성
         ↓
Stage 3: Task 필터링
    FAISS 의미론적 중복 제거 → LLM Judge 품질 점수 (quality + realism)
         ↓
Stage 4: Agent 궤적 생성
    BLOCKED_TOOLS 제외 → OpenAI 함수 호출 형식 → MCP 도구 실행 → 다중 턴 대화
         ↓
Stage 5: 후처리 필터링
    규칙 기반 → LLM 3차원 점수 (완성도/간결성/신뢰성) → 스키마 검증
         ↓
Stage 6: 신뢰성 후처리 (선택)
    완성된 데이터셋 전체에 대해 신뢰성 재검증
         ↓
    최종: data/completed/law_sft_{ts}.jsonl
```

---

## 각 Stage 상세

### Stage 1 — MCP 도구 온보딩 (`stage1_onboarding.py`)

`korean-law-mcp` Node.js 패키지를 서브프로세스로 구동하여 MCP JSON-RPC 프로토콜을 통해 도구 목록을 가져옵니다.

```
python stage1_onboarding.py
    → npx -y korean-law-mcp@latest (서브프로세스)
    → MCP tools/list 요청
    ← 92개 도구 스키마 (name, description, inputSchema)
    → data/{timestamp}/tool_registry.json 저장
```

**등록되는 도구 유형 (92개):**
- 법령 검색 (`search_law`, `get_law_text`, `get_article`)
- 판례 검색 (`search_precedent`, `get_precedent_text`)
- 행정심판 결정문 (`search_appeal_review_decisions`)
- 법령 개정 이력 (`search_historical_law`)
- 영문 법령 (`search_english_law`, `get_english_law_text`)
- 법령 비교 (`compare_old_new`)
- 등 다수

**`McpClient` 클래스:**

JSON-RPC over stdio 방식으로 서브프로세스와 통신합니다. 스레드 안전성을 위해 요청 ID 생성과 응답 대기에 잠금을 사용합니다.

```python
class McpClient:
    def __init__(self):
        self._proc = subprocess.Popen(
            ["npx", "-y", "korean-law-mcp@latest"],
            stdin=PIPE, stdout=PIPE, ...
        )
    
    def list_tools(self) -> list[dict]: ...
    def call_tool(self, name, arguments, timeout=120) -> str: ...
```

**출력:** `data/{timestamp}/tool_registry.json`

---

### Stage 2 — 법률 Task 합성 (`stage2_task_synthesis.py`)

등록된 92개 도구 중 1~3개를 무작위 선택하여 해당 도구를 사용해야 풀 수 있는 법률 질문을 생성합니다.

**페르소나 기반 생성:**

`~/persona/Nemotron-Personas-Korea/data`의 parquet 파일에서 페르소나를 샘플링합니다(성별, 연령, 직업, 결혼 여부, 교육 수준 등). 페르소나는 질문의 맥락을 구체화합니다.

예시:
- "30대 자영업자가 임대차 분쟁에서 자신의 권리를 알고 싶어 합니다"
- "50대 공무원이 공직자 이해충돌 방지법 위반 사례를 조사합니다"

**도구 이름 난독화:**

프롬프트에 도구의 실제 내부 API 이름 대신 사람이 이해하기 쉬운 설명을 제공합니다. 모델이 특정 API 명칭에 과적합하지 않도록 합니다.

**재개 가능성:**

완료된 `task_id`를 추적하여 중단 후 재실행 시 이미 생성된 Task는 스킵합니다.

**출력:** `data/{timestamp}/tasks_raw.jsonl`
```json
{
  "id": "task-uuid",
  "task_id": "t_001",
  "question": "임대차 계약 만료 후 보증금 반환 거부 시 법적 대응 방법은?",
  "tool_names": ["search_law", "search_precedent"],
  "domain": "민법",
  "persona": { "sex": "female", "age": "30", "occupation": "자영업" }
}
```

---

### Stage 3 — Task 필터링 (`stage3_task_filtering.py`)

LLM Judge 호출 전에 의미론적 중복을 먼저 제거하여 비용을 절감하고, 이후 LLM으로 품질을 평가합니다.

#### Step 1: FAISS 의미론적 중복 제거

```python
def deduplicate_by_similarity(texts, threshold=0.05, model_name="paraphrase-multilingual-mpnet-base-v2"):
    # 1. SentenceTransformer로 임베딩 생성
    embeddings = model.encode(texts, normalize_embeddings=True)
    
    # 2. FAISS IndexFlatIP로 코사인 유사도 검색
    index = faiss.IndexFlatIP(embeddings.shape[1])
    
    # 3. 각 항목을 추가할 때, 기존 항목들과의 최대 유사도가
    #    threshold(기본 0.05, 즉 95% 유사성 차단) 미만인 경우만 보존
    kept = []
    for emb, item in zip(embeddings, items):
        if not index.ntotal or max_similarity < (1.0 - threshold):
            index.add(emb)
            kept.append(item)
    return kept
```

`paraphrase-multilingual-mpnet-base-v2` 모델은 한국어를 포함한 다국어 의미론적 유사성을 잘 포착합니다. 이 단계에서 보통 전체 Task의 약 50%가 통과합니다.

#### Step 2: LLM 품질 평가

`task_quality_check.md` 프롬프트로 두 차원을 1~5점으로 평가합니다.

| 차원 | 설명 |
|---|---|
| `quality` | 질문의 법적 명확성, 구체성, 도구 필요성 |
| `realism` | 실제 한국인이 제기할 법한 상황인지 |

**임계값:** avg(quality, realism) ≥ 2.0

**출력:** `data/{timestamp}/tasks_filtered_{judge}.jsonl`

---

### Stage 4 — 궤적 생성 (`stage4_trajectory_gen.py`)

파이프라인의 핵심 단계입니다. 각 법률 질문에 대해 Agent가 실제로 법률 도구를 호출하며 답변하는 대화 궤적을 생성합니다.

#### BLOCKED_TOOLS — 고장난 API 사전 차단

실운영 중 발견된 지속적으로 오류를 반환하는 12개 도구를 Agent에게 노출하지 않습니다.

```python
BLOCKED_TOOLS = frozenset({
    "compare_old_new",              # HTTP 500 오류
    "get_appeal_review_decision_text",  # 500 + not found
    "search_appeal_review_decisions",
    "get_english_law_text",         # 영문 법령 엔드포인트 미작동
    "search_english_law",
    "get_nlrc_decision_text",
    "search_historical_law",        # XML 엔드포인트 장애
    "get_article_with_precedents",
    # ...
})
```

이 도구들을 완전히 차단하지 않으면 Agent가 이를 시도하다 오류 메시지를 받고, 해당 오류 패턴이 학습 데이터에 포함됩니다. 잘못된 도구 사용 패턴을 모델이 학습하지 않도록 사전에 제거합니다.

#### Agent 실행 루프 (최대 10턴)

```
[System Prompt] 법률 도구 사용 지침
[User Message] 법률 질문
    ↓
OpenAI API 호출 (함수 호출 형식, BLOCKED_TOOLS 제외)
    ↓
응답에 tool_call이 있으면:
    → MCP 클라이언트로 law.go.kr API 호출
    ← 도구 실행 결과
    → 긴 결과는 LLM 요약
    → 다음 턴 계속
    ↓
tool_call 없이 텍스트만 있으면:
    → 궤적 저장 (Agent가 답변 완료로 판단)
```

#### 잘못된 도구 호출 복구

Agent가 허용된 도구 목록에 없는 도구를 호출하면 (환각 또는 도구명 오타):

1. 마지막 어시스턴트 메시지를 메시지 히스토리에서 롤백
2. 해당 턴을 다시 실행
3. 최대 3회 재시도 (`invalid_tool_retry_limit`)

이 처리가 없으면 잘못된 도구 호출 → 오류 메시지 → Agent가 혼란에 빠지는 악순환이 발생합니다.

#### 일시적 도구 오류 복구

Rate limit, 타임아웃, 서버 일시 불가 등의 오류 감지 시:

1. 마지막 어시스턴트 메시지 롤백
2. 최대 2회 재시도 (`retryable_tool_error_retry_limit`)

`[NOT_FOUND]` 결과 (법령이 실제로 없는 경우)는 재시도 없이 그대로 Agent에게 전달합니다. Agent가 다른 도구로 접근 방법을 바꾸도록 유도합니다.

#### API 키 마스킹

law.go.kr API 키(`LAW_OC` 환경 변수)가 도구 호출 URL에 포함되어 도구 결과에 그대로 반환될 수 있습니다. 모든 도구 결과에서 API 키와 URL 인코딩된 변형을 `XXXXXXX`로 치환합니다.

```python
def _mask_sensitive_values(text: str) -> str:
    api_key = os.environ.get("LAW_OC", "")
    if api_key:
        text = text.replace(api_key, "XXXXXXX")
        text = text.replace(urllib.parse.quote(api_key), "XXXXXXX")
    return text
```

#### 인-트래젝터리 신뢰성 검증 (선택)

각 도구 호출 이후, Agent의 다음 응답이 도구 결과에 근거하고 있는지 즉시 평가합니다.

```python
# 도구 결과 수신 후
faithfulness_result = judge_faithfulness(messages_so_far)

if faithfulness_result == 0:  # 환각 탐지
    # 해당 궤적 전체 폐기 (저장하지 않음)
    raise TrajectoryFailed("Faithfulness check failed")
```

이를 통해 이미 환각이 발생한 궤적이 5~6턴 더 진행되는 낭비를 방지합니다.

#### Toucan SFT 포맷 변환

OpenAI native 메시지 포맷을 학습에 사용하는 Toucan SFT 포맷으로 변환합니다.

```python
# OpenAI 포맷 (입력)
{"role": "assistant", "tool_calls": [{"function": {"name": "search_law", "arguments": "..."}}]}

# Toucan SFT 포맷 (출력)
{
  "role": "assistant",
  "content": [
    {"type": "reasoning", "value": "임대차보호법을 먼저 검색해야 합니다..."},
    {"type": "tool_call", "tool_call_id": "call_001",
     "value": "{\"name\": \"search_law\", \"arguments\": {\"query\": \"임대차보호법\"}}"}
  ]
}
```

**체크포인팅:**

각 Task 완료 후 `trajectories_raw_checkpoint.json`에 진행 상황을 저장합니다. 파이프라인 중단 시 마지막 완료 Task부터 재개합니다.

**출력:** `data/{timestamp}/trajectories_raw_{agent}.jsonl`

---

### Stage 5 — 후처리 필터링 (`stage5_post_filtering.py`)

3단계 필터링으로 최종 품질을 보장합니다.

#### Step 1: 규칙 기반 필터

| 검사 항목 | 기준 |
|---|---|
| `has_system_prompt` | 시스템 메시지 존재 여부 |
| `has_tool_call` | 최소 1회 도구 호출 여부 |
| `has_final_answer` | 마지막 어시스턴트 메시지에 텍스트 존재 |
| `tool_error_count` | 도구 오류 횟수 ≤ failure_threshold |

도구를 한 번도 사용하지 않은 궤적은 Tool-use 학습 데이터로 가치가 없습니다.

#### Step 2: LLM 3차원 품질 평가

`response_quality_check.md` 프롬프트로 전체 궤적을 평가합니다.

| 차원 | 설명 |
|---|---|
| `completeness` (완성도) | 법률 질문이 완전히 해결되었는가? |
| `conciseness` (간결성) | 도구를 효율적으로 사용했는가? |
| `faithfulness` (신뢰성) | 답변이 도구 결과에 근거했는가? |

**임계값:** 세 점수 모두 ≥ 3.0

#### Step 3: 스키마 검증

외부 `datakit` 검증기를 통해 출력 포맷의 스키마 적합성을 확인합니다. 검증 통과 샘플은 `data/completed/`, 실패 샘플은 `data/rejected/`로 분류됩니다.

**감사 파일 생성:**

각 필터링 단계의 결과를 별도 파일로 저장하여 어느 단계에서 얼마나 제거되었는지 분석 가능합니다.

```
stage5_rule_evaluated.jsonl     → 규칙 필터 통과
stage5_judged.jsonl             → LLM 점수 부여
stage5_response_evaluated.jsonl → 임계값 통과
stage5_removed.jsonl            → 탈락 전체
stage5_kept.jsonl               → 통과 전체 (검증 전)
```

**출력:** `data/completed/law_sft_{agent}_{judge}_{ts}.jsonl`

---

### Stage 6 — 신뢰성 후처리 필터 (`stage6_faithfulness_filter.py`)

완성된 데이터셋에 대해 추가적인 신뢰성 검증을 독립적으로 실행하는 선택적 단계입니다.

**동작 방식:**

각 Task에서 "판단 가능한" 어시스턴트 턴을 찾습니다. 판단 가능한 턴이란 최소 1회 도구 호출 이후, reasoning 또는 text 내용이 있는 어시스턴트 메시지입니다.

각 턴에 대해 `faithfulness_judge.md`로 평가하고, `<faithfulness>0</faithfulness>` 응답이 오면 해당 Task를 탈락시킵니다. 첫 번째 실패 시 해당 Task의 나머지 턴은 검사하지 않고 즉시 종료합니다(단락 평가).

이 단계는 Stage 5를 통과했지만 세밀한 신뢰성 검사를 더 필요로 하는 경우에 실행합니다.

---

## 주요 컴포넌트

### MCP 클라이언트 (`mcp_client.py`)

```
python stage4.py
    → McpClient.__init__()
    → subprocess: npx -y korean-law-mcp@latest
    → stdin/stdout으로 JSON-RPC 통신
    → call_tool("search_law", {"query": "임대차보호법"})
    ← "주택임대차보호법 [시행 2024....]..."
```

**스레드 안전성:** 요청 ID 생성(`_req_id_lock`)과 응답 대기(`_pending` dict + Event)에 잠금을 사용하여 여러 스레드에서 동시 호출 가능

**타임아웃:** 기본 120초. 초과 시 `TimeoutError` 발생

**모듈 싱글톤:** `get_client()` 함수로 프로세스당 하나의 클라이언트 인스턴스 유지

### 도구 레지스트리 (`law_tools.py`)

```python
class _LazyTools(list):
    def __getitem__(self, idx):
        if not self._loaded:
            self._load()  # 첫 접근 시 MCP 서브프로세스 구동
        return super().__getitem__(idx)

TOOLS = _LazyTools()  # 모듈 임포트 시점에는 서브프로세스 없음
```

import 시점이 아닌 실제 사용 시점에 서브프로세스를 구동하여 모듈 로드 시간을 단축합니다.

### LLM 클라이언트 (`llm_client.py`)

세 가지 엔진을 통합 인터페이스로 지원합니다.

```python
# OpenAI 표준
client = OpenAIClient(engine="openai", model="gpt-4o")

# OpenRouter (오픈소스 모델)
client = OpenRouterClient(engine="openrouter", model="moonshotai/kimi-k2.6",
                          provider="io-net/int4")  # int4 양자화

# Anthropic
client = AnthropicClient(engine="anthropic", model="claude-sonnet-4-6")
```

**비용 추적:** 각 API 호출의 프롬프트/완성 토큰 수를 집계하고 모델별 가격표로 USD 비용을 계산합니다. 파이프라인 완료 시 단계별 비용 요약을 출력합니다.

**Kimi 모델 특수 처리:**

```python
# Kimi 모델은 thinking 기능 활성화 필요
if "kimi" in model.lower():
    extra_body = {"thinking": {"type": "enabled", "keep": "all"}}
```

---

## 설정 (`pipeline_config.json`)

```json
{
  "synthesis_model": {
    "engine": "openrouter",
    "model": "xiaomi/mimo-v2.5-pro",
    "provider": "xiaomi/fp8",
    "reasoning_effort": "high",
    "temperature": 0.9,
    "top_p": 0.95,
    "timeout": 300,
    "max_retries": 2,
    "max_workers": 10
  },
  "agent_model": {
    "engine": "openrouter",
    "model": "moonshotai/kimi-k2.6",
    "provider": "io-net/int4",
    "timeout": 600,
    "max_workers": 6,
    "tool_result_token_threshold": 100000,
    "tool_result_summary_max_chars": 80000,
    "tool_result_summary_max_tokens": 8192,
    "invalid_tool_retry_limit": 3,
    "retryable_tool_error_retry_limit": 2
  },
  "judge_model": {
    "engine": "openrouter",
    "model": "deepseek/deepseek-v4-flash"
  },
  "pipeline": {
    "total_tasks": 10000,
    "min_tools": 1,
    "max_tools": 3,
    "sentence_model": "sentence-transformers/paraphrase-multilingual-mpnet-base-v2",
    "distance_threshold": 0.05,
    "task_quality_threshold": 2.0,
    "response_quality_threshold": 3.0
  },
  "post_filter": {
    "failure_threshold": 10000
  },
  "law_api": {
    "api_key_env": "LAW_OC"
  }
}
```

---

## 실행 방법

```bash
# 전체 파이프라인 실행
bash datagen/run_pipeline.sh

# 특정 Stage부터 재개 (RUN_TS는 이전 실행의 타임스탬프)
START_STAGE=stage4 RUN_TS=1746000000 bash datagen/run_pipeline.sh

# Stage 5까지만 실행
STOP_AFTER=stage5 bash datagen/run_pipeline.sh

# 테스트 실행
pytest tests/
```

**중요:** 프록시 환경에서는 `HTTP_PROXY`/`HTTPS_PROXY` 환경 변수를 해제해야 MCP 서브프로세스(Node.js)가 정상 동작합니다. `run_pipeline.sh`는 이를 자동으로 처리합니다.

---

## 고려한 주요 이슈

### 이슈 1: 고장난 API 엔드포인트

**문제:** law.go.kr Open API의 12개 엔드포인트가 지속적으로 HTTP 500 오류, "not found", 또는 잘못된 XML을 반환함. 이 도구들을 Agent에게 노출하면 오류 응답이 학습 데이터에 포함되어 "오류를 받으면 이렇게 행동하라"는 잘못된 패턴을 학습시킬 수 있음

**해결:** `BLOCKED_TOOLS` frozenset에 12개 도구 하드코딩. 레지스트리에는 존재하지만 Agent에게 절대 노출되지 않음

### 이슈 2: 법령 문서의 토큰 오버플로우

**문제:** 법령 전문, 판례 전문 등이 수만 토큰에 달해 LLM 컨텍스트를 초과함

**해결:**
- 100,000 토큰 초과 시 `context_to_summary.md` 프롬프트로 LLM 요약
- 입력 최대 80,000자, 출력 최대 8,192 토큰으로 제한
- `<observation>...</observation>` 블록으로 요약 결과 추출

### 이슈 3: API 키 유출

**문제:** law.go.kr API 호출 URL에 API 키가 쿼리 파라미터로 포함되어, 도구 결과에 API 키가 그대로 노출됨

**해결:** 모든 도구 결과 텍스트에서 원문 API 키와 URL 인코딩된 변형 모두를 `XXXXXXX`로 치환 (Stage 4 도구 호출 직후 적용)

### 이슈 4: Agent의 환각 (신뢰성 위반)

**문제:** Agent가 도구 결과를 무시하고 사전 지식으로 답변하거나, 도구 결과를 왜곡하여 답변을 생성함. 이런 궤적이 학습 데이터에 포함되면 "도구 결과와 무관한 답변도 괜찮다"고 학습

**해결 (이중 방어):**
1. **인-트래젝터리:** 각 도구 호출 이후 즉시 신뢰성 검증. 실패 시 궤적 즉시 폐기
2. **Stage 5:** LLM judge의 faithfulness 차원 (≥3.0 필요)
3. **Stage 6:** 완성된 데이터셋 전체 재검증 (선택적 추가 게이트)

### 이슈 5: 의미론적 중복

**문제:** LLM 기반 합성으로 생성한 질문들은 표면적으로 다르지만 의미적으로 동일한 경우가 많음 (예: "임대차 보증금 반환 방법" vs "전세 보증금 못 받을 때 어떻게 하나요?")

**해결:** FAISS + SentenceTransformer로 코사인 유사도 계산. 코사인 유사도가 95% 이상인 쌍은 중복으로 처리하여 제거

### 이슈 6: 페르소나 부족

**문제:** 페르소나 데이터셋의 크기보다 더 많은 Task를 생성하면 페르소나가 부족해짐

**해결:** 페르소나 풀이 소진되면 기존 페르소나를 순환 재사용 (한 페르소나가 다수 Task에 쓰이더라도 Task 내용 자체는 다름)

### 이슈 7: 재개 가능성

**문제:** 10,000개 이상의 Task를 처리하는 긴 파이프라인이 중간에 실패하면 전체를 다시 실행해야 함

**해결:**
- Stage 2: 완료된 `task_id` 세트를 추적, 재실행 시 스킵
- Stage 4: 매 Task 완료 후 `_checkpoint.json` 저장. 재실행 시 체크포인트 로드, 완료 Task 제외한 나머지만 처리. 파이프라인 정상 완료 시 체크포인트 삭제

### 이슈 8: 동시 실행 안전성

**문제:** 여러 스레드가 동시에 MCP 클라이언트를 사용할 때 응답 혼선 발생 가능

**해결:** `McpClient`의 요청 ID 생성(`_req_id_lock`)과 응답 매핑(`_pending` dict의 각 Entry에 `threading.Event`) 을 스레드 안전하게 구현. 각 스레드는 자신의 요청 ID로만 응답을 수신
