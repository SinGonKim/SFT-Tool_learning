# Toucan

## 개요

**Toucan**은 Smithery MCP 생태계의 실제 서버들을 대상으로 대규모 Tool-use SFT 데이터를 생성하는 파이프라인입니다. 2,120개 이상의 Smithery MCP 서버 메타데이터를 활용하며, 실제 HTTP 엔드포인트에 연결하는 OpenAI Agents 기반 실행 방식을 사용합니다.

앞선 세 파이프라인들이 로컬 MCP 서버 또는 특정 도메인 서버를 대상으로 한다면, Toucan은 공개된 Smithery 생태계 전체를 활용하여 매우 다양한 도구 조합과 시나리오를 생성합니다.

**핵심 특징:**
- 2,120개 Smithery MCP 서버 메타데이터 활용
- 실제 HTTP MCP 엔드포인트 연동 (로컬 서버 불필요)
- 병렬 파이프라인 실행 (`run_parallel_pipeline.sh`)
- 대용량 파일 자동 샤딩 (>10,000 레코드)
- Smithery API 키 기반 MCP 접근 제어
- MCP 서버 사전 연결성 검증 및 실패 추적

---

## 디렉토리 구조

```
Toucan/
├── datagen/
│   ├── run_full_pipeline.sh                   # 메인 파이프라인 오케스트레이터
│   ├── run_parallel_pipeline.sh               # 병렬 실행 모드
│   ├── run_sequential_pipeline.sh             # 순차 실행 모드
│   ├── pipeline_config.json                   # 파이프라인 설정
│   ├── pipeline_config_multiprocess.json      # 멀티프로세스 설정 변형
│   ├── model_configs.json                     # 모델 약어 매핑
│   ├── smithery_live_servers.json             # MCP 서버 화이트리스트
│   ├── mcp_failure_tracking.json              # 서버별 실패 횟수 추적
│   ├── persona_state.json                     # 페르소나 로더 상태
│   │
│   ├── step1.1_gen_questions.py               # Step 1: 질문 생성
│   ├── step1.2_completion.sh                  # Step 1 LLM 완성 래퍼
│   ├── step1.3_process_completion.py          # Step 1 후처리 (FAISS 중복제거)
│   │
│   ├── step2.1_question_quality_check.py      # Step 2: 질문 품질 평가 준비
│   ├── step2.2_completion_quality_check.sh    # Step 2 Judge LLM 래퍼
│   ├── step2.3_process_completion.py          # Step 2 결과 처리
│   │
│   ├── step3.1_completion_agent.sh            # Step 3: Agent 실행 래퍼
│   ├── step3.2_process_completion.py          # Step 3 규칙 기반 필터
│   │
│   ├── step4.1_response_quality_check.py      # Step 4: 응답 품질 평가 준비
│   ├── step4.2_completion_response_check.sh   # Step 4 Judge LLM 래퍼
│   ├── step4.3_process_completion.py          # Step 4 결과 처리 + 지표
│   │
│   ├── step5.1_convert_chatsample.py          # Step 5: ChatSample 변환
│   │
│   ├── completion_endpoint.py                 # 재사용 가능한 LLM 완성 엔진
│   ├── completion_openai_agent.py             # OpenAI Agent + MCP 실행 (Step 3)
│   ├── completion_qwen_agent.py               # Qwen Agent 변형 (레거시)
│   ├── check_mcp_connectivity.py              # MCP 서버 연결성 사전 검증
│   ├── crawl_smithery_servers.py              # Smithery 서버 메타데이터 수집
│   ├── persona_loader.py                      # 페르소나 샘플링 (Nemotron-Personas-USA)
│   ├── add_tool_distractors.py                # Distractor 도구 추가
│   ├── merge_pipeline_outputs.py              # 병렬 실행 결과 병합
│   └── utils.py                               # 공유 유틸리티
│
├── mcp_servers/                               # ~2,120개 MCP 서버 메타데이터
│   └── <id>.<server-identifier>_labeled.json
│
├── data/
│   ├── ToolUse_smithery_{n}_{range}tool_{mode}_{ts}/  # 실행별 디렉토리
│   ├── completed/                                      # 최종 출력 (Step 5)
│   ├── rejected/                                       # 탈락 샘플
│   └── finished/                                       # 완료 마커
│
├── logs/
│   ├── full_pipeline/<YYYYMMDD_HHMMSS_model>/
│   └── parallel_pipeline/
│
├── Qwen-Agent/                                # Qwen Agent 프레임워크 (서브모듈)
└── prompts/
    ├── agent_instructions.md
    └── question_quality_check.md
```

