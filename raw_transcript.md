# Raw Transcript — Groupbuying Agent 개발 로그

Tool: Claude Code (claude-sonnet-4-6) via Anthropic Claude Code CLI  
Project: 이커머스 공동구매 에이전트 (채널코퍼레이션)  
Platform: n8n (self-hosted, Docker + Traefik) × Supabase × Google Workspace  
작성일: 2026년 4월 25일

---

## PHASE 1 — 기능 정의 및 아키텍처 설계

### [배경] 고객 페인포인트 인터뷰 3단계 프로세스

에이전트 기능 정의는 LLM에 요구사항을 한 번에 던지는 방식이 아닌, **고객 인터뷰 2회 + 골든 데이터셋 수집**이라는 3단계 방법론을 통해 도출되었습니다.

---

### STEP 1 — 1차 고객 인터뷰 (2026년 4월 21일)

**인터뷰 대상:** 그로스 마케터 (인플루언서 공동구매 캠페인 전 과정 담당)  
**방식:** 1:1 심층 인터뷰 (8분 02초)  
**목적:** 반복 업무 및 Pain Point 발굴

**발굴된 핵심 페인포인트 3가지:**

1. **인게이지먼트 검증 비효율** — 팔로워 10만 계정이어도 실제 반응률이 현저히 낮은 경우가 빈번. 계정에 직접 접속해 게시물을 하나씩 눈으로 확인하는 수작업(건당 10~15분). 잘못된 선택 시 예상 매출(2,000만원)의 85%가 기회비용으로 손실된 실제 사례 존재.
2. **브랜드 핏 판별 수작업** — 운동 카테고리 인플루언서라도 팔로워 연령대가 브랜드 타겟(2030 여성)과 불일치할 수 있음. 콘텐츠 성향·댓글 성향을 수동으로 열람해 판단.
3. **수익 시뮬레이션 부재** — RS율·제품 구성에 따라 순수익이 크게 달라지나, 현재는 스프레드시트에 수식을 직접 입력하거나 경험치 기반으로 어림잡아 결정. 역마진 발생 리스크 존재.

> "공구를 열기 전에 이 조합으로 하면 실제로 얼마가 남는지, RS율을 얼마로 해야 우리가 남는 장사인지를 미리 한번 시뮬레이션해서 볼 수 있었으면 좋겠다."

**1차 인터뷰 결과로 도출된 초기 기능 정의:**

| 기능 | 자동화 난이도 | 기대 효과 | 우선순위 |
|------|-------------|----------|----------|
| 인게이지먼트 수치 재검증 | 낮음 | 높음 | ★★★ 즉시 |
| 브랜드 핏 자동 필터링 | 중간 | 매우 높음 | ★★★ 즉시 |
| RS율·수익 시뮬레이션 자동화 | 낮음 | 매우 높음 | ★★★ 즉시 |

---

### STEP 2 — 2차 고객 인터뷰 (2026년 4월 22일)

**배경:** 1차 인터뷰 기반 워크플로우 초안을 고객에게 시각화 자료로 공유 후 피드백 수렴  
**방식:** 1:1 심층 인터뷰 (7분 11초)  
**목적:** 추가 Pain Point 및 개선 요구사항 수렴

**핵심 피드백:**
> "지금 설계한 에이전트만으로는 우리가 들이는 비용에 비해 효용성이 살짝 부족합니다. 공동 구매의 전체 흐름을 에이전트가 커버해줬으면 합니다."

**추가 발굴된 페인포인트 3가지:**

1. **공구 이벤트 기획 수작업** — 인플루언서 팔로워 규모·RS율에 따라 선착순 증정, 베스트 댓글 체험단 등 이벤트 조합을 담당자가 직접 고민해 제안(건당 30분~1시간). 경험치 의존으로 일관성 부족.
2. **정산 메일 수작업** — 공구 종료 다음 날 오전 9시까지 네이버 스마트스토어 주문 내역 다운로드 → 반품·취소 제외 실 매출 계산 → RS율 기반 수수료 산출 → 메일 작성·발송을 전부 수작업 처리. 야간 종료 공구는 새벽에 처리해야 하는 부담.
3. **협업 히스토리 비자산화** — 인플루언서별 공구 결과(달성률·순수익)가 체계적으로 저장되지 않아 재협업 여부를 개인 기억 또는 단편적 메모에 의존.

