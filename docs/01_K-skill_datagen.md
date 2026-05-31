# K-skill Datagen

## 개요

**K-skill Datagen**은 한국 공공 서비스 MCP(Model Context Protocol) 도구를 활용하는 한국어 Tool-use Agent 학습 데이터를 생성하는 파이프라인입니다.

교통, 날씨, 부동산, 법률, 뉴스, 금융, 의료 등 약 50개의 한국 공공 서비스 FastMCP 서버를 대상으로, 실제 한국인 사용자의 요청 시나리오를 반영한 고품질 도구 사용 궤적(trajectory)을 합성합니다.

**핵심 특징:**
- 한국 공공 데이터 포털 기반 FastMCP 서버 (~50개)
- Nemotron-Personas-Korea 기반 페르소나 합성
- 6단계 품질 게이트 파이프라인
- Distractor 도구 증강으로 위치 상관 편향 방지
- XML, OpenAI structured, Kimi-style 등 다양한 도구 호출 형식 처리

---

## 디렉토리 구조

```
K-skill_datagen/
├── datagen/
│   ├── stage1_onboarding.py            # Stage 1: 서버 카탈로그 등록
│   ├── stage2_task_synthesis.py        # Stage 2: 합성 Task 생성
│   ├── stage3_task_filtering.py        # Stage 3: Task 품질 필터링
│   ├── stage4_free_trajectory_gen.py   # Stage 4: Agent 실행 및 궤적 생성
│   ├── stage5_post_filtering.py        # Stage 5: 궤적 품질 필터링
│   ├── stage6_tool_selection.py        # Stage 6: Distractor 도구 증강
│   ├── agent_loop.py                   # 핵심 Agent 실행 루프
│   ├── llm_client.py                   # 다중 엔진 LLM 클라이언트
│   ├── chatsample.py                   # ChatSample 포맷 변환
│   ├── mcp/
│   │   └── catalog.py                  # MCP 서버 카탈로그 로드
│   ├── personas.py                     # 페르소나 생성
│   ├── utils.py                        # 유틸리티 함수
│   ├── pipeline_config.json            # 파이프라인 설정
│   ├── run_full_free_trajectory.sh     # 전체 파이프라인 실행 스크립트
│   └── run_free_trajectory.sh          # 하위 파이프라인 스크립트
├── servers/
│   ├── kskill_fastmcp.py               # K-skill FastMCP 서버
│   └── publicdata_fastmcp.py           # 공공 데이터 FastMCP 서버
├── configs/
│   └── kskill_servers.json             # 서버 카탈로그 및 인증 정책
├── prompts/
│   ├── stage2_task_synthesis.md        # Task 합성 프롬프트
│   ├── stage3_task_filtering.md        # Task 품질 평가 프롬프트
│   ├── stage4_agent.md                 # Agent 시스템 프롬프트
│   ├── stage5_trajectory_judge.md      # 궤적 품질 평가 프롬프트
│   ├── prior_knowledge_judge.md        # 환각 탐지 Judge 프롬프트
│   ├── server_selection.md             # 서버 선택 프롬프트
│   └── tool_result_summary.md          # 도구 결과 요약 프롬프트
├── data/
│   ├── tool_registry.json              # 등록된 도구 목록
│   ├── synthetic_runs/run_{ts}/        # 실행별 중간 산출물
│   └── completed/                      # 최종 완성 데이터
├── vendor/k-skill/                     # k-skill 도구 소스
└── .env.example
```

---

## 파이프라인 전체 흐름

```
Stage 1: 서버 온보딩
    tool_registry.json 생성
         ↓
Stage 2: Task 합성
    페르소나 + 서버 능력 기반 한국어 Task 생성
         ↓
Stage 3: Task 필터링
    LLM judge로 질문 품질 / 시나리오 현실성 평가
         ↓
Stage 4: 궤적 생성
    서버 선택 → Agent 실행 → 도구 호출 → 다중 턴 대화 수집
         ↓
Stage 5: 궤적 후처리 필터링
    완성도 / 간결성 / 신뢰성 3차원 평가
         ↓
Stage 6: Distractor 도구 증강
    미사용 서버 도구를 추가하여 위치 편향 방지
         ↓
    최종: data/completed/kskill_free_{ts}.jsonl
```

---

## 각 Stage 상세