---

## 파이프라인 전체 흐름

```
사전 단계: MCP 연결성 검증
    check_mcp_connectivity.py → smithery_live_servers.json 갱신
         ↓
Step 1: 질문 생성
    1.1 MCP 서버 메타데이터 기반 질문 구조 생성
    1.2 LLM(Grok-4.3)으로 자연어 질문 완성
    1.3 FAISS 의미론적 중복제거 + 정제
         ↓
Step 2: 질문 품질 평가
    2.1 품질 평가 프롬프트 준비
    2.2 LLM Judge(DeepSeek) 실행
    2.3 점수 파싱 및 임계값 필터링
         ↓
Step 3: Agent 실행 (실제 MCP 도구 호출)
    3.1 OpenAI Agents + Smithery 실제 HTTP MCP
    3.2 규칙 기반 필터 (시스템 프롬프트 없는 타임아웃 제거)
         ↓
Step 4: 응답 품질 평가
    4.1 응답 품질 평가 프롬프트 준비
    4.2 LLM Judge 실행
    4.3 점수 파싱 + 도구 정확도 계산
         ↓
Step 5: ChatSample 변환
    5.1 OpenAI 형식 → ChatSample 포맷 변환
         ↓
    최종: data/completed/*_chatsample.jsonl
```

---

## 각 Step 상세

### 사전 단계 — MCP 연결성 검증 (`check_mcp_connectivity.py`)

실제 Smithery HTTP 엔드포인트에 접속하여 살아있는 서버만 화이트리스트에 유지합니다. 응답 없는 서버에 Agent를 연결하면 타임아웃이 발생하여 Step 3의 처리량이 급감하므로, 파이프라인 실행 전에 반드시 수행합니다.

**동작 방식:**
1. `smithery_live_servers.json`의 서버 목록에 대해 HTTP 연결 테스트
2. 실패 분류: `auth_failed`, `forbidden`, `not_found`, `connection_error`, `timeout`
3. 실패 횟수를 `mcp_failure_tracking.json`에 누적
4. `--update_whitelist` 플래그 사용 시 지속적 실패 서버를 화이트리스트에서 제거

**Smithery URL 구성:**

```python
# Smithery MCP 서버는 다음 형식의 HTTP 엔드포인트를 사용
base_url = "https://server.smithery.ai"
server_path = f"/{server_id}/mcp"
headers = {"Authorization": f"Bearer {SMITHERY_API_KEY}"}
```

**중복 서버 정규화:**

동일 MCP 서버가 여러 URL로 등록된 경우, URL을 정규화하여 중복을 제거합니다.

---

### Step 1.1 — 질문 생성 (`step1.1_gen_questions.py`)

`mcp_servers/` 디렉토리의 JSON 파일에서 서버 메타데이터를 읽고, 해당 서버의 도구들을 사용해야 하는 질문 구조를 생성합니다.

#### 서버 샘플링 전략

```python
SAMPLING_STRATEGIES = {
    "random": 균등 무작위,
    "uniform": 카테고리별 균등 분배,
    "power_law": 인기 서버에 높은 가중치,
    "featured": Smithery 추천 서버 우선
}
```

#### 두 가지 실행 모드

