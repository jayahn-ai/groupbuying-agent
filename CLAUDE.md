# CLAUDE.md

## Project Purpose
n8n 워크플로우 기반 공동구매 에이전트
워크플로우는 **n8n MCP**를 통해 클라우드 n8n 인스턴스에 생성·관리된다.

## 절대 하면 안 되는 것 ❌
- n8n credential 직접 수정 금지
- 검증(`n8n_validate_workflow`) 없이 워크플로우 완료 선언 금지
- 명시적으로 지정되지 않은 노드 수정 금지 (요청된 노드만 수정)

## 노드 수정 규칙 🔧
- 사용자가 문제를 제기하거나 수정 요청한 노드만 변경한다
- 작업 완료 후 반드시 수정된 노드 목록을 아래 형식으로 보고한다
  > 수정된 노드: `노드명1`, `노드명2`
- 연관 노드 수정이 불가피할 경우 사전에 사용자에게 확인을 받는다

## 검증 실패 처리
- 검증 오류 발생 시 자동 수정 1회 재시도
- 재시도 후에도 실패하면 오류 내용을 사용자에게 보고하고 중단

## Workflow Architecture (현행)
- 단일 워크플로우로 구성
- Execute Workflow 노드 사용 금지
- 워크플로우명: 기능을 나타내는 명사구 (`공동구매 신청 처리`)

## n8n MCP
- Instance: `https://n8n.srv1001864.hstgr.cloud`

## nodeType Format
| 용도 | 형식 |
|------|------|
| search/validate | `nodes-base.httpRequest` |
| create/update | `n8n-nodes-base.httpRequest` |
| LangChain | `@n8n/n8n-nodes-langchain.agent` |

## Skills (.claude/skills/)

사용 순서: `n8n-mcp-tools-expert` → 작업별 스킬 → `n8n-validation-expert`

| Skill | 사용 시점 |
|-------|-----------|
| `n8n-mcp-tools-expert` | MCP 도구 선택·호출 |
| `n8n-workflow-patterns` | 워크플로우 패턴 설계 |
| `n8n-node-configuration` | 노드 설정·옵션 확인 |
| `n8n-code-javascript` | Code 노드 JS 작성 |
| `n8n-code-python` | Code 노드 Python 작성 |
| `n8n-expression-syntax` | n8n 표현식 문법 |
| `n8n-validation-expert` | 검증·오류 수정 |

## Node 작성 규칙
- 노드명: 명확한 영어 동사구 (`Download Certificate`, `Parse OCR Result`)
- credential · operation · 필수 필드 누락 없이 완성
- AI 노드 operation은 반드시 `get_node`로 존재 여부 확인 후 사용