### Stage 1 — 서버 온보딩 (`stage1_onboarding.py`)

`configs/kskill_servers.json`에 정의된 k-skill 서버 카탈로그를 읽어 실제로 공개 접근 가능한 서버들을 필터링하고, 각 서버의 도구 스키마를 `data/tool_registry.json`에 등록합니다.

**인증 정책 필터링:**

```python
# 포함 (공개 접근 가능)
AUTH_ALLOW = {"none", "public_api", "local_file"}

# 제외 (개인 인증 필요)
AUTH_DENY = {"login", "certificate", "personal_account", "manual_browser_auth"}
```

로그인이나 개인 인증이 필요한 서버는 합성 환경에서 실행 불가능하므로 처음부터 목록에서 제외합니다.

**등록 서버 예시 (~50개):**
- 교통: 버스, 지하철, 기차 시간표
- 날씨: 기상청 단기/중기 예보
- 부동산: 실거래가, 청약 정보
- 법률: 법령 검색, 판례
- 금융: 주가, 환율, 코스피
- 뉴스: 뉴스 검색
- 의료: 병원, 약국 찾기
- 교육: 학교 정보, 수능 등급

**출력:** `data/tool_registry.json`

---

### Stage 2 — Task 합성 (`stage2_task_synthesis.py`)

등록된 서버 중 1~4개를 무작위 조합하여 실제 한국인 사용자가 할 법한 Task를 생성합니다.

**페르소나 기반 생성:**

Nemotron-Personas-Korea parquet 파일에서 페르소나를 샘플링합니다. 각 페르소나는 성별, 연령, 직업, 교육 수준, 디지털 리터러시 등 인구통계학적 정보를 담고 있습니다. 이를 통해 Task에 현실적인 맥락(예: "60대 주부", "20대 취업준비생")이 반영됩니다.

**생성 패턴 3가지:**
1. 공공 정보 조회 — 날씨, 교통, 정부 서비스 등 단순 조회
2. 문서 작성 + 사실 검증 — 내용 생성 후 도구로 사실 확인
3. 다단계 추론 — 여러 서버를 순차 사용하는 복합 Task

**서버 능력 검증:**

각 서버의 `SKILL.md`(최대 3,000자)를 프롬프트에 포함하여, 실제로 해당 서버로 수행 가능한 Task만 생성되도록 합니다. 존재하지 않는 기능을 Task에 요구하는 것을 사전 차단합니다.

**두 가지 생성 모드:**
- **Heuristic 모드**: 템플릿 기반, 빠른 생성
- **LLM 모드** (`--use-llm`): synthesis_model을 호출하여 페르소나 맥락이 반영된 세련된 Task 생성

**출력:** `stage2_tasks.jsonl`
```json
{
  "id": "task_000001",
  "persona": { "age": "30대", "occupation": "직장인", "digital_literacy": "중" },
  "task": "다음 주 서울 날씨를 확인하고 출장 일정에 맞는 교통편을 검색해줘",
  "mcp_servers": ["weather_server", "transit_server"],
  "quality_constraints": { "concrete_location": true, "time_bound": true }
}
```

---

### Stage 3 — Task 필터링 (`stage3_task_filtering.py`)

Agent 실행(비용이 큰 Stage 4) 전에 낮은 품질의 Task를 미리 제거합니다.

**두 가지 필터링 방식:**

1. **Heuristic 모드**: 로그인, 결제, 인증서 등 실행 불가능한 키워드 포함 여부 체크

2. **LLM Judge 모드** (`--judge-llm`): judge_model을 호출하여 2개 차원을 1~5점으로 평가
   - `question_quality`: 질문 자체의 명확성, 구체성
   - `scenario_realism`: 실제 한국인 사용자가 할 법한 시나리오인지

**임계값:** 두 점수의 평균 ≥ 2.0 (기본값)

**출력:**
- `stage3_tasks_filtered.jsonl` — 통과한 Task
- `stage3_tasks_rejected.jsonl` — 탈락한 Task (분석용)

---

### Stage 4 — 궤적 생성 (`stage4_free_trajectory_gen.py`, `agent_loop.py`)

파이프라인의 핵심 단계입니다. 각 Task에 대해 실제 MCP 서버에 도구를 호출하며 Agent 대화 궤적을 생성합니다.

#### 2단계 실행 구조