**`single_server` 모드:** 한 서버의 도구만 사용하는 질문 생성
```
WooCommerce 서버 선택
→ 이 서버의 도구 2~4개를 사용하는 질문 구조 생성
```

**`multi_server` 모드 (기본):** 서버별로 하나의 도구를 선택하여 여러 서버를 조합
```
[weather-mcp, google-maps-mcp, calendar-mcp] 선택
→ 각 서버에서 하나씩 도구 선택
→ 세 도구를 모두 사용하는 질문 구조 생성
```

#### CJK 메타데이터 필터링

```python
if exclude_cjk_metadata:
    # 서버 설명이나 도구 설명에 중국어/일본어가 포함된 서버 제외
    # 한국어(한글)는 포함 허용
```

비영어권 서버 중 중국어/일본어 메타데이터를 가진 서버를 사전에 제외하여 질문 생성 단계부터 언어 순도를 유지합니다.

#### 페르소나 연동

```python
if use_persona:
    persona = persona_loader.sample_one()
    # 예: {"persona": "...", "sex": "female", "age": "28",
    #      "education_level": "bachelor", "occupation": "software engineer"}
    # 질문 생성 프롬프트에 페르소나 맥락 포함
```

`persona_state.json`으로 로더 상태를 유지하여 재실행 시에도 동일 페르소나를 중복 사용하지 않습니다.

**출력:** `*_prepared.jsonl`
```json
{
  "question": "I need to book a restaurant and add it to my calendar...",
  "target_tools": "restaurant_search,create_event",
  "metadata": {
    "mcp_servers": [
      { "server_name": "yelp-mcp", "remote_server_response": { ... } },
      { "server_name": "google-calendar-mcp", "remote_server_response": { ... } }
    ],
    "question_gen_args": { "mode": "multi_server", "persona": { ... } }
  }
}
```

---

### Step 1.2~1.3 — 질문 완성 및 후처리

**Step 1.2 (`completion_endpoint.py`):**

LLM(Grok-4.3)을 호출하여 질문 구조를 자연스러운 최종 질문으로 완성합니다.

```python
class CompletionEngine:
    # 지원 엔진: openai, openrouter, vllm, together
    # 배치 처리 + 병렬 워커
    # N 배치마다 체크포인트 저장
    # 토큰 한도 강제 (model별 max_tokens)
```

**Step 1.3 (`step1.3_process_completion.py`):**

FAISS 코사인 유사도 검색으로 중복 질문을 제거합니다.

```python
# 임계값: cosine distance < 0.1 (90% 이상 유사하면 중복)
# Toucan은 다양한 도메인의 대규모 생성이므로 K-skill(0.05)보다 느슨한 기준 사용
deduplicate_by_similarity(questions, threshold=0.1)
```

추가로 다음 정제 작업을 수행합니다.
- XML 형식 응답에서 구조화된 필드 추출 (`server_analysis`, `target_tools`, `question`)
- 비정상 유니코드 줄 종결자 제거 (`U+2028`, `U+2029` 등)

**출력:** `*_4prepared.jsonl`

---

### Step 2 — 질문 품질 평가

**Step 2.1:** 각 질문에 대한 judge 프롬프트 준비 (`question_quality_check.md` 렌더링)

**Step 2.2:** DeepSeek judge 모델 호출. 프롬프트당 65K 토큰 한도로 긴 서버 메타데이터도 처리

**Step 2.3:** XML 응답에서 점수 파싱

```
difficulty: 1~5 (도구 사용의 복잡성)
quality:    1~5 (질문 명확성, 구체성)
realism:    1~5 (실제 사용자 요청 가능성)
```

임계값 미달 질문을 제거하여 Step 3 (가장 비용이 큰 단계)에 전달할 질문 수를 줄입니다.

**출력:** `*_2prepared.jsonl`

---

### Step 3 — Agent 실행 (`completion_openai_agent.py`)

가장 비용이 크고 핵심적인 단계입니다. OpenAI Agents 프레임워크를 사용하여 실제 Smithery MCP 서버에 도구를 호출하며 대화 궤적을 생성합니다.

