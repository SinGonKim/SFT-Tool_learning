# Tool-athlon Datagen

## 개요

**Tool-athlon Datagen**은 23개의 다양한 MCP 서버(파일시스템, Google Calendar, Snowflake, WooCommerce, Arxiv, YouTube 등)를 활용하는 범용 Tool-use Agent 학습 데이터를 생성하는 파이프라인입니다.

핵심 차별점은 **Docker 기반 완전 격리 실행 환경**입니다. 각 Task를 독립된 Docker 컨테이너와 전용 PostgreSQL 인스턴스에서 실행하여, 테스트 데이터와 실행 환경이 Task 간에 오염되지 않도록 보장합니다.

**핵심 특징:**
- 23개 MCP 서버 (Node.js + Python 혼합)
- Task별 Docker 네트워크 + PostgreSQL 격리 (run_isolated.sh)
- 실제 데이터가 담긴 PostgreSQL 스냅샷 (`db/init.sql.gz`)
- LibreOffice, Playwright 브라우저 포함 컨테이너
- CAMEL 프레임워크 기반 Agent 실행
- 7단계 파이프라인 (언어 필터 + 오류 추출 포함)

---

## 디렉토리 구조

```
Tool-athlon-datagen/
├── datagen/
│   ├── stage1_onboard_mcp_servers.py       # Stage 1: MCP 서버 카탈로그 구성
│   ├── stage2_synthesize_tasks.py          # Stage 2: 합성 Task 생성
│   ├── stage3_filter_tasks.py              # Stage 3: Task 필터링 (중복제거 + 품질)
│   ├── stage4_generate_trajectories.py     # Stage 4: Docker 실행으로 궤적 생성
│   ├── stage5_post_filter.py               # Stage 5: 궤적 후처리 필터
│   ├── stage6_export_chatsample.py         # Stage 6: ChatSample 포맷 변환
│   ├── filter_and_extract.py               # Stage 7: 언어 필터 + 오류 추출
│   ├── extract_failed_stage4.py            # 실패한 Stage 4 Task 재추출 유틸
│   ├── run_full_pipeline.sh                # 전체 파이프라인 오케스트레이터
│   ├── pipeline_config.json               # 파이프라인 설정
│   ├── llm_client.py                       # LLM 클라이언트
│   ├── utils.py                            # 유틸리티 (JSONL I/O, PII 마스킹 등)
│   ├── validator_runner.py                 # n-gram 반복/엔트로피 검증기
│   └── pipeline_logger.py                  # 구조화된 로깅
├── scripts/
│   ├── run_isolated.sh                     # Task별 격리 실행 (병렬용)
│   ├── run_containerized.sh                # 공유 Postgres 실행 (순차용)
│   └── test_containerized.sh              # Docker 환경 스모크 테스트
├── configs/
│   └── mcp_servers/
│       ├── filesystem.yaml
│       ├── google_calendar.yaml
│       ├── snowflake.yaml
│       ├── woocommerce.yaml
│       ├── arxiv-latex-mcp.yaml
│       ├── youtube.yaml
│       ├── playwright_with_chunk.yaml
│       └── ... (총 23개 YAML)
├── local_servers/                          # 25개 MCP 서버 구현체
├── utils/                                  # Agent 실행 지원 (13개 하위 디렉토리)
│   ├── api_model/                          # LLM 제공자 (OpenAI, OpenRouter 등)
│   ├── task_runner/                        # Task 실행 오케스트레이션
│   ├── conversation/                       # 메시지 관리 및 도구 호출 처리
│   ├── data_structures/                    # Task 설정, MCP 서버 정의
│   ├── mcp/                               # MCP 클라이언트/서버 통신
│   ├── evaluation/                         # Task 평가 로직
│   ├── system_prompts/                     # Agent 역할별 프롬프트 템플릿
│   ├── roles/                              # Agent 역할 정의
│   ├── aux_tools/                          # 보조 도구 (claim_done, 긴 출력 처리 등)
│   ├── logging/                            # 구조화 Agent 로깅
│   └── general/                            # 공통 헬퍼 (JSON, 색상, 경로)
├── db/
│   └── init.sql.gz                         # PostgreSQL 스냅샷 (목 데이터 포함)
├── explorer/
│   ├── index.html                          # 결과 탐색 웹 UI
│   └── server.py                           # 간단한 HTTP 서버
├── Dockerfile                              # 에이전트 컨테이너 이미지 정의
├── docker-compose.yml                      # 서비스 정의
├── main.py                                 # Agent 컨테이너 진입점
├── run_full_pipeline.sh                    # 최상위 실행 스크립트
└── .env.example
```

