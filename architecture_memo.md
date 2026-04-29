# 아키텍처 메모 — Groupbuying Agent

Project: 이커머스 공동구매 에이전트 (채널코퍼레이션)  
작성일: 2026년 4월 25일

---

## 1. 도구 선택 이유

### n8n (워크플로우 오케스트레이터)

| 항목 | 내용 |
|---|---|
| 선택 이유 | 채널톡의 ALF 구현 방식이 사용자 최적화 기반 워크플로우 구축이기에 이와 유사하게 시각적 워크플로우 구축 내용을 보여줄 수 있는 n8n을 선택. 에이전트 구축에 필요한 내장 기능들이 충분히 갖춰져 있음 (Google Sheets, Gmail, Supabase 등). |
| Self-hosted 선택 이유 | n8n Cloud 대비 월 비용 절감 (~$24/월 → ~$10/월 VPS). 민감한 정산 데이터·API 키가 외부 SaaS로 전송되지 않음. |
| 배포 방식 | Docker Compose + Github Pages |

---

### Gemini API (Google AI)

| 항목 | 내용 |
|---|---|
| 선택 이유 | 인텐트 분류·RAG 응답·정산 리포트 생성·인게이지먼트 분석 등 모든 LLM 작업을 단일 API 키로 통합 관리. Google Workspace(Sheets, Drive, Gmail)와 동일 GCP 생태계로 인증 일관성 유지. |
| 사용 모델 | `gemini-2.0-flash`: 인텐트 분류, 분석 리포트 생성 (빠른 응답, 저비용) |
| | `gemini-2.0-flash`: AI Agent Chat Model (RAG 대화) |
| | `text-embedding-001`: 문서 벡터 임베딩 (3072차원, 높은 한국어 품질) |
| 대안 검토 | OpenAI GPT-4o: 성능 우수하나 비용 3~5배. Claude API: 한국어 품질 우수하나 n8n 내장 노드 미지원으로 커스텀 HTTP 노드 필요. |

---

### Supabase (데이터베이스 + 벡터 스토어)

| 항목 | 내용 |
|---|---|
| 선택 이유 | PostgreSQL + pgvector 확장으로 그래프 DB와 벡터 DB를 단일 인스턴스에서 운영. n8n에 Supabase 내장 노드 존재 (별도 HTTP 구성 불필요). 세션 메모리(PostgreSQL Chat Memory), 엔티티·관계 저장, 벡터 유사도 검색 모두 단일 DB로 처리. |
| 테이블 구성 | `channel`: 문서 벡터 청크 저장 (content, embedding, metadata) |
| | `entities`: 추출된 엔티티 저장 (인플루언서, 브랜드, 상품) |
| | `relationships`: 엔티티 간 관계 저장 (source, target, relation_type) |
| | `n8n_chat_histories`: 세션별 대화 이력 저장 |
| 대안 검토 | Pinecone: 벡터 전용이라 그래프 데이터 별도 DB 필요. Weaviate: 자체 호스팅 복잡도 증가. |

---

### Google Workspace (Sheets · Drive · Gmail)

| 항목 | 내용 |
|---|---|
| 선택 이유 | 정산 데이터·인플루언서 정보·상품 DB를 Google Sheets에 저장함으로써 비개발자 팀원이 별도 도구 없이 직접 조회·수정 가능. Google Drive Trigger로 PDF 업로드 시 자동 지식 수집 파이프라인 실행. Gmail로 정산 메일 발송 시 발신자 도메인 신뢰도 유지. |
| Google Sheets 역할 | 인플루언서 컨택 정보, 상품 DB, 인게이지먼트 데이터, 공구 정산 기록 저장 |
| Google Drive 역할 | 계약서·가이드라인 PDF 수집 트리거 (신규 파일 감지 → 자동 임베딩) |

---

## 2. 데이터 흐름

### 전체 아키텍처 다이어그램

```
[인플루언서 / HTML 클라이언트]
        │
        │ POST /webhook/influencer-query
        │ { "query": "...", "influencer_id": "..." }
        ▼
[Webhook Trigger]
        │
        ▼
[Intent 분류] ── Gemini API ──▶ gemini-2.0-flash
        │           응답: {"intent": "engagement"|"simulation"|"settlement"|"settlement_confirm"|"rag"}
        ▼
[Intent 파싱 Code] ── 마크다운 코드블록 제거 후 JSON 파싱
        │
        ▼
[Switch] ──────┬─ [0] engagement ──────────────────────────────────────────────┐
               ├─ [1] simulation ──────────────────────────────────────────┐   │
               ├─ [2] settlement ──────────────────────────────────────┐   │   │
               ├─ [3] settlement_confirm ──────────────────────────┐   │   │   │
               └─ [4] rag ─────────────────────────────────────┐   │   │   │   │
                                                               │   │   │   │   │
                                                               ▼   ▼   ▼   ▼   ▼
                                                         [각 기능 파이프라인]
```

---