#### OpenAI Agents + Smithery MCP

```python
from agents import Agent, Runner
from agents.mcp import MCPServerSse

# Smithery MCP 서버를 SSE 방식으로 연결
mcp_servers = [
    MCPServerSse(
        url=f"https://server.smithery.ai/{server_id}/mcp",
        headers={"Authorization": f"Bearer {SMITHERY_API_KEY}"}
    )
    for server_id in task["mcp_server_ids"]
]

agent = Agent(
    name="tool-use-agent",
    model="moonshotai/kimi-k2.6",
    instructions=open("prompts/agent_instructions.md").read(),
    mcp_servers=mcp_servers
)

result = Runner.run_sync(agent, task["question"])
```

실제 인터넷상의 MCP 서버에 연결하여 진짜 도구 호출을 수행합니다. 목(mock) 데이터가 아닌 실제 API 응답이 학습 데이터에 포함됩니다.

#### 동시성 제어

**두 가지 레벨의 병렬화:**
1. **Task 수준**: `max_workers` 개의 Task를 동시 처리 (ThreadPool)
2. **MCP 수준**: 각 Task 내에서 `mcp_concurrency` 개의 MCP 서버를 동시 연결

```python
# MCP 세마포어로 동시 연결 수 제한
mcp_semaphore = threading.Semaphore(mcp_concurrency)

with mcp_semaphore:
    result = mcp_server.call_tool(...)
```

MCP 서버마다 동시 연결 제한이 있으므로, 세마포어로 Smithery 서버에 과부하를 주는 것을 방지합니다.

#### 타임아웃 처리

```python
try:
    result = run_with_timeout(agent_func, timeout=600)
    # 성공 시: 시스템 프롬프트 포함하여 저장
    add_system_prompt(result)
except TimeoutError:
    # 타임아웃 시: 시스템 프롬프트 없이 저장
    # Step 3.2에서 시스템 프롬프트 없는 레코드를 필터링하므로 실질적으로 제거됨
    save_without_system_prompt(result)
```

시스템 프롬프트를 타임아웃 마커로 활용하는 영리한 설계입니다. 별도의 상태 플래그 없이 Step 3.2에서 자동으로 필터링됩니다.

#### 자식 프로세스 정리

```python
def cleanup_child_processes():
    # wrapt_timeout_decorator가 포크한 고아 자식 프로세스 탐지
    # /proc를 스캔하여 부모 PID가 현재 프로세스인 자식 찾기
    # SIGKILL 전송
```

OpenAI Agents + timeout decorator 조합에서 자식 프로세스가 정리되지 않는 버그를 직접 수정합니다.

**Step 3.2 필터:**
- 시스템 프롬프트 없는 레코드 제거 (타임아웃으로 실패한 경우)
- 도구 호출이 없는 레코드 제거
- 도구 결과에 오류 패턴이 있는 레코드 제거

**출력:** `*_rule_filtered.jsonl`

---

### Step 4 — 응답 품질 평가

**Step 4.1:** 전체 대화 궤적을 포함한 응답 평가 프롬프트 준비

**Step 4.2:** Judge LLM 호출. 최대 65,000 토큰의 긴 대화도 처리

**Step 4.3:** 점수 파싱 + 도구 정확도 계산

```
completeness: 1~5 (사용자 요청 완전 해결 여부)
conciseness:  1~5 (도구 사용 효율성)

desired_tools_used_percentage: 0.0~1.0 (목표 도구 사용률)
order_correctness: true/false (도구 호출 순서 적절성)
```

**임계값 초과 샘플만 `_top_rated_processed.jsonl`로 저장**

**출력:** `final/*_processed.jsonl`

---

### Step 5 — ChatSample 변환 (`step5.1_convert_chatsample.py`)

내부 포맷을 최종 ChatSample 포맷으로 변환합니다.