---

## 파이프라인 전체 흐름

```
Stage 1: MCP 서버 온보딩
    23개 YAML 설정 로드 → 도구 정의 수집
         ↓
Stage 2: Task 합성
    서버 조합(2~5개) + 템플릿/LLM → 합성 Task 생성
         ↓
Stage 3: Task 필터링
    규칙 체크 → FAISS 중복제거 → LLM 품질 평가
         ↓
Stage 4: 궤적 생성 (Docker 실행)
    run_isolated.sh 또는 run_containerized.sh
    → Docker 컨테이너 + PostgreSQL 격리 실행
    → CAMEL Agent + MCP 도구 호출
    → traj.json 수집
         ↓
Stage 5: 궤적 후처리 필터
    도구 실패율 / 완료 상태 / LLM 품질 평가
         ↓
Stage 6: ChatSample 변환
    OpenAI 호환 메시지 포맷 → 최종 데이터셋
         ↓
Stage 7: 언어 필터 + 오류 추출
    CJK 분리 → 도구 오류 추출 → 검증 (n-gram, 엔트로피)
         ↓
    최종: data/filtered/toolathlon_gym_synthetic_{ts}.jsonl
```

---

## 각 Stage 상세

### Stage 1 — MCP 서버 온보딩 (`stage1_onboard_mcp_servers.py`)

`configs/mcp_servers/*.yaml` 파일을 읽고, 각 서버를 실제로 프로빙하여 도구 정의를 수집합니다.

**처리 과정:**
1. YAML 파일 파싱 (launch command, 환경 변수, 타임아웃 등)
2. 해당 서버 구현체가 `local_servers/`에 존재하는지 검증
3. 서버를 실제로 구동하여 도구 목록 조회
4. CJK 텍스트 감지로 언어 오염 여부 사전 체크

**23개 MCP 서버 분류:**

| 카테고리 | 서버 |
|---|---|
| 파일/문서 | filesystem, excel, word, pdf-tools, pptx |
| 웹/브라우저 | fetch(npx-fetch), playwright_with_chunk |
| Google | google_calendar, google_sheet, google_forms |
| 연구 | arxiv-latex-mcp, arxiv_local, scholarly_search |
| 데이터 | snowflake, yahoo-finance |
| 커머스/LMS | woocommerce, canvas |
| 커뮤니케이션 | emails, youtube, youtube_transcript |
| 인프라 | terminal, memory, notion |

**MCP YAML 설정 예시 (`google_calendar.yaml`):**
```yaml
name: google_calendar
command: node
args:
  - ${local_servers_paths}/google-calendar-mcp/build/index.js
env:
  GOOGLE_CLIENT_ID: ${GOOGLE_CLIENT_ID}
  GOOGLE_CLIENT_SECRET: ${GOOGLE_CLIENT_SECRET}
  GOOGLE_REFRESH_TOKEN: ${GOOGLE_REFRESH_TOKEN}
timeout: 30
```

**출력:** `stage1_mcp_servers.jsonl`

---

### Stage 2 — Task 합성 (`stage2_synthesize_tasks.py`)