**Phase 1 — 서버 선택 (synthesis_model 사용):**

전체 레지스트리에서 현재 Task와 가장 관련 있는 서버 1~5개를 선택합니다. 서버 설명 요약본을 컨텍스트로 제공하여 빠르게 판단하게 합니다. 불필요한 서버를 사전에 제거해 Agent의 컨텍스트 부담을 줄입니다.

**Phase 2 — Agent 실행 (agent_model 사용, `agent_loop.py`):**

선택된 서버들의 `skill_instructions`를 미리 가져온 뒤(pre-fetch), 최대 10턴의 대화를 진행합니다.

```
시스템 프롬프트
사용자 메시지 (Task)
    → LLM 호출 → tool_call 포함 응답
    → MCP 서버에 도구 실행
    → 결과를 다음 메시지로 추가
    → 반복 (최대 10턴)
    → 최종 답변 생성
```

#### 핵심 처리 로직

**도구 호출 형식 다형성 지원:**

다양한 모델이 서로 다른 형식으로 도구를 호출하므로, 세 가지 형식을 모두 파싱합니다.

```python
# 1. OpenAI structured (표준 API)
{"type": "function", "function": {"name": "...", "arguments": "..."}}

# 2. XML 형식 (GLM 계열 모델)
<tool_call>server__tool
  <arg_key>query</arg_key>
  <arg_value>서울 날씨</arg_value>
</tool_call>

# 3. Kimi 특수 토큰 형식
functions.TOOL:0 <|tool_call_argument_begin|>{"query": "서울 날씨"}<|tool_call_end|>
```

**도구 결과 요약 (Token Overflow 방지):**

일부 공공 API는 매우 긴 응답을 반환합니다(목록 데이터, 전체 법령문 등). 결과가 100,000 토큰을 초과하면 `tool_result_summary.md` 프롬프트로 LLM 요약을 실행하여 80,000자 이내로 압축합니다.

**Rate-limit 재시도:**

MCP 서버나 LLM API에서 rate-limit 오류 발생 시 지수 백오프(1초 → 2초 → 4초)로 최대 3회 재시도합니다.

**사전 지식 환각 탐지 (Prior Knowledge Filter):**

Agent가 도구를 사용하지 않고 사전 학습 지식으로 답하는 경우를 탐지합니다. 다음 16개 한국어 패턴을 정규식으로 검사합니다.

```python
PRIOR_KNOWLEDGE_PATTERNS = [
    "알려져 있", "것으로 알려", "것으로 알고",
    "알고 있는", "제가 알기로", "알기로는",
    "일반적으로", "보통", "통상적으로",
    ...
]
```

패턴이 감지되면 `prior_knowledge_judge.md` 프롬프트로 환각 여부를 재확인합니다. 환각으로 판정되면 `PriorKnowledgeDetected` 예외를 발생시켜 해당 Task를 **영구 실패**로 처리합니다(재시도 없음, 비용 낭비 방지).

**재개 안전성:**

이미 완료된 Task ID를 추적하여 중단 후 재실행 시 중복 처리를 방지합니다.

**출력:** `stage4_free_trajectories.jsonl`

---

### Stage 5 — 궤적 후처리 필터링 (`stage5_post_filtering.py`)

생성된 궤적의 품질을 다시 평가합니다.

#### 사전 규칙 필터

LLM Judge 호출 전에 구조적 결함을 먼저 제거합니다.

- `has_final_answer`: 최종 답변 메시지 존재 여부
- `has_valid_trajectory`: 최소 1회 이상 도구 호출 여부
- **CJK 오염 사전 필터**: 중국어/일본어 텍스트 포함 여부

> **CJK 필터의 세부 처리:**
> 중국어(간체/번체)와 일본어(히라가나/가타카나)는 제거하되, 한국어 본문에 자연스럽게 섞이는 한자(事實, 民主主義 등)는 보존합니다. 유니코드 범위를 세밀하게 구분하여 처리합니다.

#### LLM 3차원 품질 평가

`stage5_trajectory_judge.md` 프롬프트로 각 궤적을 1~5점으로 평가합니다.

| 차원 | 설명 |
|---|---|
| **Completeness (완성도)** | 사용자 요청이 완전히 해결되었는가? 핵심 질문에 모두 답했는가? |
| **Conciseness (간결성)** | 도구를 효율적으로 사용했는가? 불필요한 반복 호출은 없는가? |
| **Faithfulness (신뢰성)** | 답변이 도구 결과에 근거했는가? 환각이 없는가? |