**변환 과정:**
1. 시스템 프롬프트에서 도구 설명 텍스트 제거 (도구는 `tools` 필드에 별도 표현)
2. 메시지를 multimodal content 배열 형식으로 변환
3. 제한된 도구(`RESTRICTED_TOOLS`) 목록에 포함된 도구를 사용한 샘플 제거

```python
RESTRICTED_TOOLS = {
    "send_email",    # 실제 이메일 발송 방지
    "post_tweet",    # 실제 트윗 방지
    # ...
}
```

**출력:** `data/completed/*_chatsample.jsonl`

---

## 병렬 파이프라인 실행 (`run_parallel_pipeline.sh`)

대규모 데이터 생성을 위한 병렬 실행 모드입니다.

```bash
# 4개 워커로 24,000개 질문 병렬 생성
NUM_WORKERS=4 TOTAL_PROMPTS=24000 bash datagen/run_parallel_pipeline.sh
```

**동작 방식:**
1. `total_prompts`를 N개 워커에 균등 분배 (예: 24,000 / 4 = 6,000개/워커)
2. 각 워커는 고유한 `RUN_TS`로 독립 파이프라인 실행
3. 워커 시작을 `STAGGER_SECS` 간격으로 엇갈리게 배치 (thundering herd 방지)
4. 모든 워커 완료 후 `merge_pipeline_outputs.py`로 결과 통합

```bash
# 출력 병합
python datagen/merge_pipeline_outputs.py \
  --run-ts-list $RUN_TS_LIST \
  --output data/completed/merged_chatsample.jsonl
```

**대용량 파일 자동 샤딩:**

단일 파일이 10,000 레코드를 초과하면 자동으로 샤딩합니다.

```
base_output.jsonl (10,001개) → 
    base_output_part001.jsonl (10,000개)
    base_output_part002.jsonl (1개)
    base_output.manifest.json (메타데이터)
```

파이프라인의 다음 단계는 매니페스트를 읽어 모든 파트를 하나의 논리적 파일로 처리합니다.

---

## 주요 컴포넌트

### `completion_endpoint.py` — 범용 LLM 완성 엔진

Step 1.2, 2.2, 4.2에서 재사용됩니다.

```python
class CompletionEngine:
    engines = ["openai", "openrouter", "vllm", "together"]
    
    def complete_batch(self, prompts: list[str]) -> list[str]:
        # max_workers 병렬 처리
        # N 배치마다 checkpoint_{n}.jsonl 저장
        # rate limit 오류 → 지수 백오프
        # JSON/JSONL 입력 자동 감지
```

**비용 추적:**
```python
COST_TABLE = {
    "x-ai/grok-4.3": {"input": 3.0, "output": 15.0},  # USD/1M 토큰
    "moonshotai/kimi-k2.6": {"input": 0.6, "output": 2.5},
    "deepseek/deepseek-v4-flash": {"input": 0.1, "output": 0.3},
}
```

각 단계 완료 후 소비한 토큰 수와 USD 비용을 출력합니다.

### `persona_loader.py` — 페르소나 샘플링

```python
class PersonaLoader:
    def __init__(self, persona_dir: str, state_file: str = "persona_state.json"):
        # 11개 parquet 파일을 순환 (Nemotron-Personas-USA)
        # 각 파일에서 최대 N개 페르소나 로드
        # 현재 파일 인덱스와 레코드 인덱스를 state_file에 저장
        # 재실행 시 중단된 위치부터 재개
    
    def sample_batch(self, n: int) -> list[dict]:
        # n개의 고유한 페르소나 반환
        # 풀 소진 시 자동 순환
```

**상태 파일 (`persona_state.json`):**
```json
{
  "file_index": 3,
  "record_index": 1247
}
```

### `utils.py` — 공유 유틸리티

