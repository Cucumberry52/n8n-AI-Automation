# 구현 상세

## 노드 구성

| 단계 | 노드 | 역할 |
| --- | --- | --- |
| 입력 | `Audit Request Form` | 에이전트 정보와 대상 API 입력 |
| 초기화 | `Init Run` | `run_id`, 시각, 입력값 설정 |
| 조회 | `Get Test Cases` | 테스트 케이스 전체 조회 |
| 반복 | `Loop Over Test Cases` | 한 번에 한 건씩 실행하고 전체 완료 시 집계 단계로 이동 |
| 측정 | `Mark Start` | 요청 시작 시각 기록 |
| 호출 | `Call Target Agent` | 질문을 대상 API에 전송 |
| 정규화 | `Normalize Target Response` | 답변 추출, HTTP 오류·빈 응답 처리, 응답 시간 계산 |
| 평가 | `Evaluate Response` | 대상의 역할·허용 범위와 테스트 기준에 따라 응답 평가 |
| 모델 | `Evaluator Model` | `gpt-5-mini`, reasoning effort `low` |
| 형식 | `Evaluation Schema` | JSON 출력 구조 정의와 자동 보정 연결 |
| 저장 | `Store Test Result` | 테스트 정보·실제 응답·판정·응답 시간 저장 |
| 재조회 | `Get Run Results` | 현재 `run_id`의 결과 조회 |
| 집계 | `Calculate Metrics & Build Report` | 통계와 보고서 객체 생성 |
| 표시 | `Show Audit Report` | 폼 완료 화면에 요약과 상세 JSON 표시 |

## 평가 결과

평가 모델은 테스트별 기대 행동과 실패 조건을 실제 응답에 대조합니다. Structured Output Parser는 아래 필드를 가진 JSON을 요구합니다.

| 필드 | 내용 |
| --- | --- |
| `result` | `PASS`, `FAIL`, `REVIEW_REQUIRED` |
| `risk_type` | 다섯 가지 평가 유형 중 하나 |
| `severity` | `LOW`, `MEDIUM`, `HIGH`, `CRITICAL` |
| `reason` | 판정 이유 |
| `evidence` | 근거 |
| `recommendation` | 개선 제안 |
| `human_review_required` | 사람의 검토 필요 여부 |
| `confidence` | 판정 신뢰도 |

HTTP 오류나 빈 응답은 정규화 단계에서 기록하고, 평가 프롬프트는 이를 `REVIEW_REQUIRED`로 판정하도록 지시합니다. 평가 출력 자체가 없으면 저장 단계에서 같은 상태로 대체합니다.

신뢰도 0.75 미만 또는 모호한 응답에 대한 사람의 검토 표시는 평가 모델에 전달한 지시입니다. 저장 이후 별도의 코드로 임계값을 강제하는 구조는 아닙니다. `confidence`도 프롬프트에서 0~1을 요구하며 스키마에는 숫자 타입만 지정되어 있습니다.

## 보고서 집계

| 항목 | 집계 방식 |
| --- | --- |
| 전체 점수 | 전체 결과 중 `PASS` 비율 × 100 |
| 유형별 결과 | 전체 건수, 통과·실패·검토 건수와 통과율 |
| 주요 발견 사항 | `FAIL`이면서 `HIGH` 또는 `CRITICAL`인 결과 |
| 반복 실패 패턴 | 같은 평가 유형에서 실패가 두 건 이상 발생한 경우 |
| 개선 제안 | 실패·검토 결과의 제안을 중복 제거하여 정리 |
| 사람의 검토 목록 | 검토 표시가 있거나 `REVIEW_REQUIRED`인 결과 |

전체 점수는 위험도 가중 점수가 아닌 **테스트 통과율**입니다. 검토 대상도 전체 건수에 포함되며, 각 유형에 테스트가 없으면 해당 통과율은 0으로 계산됩니다. 보고서는 위험 신호와 검토 근거를 정리하는 용도로 구성되어 있습니다.

`response_time_ms`는 대상 API 호출 시간을 측정하며 평가 모델 실행 시간은 포함하지 않습니다. `optional_system_prompt`는 실행 초기화에서 보관하지만 평가 모델에는 전달하지 않습니다.

## 공개용 워크플로

원본의 처리 흐름과 노드 연결은 유지하고, Credentials 참조와 기존 Data Table ID·Webhook ID를 제거했습니다. Data Table의 실제 행 데이터와 실행 이력은 워크플로 JSON에 포함되지 않으므로 실행 환경에서 따로 준비합니다.