**임계값:** 세 점수 모두 ≥ 3.0 (평균이 아닌 개별 차원 기준)

**출력:**
- `stage5_free_completed.jsonl` — 통과 (ChatSample 포맷으로 변환)
- `stage5_free_trajectories_rejected.jsonl` — 탈락

---

### Stage 6 — Distractor 도구 증강 (`stage6_tool_selection.py`)

최종 학습 데이터에 Distractor 도구를 추가하여 모델이 도구 위치에 의존하는 편향을 학습하지 않도록 합니다.

**동작 방식:**
1. 현재 샘플에서 사용된 서버를 파악
2. 레지스트리에서 미사용 서버 4개를 무작위 선택
3. 해당 서버의 모든 도구를 도구 목록에 추가
4. 전체 도구 목록을 무작위로 셔플

모델이 "도구 목록에서 첫 번째 도구가 항상 사용된다"는 편향을 학습하면 실제 서비스에서 오작동합니다. Distractor 증강은 이를 방지하고 실제로 관련 있는 도구를 선택하는 능력을 강화합니다.

**출력:** `stage6_free_filtered.jsonl` (최종 학습 데이터)

---

## 주요 컴포넌트

### LLM 클라이언트 (`llm_client.py`)

세 가지 LLM 제공자를 통합 인터페이스로 지원합니다.

```python
# OpenAI
client = OpenAIClient(model="gpt-4o", temperature=0.9)

# OpenRouter (다양한 오픈소스 모델)
client = OpenRouterClient(model="moonshotai/kimi-k2.6", provider="io-net/int4")

# Anthropic
client = AnthropicClient(model="claude-sonnet-4-6")
```

**주요 기능:**
- `<think>` 태그 추출 (DeepSeek-R1 등 추론 모델)
- thinking/reasoning 블록 파싱 (Anthropic extended thinking)
- `complete_many()` — ThreadPool 기반 병렬 처리
- OpenAI Batch API 지원 — 모든 요청 일괄 제출 후 60초 간격 폴링
- OpenRouter provider 오버라이드 — 양자화 설정 (`fp8`, `int4`)

### MCP 카탈로그 (`datagen/mcp/catalog.py`)

`configs/kskill_servers.json`을 파싱하여 서버 메타데이터를 관리합니다.

```python
@dataclass
class KSkillServer:
    name: str
    auth: str  # "none" | "public_api" | "login" | ...
    description: str
    
    @property
    def is_public(self) -> bool:
        return self.auth in {"none", "public_api", "local_file"}
```

SKILL.md 파일의 YAML front-matter에서 서버 설명을 추출하는 기능도 포함합니다.

### ChatSample 변환 (`chatsample.py`)

raw 궤적 데이터를 ChatSample 포맷으로 변환합니다.

- 도구 이름을 `{server}__{tool}` 형식으로 네임스페이싱
- 멀티턴 메시지 시퀀스 구성 (시스템 → 사용자 → 어시스턴트/도구 교대)
- 최종 추론(`final_reasoning`) 포함 여부 처리
- `datakit.validator_runner`로 스키마 유효성 검증

---

## 설정 (`pipeline_config.json`)

```json
{
  "synthesis_model": {
    "engine": "openrouter",
    "model": "xiaomi/mimo-v2.5-pro",
    "provider": "xiaomi/fp8",
    "temperature": 0.9,
    "timeout": 300,
    "max_workers": 10
  },
  "agent_model": {
    "engine": "openrouter",
    "model": "moonshotai/kimi-k2.6",
    "timeout": 600,
    "max_workers": 4,
    "mcp_concurrency": 4,
    "tool_result_token_threshold": 100000,
    "tool_result_summary_max_chars": 80000,
    "prior_knowledge_filter": true
  },
  "judge_model": {
    "engine": "openrouter",
    "model": "deepseek/deepseek-v4-flash"
  },
  "pipeline": {
    "total_tasks": 10,
    "min_tools": 1,
    "max_tools": 2,
    "persona_data_dir": "...",
    "task_quality_threshold": 2.0,
    "response_quality_threshold": 3.0
  },
  "post_filter": {
    "failure_threshold": 1,
    "retry_limit": 3
  }
}
```