**파일 I/O:**
```python
save_jsonl_sharded(records, base_path, shard_size=10000)
# 10,000개 초과 시 자동 샤딩 + 매니페스트 생성

iter_jsonl_records(path)  # 스트리밍 반복자

safe_save_checkpoint(path, data)
# 임시 파일에 저장 후 원자적 교체 (데이터 손실 방지)
```

**언어 감지:**
```python
def has_cjk_non_korean(text: str) -> bool:
    # 히라가나: U+3040-U+309F
    # 가타카나: U+30A0-U+30FF
    # CJK 통합 한자: U+4E00-U+9FFF
    # 한글(U+AC00-U+D7AF)은 포함하지 않음
```

**레이블 매칭:**
```python
CATEGORIES = [
    "Web Search & Research",
    "Cryptocurrency & Blockchain",
    "File Management",
    "Calendar & Scheduling",
    "E-commerce & Shopping",
    "Social Media & Communication",
    # ... (24개 카테고리)
]

def find_matching_category(description: str) -> str:
    # 도구 설명을 24개 카테고리로 분류
    # 케이스 비감지 매칭
```

---

## 설정 (`pipeline_config.json`)

```json
{
  "question_model": {
    "engine": "openrouter_api",
    "model": "x-ai/grok-4.3",
    "reasoning_effort": "high",
    "max_workers": 15
  },
  "agent_model": {
    "model": "moonshotai/kimi-k2.6",
    "engine": "openrouter_api",
    "timeout": 600,
    "max_workers": 2,
    "mcp_concurrency": 2
  },
  "judge_model": {
    "model": "deepseek/deepseek-v4-flash",
    "engine": "openrouter_api",
    "reasoning_effort": "high",
    "max_workers": 20
  },
  "pipeline": {
    "total_prompts": 6000,
    "min_tools": 2,
    "max_tools": 4,
    "mode": "multi_server",
    "exclude_cjk_metadata": true,
    "use_persona": true,
    "persona_dir": "/home/singon_kim/persona/Nemotron-Personas-USA/data"
  }
}
```

---

## 실행 방법

### 전체 파이프라인

```bash
# 기본 실행
bash datagen/run_full_pipeline.sh

# MCP 연결성 사전 검증 (권장)
python datagen/check_mcp_connectivity.py --update_whitelist

# 특정 Step부터 재개
START_STEP=step3_1 bash datagen/run_full_pipeline.sh

# 이전 실행 재개
RUN_TS=1779264500 bash datagen/run_full_pipeline.sh
```

### 병렬 파이프라인

```bash
# 4개 워커, 24,000개 질문
NUM_WORKERS=4 TOTAL_PROMPTS_OVERRIDE=24000 bash datagen/run_parallel_pipeline.sh

# 워커 시작 간격 조정 (기본: 30초)
STAGGER_SECS=60 NUM_WORKERS=4 bash datagen/run_parallel_pipeline.sh
```

### 환경 변수

| 변수 | 기본값 | 설명 |
|---|---|---|
| `RUN_TS` | `date +%s` | 실행 ID (재개 시 동일 값 사용) |
| `TOTAL_PROMPTS_OVERRIDE` | — | 질문 수 오버라이드 |
| `MIN_TOOLS_OVERRIDE` | — | 최소 도구 수 |
| `MAX_TOOLS_OVERRIDE` | — | 최대 도구 수 |
| `CONFIG_FILE_OVERRIDE` | — | 대체 설정 파일 경로 |
| `START_STEP` | step1_1 | 시작 Step |
| `STOP_AFTER_STEP` | — | 종료 Step |
| `PYTHON_BIN` | 자동 감지 | Python 인터프리터 경로 |
| `OPENAI_API_KEY` | — | OpenAI API 키 |
| `OPENROUTER_API_KEY` | — | OpenRouter API 키 |
| `SMITHERY_API_KEY` | — | Smithery MCP 접근 키 |
| `SMITHERY_PROFILE` | default | Smithery 프로필 이름 |

---

## 고려한 주요 이슈

### 이슈 1: Smithery MCP 서버 가용성