> "엑셀 파일을 에이전트에게 보내면 에이전트가 그거를 읽고 전체 판매 수량, 취소된 거 이런 것들을 분석한 다음에 수수료 비용을 알려주고, 담당자가 승인하면 메일까지 직접 써서 보내줬으면 좋겠습니다."

**2차 인터뷰 결과로 확장된 기능 정의:**

| 기능 | 우선순위 |
|------|----------|
| 공구 이벤트 조합 자동 제안 | ★★★ 즉시 |
| 매출 엑셀 분석 + RS 수수료 자동 계산 | ★★★ 즉시 |
| 정산 메일 자동 작성 및 Gmail 발송 | ★★★ 즉시 |
| 협업 결과 Google Sheets 자동 누적 저장 | ★★★ 즉시 |
| 누적 히스토리 기반 재협업 제안 | ★★ 2단계 |
| 지식 기반 RAG Q&A (계약서·가이드라인) | ★★★ 즉시 |

---

### STEP 3 — 골든 데이터셋 수집 (2026년 4월 25일)

**목적:** 에이전트가 실제로 답변해야 할 답변 방향성과 품질 기준을 구체화  
**방법:** 고객에게 인텐트별 이상적인 Q&A 쌍을 직접 작성하도록 요청 → 엑셀로 전달

**수집된 골든 데이터셋 구성:**

| intent | 샘플 수 | 특징 |
|--------|--------|------|
| engagement | 5개 | ER·팔로워 수치 기반 인플루언서 추천 형식 |
| simulation | 5개 | 구체적 수치(매출·RS율·순수익) 계산 포함 |
| settlement | 5개 | 월간 요약·인플루언서별 내역·전환율 분석 포함 |
| settlement_confirm | 5개 | 메일 발송 완료 확인 형식 |
| RAG | 10개 | 계약서·운영 가이드·정책 기반 답변 |

**골든 데이터셋이 설계에 미친 영향:**
- 에이전트 답변이 사용자 질의에 따라 일정한 답변 출력 방식을 원한다는 것을 확인
- settlement_confirm 인텐트: 메일 발송 완료 후 발송 시각·대상을 명시하는 확인 형식으로 통일
- RAG 답변: 규정·절차를 번호/단계별로 구조화하여 전달하는 패턴 채택
- 인텐트 분류 기준(engagement / simulation / settlement / settlement_confirm / rag) 최종 확정

---

### [User Prompt — 아키텍처 확정]
> 3단계 인터뷰와 골든 데이터셋을 바탕으로 단일 워크플로우로 구성하고, Gemini API로 인텐트 분류 후 Switch로 5개 분기하는 방식으로 진행해주세요.

### [Model Response]
인터뷰에서 도출된 요구사항과 골든 데이터셋 패턴을 반영해 최종 기능 5가지와 아키텍처를 확정합니다:

1. **인게이지먼트 분석** — 팔로워 대비 좋아요/댓글 비율로 인플루언서 영향력 수치화
2. **수익 시뮬레이션** — RS율 × 예상 판매량으로 브랜드·인플루언서 수익 사전 예측
3. **정산 계산 및 메일 발송** — 공구 종료 후 확정 판매액 기준 자동 정산 및 Gmail 발송
4. **지식 기반 RAG Q&A** — 계약서·가이드라인 문서를 벡터 DB에 저장하여 자연어 질의응답
5. **자동 지식 수집** — Google Drive 파일 감지 → 청킹 → 임베딩 → Supabase 자동 저장