---

## 실행 방법

### 전체 파이프라인

```bash
# 기본 실행 (pipeline_config.json 기준)
bash datagen/run_full_free_trajectory.sh

# Task 수 지정
KS_DATAGEN_TOTAL_TASKS=500 bash datagen/run_full_free_trajectory.sh

# LLM 기반 Task 합성 활성화
KS_DATAGEN_USE_LLM_SYNTHESIS=1 KS_DATAGEN_TASK_FILTER_LLM=1 bash datagen/run_full_free_trajectory.sh
```

### 특정 Stage부터 재개

```bash
# Stage 3부터 시작 (Stage 1, 2는 이미 완료)
START_STAGE=stage3 bash datagen/run_full_free_trajectory.sh

# Stage 2에서 멈추기
STOP_AFTER=stage2 bash datagen/run_full_free_trajectory.sh
```

### 환경 변수 전체 목록

| 변수 | 기본값 | 설명 |
|---|---|---|
| `RUN_TS` | `date +%s` | 실행 ID (재개 시 동일 값 사용) |
| `KS_DATAGEN_TOTAL_TASKS` | 50 | 생성할 Task 수 |
| `KS_DATAGEN_USE_LLM_SYNTHESIS` | 1 | LLM Task 합성 활성화 |
| `KS_DATAGEN_TASK_FILTER_LLM` | 1 | LLM Task 필터링 활성화 |
| `START_STAGE` | — | 시작 Stage (stage1\|stage2\|stage3\|free_traj) |
| `STOP_AFTER` | — | 종료 Stage |
| `KS_SKILL_ROOT` | `vendor/k-skill` | k-skill 소스 경로 |

---

## 고려한 주요 이슈

### 이슈 1: CJK 언어 오염

**문제:** 일부 서버의 설명이나 도구 결과에 중국어/일본어가 포함되어 한국어 학습 데이터 순도를 낮춤

**해결:**
- Stage 5에서 중국어/일본어 유니코드 범위(히라가나, 가타카나, CJK 통합 한자)를 검출하여 궤적 전체를 제거
- 단, 한국어 문장 내의 한자(한국식 한자어)는 보존 — 유니코드 범위를 세밀하게 구분

### 이슈 2: 사전 지식 환각

**문제:** Agent가 실제로 도구를 호출하지 않고 학습 데이터에서 얻은 지식으로 답변을 생성하면, 해당 궤적은 "도구를 사용하여 답을 얻었다"는 학습 신호를 잘못 전달함

**해결:**
- 16개 한국어 환각 패턴을 정규식으로 사전 탐지
- 탐지 시 judge_model로 재확인하여 false positive 방지
- 환각으로 판정 시 해당 Task 영구 실패 처리 (재시도 없음 — 동일 Task는 동일 환각 경향이 있어 재시도가 의미 없음)

### 이슈 3: 도구 호출 형식 다형성

**문제:** OpenRouter에서 서빙되는 모델들이 각자 다른 형식으로 도구를 호출함 (OpenAI 표준 API, GLM XML 방식, Kimi 특수 토큰 등)

**해결:** `agent_loop.py`에서 세 가지 형식을 모두 파싱하는 멀티 포맷 파서 구현

### 이슈 4: 도구 결과 토큰 오버플로우

**문제:** 기상청, 법령, 부동산 API 등에서 반환하는 데이터가 매우 길어 LLM 컨텍스트 초과 발생

**해결:** 100,000 토큰 초과 시 `tool_result_summary.md` 프롬프트로 LLM 요약 실행. 최대 80,000자 입력, 8,192 토큰 출력 제한

### 이슈 5: 도구 위치 편향

**문제:** 학습 데이터에서 항상 사용된 도구가 목록 앞쪽에 위치하면 모델이 위치로 도구를 선택하는 편향 학습

**해결:** Stage 6에서 미사용 서버의 도구 4개 세트를 추가하고 전체 도구 목록을 무작위 셔플

### 이슈 6: 병렬 처리 중 데이터 손실

**문제:** 병렬 처리 도중 프로세스가 종료되면 완료된 결과를 잃을 수 있음

**해결:** 각 Task 완료 직후 결과를 파일에 즉시 append (in-place 쓰기). 완료된 Task ID를 추적하여 재실행 시 스킵