**문제:** Smithery에는 2,120개의 서버가 등록되어 있지만, 언제든 일부 서버가 오프라인이거나 인증 오류가 발생할 수 있음. 이를 파이프라인 실행 중에 발견하면 Agent가 타임아웃으로 실패하여 처리량이 급감함

**해결:**
- `check_mcp_connectivity.py`로 파이프라인 실행 전 전체 서버 연결 테스트
- 실패 분류: `auth_failed` (인증 오류), `forbidden` (권한 없음), `not_found` (서버 없음), `timeout` (응답 없음)
- `mcp_failure_tracking.json`으로 서버별 실패 횟수 누적 추적
- `smithery_live_servers.json`에서 불안정한 서버 제거

### 이슈 2: Agent 실행 타임아웃

**문제:** OpenAI Agents가 응답이 느린 MCP 서버와 통신하다 타임아웃이 발생. `wrapt_timeout_decorator` 사용 시 자식 프로세스가 정리되지 않아 메모리 누수 발생

**해결:**
- 시스템 프롬프트를 성공 마커로 활용: 타임아웃 시 시스템 프롬프트 없이 저장, Step 3.2에서 자동 필터링
- `cleanup_child_processes()` 함수가 `/proc`를 스캔하여 고아 자식 프로세스에 SIGKILL 전송

### 이슈 3: 대규모 파일 처리

**문제:** 6,000~24,000개 레코드의 JSONL 파일을 다음 단계로 전달할 때 단일 파일이 너무 커져서 메모리 문제 및 처리 지연 발생

**해결:**
- 10,000 레코드 초과 시 자동 샤딩 (`_part001.jsonl`, `_part002.jsonl`, ...)
- 매니페스트 파일(`*.manifest.json`)로 샤드 메타데이터 관리
- 다음 단계는 매니페스트를 읽어 모든 샤드를 단일 스트림으로 처리

### 이슈 4: 중국어/일본어 메타데이터 오염

**문제:** Smithery의 2,120개 서버 중 일부는 중국어나 일본어 설명을 가짐. 이런 서버 기반으로 생성된 질문은 CJK 문자를 포함하게 됨

**해결:**
- Step 1.1에서 `exclude_cjk_metadata: true`로 중국어/일본어 메타데이터 서버 사전 제외
- 만약 통과하더라도 최종 필터에서 CJK 감지로 제거

### 이슈 5: MCP 세마포어 부재 시 서버 과부하

**문제:** Task당 최대 4개의 MCP 서버에 동시 연결하고, 8개의 Task를 병렬 처리하면 최대 32개의 동시 MCP 연결이 발생. Smithery 서버가 이를 rate-limit으로 차단함

**해결:** `mcp_concurrency` 세마포어로 각 Task 내의 동시 MCP 연결을 2개로 제한. 전체 동시 MCP 연결을 `max_workers × mcp_concurrency`로 제어 가능


### 이슈 6: 프록시 환경 간섭

**문제:** 회사/서버 환경의 HTTP 프록시 설정이 Smithery MCP 연결을 방해하거나 우회함

**해결:** 파이프라인 시작 시 `HTTP_PROXY`, `HTTPS_PROXY`, `NO_PROXY` 환경 변수를 명시적으로 해제 (run_full_pipeline.sh 상단)

### 이슈 7: 도구 정확도 측정

**문제:** Task에서 `target_tools`를 지정했지만 Agent가 다른 도구를 사용했을 때, 이것이 실패인지 창의적 해결인지 구분하기 어려움

**해결:** Step 4.3에서 두 가지 지표를 별도로 측정
- `desired_tools_used_percentage`: 목표 도구 중 실제로 사용된 비율 (완전 일치를 요구하지 않음)
- `order_correctness`: 도구 호출 순서의 논리적 타당성 (Judge LLM이 평가)

두 지표를 품질 점수와 함께 메타데이터로 저장하여 학습 데이터 선별 기준으로 활용
