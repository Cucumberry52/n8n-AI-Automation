# 실행 방법

## 1. 워크플로 가져오기

n8n Editor의 워크플로 메뉴에서 **Import from File**을 선택하고 [ai-agent-safety-auditor.json](../workflows/ai-agent-safety-auditor.json)을 가져옵니다.

Form, Data Table, LLM Chain과 Structured Output Parser 노드를 지원하는 n8n 환경이 필요합니다. 가져온 뒤 OpenAI의 `gpt-5-mini` 모델을 사용할 수 있는 Credentials를 연결합니다.

## 2. 테스트 케이스 테이블

Data tables에서 `agent_safety_test_cases`를 만들고 다음 컬럼을 추가합니다. 한 행이 하나의 테스트입니다.

| 컬럼 | 타입 | 내용 |
| --- | --- | --- |
| `test_id` | String | 테스트 식별자 |
| `risk_category` | String | 아래 다섯 가지 평가 유형 중 하나 |
| `test_question` | String | 대상 에이전트에 보낼 질문 |
| `expected_behavior` | String | 정상 응답의 기준 |
| `failure_condition` | String | 실패로 판단할 조건 |
| `severity` | String | `LOW`, `MEDIUM`, `HIGH`, `CRITICAL` 중 하나 |

`risk_category`는 `ROLE_DEVIATION`, `HALLUCINATION`, `PROMPT_INJECTION`, `PRIVACY`, `OVER_REFUSAL`을 사용합니다. 질문과 기대 행동은 대상 에이전트의 실제 역할에 맞춰 입력합니다.

## 3. 결과 테이블

`agent_safety_results`를 만들고 다음 컬럼을 추가합니다.

| 타입 | 컬럼 |
| --- | --- |
| String | `run_id`, `timestamp`, `agent_name`, `test_id`, `risk_category` |
| String | `test_question`, `expected_behavior`, `actual_response` |
| String | `result`, `severity`, `reason`, `evidence`, `recommendation`, `execution_error` |
| Boolean | `human_review_required` |
| Number | `confidence`, `response_time_ms` |

다음 노드의 Data Table 선택값을 실제 테이블로 바꿉니다. 공개용 JSON의 `REPLACE_WITH_...` 값은 연결 위치를 표시하는 자리표시자입니다.

| 노드 | 연결할 테이블 |
| --- | --- |
| `Get Test Cases` | `agent_safety_test_cases` |
| `Store Test Result` | `agent_safety_results` |
| `Get Run Results` | `agent_safety_results` |

## 4. 인증과 대상 API

| 노드 | 설정 |
| --- | --- |
| `Evaluator Model` | 본인의 OpenAI Credentials 연결 |
| `Call Target Agent` | 대상 API의 인증 방식에 맞는 Credentials 연결 |

`Call Target Agent`는 현재 `POST` 요청에 `multipart/form-data`로 `message`와 `history`를 보냅니다. `message`에는 테스트 질문, `history`에는 `[]`가 들어갑니다. 대상 API가 JSON 본문이나 다른 필드를 요구하면 이 노드의 전송 형식을 맞춥니다.

응답은 일반 텍스트 또는 다음 필드에서 추출합니다.

- `data.message`
- `answer`, `message`, `output`, `response`, `text`

다른 구조를 사용하는 API는 `Normalize Target Response`의 추출 경로를 맞춥니다. 요청 타임아웃은 20초입니다.

## 5. 폼 실행

폼에서 다음 정보를 입력합니다.

| 필드 | 필수 | 내용 |
| --- | --- | --- |
| `agent_name` | O | 대상 에이전트 이름 |
| `agent_role` | O | 수행해야 하는 역할 |
| `agent_description` |  | 서비스 설명 |
| `target_webhook_url` | O | 테스트용 대상 API 주소 |
| `optional_system_prompt` |  | 참고용 시스템 프롬프트 |
| `allowed_data_scope` |  | 허용된 데이터 범위 |
| `allowed_action_scope` |  | 허용된 행동 범위 |

`optional_system_prompt`는 입력값으로 보관되지만 현재 평가 프롬프트에서는 사용하지 않습니다. 허용 데이터·행동 범위는 평가 기준으로 전달되며 대상 API의 접근 권한을 직접 제한하는 기능은 아닙니다.

실행이 끝나면 폼에서 요약과 JSON 보고서를 확인할 수 있습니다. `run_id`로 결과 테이블을 조회하면 같은 실행의 개별 판정을 확인할 수 있습니다.

현재 `test_mode`는 실행 메타데이터이며 외부 호출이나 대상 에이전트의 변경 작업을 차단하지 않습니다. 변경 기능을 가진 에이전트는 테스트 계정과 테스트 데이터에 연결해 실행합니다.

## 참고

- [n8n 워크플로 가져오기·내보내기](https://docs.n8n.io/workflows/export-import/)
- [n8n Data tables](https://docs.n8n.io/data/data-tables/)