등록된 MCP 서버 중 2~5개를 조합하여 해당 서버들을 동시에 사용해야 풀 수 있는 Task를 생성합니다.

**두 가지 생성 모드:**

1. **Template 모드:** 미리 정의된 시나리오 표(scenario table)에서 패턴을 선택하여 결정론적으로 생성. 빠르고 비용이 없지만 다양성이 제한적

2. **LLM 모드:** `task_model`(기본: `x-ai/grok-4.3`)을 호출하여 서버 조합에 맞는 자연스러운 Task 생성. 옵션으로 Nemotron-Personas 페르소나 추가 가능

**페르소나 연동:**

`PERSONA_DIR` 환경 변수가 설정되면 `~/persona/Nemotron-Personas-USA/data`의 parquet 파일에서 페르소나를 샘플링합니다. 실제 사용자 맥락(직업, 연령 등)을 Task에 반영하여 현실적인 시나리오 생성을 촉진합니다.

**출력:** `stage2_tasks.jsonl`

---

### Stage 3 — Task 필터링 (`stage3_filter_tasks.py`)

3단계 필터링으로 낮은 품질의 Task를 사전 제거합니다.

#### Step 1: 규칙 기반 체크

```python
# 최소 조건
assert len(question) >= 40          # 너무 짧은 질문 제거
assert len(tool_names) >= 2         # 최소 2개 도구 사용 요구
assert contains_action_term(question)  # 실행 가능한 동사 포함 여부
```

#### Step 2: 의미론적 중복 제거

```python
# FAISS + SentenceTransformer
# distance_threshold = 0.05 (코사인 유사도 95% 이상이면 중복)
# FAISS 실패 시 lexical difflib로 폴백
```

#### Step 3: LLM 품질 평가 (선택)

`llm_filter_enabled: true`이면 judge_model로 Task 품질을 평가합니다.

**출력:**
- `stage3_tasks_filtered.jsonl`
- `stage3_tasks_rejected.jsonl`

---

### Stage 4 — 궤적 생성 (`stage4_generate_trajectories.py`)

파이프라인에서 가장 비용이 크고 복잡한 단계입니다. 각 Task를 Docker 컨테이너 안에서 실행합니다.

#### Task 머티리얼라이제이션

각 Task를 `tasks/finalpool/synthetic/{task_id}/`에 파일로 저장합니다.
이 디렉토리가 컨테이너의 workspace로 마운트됩니다.

#### 두 가지 실행 방식

**`run_isolated.sh` — Task별 격리 실행 (병렬용, 기본)**

```bash
# Task별 실행 구조
docker network create toolathlon_net_${TASK_ID}  # 전용 네트워크
docker run postgres toolathlon_pg_${TASK_ID}     # 전용 Postgres

# init.sql.gz로 목 데이터 초기화
zcat db/init.sql.gz | psql ...

# Agent 컨테이너 실행
docker run toolathlon \
  --network toolathlon_net_${TASK_ID} \
  -v tasks/finalpool/synthetic/${TASK_ID}:/workspace/task \
  python main.py

# 완료 후 정리
docker rm toolathlon_pg_${TASK_ID}
docker network rm toolathlon_net_${TASK_ID}
```

Task들이 서로의 데이터베이스 상태를 오염시키지 않도록 각각 독립된 Postgres 인스턴스를 사용합니다. 동시에 여러 Task를 실행할 때 발생하는 교차 오염(cross-contamination)을 완전히 차단합니다.

**`run_containerized.sh` — 공유 Postgres 실행 (순차용)**

순차 실행을 위한 간소화된 방식입니다. `flock`으로 공유 Postgres 접근을 직렬화합니다. 각 Task 실행 전 외래 키 제약 조건(`email.sent_log → email.messages`)을 수동으로 수정합니다.

#### Docker 컨테이너 구성

**Dockerfile 핵심 구성:**

