# SFT Tool Learning — 데이터 파이프라인 문서

이 Repository는 Tool-use Agent 학습을 위한 합성 SFT(Supervised Fine-Tuning) 데이터 생성 파이프라인들에 대한 기술 문서 모음입니다.

---

## 파이프라인 목록

| 파이프라인 | 도메인 | 특징 | 문서 |
|---|---|---|---|
| [K-skill Datagen](docs/01_K-skill_datagen.md) | 한국 공공 서비스 MCP | FastMCP 기반, 한국어 페르소나, 6단계 파이프라인 | [→](docs/01_K-skill_datagen.md) |
| [Korean Law Datagen](docs/02_korean-law-datagen.md) | 한국 법률 (92개 도구) | FAISS 중복 제거, 신뢰성 검증, MCP stdio 클라이언트 | [→](docs/02_korean-law-datagen.md) |
| [Tool-athlon Datagen](docs/03_Tool-athlon_datagen.md) | 범용 MCP (23개 서버) | Docker 실행 격리, PostgreSQL 목 DB, 7단계 파이프라인 | [→](docs/03_Tool-athlon_datagen.md) |
| [Toucan](docs/04_Toucan.md) | Smithery MCP (2,120개 서버) | 실제 MCP 서버 연동, 병렬 실행, 대규모 생성 | [→](docs/04_Toucan.md) |

---

## 공통 개요

네 파이프라인은 모두 **Tool-use 학습 데이터** 생성을 목표로 하며, 다음 공통 구조를 따릅니다.

```
Task 합성 → Task 필터링 → Agent 실행 (실제 도구 호출) → 궤적 품질 평가 → 최종 ChatSample 변환
```

각 파이프라인은 대상 도메인과 실행 환경에 맞게 설계되었으며, 공통적으로 아래 요소들을 처리합니다.

- **LLM 클라이언트 추상화** — OpenAI / OpenRouter / Anthropic 통합 인터페이스
- **재개 가능한 실행** — 중간 산출물 기반 체크포인팅
- **다중 품질 게이트** — Task 품질 → 궤적 완성도/간결성/신뢰성
- **CJK 언어 오염 필터** — 중국어/일본어 혼입 감지 및 제거
- **ChatSample 포맷** — 최종 출력은 동일한 multimodal content 구조

---

## 데이터 출력 포맷 (ChatSample)

모든 파이프라인의 최종 출력은 아래 구조를 따릅니다.

```json
{
  "id": "task-uuid",
  "messages": [
    {
      "role": "system",
      "content": [{ "type": "text", "value": "당신은 도구를 사용하는 어시스턴트입니다..." }]
    },
    {
      "role": "user",
      "content": [{ "type": "text", "value": "사용자 요청..." }]
    },
    {
      "role": "assistant",
      "content": [
        { "type": "reasoning", "value": "추론 과정..." },
        { "type": "tool_call", "tool_call_id": "call_001",
          "value": "{\"name\": \"server__tool\", \"arguments\": {...}}" }
      ]
    },
    {
      "role": "tool",
      "content": [
        { "type": "tool_result", "tool_call_id": "call_001", "value": "도구 실행 결과..." }
      ]
    },
    {
      "role": "assistant",
      "content": [{ "type": "text", "value": "최종 답변..." }]
    }
  ],
  "tools": [
    {
      "type": "function",
      "function": {
        "name": "server__tool_name",
        "description": "도구 설명",
        "parameters": { "type": "object", "properties": { ... } }
      }
    }
  ],
  "metadata": { ... }
}
```

---

## 빠른 탐색

- 한국어 공공 서비스 도구 학습 데이터가 필요하다면 → **[K-skill Datagen](docs/01_K-skill_datagen.md)**
- 한국 법률 Q&A 학습 데이터가 필요하다면 → **[Korean Law Datagen](docs/02_korean-law-datagen.md)**
- 다양한 MCP 도구 조합의 Docker 기반 실행 궤적이 필요하다면 → **[Tool-athlon Datagen](docs/03_Tool-athlon_datagen.md)**
- Smithery 생태계의 실제 MCP 서버 연동 대규모 데이터가 필요하다면 → **[Toucan](docs/04_Toucan.md)**
