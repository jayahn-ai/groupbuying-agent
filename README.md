# Groupbuying Agent                                                                                                                             
                                                                                                                                                  
  인플루언서 공동구매 마케팅 업무를 돕는 AI 에이전트.                                                                                       
                                                                                                                                                  
  **[→ 라이브 데모](https://jayahn-ai.github.io/groupbuying-agent/)**                                                                             
                                                                                                                                                  
  ---                                                                                                                                             
                                                            
  ## 주요 기능

  | 인텐트 | 기능 | 예시 질문 |                                                                                                                   
  |--------|------|-----------|
  | `engagement` | 인플루언서 인게이지먼트 분석 | "팔로워 대비 반응률 분석해줘" | 
  | `simulation` | 공동구매 예상 수익 분석 | "공구 수익 시뮬레이션 진행해줘" | 
  | `settlement` | 주문 데이터 기반 정산 계산 | "이번 공구 정산 계산해줘" |                                                                       
  | `settlement_confirm` | 정산 메일 자동 발송 | "정산 메일 보내줘" |                                                                             
  | `rag` | 계약서·가이드라인 지식 Q&A | "공구 계약서 필수 항목이 뭐야?" |                                                                        
                                                                                                                                                  
  ---                                                                                                                                             
                                                                                                                                                  
  ## 기술 스택                                                                                                                                    
                                                            
  | 레이어 | 도구 |
  |--------|------|
  | 워크플로우 오케스트레이션 | n8n (Self-hosted, Docker + Traefik) |
  | LLM | Gemini 2.0 Flash (인텐트 분류, 분석 리포트, RAG 응답) |    
  | 임베딩 | Gemini text-embedding-001 |                                                                                                          
  | 벡터 DB + 그래프 DB | Supabase (PostgreSQL + pgvector) |                                                                                      
  | 운영 데이터 | Google Sheets (인플루언서 정보, 상품 DB, 정산 기록) |                                                                           
  | 문서 수집 트리거 | Google Drive (PDF 업로드 감지 → 자동 임베딩) |                                                                             
  | 메일 발송 | Gmail |                                                                                                                           
  | 클라이언트 | HTML + GitHub Pages |                                                                                                            
                                                                                                                                                  
  ---                                                                                                                                             
                                                            
  ## 아키텍처                                                                                                                                     
                                                                                                                                                  
  **요청 흐름**                                                                                                                                   
                                                                                                                                                  
  HTML 클라이언트                                                                                                                                 
    → POST /webhook/influencer-query                        
    → Webhook Trigger                                                                                                                             
    → Intent 분류 (Gemini)
    → Switch (5개 분기)                                                                                                                           
                                                            
  **분기별 파이프라인**                                                                                                                           
                                                                                                                                                  
  | 분기 | 파이프라인 |
  |------|-----------|                                                                                                                            
  | `engagement` | Google Sheets → Gemini → 응답 |          
  | `simulation` | Google Sheets → 수익 계산 → Gemini → 응답 |                                                                                    
  | `settlement` | Google Sheets → 정산 계산 → Gemini → 응답 |                                                                                    
  | `settlement_confirm` | Google Sheets → Gmail 발송 → 응답 |                                                                                    
  | `rag` | PostgreSQL 관계 검색 → AI Agent (Gemini + Vector Store) → 응답 |                                                          
                                                                                                                                                  
  ---                                                                                                                                             
                                                                                                                                                  
  ## 개발 프로세스                                          

  에이전트 기능 정의는 **고객 인터뷰 2회 + 골든 데이터셋 수집** 3단계를 거쳐 도출되었습니다.                                                      
  
  1. **1차 인터뷰 (4/21)** — 인게이지먼트 검증·브랜드 핏 판별·수익 시뮬레이션 부재 3가지 페인포인트 발굴                                          
  2. **2차 인터뷰 (4/22)** — 정산 메일 수작업·히스토리 비자산화 등 3가지 추가 페인포인트 발굴, 기능 확장
  3. **골든 데이터셋 (4/25)** — 인텐트별 골든데이터셋 확보 → 답변 형식·수치 포함 기준 확정                                                       
                                                                                                                                                  
  ---                                                                                                                                             
                                                                                                                                                  
  ## 문서                                                   

  | 파일 | 내용 |
  |------|------|
  | [`raw_transcript.md`](./raw_transcript.md) | Claude Code와의 실제 개발 로그 (프롬프트, 에러, 설계 변경) |                                     
  | [`architecture_memo.md`](./architecture_memo.md) | 도구 선택 이유, 데이터 흐름, 예상 비용 (~$13/월) |                                         
  | [`kpi_sheet.md`](./kpi_sheet.md) | 6개 KPI 정의 및 목표치 |                                                                                   
  | [`CLAUDE.md`](./CLAUDE.md) | 개발 시 Claude Code에 부여한 프로젝트 규칙 |                                                                     
                                                                                                                                                  
  ---                                                       
                                                                                                                                                  
  ## 예상 운영 비용                                                                                                                               
  
  | 항목 | 초기 (소규모) | 성장기 |                                                                                                               
  |------|-------------|--------|                           
  | Gemini API | ~$0.17/월 | ~$2~5/월 |                                                                                                           
  | VPS (Hostinger) | ~$13/월 | ~$15~20/월 |
  | Supabase | $0 (Free tier) | $25/월 |                                                                                                          
  | **합계** | **~$10/월** | **~$30~50/월** |                                                                                                     
                                                                                                                                                  
  ---                                                         