### Flow A — 인게이지먼트 분석

```
Switch[0]
  → 인게이지먼트 시트 읽기 (Google Sheets: 인플루언서_컨택 시트)
  → 인게이지먼트 포맷변환 (Code: 팔로워수·좋아요·댓글 정제)
  → Message a model (Gemini: 인게이지먼트율 계산 및 평가 리포트 생성)
  → 인게이지먼트 응답파싱 (Code: 응답 텍스트 추출)
  → 인게이지먼트 응답 (respondToWebhook)
```

**데이터 형태:**
```
입력: { query: "인게이지먼트 분석해줘", influencer_id: "andantenine" }
중간: { 팔로워: 52000, 좋아요평균: 2184, 댓글평균: 312 }
출력: { message: "인게이지먼트율 4.2% — 동일 규모 인플루언서 평균(2.8%) 대비 우수..." }
```

---

### Flow B — 수익 시뮬레이션

```
Switch[1]
  → 상품DB 읽기 (Google Sheets: 상품DB 시트)
  → 공구 RS 파악 (Google Sheets: RS율·원가율 조회)
  → 수익 계산 (Code: 인플루언서 수수료·브랜드 순수익 계산)
  → Message a model1 (Gemini: 시뮬레이션 결과 리포트 생성)
  → 시뮬레이션 응답파싱 (Code)
  → 수익 계산 응답 (respondToWebhook)
```

**데이터 형태:**
```
입력: { query: "수익 시뮬레이션 해줘", influencer_id: "andantenine" }
중간: { RS율: 30, 원가율: 40, 예상판매액: 30000000 }
출력: { message: "인플루언서 수수료: 9,000,000원 / 브랜드 순수익: 9,000,000원..." }
```

---

### Flow C — 정산 계산

```
Switch[2]
  → 주문 데이터 읽기 (Google Sheets: 주문 시트)
  → 주문 집계 (Code: 상품별 판매량 집계)
  → 공구 가격 조회 (Google Sheets: 상품 가격 매핑)
  → 인플루언서 정보 파악 (Google Sheets: RS율·담당자 조회)
  → 정산 계산 (Code: 확정 판매액 × RS율 계산)
  → 공구 정산 기록 (Google Sheets: 정산 결과 upsert)
  → Message a model2 (Gemini: 정산 리포트 텍스트 생성)
  → 정산 응답파싱 (Code)
  → 정산 응답 (respondToWebhook)
```

---

### Flow D — 정산 메일 발송

```
Switch[3]
  → 공구 정산 기록 조회 (Google Sheets: 미발송 행 필터)
  → 정산 데이터 매칭 (Code: 미발송 행 추출, influencer_id 해석)
  → 이메일 조회 (Google Sheets: 아이디로 담당자 이메일 조회)
  → 정산 메일 발송 (Gmail: 정산 내역 이메일 발송)
  → 메일 발송 업데이트 (Google Sheets: 발송 여부 "발송 완료"로 upsert)
  → 정산 메일 발송 응답 (respondToWebhook)
```

**데이터 형태:**
```
입력: { query: "정산 메일 보내줘", influencer_id: "andantenine" }
중간: { 아이디: "andantenine", 총매출: 30000000, RS: 30, RS금액: 9000000 }
발송: Gmail → winsjf11@gmail.com (정산 내역 6건) → 인플루언서 혹은 공구 담당자 이메일
출력: "정산 메일 발송 완료"
```

---

### Flow E — RAG 지식 질의응답

```
Switch[4]
  → 관계 및 이력 검색 (PostgreSQL: relationships 테이블 ILIKE 검색)
      ↓ 결과를 AI Agent 시스템 프롬프트에 주입
  → AI Agent
      ├── Chat Model: gemini-2.0-flash
      ├── Memory: 세션ID 기억 (PostgreSQL n8n_chat_histories)
      └── Tool: Supabase Vector Store (pgvector 유사도 검색)
              └── Embeddings: text-embedding-004
  → 지식데이터 응답 (respondToWebhook)
```

**데이터 형태:**
```
입력: { query: "계약서 필수 항목이 뭐야?" }
PostgreSQL 조회: relationships WHERE source_entity ILIKE '%계약서%'
벡터 검색: channel WHERE embedding <-> query_vector < threshold
출력: { message: "공구 계약서 필수 항목은 기본 정보, 캠페인 조건, 콘텐츠 조건..." }
```

---

### Flow F — 문서 자동 수집 (백그라운드)

```
구글 드라이브 파일 감지 (Trigger: 폴더 내 신규 파일)
  → 파일 다운로드 (Google Drive: binary 다운로드)
  → Extract from File (PDF 텍스트 추출)
  → Code in JavaScript (청킹: 1000자 단위 분할)
        │
        ├─▶ 임베딩 API 호출 (HTTP Request → Gemini text-embedding-001)
        │     → 벡터 포맷 (Code: values 배열 → PostgreSQL 형식 문자열)
        │     → 채널 벡터 저장 (Supabase: channel 테이블 insert)
        │
        └─▶ Basic LLM Chain (Gemini: 청크에서 엔티티·관계 추출)
              → Code in JavaScript1 (추출 결과 파싱)
              → 엔티티/관계 분기 (Switch: 엔티티 vs 관계)
                    ├─▶ Create a row (Supabase: entities 테이블)
                    └─▶ 관계 저장 (Supabase: relationships 테이블)
```