아키텍처 결정: 단일 워크플로우 + 인텐트 라우팅(Switch 노드). RAG 챗봇에서 tool calling 방식을 n8n에서는 Switch 노드를 활용하여 각각의 설계된 기능을 명확하게 구현하도록 설정.

---

## PHASE 2 — 인텐트 라우팅 구현

### [User Prompt]
> Webhook으로 query가 들어오면 Gemini가 intent를 분류하고 Switch로 분기하는 흐름을 만들어주세요.

### [Model Response]
```
Webhook Trigger → Intent 분류(HTTP Request → Gemini) → Intent 파싱(Code) → Switch
```

Intent 분류를 위한 Gemini 프롬프트:
```
다음 사용자 메시지의 의도를 아래 중 하나로 분류하고 JSON으로만 응답하세요.
- engagement: 인게이지먼트 분석
- simulation: 수익 시뮬레이션
- settlement: 정산 계산
- settlement_confirm: 정산 메일 발송
- rag: 지식 기반 질의

사용자 메시지: {{ $json.body.query }}
응답 형식: {"intent": "분류결과"}
```

Switch 노드 5개 분기(index 0~4)로 각 기능에 연결.

---

### [에러/디버깅 #1] Intent 파싱 실패

**1차 시도 — 단순 JSON.parse 적용:**
```javascript
// Code 노드 초기 코드
const text = $json.candidates[0].content.parts[0].text;
return [{ json: JSON.parse(text) }];
```

**에러 발생:**
```
SyntaxError: Unexpected token ` in JSON at position 0
```

**원인 파악:** Gemini가 순수 JSON 대신 마크다운 코드블록으로 응답을 감쌈:
```
```json
{"intent": "rag"}
```
```

**재시도 — 코드블록 제거 로직 추가:**
```javascript
const raw = $json.candidates?.[0]?.content?.parts?.[0]?.text || '';
const cleaned = raw.replace(/```json\n?/g, '').replace(/```\n?/g, '').trim();
return [{ json: JSON.parse(cleaned) }];
```

**결과:** ✅ 인텐트 파싱 정상 동작

---

## PHASE 3 — 인게이지먼트 분석 / 수익 시뮬레이션

### [User Prompt]
> 인게이지먼트는 Google Sheets에서 읽어서 계산하고, 수익 시뮬레이션도 상품DB 시트에서 RS율·원가율을 가져와서 계산해주세요.

### [Model Response]

**인게이지먼트 흐름:**
```
Switch[0] → 인게이지먼트 시트 읽기 → 인게이지먼트 포맷변환(Code)
  → Message a model(Gemini) → 인게이지먼트 응답파싱 → respondToWebhook
```

**수익 시뮬레이션 흐름:**
```
Switch[1] → 상품DB 읽기 → 공구 RS 파악 → 수익 계산(Code)
  → Message a model1(Gemini) → 시뮬레이션 응답파싱 → 수익 계산 응답
```

**수익 계산 공식 (Code 노드):**
```javascript
const 확정판매액 = toNum(row['확정 판매액']);
const RS율 = toNum(row['RS율']) / 100;
const 원가율 = toNum(row['원가율']) / 100;
const 인플루언서수수료 = 확정판매액 * RS율;
const 원가합계 = 확정판매액 * 원가율;
const 브랜드순수익 = 확정판매액 - 원가합계 - 인플루언서수수료;
```

**실제 모델 응답 예시 (수익 시뮬레이션 결과):**
```
📊 수익 시뮬레이션 결과

상품: 딥클린 초음파 클렌저
확정 판매액: 30,000,000원
RS율: 30%

💰 정산 내역
인플루언서 수수료: 9,000,000원
원가 합계 (40%): 12,000,000원
브랜드 순수익: 9,000,000원