```dockerfile
FROM ubuntu:22.04

# Python 3.12 (uv로 설치)
# Node.js 22
# LibreOffice (PDF 변환용)
# Playwright + Chromium (브라우저 자동화용)

# 23개 MCP 서버 빌드 (Node + Python 혼합)
RUN npm install --prefix local_servers/google-calendar-mcp
RUN npm install --prefix local_servers/filesystem-mcp
# ...

# venv는 /workspace 외부에 위치 (마운트 시 덮어씌워지지 않도록)
ENV VIRTUAL_ENV=/opt/venv
```

컨테이너 내부에서 실제 파일 생성, 브라우저 조작, 이메일 전송 시뮬레이션 등을 수행할 수 있는 완전한 실행 환경을 제공합니다.

#### PostgreSQL 목 데이터 (`db/init.sql.gz`)

다음 스키마의 목 데이터를 포함합니다.

| 스키마 | 내용 |
|---|---|
| Canvas | 강의, 과제, 학생 데이터 |
| Snowflake | 비즈니스 분석 데이터 |
| WooCommerce | 이커머스 주문, 상품, 고객 |
| Yahoo Finance | 주가, 금융 지표 |
| YouTube | 채널, 영상 메타데이터 |
| Emails | 받은 편지함, 발신 로그 |

**외래 키 제약 이슈:** `email.sent_log` 테이블이 `email.messages` 테이블을 참조하는 외래 키 제약 조건이 있어, init.sql 실행 시 순서 문제로 오류 발생. `run_containerized.sh`에서 Task 실행 전 이 제약을 일시 해제하여 처리.

#### CAMEL Agent 실행 (`main.py`)

`utils/` 서브시스템을 통해 CAMEL 프레임워크로 Agent를 실행합니다.

- 최대 스텝: `max_steps` (기본값: 10,000, 사실상 무제한)
- 타임아웃: `timeout_seconds` (기본값: 1,800초 = 30분)
- Agent 실행 결과: `traj.json`으로 workspace에 저장

**궤적 수집:**

`stage4_generate_trajectories.py`가 각 컨테이너의 `traj.json`을 읽어 메시지, 도구 호출, 결과를 추출합니다.

**병렬화:**

`max_workers` 설정에 따라 여러 Docker 컨테이너를 동시에 실행합니다.

| max_workers | 권장 RAM |
|---|---|
| 4 | ~8 GB |
| 8 | ~16 GB |
| 16+ | ~32 GB |

**체크포인팅:**

완료된 Task ID를 추적합니다. 중단 후 재실행 시 이미 성공/실패가 확정된 Task는 스킵합니다.

**출력:** `stage4_trajectories.jsonl`

---

### Stage 5 — 궤적 후처리 필터 (`stage5_post_filter.py`)

Docker 실행으로 생성된 궤적의 품질을 검사합니다.

#### 필터 조건

```python
# 필수 요소
has_assistant_activity = len(assistant_turns) > 0
has_tool_activity = len(tool_calls) > 0

# 도구 실패율
tool_failure_rate = failed_tool_calls / total_tool_calls
assert tool_failure_rate <= failure_threshold  # 기본값: 0.8 (80%)

# 실행 완료 상태 (선택)
if require_success_status:
    assert trajectory.status == "success"
```

**도구 실패율 임계값 80%:** 처음에는 엄격한 기준을 적용했으나, 일부 현실적인 Task (예: "여러 파일을 처리하되 일부는 존재하지 않을 수 있다")에서 도구 오류가 불가피하게 발생함. 80%라는 완화된 기준으로 이런 케이스도 포함하되, 완전히 실패한 궤적은 제거.

#### LLM 품질 평가 (선택)

`post_llm_filter_enabled: true`이면 judge_model로 궤적 품질을 추가 평가합니다.

**출력:**
- `stage5_raw_accepted.jsonl`
- `stage5_trajectories_rejected.jsonl`

---