---

## 3. 예상 비용

### 월간 사용량 가정 (운영 초기 기준)

| 항목 | 가정치 |
|---|---|
| 웹훅 요청 수 | 600건/월 (30건/일, 20일 영업일 기준) |
| 문서 자동 수집 | 20건/월 (PDF 업로드) |
| 청크당 평균 토큰 | 500토큰 |
| RAG 응답 평균 토큰 | 입력 2,000 + 출력 500 |

---

### 항목별 비용 산출

#### Gemini API

| 용도 | 월 요청 | 입력 토큰 | 출력 토큰 | 예상 비용 |
|---|---|---|---|---|
| 인텐트 분류 | 300회 | 300 × 300 = 90K | 300 × 20 = 6K | ~$0.01 |
| 인게이지먼트·시뮬레이션·정산 분석 | 150회 | 150 × 1,500 = 225K | 150 × 800 = 120K | ~$0.07 |
| RAG 응답 (AI Agent) | 100회 | 100 × 2,000 = 200K | 100 × 500 = 50K | ~$0.04 |
| 엔티티 추출 (문서 수집) | 200청크 | 200 × 600 = 120K | 200 × 200 = 40K | ~$0.03 |
| **gemini-embedding-001** | 200청크 | 200 × 500 = 100K tokens | — | ~$0.02 |
| **합계** | | | | **~$0.17/월** |

> Gemini 2.5 Flash-Lite 기준: 입력 $0.10/1M tokens, 출력 $0.40/1M tokens  
> ※ Gemini 2.0 Flash는 2026년 6월 1일부로 서비스 종료 예정 — Gemini 2.5 Flash-Lite로 대체 권장  
> gemini-embedding-001: $0.15/1M tokens (characters 아닌 tokens 기준)  
> ※ 무료 할당량(1분 60회, 일 1,500회) 내 운영 시 $0
---

#### 인프라 (VPS — Hostinger)

| 항목 | 비용 |
|---|---|
| VPS 호스팅 (2 vCPU, 8GB RAM) | ~$13/월 |
| 도메인 (srv1001864.hstgr.cloud) | 포함 |
| TLS 인증서 (Let's Encrypt) | 무료 |
| **합계** | **~$13/월** |

---

#### Supabase

| 플랜 | 비용 | 적용 시점 |
|---|---|---|
| Free tier (500MB DB, 1GB storage) | $0/월 | 초기 운영 |
| Pro plan (8GB DB, 100GB storage) | $25/월 | 문서 1,000건 이상 축적 시 |

> 현재 운영 규모 기준 Free tier로 충분히 운영 가능

---

#### Google Workspace

| 항목 | 비용 |
|---|---|
| Google Workspace Business Starter | 기존 구독 활용 |
| Sheets / Drive / Gmail API | 무료 (할당량 내) |
| **추가 비용** | **$0/월** |

---

### 월간 총 예상 비용 요약

| 항목 | 초기 (소규모) | 성장기 (문서 1,000건+) |
|---|---|---|
| Gemini API | ~$0.12 | ~$2~5 |
| VPS (Hostinger) | ~$10 | ~$15~20 (업그레이드 시) |
| Supabase | $0 | $25 |
| Google Workspace | 기존 구독 | 기존 구독 |
| **합계** | **~$10/월** | **~$30~50/월** |

---

### 비용 최적화 전략

1. **Gemini 무료 할당량 최대 활용:** 일 1,500회 무료 요청 한도 내에서는 API 비용 $0
2. **청킹 크기 조정:** 청크를 크게 할수록 임베딩 횟수 감소 → 비용 절감 (단, 검색 정밀도 트레이드오프)
3. **응답 캐싱:** 동일 질의에 대해 PostgreSQL 세션 메모리 활용으로 중복 LLM 호출 방지
4. **Supabase 무료 티어 유지:** 불필요한 벡터 데이터 주기적 정리로 500MB 내 운영

---

## 4. 확장성 고려사항

| 시나리오 | 대응 방안 |
|---|---|
| 인플루언서 수 증가 (10명 → 100명) | Google Sheets → Supabase 테이블로 마이그레이션 |
| 문서 수 급증 (100건 → 10,000건) | Supabase Pro 업그레이드, 청크 인덱스 최적화 |
| 동시 요청 증가 | n8n 큐 모드 활성화, VPS 수직 확장 |
| 다국어 지원 | text-embedding-001는 한국어 포함 100개 언어 지원으로 추가 비용 없음 |