✅ 수익성 평가: 브랜드 마진 30% — 정상 범위입니다.
```

---

## PHASE 4 — 정산 계산 및 메일 자동 발송

### [User Prompt]
> 정산 계산이 제대로 안 됩니다. 이메일 조회 노드에서 항상 0건이 반환돼요.

### [Model Response]
n8n 실행 API로 최근 실행 로그를 확인하겠습니다.

```bash
curl https://n8n.srv1001864.hstgr.cloud/api/v1/executions/[id]?includeData=true
# → lastNodeExecuted: "이메일 조회", status: "error"
```

세 가지 문제를 발견했습니다.

---

### [에러/디버깅 #2] n8n 태스크 러너 fatal crash

**1차 시도 — 시트명으로 인플루언서 ID 판별:**
```javascript
// 정산 계산 Code 노드 초기 버전
const sheetName = $node["주문 데이터 읽기"].parameter.sheetName;
const inf = infList.find(r => r['아이디'] === sheetName);
```

**에러 발생:**
```
InternalTaskRunnerDisconnectAnalyzer.toDisconnectError
워크플로우 실행이 응답 없이 중단됨 (exit code 28)
```

**원인 파악:** n8n 외부 태스크 러너 모드에서 `$node["nodeName"].parameter` 접근은 JavaScript try-catch로도 잡히지 않는 프로세스 수준의 치명적 오류. 실행 자체가 죽어버림.

**재시도 — 웹훅 바디에서 influencer_id 직접 수신으로 설계 변경:**

> **[설계 변경 #1] 인플루언서 ID 탐지 방식 변경**  
> - 변경 전: 노드 파라미터(`sheetName`)에서 ID 추출 시도  
> - 변경 후: 웹훅 호출 시 `influencer_id` 필드를 명시적으로 포함하도록 인터페이스 변경  
> - 이유: `$node[].parameter`는 n8n 외부 태스크 러너에서 사용 불가. 데이터는 웹훅 바디를 통해 전달하는 것이 n8n 표준 패턴

```javascript
// 수정된 코드
let 인플루언서ID = null;
try {
  인플루언서ID = $('Webhook Trigger').first().json.body?.influencer_id || null;
} catch(e) {}

// fallback: 가장 최근 공구 날짜 행 사용
if (!인플루언서ID) {
  inf = infList.slice().sort((a, b) =>
    new Date(b['공구 날짜'] || 0) - new Date(a['공구 날짜'] || 0)
  )[0] || {};
}
```

**결과:** ✅ 태스크 러너 crash 해소

---

### [에러/디버깅 #3] 이메일 조회 0건 반환

**원인 파악:** `이메일 조회` 노드의 filtersUI에 `lookupValue`가 없는 필터 항목이 존재:
```json
[
  { "lookupColumn": "아이디", "lookupValue": "={{ $json['아이디'] }}" },
  { "lookupColumn": "담당자 이메일" }   ← lookupValue 누락
]
```

Google Sheets 노드는 `lookupValue`가 비어있으면 조건을 만족하는 행이 없다고 판단 → 0건 반환.

**수정:** `담당자 이메일` 필터 제거, `아이디` 단일 필터만 유지.

**결과:** ✅ 이메일 조회 정상 반환

---

### [에러/디버깅 #4] appendOrUpdate 중복 행 삽입

**1차 수정 후 재실행 결과:** 정산 시트에 기존 6행 외에 새로운 6행이 추가로 삽입됨 (총 12행).

**원인 파악:**
```
정산 시트 상태:
  행 1~3: 아이디="미확인", 공구날짜=2026.4.25
  행 4~6: 아이디="andantenine", 공구날짜=2026.5.15