### Stage 6 — ChatSample 변환 (`stage6_export_chatsample.py`)

내부 포맷을 학습에 사용하는 OpenAI 호환 ChatSample 포맷으로 변환합니다.

**변환 과정:**
1. `traj.json`에서 도구 정의 추출 (없으면 Stage 1 인벤토리 폴백)
2. 메시지를 multimodal content 배열 형식으로 변환
3. `tool_call_errors` 메타데이터 추가 (도구 오류 기록)
4. 도구 이름을 `{server}__{tool}` 형식으로 네임스페이싱

**메타데이터 포함:**
```json
{
  "metadata": {
    "source": "toolathlon_gym",
    "tool_call_errors": [
      {"turn": 3, "tool": "filesystem__read_file", "error": "File not found"}
    ]
  }
}
```

**출력:** `data/completed/toolathlon_gym_synthetic_{ts}.jsonl`

---

### Stage 7 — 언어 필터 + 오류 추출 (`filter_and_extract.py`)

최종 학습 데이터의 순도를 높이는 후처리 단계입니다.

#### CJK 언어 필터

```python
def has_cjk_non_korean(text: str) -> bool:
    # 히라가나, 가타카나, CJK 통합 한자 범위 검출
    # 한국어 한글(가-힣)은 제외
    ...
```

중국어/일본어가 감지된 레코드를 `data/chinese_excluded/`로 이동합니다.

#### 도구 오류 추출

도구 결과에 오류가 포함된 레코드를 `data/Error/`로 분리합니다. 이 데이터는 별도로 분석하거나 오류 복구 학습 데이터로 활용할 수 있습니다.

#### 검증기 (`validator_runner.py`)

세 가지 품질 검사를 수행합니다.

```python
class NGramValidator:
    # 5-gram이 5회 이상 반복되면 거부
    # 모델이 같은 문장을 무한 반복 생성한 경우 탐지

class CompressionValidator:
    # zlib 압축률 > 0.2 미만이면 거부
    # 텍스트 엔트로피가 너무 낮으면 (반복/단조로운 텍스트) 거부

class ReasoningFormatValidator:
    # <think> 태그 포함 시 거부
    # 추론 블록이 최종 출력에 노출된 경우 제거
```

**출력:** `data/filtered/toolathlon_gym_synthetic_{ts}.jsonl` (최종 학습 데이터)

---

## 주요 컴포넌트

### 파이프라인 오케스트레이터 (`datagen/run_full_pipeline.sh`)

2,800줄 이상의 bash 스크립트로, 파이프라인의 모든 측면을 관리합니다.

**핵심 기능:**
- `pipeline_config.json`과 환경 변수 병합 (환경 변수가 우선)
- 타임스탬프 기반 실행 디렉토리 생성 (`data/synthetic_runs/run_{ts}/`)
- 각 Stage 출력 파일 존재 여부 확인으로 자동 스킵
- 장시간 실행 프로세스를 위한 watchdog 진단 로깅
- 오류 트랩 및 정리(cleanup)

**실행 제어 변수:**
```bash
START_STAGE=stage4     # Stage 4부터 시작
STOP_AFTER_STAGE=stage5  # Stage 5 이후 중단
RUN_TS=1746000000      # 이전 실행 재개
```

### LLM 클라이언트 (`datagen/llm_client.py`)

Stage 2, 3, 5에서 사용하는 비동기 LLM 호출 관리자입니다.

**두 가지 실행 모드:**
- `standard`: ThreadPool 기반 병렬 요청
- `batch`: OpenAI Batch API — 모든 요청을 일괄 제출 후 폴링

**지원 엔진:** OpenAI, OpenRouter, Anthropic, DeepSeek

**비용 추적:** 단계별 토큰 사용량 및 USD 비용 집계

### 유틸리티 (`datagen/utils.py`)

