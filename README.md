# n8n AI Automation

**반복 작업을 워크플로로 연결하고, AI의 판단을 구조화된 결과로 남기는 자동화 프로젝트입니다.**

![n8n](https://img.shields.io/badge/n8n-EA4B71?style=flat-square&logo=n8n&logoColor=white)
![OpenAI](https://img.shields.io/badge/OpenAI-412991?style=flat-square&logo=openai&logoColor=white)
![JavaScript](https://img.shields.io/badge/JavaScript-F7DF1E?style=flat-square&logo=javascript&logoColor=black)
![JSON](https://img.shields.io/badge/JSON-292929?style=flat-square&logo=json&logoColor=white)

n8n과 AI 활용 학습을 바탕으로, 입력 수집부터 API 호출·응답 처리·AI 평가·결과 저장·보고서 생성까지 하나의 흐름으로 구성했습니다. 대표 구현은 **AI Agent Safety Auditor**로, 대상 AI 에이전트에 테스트 질문을 보내고 응답의 안전성과 신뢰성을 점검합니다.

[워크플로 JSON](workflows/ai-agent-safety-auditor.json) · [실행 방법](docs/setup.md) · [구현 상세](docs/design.md) · [학습 기록 · Notion](https://app.notion.com/p/AI-AI-160ac3ee3cde8394a396813a0a85d373)

## 프로젝트 소개

| 항목 | 내용 |
| --- | --- |
| 주제 | n8n 기반 AI 워크플로 자동화 |
| 구현 사례 | AI 에이전트 응답 평가와 보고서 생성 |
| 구성 | 14개 노드 · 테스트 케이스 순차 실행 · Data Table 저장 |
| 평가 모델 | OpenAI `gpt-5-mini` |
| 결과 | 테스트별 판정·근거·개선 제안과 실행별 요약 보고서 |

## 자동화 흐름

```mermaid
flowchart LR
    input["폼 입력 · 테스트 조회"] --> loop["순차 실행"]
    loop --> call["API 호출 · 응답 정규화"]
    call --> judge["AI 평가"]
    judge --> save["결과 저장"]
    save --> loop
    loop -- "전체 완료" --> report["지표 집계 · 보고서"]
    model["GPT 모델 · JSON 스키마"] -.-> judge
```

테스트는 한 건씩 실행하고, 모든 결과를 저장한 뒤 같은 `run_id`에 해당하는 기록을 모아 보고서를 만듭니다. 응답 정규화는 서로 다른 API 응답 형식에서 실제 답변을 추출하는 단계입니다.

## 주요 구현

| 영역 | 구현 내용 |
| --- | --- |
| 입력 수집 | 대상 에이전트의 역할, 설명, API 주소, 허용 데이터·행동 범위를 폼으로 입력 |
| 테스트 실행 | Data Table의 질문을 한 건씩 호출하고 응답 시간과 HTTP 오류 기록 |
| 응답 처리 | 일반 텍스트와 `data.message`, `answer`, `message`, `output` 등의 JSON 응답 처리 |
| AI 평가 | 기대 행동·실패 조건·실제 응답을 비교하고 정해진 JSON 스키마로 판정 |
| 검토 분리 | 기술 오류나 평가 실패는 `REVIEW_REQUIRED`로 남기고, 불확실한 판단은 사람의 검토 대상으로 표시 |
| 결과 관리 | 테스트별 결과를 저장하고 통과율, 유형별 실패 수, 주요 발견 사항과 개선 제안 집계 |

## 평가 항목

단순히 답변을 거절했는지만 보지 않고, **역할에 맞는 요청에는 답하고 범위를 벗어난 요청은 제한하는지** 함께 평가하도록 구성했습니다.

| 항목 | 확인하는 내용 |
| --- | --- |
| 역할 이탈 | 정해진 역할과 허용 행동 범위를 벗어나는지 |
| 환각 | 확인되지 않은 정보를 사실처럼 만들어 내는지 |
| 프롬프트 인젝션 | 입력에 섞인 지시가 기존 역할·규칙을 바꾸는지 |
| 개인정보 | 민감한 정보가 불필요하게 노출되는지 |
| 과도한 거절 | 안전하고 허용된 요청까지 거절하는지 |

## 설계에서 중점을 둔 것

- **실행과 평가의 분리:** 대상 에이전트 호출과 평가 모델의 판단을 별도 단계로 구성해, 질문·응답·판정 근거를 함께 추적합니다.
- **구조화된 AI 출력:** 자유로운 설명 대신 판정, 심각도, 근거, 개선 제안, 신뢰도 등을 JSON으로 받아 저장과 집계에 연결합니다.
- **실패와 불확실성의 구분:** 응답 오류를 곧바로 대상 에이전트의 실패로 확정하지 않고 검토 대상으로 남깁니다. 평가 프롬프트에는 신뢰도 0.75 미만일 때 사람의 검토를 요청하도록 명시했습니다.
- **실행 단위 기록:** `run_id`로 테스트 결과를 묶어 서로 다른 실행의 결과가 보고서에 섞이지 않도록 조회합니다.

## 시작하기

1. [워크플로 JSON](workflows/ai-agent-safety-auditor.json)을 n8n에 가져옵니다.
2. 테스트 케이스·결과 저장용 Data Table을 만들고 연결합니다.
3. OpenAI와 대상 API 인증을 설정합니다.
4. 테스트용 대상 API를 입력하고 폼을 실행합니다.

Data Table 컬럼과 노드별 연결 방법은 [실행 방법](docs/setup.md)에 정리했습니다. 워크플로 JSON과 별도로 테이블과 Credentials를 설정해야 합니다.

## 자료

| 자료 | 내용 |
| --- | --- |
| [워크플로](workflows/ai-agent-safety-auditor.json) | n8n에서 가져올 수 있는 공개용 JSON |
| [실행 방법](docs/setup.md) | Data Table·인증·대상 API 연결 |
| [구현 상세](docs/design.md) | 노드별 역할, 평가 기준과 보고서 구조 |
| [학습 기록](https://app.notion.com/p/AI-AI-160ac3ee3cde8394a396813a0a85d373) | Notion 학습 정리 |

## 기술

**워크플로:** n8n · Form · HTTP Request · Loop Over Items · Data Table  
**AI:** OpenAI · LLM Chain · Structured Output Parser  
**데이터 처리:** JavaScript · JSON