메일 발송 업데이트 matchingColumns: ["아이디", "공구 날짜", "상품명"]
→ 재실행 시 "andantenine" 값으로 매칭 시도
→ "미확인"으로 저장된 행 1~3은 매칭 실패
→ upsert 대신 insert 실행 → 중복 삽입
```

**재시도 — matchingColumns에서 아이디 제거:**

> **[설계 변경 #2] 정산 업데이트 매칭 키 변경**  
> - 변경 전: `matchingColumns: ["아이디", "공구 날짜", "상품명"]`  
> - 변경 후: `matchingColumns: ["공구 날짜", "상품명"]`  
> - 이유: 아이디가 "미확인"으로 저장된 케이스에서 매칭이 깨지는 문제 회피. 날짜+상품명 조합이 정산 행의 실질적 고유키

**결과:** ✅ 정산 업데이트 정상 upsert

---

### [User Prompt]
> 정산 워크플로우 재실행 테스트해봅시다.

### [Model Response — 테스트 실행]

```bash
curl -X POST https://n8n.srv1001864.hstgr.cloud/webhook/influencer-query \
  -H "Content-Type: application/json" \
  -d '{"query":"정산 메일 보내줘","influencer_id":"andantenine"}' \
  --max-time 90
```

**실제 응답:**
```
정산 메일 발송 완료 / 인플루언서: @andantenine / 발송 완료 처리된 상품 수: 6건
```

실행 확인:
```bash
curl https://n8n.srv1001864.hstgr.cloud/api/v1/executions?workflowId=iEr5iv4VpURCS303&limit=3
# id=351 | status=success
# id=350 | status=success
```

✅ 정산 계산 → Gmail 발송 → 시트 업데이트 전 구간 성공

---

## PHASE 5 — LightRAG 지식 기반 구현

### [User Prompt]
> 계약서나 가이드라인 PDF를 Google Drive에 올리면 자동으로 벡터 DB에 저장되고, 인플루언서가 질문하면 RAG로 답변하는 기능을 만들어주세요.

### [Model Response]
두 개의 독립 파이프라인으로 설계합니다.

**문서 수집 파이프라인 (Drive Trigger):**
```
구글 드라이브 파일 감지 → 파일 다운로드 → Extract from File
  → Code in JavaScript (청킹, ~500자 단위)
    ├── 임베딩 API 호출 (Gemini text-embedding-004 HTTP Request)
    │     → 벡터 포맷 (Code) → 채널 벡터 저장 (Supabase)
    └── Basic LLM Chain (엔티티·관계 추출)
          → Code in JavaScript1 → 엔티티/관계 분기 (Switch)
                ├── Create a row (entities 테이블)
                └── 관계 저장 (relationships 테이블)
```

**RAG 질의 파이프라인 (Webhook):**
```
Switch[4] → 관계 및 이력 검색 (PostgreSQL)
  → AI Agent (Gemini + Supabase Vector Store + 세션 메모리)
  → 지식데이터 응답
```

---

### [에러/디버깅 #5] 임베딩 노드 invalid syntax

**에러 발생:**
```
노드: 임베딩 API 호출
메시지: invalid syntax
```

**원인 파악:** `jsonBody` 필드에 JavaScript 객체 리터럴 + 단따옴표 사용:
```
{{ {model: 'models/text-embedding-004', content: {parts: [{text: $json.chunk}]}} }}
```
n8n의 HTTP Request 노드는 `specifyBody: "json"` 모드에서 필드를 JSON으로 파싱하려 시도 → JS 단따옴표가 JSON 스펙 위반으로 파싱 실패.

**1차 수정 시도 — JSON.stringify 래핑:**
```
={{ JSON.stringify({model: "models/text-embedding-004", content: {parts: [{text: $json.chunk}]}}) }}
```

**재시도 결과:** 여전히 동일 오류. HTTP Request 노드에서 `specifyBody: "json"` + 전체 필드를 expression으로 처리하는 방식의 호환성 문제.

**2차 수정 — JSON 구조를 직접 작성 + 동적 값만 인라인 표현식으로:**
```json
{
  "model": "models/text-embedding-004",
  "content": {
    "parts": [{ "text": "{{ $json.chunk }}" }]
  }
}
```

**결과:** ✅ Gemini 임베딩 API 정상 호출, 768차원 벡터 반환

---

### [에러/디버깅 #6] respondToWebhook JSON 파싱 오류

**에러 발생:**
```
노드: 지식데이터 응답
메시지: Invalid JSON in 'Response Body' field
Check that the syntax of the JSON in the 'Response Body' parameter is valid
```

**원인 파악:** `responseBody` 설정값:
```
={\n  "message": "{{ $json.output }}"\n}
```
`$json.output`(LLM 답변)에 줄바꿈(`\n`)이나 쌍따옴표(`"`)가 포함되면 JSON 구조 파괴.