**PII 마스킹:**
```python
def sanitize_environment(env: dict) -> dict:
    # API 키, 사용자명 등을 로그에서 제거
    SENSITIVE_KEYS = {"OPENAI_API_KEY", "GOOGLE_CLIENT_SECRET", ...}
    return {k: "***" if k in SENSITIVE_KEYS else v for k, v in env.items()}
```

**JSONL 스트리밍:**
```python
def iter_jsonl_records(path) -> Iterator[dict]:
    # 메모리 효율적 스트리밍 반복자
    # 대용량 파일(수십만 레코드)을 메모리에 전부 로드하지 않고 처리
```

---

## 설정 (`pipeline_config.json`)

```json
{
  "task_model": {
    "engine": "openrouter_api",
    "model": "x-ai/grok-4.3",
    "reasoning_effort": "high",
    "max_workers": 20
  },
  "agent_model": {
    "model": "moonshotai/kimi-k2.6",
    "engine": "openrouter_api",
    "timeout": 600,
    "max_workers": 8,
    "mcp_http_timeout": 600,
    "mcp_stdio_timeout": 120,
    "tool_execution_timeout": 120,
    "step_timeout": 1200
  },
  "judge_model": {
    "engine": "openrouter_api",
    "model": "deepseek/deepseek-v4-pro",
    "max_workers": 20
  },
  "trajectory": {
    "mode": "docker_run",
    "runner_script": "scripts/run_isolated.sh",
    "max_steps": 10000,
    "timeout_seconds": 1800
  },
  "post_filter": {
    "failure_threshold": 80,
    "require_success_status": true
  },
  "pipeline": {
    "total_tasks": 10000,
    "min_servers": 2,
    "max_servers": 5,
    "semantic_distance_threshold": 0.05,
    "task_llm_enabled": true,
    "llm_filter_enabled": true,
    "post_llm_filter_enabled": true
  }
}
```

---

## 실행 방법

### 전체 파이프라인

```bash
# 기본 실행 (10,000개 Task, 8개 병렬 컨테이너)
bash datagen/run_full_pipeline.sh

# Stage 1~3 테스트 (Docker 불필요)
TOTAL_TASKS_OVERRIDE=20 STOP_AFTER_STAGE=stage3 bash datagen/run_full_pipeline.sh

# 소규모 전체 테스트
TOTAL_TASKS_OVERRIDE=5 bash datagen/run_full_pipeline.sh
```

### 특정 Stage 재개

```bash
# Stage 4부터 재개
START_STAGE=stage4 bash datagen/run_full_pipeline.sh

# Stage 4만 실행
START_STAGE=stage4 STOP_AFTER_STAGE=stage4 bash datagen/run_full_pipeline.sh
```

### 실패한 Stage 4 Task 재처리

```bash
# Stage 4에서 실패한 Task만 추출하여 재처리
python datagen/extract_failed_stage4.py \
  --stage4-output data/synthetic_runs/run_xxx/stage4_trajectories.jsonl \
  --stage3-input data/synthetic_runs/run_xxx/stage3_tasks_filtered.jsonl \
  --output data/synthetic_runs/run_xxx/stage4_failed_tasks.jsonl
```

### 환경 변수

| 변수 | 기본값 | 설명 |
|---|---|---|
| `TOTAL_TASKS_OVERRIDE` | — | Task 수 오버라이드 |
| `START_STAGE` | stage1 | 시작 Stage |
| `STOP_AFTER_STAGE` | — | 종료 Stage |
| `TRAJECTORY_MODE_OVERRIDE` | docker_run | `docker_run` 또는 `mock` |
| `INCLUDE_UNAVAILABLE_MCPS` | false | 구현 없는 서버 포함 여부 |
| `PERSONA_DIR` | — | Nemotron-Personas 경로 |
| `OPENROUTER_API_KEY` | — | OpenRouter API 키 |
| `MODEL_NAME` | — | Agent 모델명 오버라이드 |

---

## 고려한 주요 이슈