**수정:**
```
={{ JSON.stringify({message: $json.output}) }}
```
`JSON.stringify`가 특수문자를 자동 이스케이프하여 항상 유효한 JSON 보장.

**실제 응답 예시 (RAG 질의 결과):**
```json
{
  "message": "공구 계약서 필수 항목은 다음과 같습니다.\n\n기본 정보: 계약 당사자 정보, 계약 체결일 및 유효 기간\n캠페인 조건: RS율 또는 고정 수수료, 최소 판매 보장 수량\n콘텐츠 관련: 게시 채널 및 형식, 광고 표기 의무\n정산 관련: 정산 기준일 및 반품 처리 기준\n위약 관련: 계약 해지 조건 및 분쟁 관할 법원"
}
```

**결과:** ✅ 정상 응답 반환

---

### [에러/디버깅 #7 + 설계 변경 #3] 관계 및 이력 검색 미실행 문제

### [User Prompt]
> AI Agent가 관계 및 이력 검색 툴을 항상 쓰지 않아요. 강제로 실행되게 할 수 있나요? 혹은 rag 부분 워크플로우 수정이 이뤄져야 할까요?

### [Model Response]

**현상 분석:**

현재 구조에서 `관계 및 이력 검색`은 AI Agent의 tool로 연결됨:
```
Switch[4] → AI Agent
              ├── tool: 관계 및 이력 검색 (PostgreSQL)
              └── tool: Supabase Vector Store
```

AI Agent는 각 요청에 대해 "어떤 tool을 호출할지" 스스로 판단합니다. 이미 시스템 프롬프트에 "두 도구를 함께 쓰라"고 명시했음에도 Supabase Vector Store만으로 충분하다고 판단하면 `관계 및 이력 검색`을 건너뜁니다. 이는 Agent 아키텍처의 본질적 특성으로 프롬프트만으로는 100% 보장이 불가합니다.

**해결 방안 두 가지 제시:**

1. **시스템 프롬프트 강화 (간단하지만 불안정):** "반드시 관계 및 이력 검색 툴을 먼저 호출하라" 규칙 추가 → AI가 무시할 가능성 있음
2. **워크플로우 구조 변경 (확실한 방법):** `관계 및 이력 검색`을 tool에서 분리해 항상 실행되는 독립 노드로 전환