### 이슈 1: Task 간 데이터 오염

**문제:** 여러 Task를 병렬로 실행할 때 하나의 Task가 데이터베이스를 수정하면 다른 Task의 실행 결과에 영향을 줄 수 있음. 예를 들어 Task A가 WooCommerce 주문을 삭제하면 Task B의 "주문 목록 조회"가 다른 결과를 반환함

**해결:** `run_isolated.sh`는 각 Task마다 독립된 Docker 네트워크와 PostgreSQL 컨테이너를 생성합니다. 각 Postgres는 `init.sql.gz`에서 동일한 초기 상태로 시작하므로 Task들이 완전히 독립됩니다.

```
Task A → toolathlon_pg_taskA (172.100.1.0/24)
Task B → toolathlon_pg_taskB (172.100.2.0/24)
서로 완전히 격리됨
```

### 이슈 2: 이메일 테이블 외래 키 제약

**문제:** `db/init.sql.gz`의 `email.sent_log` 테이블이 `email.messages`를 참조하는 외래 키 제약을 가지고 있어, init.sql의 테이블 생성 순서에 따라 가끔 오류 발생

**해결:** `run_containerized.sh`에서 각 Task 실행 전 해당 외래 키 제약을 임시로 DROP하고 데이터 삽입 완료 후 재생성

### 이슈 3: 컨테이너 장시간 실행 / 타임아웃

**문제:** 일부 Task가 무한 루프나 응답이 없는 MCP 서버로 인해 컨테이너가 영원히 실행될 수 있음

**해결:**
- `timeout_seconds: 1800` (30분) 설정으로 컨테이너 강제 종료
- `step_timeout: 1200` (20분)으로 개별 도구 호출 타임아웃
- Stage 5에서 `require_success_status: true`로 타임아웃 종료된 궤적 제거

### 이슈 4: n-gram 반복 생성

**문제:** LLM이 토큰을 반복 생성하는 퇴화(degeneration) 현상. 학습 데이터에 포함되면 모델도 같은 패턴을 학습

**해결:** `NGramValidator`가 5-gram 기준으로 5회 이상 반복되는 패턴을 탐지하고 해당 레코드를 제거

### 이슈 5: `<think>` 태그 노출

**문제:** 추론 모델(DeepSeek-R1 등)이 `<think>...</think>` 형식의 내부 추론을 최종 응답에 포함하여 출력하는 경우가 있음. 이를 그대로 학습 데이터에 포함하면 모델이 최종 응답에 `<think>` 태그를 사용하도록 학습됨

**해결:** `ReasoningFormatValidator`가 `<think>` 태그가 포함된 레코드를 탐지하고 제거

### 이슈 6: PII 및 API 키 유출

**문제:** 컨테이너 내부에서 실행되는 도구들이 API 키, Google OAuth 토큰 등을 로그나 도구 결과에 포함할 수 있음

**해결:** `utils.py`의 `sanitize_environment()` 함수가 민감한 환경 변수 키(`OPENAI_API_KEY`, `GOOGLE_CLIENT_SECRET` 등)를 로그 출력 전에 마스킹. 도구 결과도 별도로 민감 정보를 필터링

### 이슈 7: Stage 4 부분 실패 복구

**문제:** 10,000개 Task 중 일부가 실패했을 때, 전체를 재실행하면 비용이 크게 증가

**해결:**
- 완료된 Task ID를 추적하여 재실행 시 자동 스킵
- `extract_failed_stage4.py`로 실패한 Task만 별도 추출하여 선택적 재처리

### 이슈 8: 큰 JSONL 파일 처리

**문제:** Stage 4 궤적 파일이 수십만 레코드를 담아 전체를 메모리에 로드하면 OOM 위험

**해결:** `iter_jsonl_records()` 제너레이터로 파일을 스트리밍 처리. 전체 파일을 메모리에 올리지 않고 레코드 단위로 처리