> **[설계 변경 #3] RAG 아키텍처 — tool 기반 → 순차 실행 구조로 전환**  
> - 변경 전: 두 DB 조회 모두 AI Agent tool (AI 재량 호출)  
> - 변경 후: PostgreSQL 조회를 메인 플로우 선행 노드로 분리, 결과를 시스템 프롬프트에 주입. Supabase 벡터 검색만 tool로 유지  
> - 이유: AI Agent tool 호출은 AI의 판단에 의존하므로 결정론적 실행 보장 불가. 반드시 실행되어야 하는 데이터 조회는 워크플로우 레벨에서 강제하는 것이 올바른 설계

**변경 후 구조:**
```
Switch[4] → 관계 및 이력 검색 (항상 실행, 독립 노드, $json.query로 ILIKE 검색)
  → AI Agent
      ├── tool: Supabase Vector Store (유지)
      ├── memory: 세션ID 기억
      └── systemPrompt: 관계 데이터 동적 주입
  → 지식데이터 응답
```

**수정된 AI Agent 시스템 프롬프트:**
```javascript
={{ "당신은 채널코퍼레이션의 공동구매 전문 AI 어시스턴트입니다.\n\n"
   + "## 사전 조회된 관계/이력 데이터\n"
   + JSON.stringify($('관계 및 이력 검색').all().map(i => i.json), null, 2)
   + "\n\n## 지식 검색 전략\n"
   + "Supabase Vector Store 도구를 활용하여 관련 문서를 검색하세요.\n\n"
   + "## 답변 원칙\n"
   + "- 위의 관계/이력 데이터와 벡터 검색 결과를 함께 활용하세요\n"
   + "- 검색 결과가 없으면 모르는 것은 모른다고 답하세요\n"
   + "- 한국어로 답변하세요"
}}
```

**검증:**
```bash
curl -X POST .../webhook/influencer-query \
  -d '{"query":"andantenine 인플루언서와 관련된 브랜드나 상품이 있나요?","influencer_id":"andantenine"}'
# 실행 id=376 | status=success ✅
```

---

## PHASE 6 — 인프라 트러블슈팅 (Docker / Traefik)

### [User Prompt]
> 지금 도커 터미널에서 환경변수를 좀 바꿨거든. 그랬더니 지금 웹훅 작동이 안되고 있어. Webhook Trigger가 제대로 작동하지 않고 있거든. 문제 파악 한번 해줄 수 있어?

### [Model Response — 1차 진단 시도]

```bash
# n8n API로 워크플로우 상태 확인
curl https://n8n.srv1001864.hstgr.cloud/api/v1/workflows/iEr5iv4VpURCS303
# → 404 page not found
```

모든 엔드포인트에서 404 반환. n8n 프로세스 문제가 아닌 Traefik 라우팅 문제로 판단.

```bash
docker ps | grep n8n
# a054b0283216  traefik   Up 20 minutes  0.0.0.0:80->80, 0.0.0.0:443->443
# 95d4708682b5  n8n       Up 20 minutes  127.0.0.1:5678->5678
```

컨테이너는 정상 실행 중. n8n 로그 확인:

```bash
docker logs n8n-n8n-1 --tail 50
# → Activated workflow "Groupbuying Agent"
# → Editor is now accessible via: https://n8n.srv1001864.hstgr.cloud
```

n8n 자체는 정상. Traefik 로그 확인:

```bash
docker logs n8n-traefik-1 --tail 30
# → error="field not found, node: accessControlAllowOrigin"
```

---

### [에러/디버깅 #8] Traefik 라우터 전체 무효화

**원인 확정:** `docker-compose.yml`에 추가한 `n8n-cors` 미들웨어에 존재하지 않는 Traefik 필드명 사용:

```yaml
# 잘못된 설정 (Traefik에 없는 필드)
- traefik.http.middlewares.n8n-cors.headers.accessControlAllowOrigin=*
- traefik.http.middlewares.n8n-cors.headers.accessControlAllowMethods=GET,POST,OPTIONS
- traefik.http.middlewares.n8n-cors.headers.accessControlAllowHeaders=Content-Type,Authorization
```

Traefik이 `n8n-cors` 미들웨어 파싱 실패 → 해당 미들웨어를 참조하는 n8n 라우터(`traefik.http.routers.n8n.middlewares=n8n@docker,n8n-cors`) 전체 무효화 → 모든 엔드포인트 404.

> **[설계 변경 #4] CORS 처리 레이어 변경**  
> - 변경 전: Traefik 미들웨어 레벨에서 CORS 헤더 주입 시도  
> - 변경 후: n8n 환경변수(`N8N_CORS_ENABLED=true`, `N8N_CORS_ALLOWED_ORIGINS=*`)로 애플리케이션 레벨에서 처리  
> - 이유: Traefik의 headers 미들웨어 CORS 필드는 camelCase가 아닌 다른 명명 규칙을 사용. n8n 자체 CORS 기능이 있으므로 중복 구성 불필요

**수정 내용 (docker-compose.yml):**
```yaml
# 제거한 3줄
- traefik.http.middlewares.n8n-cors.headers.accessControlAllowOrigin=*
- traefik.http.middlewares.n8n-cors.headers.accessControlAllowMethods=GET,POST,OPTIONS
- traefik.http.middlewares.n8n-cors.headers.accessControlAllowHeaders=Content-Type,Authorization

# 수정한 1줄
# 변경 전: traefik.http.routers.n8n.middlewares=n8n@docker,n8n-cors
# 변경 후: traefik.http.routers.n8n.middlewares=n8n@docker
```

```bash
cd /docker/n8n && docker compose up -d
# ✅ Container n8n-n8n-1 Started
# ✅ Container n8n-traefik-1 Running

curl -o /dev/null -w "%{http_code}" https://n8n.srv1001864.hstgr.cloud/healthz
# → 200
```

**결과:** ✅ 전 엔드포인트 정상 복구

---

## 최종 테스트 결과

```bash
# 인게이지먼트 분석
curl -X POST .../webhook/influencer-query \
  -d '{"query":"인게이지먼트 분석해줘","influencer_id":"andantenine"}'
# → {"message": "andantenine 인게이지먼트율: 4.2% (팔로워 52,000명 기준)..."}

# 수익 시뮬레이션
curl -X POST .../webhook/influencer-query \
  -d '{"query":"수익 시뮬레이션 해줘","influencer_id":"andantenine"}'
# → {"message": "📊 수익 시뮬레이션 결과\n..."}

# 정산 메일 발송
curl -X POST .../webhook/influencer-query \
  -d '{"query":"정산 메일 보내줘","influencer_id":"andantenine"}'
# → "정산 메일 발송 완료 / 인플루언서: @andantenine / 발송 완료 처리된 상품 수: 6건"

# RAG 질의
curl -X POST .../webhook/influencer-query \
  -d '{"query":"안녕하세요","influencer_id":"andantenine"}'
# → {"message": "안녕하세요! 채널코퍼레이션 공동구매 전문 AI 어시스턴트입니다. 무엇을 도와드릴까요?"}
```

| 기능 | 결과 |
|---|---|
| 인게이지먼트 분석 | ✅ |
| 수익 시뮬레이션 | ✅ |
| 정산 계산 | ✅ |
| 정산 메일 발송 | ✅ 6건 |
| RAG 질의응답 | ✅ |
| 일반 대화 | ✅ |
| 문서 자동 수집 (Drive) | ✅ 벡터·그래프 저장 |

**Live Webhook URL:** `POST https://n8n.srv1001864.hstgr.cloud/webhook/influencer-query`

```json
{ "query": "질문 내용", "influencer_id": "andantenine" }
```

---

## 주요 설계 변경 요약

| # | 변경 내용 | 이유 |
|---|---|---|
| 1 | 인플루언서 ID 탐지: 노드 파라미터 → 웹훅 바디 | `$node[].parameter`가 외부 태스크 러너에서 fatal crash 유발 |
| 2 | matchingColumns: `[아이디, 날짜, 상품명]` → `[날짜, 상품명]` | "미확인" 아이디로 저장된 행과 매칭 실패로 중복 삽입 발생 |
| 3 | RAG: tool 기반 → 순차 실행 | AI Agent tool 호출 불확실성 제거, 관계 DB 항상 실행 보장 |
| 4 | CORS: Traefik 미들웨어 → n8n 환경변수 | Traefik 필드명 오류로 라우터 전체 404, n8n 자체 CORS로 충분 |

## 주요 기술 결정

| 결정 | 이유 |
|---|---|
| 단일 워크플로우 | Execute Workflow 사용 시 디버깅 복잡도 증가 |
| Gemini text-embedding-001 | API 키 단일 관리, 3072차원 고품질 임베딩 |
| PostgreSQL 세션 메모리 | Redis 별도 운영 없이 기존 Supabase DB 재활용 |
| Google Sheets 정산 저장 | 비개발자 팀원이 직접 확인·수정 가능한 투명한 데이터 레이어 |
| n8n self-hosted (Docker) | 데이터 외부 유출 없는 온프레미스 운영, API 비용 직접 제어 |
