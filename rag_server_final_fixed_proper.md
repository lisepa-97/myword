래그 서버용 md 작성하기..

## Project Overview

이 프로젝트는 펫푸드 웹 쇼핑몰에서 기존 NOTICE/FAQ 게시판 방식으로 제공하던 고객 안내 기능을 대체하기 위한 문서 기반 질의응답 RAG 마이크로서비스를 구현한다.

이 서비스는 REST API를 통해 호출되며, 업로드된 문서를 파싱(parse)하고 불필요한 내용을 제거(clean)한 뒤, 의미 단위로 청킹(chunking)하여 임베딩을 생성하고 벡터 데이터베이스에 저장한다.

사용자가 질문을 입력하면, 서비스는 필요 시 대화 히스토리를 기반으로 질문을 standalone query로 재작성하고, 해당 질문을 임베딩으로 변환한 뒤 벡터 검색을 수행하여 관련 문서 조각을 retrieval 한다.

검색된 문서는 LLM의 입력 context로 사용되며, LLM은 제공된 context에 기반하여 근거 있는 답변을 생성해야 한다. Context에 없는 내용은 추측하지 않아야 한다.

응답에는 생성된 답변(answer), 참조된 문서 조각(source), 그리고 관련 메타데이터가 포함된다. 또한 세션 기반 대화 히스토리를 통해 멀티턴 대화를 지원한다.

이 서비스는 인증/인가의 원장 시스템이 아니며, 사용자 식별 및 권한 정보는 API Gateway 또는 외부 인증 서비스에서 전달받는 것을 전제로 한다. 또한 원본 파일 저장소, 공통 로깅, 모니터링, 메시지 브로커 등은 MSA 공통 인프라 또는 외부 서비스와 연동하는 구조를 따른다.

대용량 문서 처리 및 재색인을 고려하여, 문서 처리 파이프라인은 비동기 작업 큐 또는 이벤트 기반 구조로 확장 가능해야 한다.

# Goals

- 펫푸드 웹 쇼핑몰에서 기존 NOTICE/FAQ 게시판 방식으로 제공하던 고객 안내 기능을 대체할 수 있는 RAG 기반 질의응답 서비스를 구현한다.
- 배송, 주문, 결제, 교환, 환불, 회원, 이벤트, 이용안내 등 고객 문의에 대응할 수 있도록 FAQ, 공지사항, 정책 문서를 지식 소스로 처리한다.
- 문서를 업로드하고 인덱싱할 수 있는 API를 제공한다.
- 업로드된 문서를 파싱, 정제, 청킹, 임베딩하여 벡터 데이터베이스에 저장한다.
- 사용자의 자연어 질문에 대해 관련 문서를 검색하고, 이를 근거로 LLM이 답변을 생성하도록 한다.
- 답변과 함께 참조된 문서 조각 및 관련 메타데이터를 반환한다.
- 세션 기반 대화 히스토리를 통해 기본적인 멀티턴 고객 문의를 지원한다.
- MSA 환경에서 기존 쇼핑몰 시스템과 연동 가능한 독립적인 RAG 마이크로서비스 구조를 제공한다.

# Out of Scope

본 서비스는 FAQ/공지/정책 기반 질의응답에 한정하며,
다른 도메인 서비스의 책임을 침범하지 않는다.

- 사용자 인증(Authentication) 및 인가(Authorization) 시스템 구현은 포함하지 않는다.
  해당 기능은 API Gateway 또는 외부 인증 서비스에서 처리하는 것을 전제로 한다.

- 상품 추천, 주문 처리, 결제, 장바구니 등 커머스 비즈니스 로직은 포함하지 않는다.

- 관리자 UI, FAQ 관리 시스템, 문서 편집 및 운영 대시보드는 포함하지 않는다.

- 원본 문서의 영구 저장 및 파일 스토리지 기능은 포함하지 않는다.
  (외부 Object Storage 사용을 전제로 한다.)

- OCR 고도화, 이미지 기반 문서 처리, 멀티모달 기능은 포함하지 않는다.

- 고급 검색 기능 (reranker, hybrid search 최적화, semantic routing 등)은 MVP 범위에서 제외한다.

- LLM fine-tuning 및 모델 학습 기능은 포함하지 않는다.

- 멀티테넌시 및 복잡한 권한 기반 문서 필터링은 MVP 범위에서 제외한다.

- 실시간 추천 챗봇 또는 상품 상담 기능은 포함하지 않는다.

- Jenkins 기반 CI/CD 파이프라인 구성은 이번 범위에 포함하지 않는다.

## Tech Stack

### Core Application
- Java 17
- Spring Boot 3.5.11
- Spring Web
- Spring Validation
- Spring Actuator

### Search / Retrieval
- Elasticsearch
- Spring Data Elasticsearch
- 본 서비스 전용 Elasticsearch 인덱스를 사용한다.
- BM25 기반 lexical 검색
- Dense Vector 기반 semantic 검색
- RRF(Reciprocal Rank Fusion) 기반 하이브리드 검색

### Session Storage
- Redis
- 세션 기반 대화 히스토리는 Redis에 저장한다.
- Redis는 본 서비스 전용 저장소를 사용한다.
- 세션 데이터는 TTL 기반으로 관리한다.
- 기본 TTL은 24시간으로 설정한다.
- 세션은 마지막 접근 시 TTL이 연장되는 sliding expiration 방식으로 관리한다.
- 최근 메시지 10~20개를 유지하는 것을 기본 정책으로 한다.

### AI Integration
- Gemini API
- Primary model: gemini-2.5-flash
- Optional low-cost batch/offline model: gemini-2.5-flash-lite
- Embedding model: gemini-embedding-001
- Embedding dimension: 1536
- Spring WebClient를 사용하여 LLM 및 Embedding API를 호출한다.

### Document Processing
- Apache Tika
- Apache POI
- Apache PDFBox
- PDF, DOCX, TXT 형식의 문서를 지원한다.

### Async Processing
- Spring Async
- 문서 인덱싱 및 재색인 작업은 비동기 처리한다.

### Serialization / Utilities
- Jackson

### Testing
- JUnit 5
- Mockito
- Spring Boot Test
- Testcontainers

### Build / Deployment
- Gradle
- Jib
- Docker Compose

### Integration Style
- REST API (JSON)
- API Gateway를 통한 호출을 전제로 한다.
- 인증/인가 정보는 외부 서비스 또는 Gateway에서 전달받는다.

## Technical Constraints

- 문서 검색 및 retrieval 저장소는 Elasticsearch로 고정한다.
- 세션 기반 대화 히스토리는 Elasticsearch에 저장하지 않고 Redis에 저장한다.
- 외부 LLM 및 Embedding 호출은 Spring WebClient를 사용한다.
- 문서 인덱싱 파이프라인은 Spring Async 기반의 비동기 처리 구조를 따른다.
- 빌드 도구는 Gradle로 고정한다.
- 컨테이너 이미지는 Jib를 사용하여 빌드한다.
- 로컬 및 개발 환경 실행은 Docker Compose를 사용한다.
- 검색 방식은 BM25 + Vector 기반 하이브리드 검색으로 고정한다.
- 답변은 한국어로만 생성한다.
- 본 서비스는 MSA의 database per service 원칙을 준수해야 한다.
- Elasticsearch와 Redis는 본 서비스 전용 저장소를 사용해야 하며, 다른 서비스와 논리적/물리적으로 공유하지 않는다.
- 다른 서비스가 사용하는 Redis 인스턴스, Elasticsearch 인덱스, 데이터베이스 스키마를 재사용하지 않는다.

## Functional Requirements

### 1. Document Upload
- 사용자는 PDF, DOCX, TXT 형식의 문서를 업로드할 수 있어야 한다.
- 업로드 API는 multipart/form-data 형식을 지원해야 한다.
- 지원하지 않는 파일 형식은 업로드를 거부해야 한다.
- 업로드 성공 시 document_id, filename, status 등의 기본 정보를 반환해야 한다.
- 업로드된 원본 파일은 문서 처리 파이프라인의 입력으로 사용되어야 한다.

### 2. Document Parsing and Cleaning
- 서버는 업로드된 문서에서 텍스트를 추출해야 한다.
- 문서 파싱 시 PDF, DOCX, TXT 형식을 지원해야 한다.
- 추출된 텍스트에서 불필요한 공백, 빈 줄, 반복 헤더/푸터, 페이지 번호 등 노이즈를 제거해야 한다.
- 파싱 또는 정제 실패 시 해당 문서 상태를 failed로 기록해야 한다.

### 3. Chunking
- 정제된 문서는 의미 단위로 청킹되어야 한다.
- 청킹은 제목/문단 기반 분할을 우선 적용해야 한다.
- chunk는 검색 및 답변 생성에 적합한 길이로 유지해야 한다.
- chunk 생성 시 chunk_index, 원문 텍스트, 문서 식별자, 카테고리 등 메타데이터를 함께 저장해야 한다.

### 4. Embedding
- 각 chunk에 대해 gemini-embedding-001 모델을 사용하여 임베딩을 생성해야 한다.
- 문서 chunk와 사용자 질문은 동일한 임베딩 모델을 사용해야 한다.
- 임베딩 결과는 Elasticsearch dense vector 필드에 저장해야 한다.
- 기본 임베딩 차원은 1536으로 설정해야 한다.

### 5. Indexing
- chunk 데이터는 Elasticsearch에 인덱싱되어야 한다.
- 인덱싱 데이터에는 최소한 다음 정보가 포함되어야 한다:
  - chunk_id
  - document_id
  - filename
  - category
  - chunk_index
  - text
  - embedding
  - created_at
- 문서 인덱싱은 Spring Async 기반의 비동기 처리로 수행되어야 한다.
- 인덱싱 완료 후 문서 상태를 processed로 변경해야 한다.
- 인덱싱 실패 시 문서 상태를 failed로 변경해야 한다.

### 6. Document Search
- 사용자의 질문에 대해 Elasticsearch 기반 하이브리드 검색을 수행해야 한다.
- lexical 검색은 BM25를 사용해야 한다.
- semantic 검색은 dense vector kNN 검색을 사용해야 한다.
- lexical 결과와 semantic 결과는 RRF로 결합해야 한다.
- 기본 retrieval top_k는 5로 설정해야 한다.
- 검색 결과는 score와 함께 반환 가능해야 하며, 답변 생성에 필요한 source 데이터가 포함되어야 한다.

### 7. Chat Query API
- 사용자는 질문을 입력하여 답변을 요청할 수 있어야 한다.
- 질문 요청에는 session_id와 question이 포함되어야 한다.
- session_id가 없으면 서버는 새로운 세션을 생성할 수 있어야 한다.
- 질문이 들어오면 서버는 대화 히스토리를 조회한 뒤 필요한 경우 standalone question으로 재작성할 수 있어야 한다.
- 재작성된 질문을 기준으로 retrieval을 수행해야 한다.
- retrieval 결과를 context로 구성하여 Gemini LLM을 호출해야 한다.
- 응답에는 answer, sources, session_id가 포함되어야 한다.

### 8. Multi-turn Conversation
- 서버는 세션 기반 대화 히스토리를 지원해야 한다.
- 세션 데이터는 Redis에 저장해야 한다.
- 세션 데이터는 TTL 기반으로 관리해야 하며 기본 TTL은 24시간이어야 한다.
- 세션은 마지막 접근 시 TTL이 연장되는 sliding expiration 방식으로 동작해야 한다.
- 최근 메시지 10~20개를 유지하는 것을 기본 정책으로 해야 한다.
- 후속 질문은 이전 대화 문맥을 반영하여 처리할 수 있어야 한다.

### 9. Answer Generation Policy
- 답변은 항상 한국어로 생성해야 한다.
- 답변은 retrieval로 제공된 context에 기반해서만 생성해야 한다.
- context에 없는 정보는 추측해서 답변하면 안 된다.
- 답을 찾을 수 없으면 명확히 모른다고 답변해야 한다.
- 답변에는 관련 source를 포함해야 한다.
- 답변 톤은 쇼핑몰 고객센터 응대에 적합한 간결하고 명확한 스타일이어야 한다.

### 10. Source Handling
- 각 답변에는 근거로 사용된 문서 조각 목록을 포함해야 한다.
- source 정보에는 최소한 다음 정보가 포함되어야 한다:
  - document_id
  - filename
  - chunk_id
  - chunk text 또는 snippet
- source는 최대 3개까지 반환하는 것을 기본 정책으로 한다.

### 11. Error Handling
- 지원하지 않는 파일 형식 업로드 시 4xx 에러를 반환해야 한다.
- 문서 파싱 실패 시 문서 상태를 failed로 기록해야 한다.
- retrieval 결과가 없을 경우 answer에는 근거 문서를 찾지 못했다는 메시지를 포함해야 한다.
- 외부 LLM 또는 Embedding API 호출 실패 시 적절한 에러 로그를 남기고 재시도 가능한 구조를 고려해야 한다.
- 예상하지 못한 예외가 발생하더라도 표준화된 에러 응답 형식을 반환해야 한다.

### 12. Observability
- 문서 업로드, 인덱싱 시작/완료/실패, 질의 처리, 외부 API 호출 실패에 대한 로그를 남겨야 한다.
- 헬스체크 및 기본 메트릭은 Spring Boot Actuator를 통해 제공해야 한다.
- retrieval latency, LLM latency, indexing latency를 추적 가능해야 한다.


## Non-Functional Requirements

### 1. Performance
- 일반적인 FAQ 질의응답 요청은 실사용 가능한 응답 속도를 제공해야 한다.
- retrieval과 answer generation은 과도한 지연 없이 동작해야 한다.
- 문서 인덱싱은 사용자 질의응답 API와 분리된 비동기 처리로 수행되어야 한다.
- retrieval 기본 top_k는 5를 유지하되, 설정값으로 조정 가능해야 한다.

### 2. Reliability
- 문서 파싱, 임베딩 생성, 인덱싱, 질의응답 처리 중 일부 단계가 실패하더라도 전체 시스템이 비정상 종료되지 않아야 한다.
- 외부 LLM 및 Embedding API 호출 실패 시 적절한 예외 처리를 수행해야 한다.
- 문서 인덱싱 실패 건은 재처리 가능하도록 상태를 추적할 수 있어야 한다.
- 세션 저장소(Redis) 또는 검색 저장소(Elasticsearch)의 일시적 장애 상황에 대해 서비스는 표준화된 오류 응답을 제공해야 한다.

### 3. Scalability
- 본 서비스는 MSA 환경에서 독립적으로 배포 및 확장 가능해야 한다.
- 질의응답 API와 문서 인덱싱 작업은 서로 다른 부하 특성을 고려하여 분리 가능한 구조여야 한다.
- 문서 처리량 증가에 대비하여 비동기 인덱싱 파이프라인을 수평 확장 가능하게 설계해야 한다.
- Elasticsearch 인덱스 구조는 FAQ, 공지, 정책 문서 증가를 고려하여 확장 가능해야 한다.

### 4. Maintainability
- 애플리케이션은 Controller, Service, Client, Repository, Domain 계층을 명확히 분리해야 한다.
- LLM 호출, Embedding 호출, Retrieval 로직, Session 관리 로직은 서로 독립적인 컴포넌트로 분리해야 한다.
- 환경별 설정(local, dev, prod)은 외부 설정값으로 분리해야 하며, 코드에 하드코딩하지 않아야 한다.
- 주요 설정값(top_k, TTL, model name, embedding dimension, timeout 등)은 변경 가능해야 한다.

### 5. Observability
- 문서 업로드, 문서 인덱싱 시작/완료/실패, 질의 처리, retrieval 수행, LLM 호출, 오류 발생에 대해 로그를 남겨야 한다.
- 애플리케이션 상태 확인을 위한 헬스체크 endpoint를 제공해야 한다.
- retrieval latency, LLM latency, indexing latency, error count 등의 기본 메트릭을 수집 가능해야 한다.
- 로그에는 추적 가능한 request id 또는 correlation id를 포함할 수 있어야 한다.

### 6. Security
- 인증(Authentication)과 인가(Authorization)는 외부 Gateway 또는 인증 서비스와 연동하는 것을 전제로 한다.
- 민감한 설정값(API Key, Secret 등)은 환경변수 또는 외부 시크릿 관리 방식으로 주입해야 한다.
- 업로드 가능한 파일 형식은 허용된 MIME type 및 확장자로 제한해야 한다.
- 로그에 민감한 사용자 입력, API Key, 전체 문서 원문을 그대로 남기지 않아야 한다.
- 답변 생성 시 retrieval context 외의 임의 정보에 의존하지 않도록 제한해야 한다.

### 7. Data Management
- 문서, chunk, 세션 데이터는 각 저장소의 역할에 맞게 분리되어야 한다.
- 검색용 데이터는 Elasticsearch에 저장하고, 세션 데이터는 Redis에 저장해야 한다.
- Redis 세션 데이터는 TTL 24시간을 기본값으로 하며 sliding expiration 방식으로 관리해야 한다.
- 인덱싱된 문서는 상태(uploaded, processed, failed)를 추적할 수 있어야 한다.

### 8. Testability
- 주요 비즈니스 로직은 단위 테스트가 가능하도록 설계해야 한다.
- retrieval, chunking, prompt building, session handling은 각각 독립적으로 테스트 가능해야 한다.
- 외부 API 연동(LLM, Embedding)은 mocking 가능한 구조로 추상화해야 한다.
- Elasticsearch 및 Redis 연동은 통합 테스트 또는 Testcontainers 기반 테스트가 가능해야 한다.

### 9. Deployment
- 애플리케이션은 Gradle 기반으로 빌드해야 한다.
- 컨테이너 이미지는 Jib를 사용하여 생성해야 한다.
- 로컬 및 개발 환경 실행은 Docker Compose를 사용해야 한다.
- 서비스는 환경 변수 기반 설정을 통해 동일한 이미지를 여러 환경에서 재사용 가능해야 한다.

### 10. Response Policy
- 사용자 응답은 항상 한국어로 제공해야 한다.
- 답변은 간결하고 명확해야 하며 쇼핑몰 고객센터 응대 톤을 유지해야 한다.
- 근거가 없는 정보는 제공하지 않아야 하며, 답을 찾을 수 없는 경우 이를 명확히 안내해야 한다.

## API Specification

### Base URL
- `https://api.eum.com`

### Base Path
- 모든 RAG 서비스 API는 `/api/v1/rag` 경로를 따른다.
- 서비스 식별은 path 기반으로 수행한다.

---

## 1. Document Upload

### POST `/api/v1/rag/documents`

문서를 업로드하고 인덱싱 파이프라인을 시작한다.  
문서 인덱싱은 비동기 처리된다.

#### Request
- Content-Type: `multipart/form-data`

#### Form Data
- `file`: PDF | DOCX | TXT
- `category` (optional): FAQ | NOTICE | POLICY

#### Response
```json
{
  "document_id": "9d8e1f4c-6c7a-4a6a-b4e2-1a2b3c4d5e6f",
  "filename": "refund_policy.pdf",
  "category": "POLICY",
  "status": "uploaded",
  "message": "Document uploaded successfully. Indexing started asynchronously."
}
```
2. Document Status
GET /api/v1/rag/documents/{document_id}

문서 처리 상태를 조회한다.

Path Variable
document_id: UUID
Response
{
  "document_id": "9d8e1f4c-6c7a-4a6a-b4e2-1a2b3c4d5e6f",
  "filename": "refund_policy.pdf",
  "category": "POLICY",
  "status": "processed",
  "chunk_count": 24,
  "created_at": "2026-04-13T10:00:00Z",
  "updated_at": "2026-04-13T10:00:12Z"
}
Error Cases
존재하지 않는 문서: 404 Not Found

3. Chat Query
POST /api/v1/rag/chat

사용자의 질문에 대해 RAG 기반 답변을 생성한다.
세션 기반 멀티턴 대화를 지원한다.

Request
Content-Type: application/json
Request Body
{
  "session_id": "2f0d7a3c-91af-4b21-9d12-3b4c5d6e7f80",
  "question": "환불은 언제까지 가능한가요?"
}
Request Rules
session_id는 optional이다.
session_id가 없으면 서버가 새로운 세션을 생성한다.
question은 필수값이다.
Response
{
  "session_id": "2f0d7a3c-91af-4b21-9d12-3b4c5d6e7f80",
  "answer": "환불은 상품 수령 후 7일 이내 가능합니다.",
  "sources": [
    {
      "document_id": "9d8e1f4c-6c7a-4a6a-b4e2-1a2b3c4d5e6f",
      "filename": "refund_policy.pdf",
      "chunk_id": "c1f5a9d0-9c33-4a7d-b0a5-5d7f8e9a1234",
      "text": "환불은 상품 수령 후 7일 이내 가능합니다."
    }
  ]
}
Response Rules
답변은 항상 한국어로 생성한다.
답변은 retrieval context 기반으로만 생성한다.
근거를 찾지 못한 경우 추측하지 않고 명확히 안내한다.
sources는 최대 3개까지 반환한다.
Example Without Session
{
  "question": "배송비는 누가 부담하나요?"
}
Example Response When No Context Found
{
  "session_id": "2f0d7a3c-91af-4b21-9d12-3b4c5d6e7f80",
  "answer": "현재 제공된 문서에서 해당 내용을 찾지 못했습니다.",
  "sources": []
}
Error Cases
질문 누락: 400 Bad Request
외부 LLM 호출 실패: 502 Bad Gateway 또는 표준 내부 오류 응답
내부 처리 오류: 500 Internal Server Error

4. Session History
GET /api/v1/rag/sessions/{session_id}

세션 기반 대화 히스토리를 조회한다.

Path Variable
session_id: UUID
Response
{
  "session_id": "2f0d7a3c-91af-4b21-9d12-3b4c5d6e7f80",
  "messages": [
    {
      "role": "user",
      "content": "환불 정책 알려줘"
    },
    {
      "role": "assistant",
      "content": "환불은 상품 수령 후 7일 이내 가능합니다."
    },
    {
      "role": "user",
      "content": "그럼 배송비는 누가 부담해?"
    },
    {
      "role": "assistant",
      "content": "배송비 부담 여부는 상품 상태와 정책에 따라 달라질 수 있으므로 관련 문서 기준으로 확인이 필요합니다."
    }
  ]
}
Error Cases
존재하지 않는 세션: 404 Not Found

## Operational Endpoints

### Health Check
- 서비스 헬스체크는 Spring Boot Actuator를 사용한다.
- 기본 헬스체크 엔드포인트는 `/actuator/health` 이다.

### Metrics
- Prometheus 수집용 엔드포인트는 `/actuator/prometheus` 이다.

### Notes
- 헬스체크는 비즈니스 API(`/api/v1/rag/**`)와 분리하여 관리한다.
- Eureka, Prometheus, Grafana, Zipkin은 헬스체크를 대체하지 않으며, 각각 서비스 디스커버리, 메트릭, 시각화, 트레이싱 용도로 사용한다.

## RAG Pipeline

본 서비스의 RAG 파이프라인은 문서 인덱싱 파이프라인과 질의응답 파이프라인으로 구성된다.

---

## 1. Document Indexing Pipeline

### 1.1 Document Upload
- 사용자가 PDF, DOCX, TXT 문서를 업로드한다.
- 서버는 document_id를 생성하고 상태를 `uploaded`로 기록한다.
- 인덱싱 작업은 비동기(Sprint Async)로 수행된다.

---

### 1.2 Document Parsing → Markdown 변환

- 문서에서 텍스트를 추출한 후 Markdown 형식으로 정규화한다.
- Markdown 변환 규칙:
  - 제목 → `#`, `##`, `###`
  - 리스트 → `-`
  - 문단 → 줄바꿈 유지
- 목적:
  - 구조 보존
  - chunk 품질 향상
  - retrieval 정확도 개선

---

### 1.3 Text Cleaning

- 제거 대상:
  - 반복 헤더/푸터
  - 페이지 번호
  - 과도한 공백
  - 불필요한 줄바꿈
- 의미 단위는 유지한다.

---

### 1.4 Chunking Strategy

#### 1) 1차: 헤더 기반 분할
- Markdown heading (`#`, `##`, `###`) 기준으로 분할

#### 2) 2차: 문단 기반 분할
- heading 내부에서 문단 단위 분할

#### 3) 3차: 길이 기반 분할 (fallback)

---

### Chunk 설정 (추천값)

- chunk size: **650 tokens**
- overlap: **120 tokens**

#### 이유
- 650: FAQ/정책 문서에서 context 유지 + noise 최소화 균형
- 120: 문맥 연결 유지 + 중복 최소화

---

### Chunk Metadata

각 chunk는 다음 정보를 포함한다:

- chunk_id
- document_id
- filename
- category
- chunk_index
- heading (optional)
- text
- created_at

---

### 1.5 Embedding

- 모델: `gemini-embedding-001`
- dimension: **1536**

규칙:
- 문서와 질문 동일 embedding 모델 사용
- embedding은 chunk 단위로 생성

---

### 1.6 Indexing

- Elasticsearch에 저장
- 필드 구성:
  - text (BM25용)
  - embedding (dense_vector)
  - metadata

- 문서 상태:
  - uploaded → processing → processed
  - 실패 시 → failed

---

## 2. Query Answering Pipeline

### 2.1 Session Handling

- Redis에서 session 조회
- 없으면 생성
- 최근 10~20개 메시지 유지
- TTL: 24시간 (sliding expiration)

---

### 2.2 Question Rewriting (Optional but Recommended)

- 이전 대화 기반 standalone question 생성

예:
"그럼 배송비는?"
→ "환불 시 배송비는 누가 부담하나요?"

---

### 2.3 Query Embedding

- 동일 모델 사용: `gemini-embedding-001`

---

### 2.4 Retrieval (Hybrid Search)

- BM25 lexical 검색
- vector kNN 검색
- RRF 기반 결합

설정:
- top_k: **5**

---

### 2.5 Context Construction

- top_k chunk 선택
- 최대 토큰 제한 내에서 context 구성
- 중복 chunk 제거

---

### 2.6 Prompt Construction (Strict QA Pattern)

LLM은 반드시 아래 규칙을 따라야 한다.

#### System Prompt (Strict QA)

- 반드시 제공된 context만 사용한다.
- context에 없는 내용은 절대 추측하지 않는다.
- 모르면 "찾을 수 없다"고 답한다.
- 답변은 한국어로 작성한다.
- 답변은 간결하고 명확해야 한다.
- 고객센터 스타일을 유지한다.

---

### 2.7 Answer Generation

- 모델: `gemini-2.5-flash`
- 입력:
  - system prompt
  - conversation history
  - retrieved context
  - user question

---

### 2.8 Response Construction

응답에는 다음이 포함된다:

- session_id
- answer
- sources (최대 3개)

---

## 3. Source Construction

- retrieval 결과 중 실제 사용된 chunk 선택
- 최대 3개 반환

구성:
- document_id
- filename
- chunk_id
- text snippet

---

## 4. Re-indexing Strategy (추천)

### 방식: Version 기반 재색인

#### 흐름

1. 기존 문서(document_id)가 존재할 경우:
   - 기존 문서는 `inactive` 상태로 유지
2. 새로운 문서를 업로드하면:
   - 새로운 version 생성
3. 새로운 chunk를 인덱싱
4. 인덱싱 완료 후:
   - 기존 index → 비활성화
   - 신규 index → 활성화

---

### 장점

- 무중단 업데이트
- 롤백 가능
- 검색 안정성 유지

---

### 정책

- 동일 document_id 재업로드 시 version 증가
- active version만 retrieval 대상
- 필요 시 이전 version 삭제 가능

---

## 5. Failure Handling

- parsing 실패 → 상태 `failed`
- embedding 실패 → 인덱싱 중단
- ES 저장 실패 → 재처리 가능 상태 유지
- retrieval 결과 없음 → "관련 문서를 찾지 못했습니다"
- LLM 실패 → fallback 응답 또는 에러 반환

## Data Model

본 서비스는 database per service 원칙을 준수하며, 저장소 역할을 다음과 같이 분리한다.

- Elasticsearch: 문서, chunk, embedding, 검색 메타데이터 저장
- Redis: 세션 기반 대화 히스토리 저장

---

## 1. Elasticsearch Data Model

Elasticsearch는 검색 및 retrieval 전용 저장소로 사용한다.  
문서 원문 전체가 아니라, 검색 가능한 단위인 chunk 중심으로 저장한다.

### 1.1 Index Strategy

- 문서 검색용 인덱스는 본 서비스 전용으로 운영한다.
- 기본 인덱스명 예시:
  - `rag_chunks`
- 향후 환경별 분리를 위해 prefix 또는 suffix 사용 가능:
  - `rag_chunks_dev`
  - `rag_chunks_prod`

---

### 1.2 Chunk Document Schema

각 Elasticsearch document는 하나의 chunk를 의미한다.

#### Fields

- `chunk_id`: chunk 고유 식별자 (UUID)
- `document_id`: 원본 문서 식별자 (UUID)
- `document_version`: 문서 버전 (integer)
- `active`: 현재 활성 버전 여부 (boolean)
- `filename`: 원본 파일명
- `category`: 문서 카테고리 (FAQ | NOTICE | POLICY)
- `chunk_index`: 문서 내 chunk 순서
- `heading`: chunk가 속한 헤더 제목
- `text`: chunk 원문 텍스트
- `text_for_search`: lexical 검색용 텍스트 필드
- `embedding`: dense vector 필드
- `status`: chunk 상태 (active | inactive)
- `created_at`: 생성 시각
- `updated_at`: 수정 시각

#### Example
json
{
  "chunk_id": "c1f5a9d0-9c33-4a7d-b0a5-5d7f8e9a1234",
  "document_id": "9d8e1f4c-6c7a-4a6a-b4e2-1a2b3c4d5e6f",
  "document_version": 2,
  "active": true,
  "filename": "refund_policy.pdf",
  "category": "POLICY",
  "chunk_index": 3,
  "heading": "## 환불 정책",
  "text": "환불은 상품 수령 후 7일 이내 가능합니다.",
  "text_for_search": "환불은 상품 수령 후 7일 이내 가능합니다.",
  "embedding": [0.0123, -0.8832, 0.4421],
  "status": "active",
  "created_at": "2026-04-13T10:00:00Z",
  "updated_at": "2026-04-13T10:00:00Z"
}

1.3 Mapping Requirements
text, text_for_search: text
chunk_id, document_id, filename, category, status: keyword
chunk_index, document_version: integer
active: boolean
created_at, updated_at: date
embedding: dense_vector (1536 dimensions)
Mapping Notes
lexical 검색은 text_for_search 필드를 기준으로 수행한다.
semantic 검색은 embedding 필드를 기준으로 수행한다.
retrieval 대상은 active=true 인 chunk로 제한한다.

1.4 Re-indexing Model

문서 재업로드 시 version 기반 재색인을 사용한다.

Rules
동일 문서를 다시 업로드하면 document_version을 증가시킨다.
신규 version의 chunk를 먼저 인덱싱한다.
인덱싱 완료 후 신규 version의 active=true 로 전환한다.
이전 version은 active=false 로 전환한다.
retrieval은 항상 active=true 조건만 사용한다.

2. Redis Data Model

Redis는 세션 기반 대화 히스토리 저장소로 사용한다.
Redis는 본 서비스 전용 저장소를 사용하며, 다른 서비스와 공유하지 않는다.

2.1 Key Strategy

세션 데이터는 다음 패턴의 key를 사용한다.

rag:session:{session_id}

예:

rag:session:2f0d7a3c-91af-4b21-9d12-3b4c5d6e7f80

2.2 Session Schema

세션은 대화 메시지 목록과 메타데이터를 포함한다.

Fields
session_id: 세션 식별자
messages: 대화 메시지 리스트
created_at: 세션 생성 시각
updated_at: 마지막 갱신 시각
Message Schema
role: user | assistant
content: 메시지 본문
created_at: 메시지 생성 시각
Example
{
  "session_id": "2f0d7a3c-91af-4b21-9d12-3b4c5d6e7f80",
  "messages": [
    {
      "role": "user",
      "content": "환불 정책 알려줘",
      "created_at": "2026-04-13T10:30:00Z"
    },
    {
      "role": "assistant",
      "content": "환불은 상품 수령 후 7일 이내 가능합니다.",
      "created_at": "2026-04-13T10:30:02Z"
    }
  ],
  "created_at": "2026-04-13T10:30:00Z",
  "updated_at": "2026-04-13T10:30:02Z"
}

2.3 Session Policy
Redis TTL은 기본 24시간으로 설정한다.
세션은 마지막 접근 시 TTL이 연장되는 sliding expiration 정책을 사용한다.
최근 메시지 10~20개만 유지한다.
오래된 메시지는 trimming 한다.

3. Logical Domain Model

서비스 내부에서는 다음 도메인 모델을 사용한다.

3.1 Document
documentId
filename
category
version
status
chunkCount
createdAt
updatedAt
3.2 Chunk
chunkId
documentId
documentVersion
chunkIndex
heading
text
embedding
active
createdAt
updatedAt
3.3 ChatSession
sessionId
messages
createdAt
updatedAt
3.4 ChatMessage
role
content
createdAt
3.5 ChatResponse
sessionId
answer
sources
3.6 Source
documentId
filename
chunkId
text

4. Data Access Rules
Elasticsearch는 검색 전용으로 사용한다.
Redis는 세션 상태 저장 전용으로 사용한다.
세션 데이터를 Elasticsearch에 저장하지 않는다.
검색 결과 생성 시 Elasticsearch chunk 데이터를 source로 사용한다.
답변 생성 후 사용자 질문과 모델 응답은 Redis 세션에 append 한다.
서비스 간 데이터 연동은 저장소 공유가 아니라 API 또는 이벤트를 통해 수행한다.

## ## Directory Structure

```
src/main/java/com/eum/rag
├── RagApplication.java
├── common
├── document
├── chat
├── retrieval
├── embedding
├── llm
├── infra
└── prompt
```

## Test Directory Structure
src
└── test
    └── java
        └── com
            └── eum
                └── rag
                    ├── document
                    ├── chat
                    ├── retrieval
                    ├── embedding
                    ├── llm
                    └── integration

Directory Structure Principles
document 패키지는 문서 업로드, 파싱, cleaning, chunking, indexing 책임을 가진다.
chat 패키지는 질의응답, 세션 관리, 멀티턴 처리 책임을 가진다.
retrieval 패키지는 Elasticsearch 하이브리드 검색과 context 구성 책임을 가진다.
embedding 패키지는 임베딩 생성 책임을 가진다.
llm 패키지는 LLM 호출 및 답변 생성 책임을 가진다.
infra 패키지는 Elasticsearch, Redis 등 외부 저장소 연동 구현을 가진다.
common 패키지는 공통 설정, 예외, 응답 모델, 유틸리티를 가진다.
프롬프트 템플릿은 코드 하드코딩 대신 resources/prompts 아래에서 관리한다.

## Package Design Rules

- Controller는 요청/응답 처리만 담당하고 비즈니스 로직을 포함하지 않는다.
- Service는 도메인 로직을 담당한다.
- Client는 외부 API(Gemini) 호출만 담당한다.
- Repository는 저장소 접근만 담당한다.
- Elasticsearch document 모델과 서비스 도메인 모델은 분리한다.
- Redis 저장 모델과 서비스 도메인 모델은 분리한다.
- Prompt 템플릿은 resources에서 관리하고 코드 내 하드코딩을 최소화한다.

## Implementation Plan

본 서비스는 아래 순서로 구현한다.

---

### Phase 1. Project Bootstrap
목표:
- Spring Boot 기반 프로젝트 기본 구조를 생성한다.
- 공통 설정과 기본 실행 환경을 준비한다.

구현 항목:
- Gradle 설정
- Spring Boot 3.5.11 프로젝트 생성
- Java 17 설정
- 기본 패키지 구조 생성
- application.yml / profile 분리
- Jib 설정
- Docker Compose 실행 환경 구성
- Spring Actuator 설정

완료 기준:
- 애플리케이션이 로컬에서 정상 실행된다.
- `/actuator/health` 응답이 정상 반환된다.

---

### Phase 2. Infrastructure Setup
목표:
- Elasticsearch, Redis, WebClient 연동 기반을 구성한다.

구현 항목:
- Elasticsearch 연결 설정
- Redis 연결 설정
- WebClient 설정
- Async 설정
- 공통 예외 처리(GlobalExceptionHandler) 구현
- 공통 응답 포맷(ApiResponse, ErrorResponse) 구현

완료 기준:
- Elasticsearch와 Redis 연결이 확인된다.
- 외부 API 호출용 WebClient Bean이 준비된다.

---

### Phase 3. Document Upload API
목표:
- 문서 업로드 API와 기본 메타데이터 처리 기능을 구현한다.

구현 항목:
- `POST /api/v1/rag/documents`
- 파일 형식 검증
- category 파라미터 처리
- document_id 생성
- 업로드 응답 DTO 구현

완료 기준:
- PDF, DOCX, TXT 업로드 요청이 정상 처리된다.
- 잘못된 파일 형식은 적절한 4xx 에러를 반환한다.

---

### Phase 4. Document Parsing / Cleaning / Markdown Conversion
목표:
- 업로드 문서를 검색 가능한 텍스트 구조로 변환한다.

구현 항목:
- PDF 파서 구현
- DOCX 파서 구현
- TXT 파서 구현
- 공통 DocumentParser 인터페이스 정의
- Text Cleaning 로직 구현
- Markdown 변환 로직 구현

완료 기준:
- 업로드된 문서가 Markdown 기반 텍스트로 변환된다.
- 노이즈 제거 규칙이 적용된다.

---

### Phase 5. Chunking
목표:
- 문서를 의미 단위 chunk로 분할한다.

구현 항목:
- 헤더 기반 분할
- 문단 기반 분할
- 길이 기반 fallback 분할
- chunk size = 650 tokens 적용
- overlap = 120 tokens 적용
- chunk metadata 생성

완료 기준:
- 하나의 문서가 chunk 리스트로 변환된다.
- chunk metadata가 함께 생성된다.

---

### Phase 6. Embedding Integration
목표:
- chunk와 query에 대한 임베딩 생성을 구현한다.

구현 항목:
- Gemini Embedding Client 구현
- `gemini-embedding-001` 호출 로직 구현
- embedding dimension 1536 적용
- request / response DTO 구현

완료 기준:
- chunk 텍스트를 embedding vector로 변환할 수 있다.
- query 텍스트를 embedding vector로 변환할 수 있다.

---

### Phase 7. Elasticsearch Indexing
목표:
- chunk를 Elasticsearch에 저장하고 검색 가능한 상태로 만든다.

구현 항목:
- ChunkDocument 모델 구현
- Elasticsearch mapping 정의
- chunk indexing 구현
- document version / active 필드 반영
- 문서 상태 관리(uploaded, processing, processed, failed)

완료 기준:
- chunk가 Elasticsearch에 저장된다.
- 활성 chunk만 retrieval 대상이 된다.

---

### Phase 8. Async Indexing Pipeline
목표:
- 문서 업로드 이후 인덱싱을 비동기 처리한다.

구현 항목:
- Spring Async 기반 indexing job 구현
- 상태 전이 처리
- 실패 로그 및 예외 처리
- 재색인 흐름 구현

완료 기준:
- 업로드 API는 빠르게 반환된다.
- 실제 인덱싱은 비동기로 수행된다.
- 실패 시 상태가 `failed`로 기록된다.

---

### Phase 9. Retrieval Layer
목표:
- 하이브리드 검색을 구현한다.

구현 항목:
- BM25 검색 구현
- dense vector kNN 검색 구현
- RRF 기반 결과 결합 구현
- top_k = 5 적용
- ContextBuilder 구현

완료 기준:
- 사용자 질문에 대해 관련 chunk 5개를 반환할 수 있다.
- lexical + semantic 하이브리드 검색이 동작한다.

---

### Phase 10. Session Management
목표:
- Redis 기반 세션 저장 및 멀티턴 히스토리 관리를 구현한다.

구현 항목:
- session key 규칙 구현
- ChatSession / ChatMessage 모델 구현
- Redis repository 구현
- TTL 24시간 적용
- sliding expiration 적용
- 최근 메시지 10~20개 유지

완료 기준:
- session_id 기반 대화 이력이 저장/조회된다.
- 세션 TTL이 갱신된다.

---

### Phase 11. Query Rewriting
목표:
- 후속 질문을 standalone question으로 변환한다.

구현 항목:
- query rewrite prompt 구현
- QueryRewriteService 구현
- 세션 히스토리 반영 로직 구현

완료 기준:
- “그럼 배송비는?” 같은 후속 질문을 독립 질문으로 재작성할 수 있다.

---

### Phase 12. Prompt Construction & Answer Generation
목표:
- Strict QA 패턴으로 답변 생성 로직을 구현한다.

구현 항목:
- strict QA system prompt 작성
- PromptBuilder 구현
- Gemini Chat Client 구현
- `gemini-2.5-flash` 호출 로직 구현
- answer policy 반영

완료 기준:
- context 기반 한국어 답변이 생성된다.
- context에 없는 내용은 추측하지 않는다.

---

### Phase 13. Chat API
목표:
- 사용자 질문을 받아 최종 응답을 반환하는 API를 구현한다.

구현 항목:
- `POST /api/v1/rag/chat`
- session_id optional 처리
- retrieval → prompt → answer 파이프라인 연결
- sources 반환
- Redis 세션 append 처리

완료 기준:
- 질문에 대해 answer + sources + session_id가 반환된다.

---

### Phase 14. Session History API
목표:
- 저장된 세션 이력을 조회할 수 있게 한다.

구현 항목:
- `GET /api/v1/rag/sessions/{session_id}`
- SessionHistoryResponse 구현

완료 기준:
- 세션별 메시지 목록을 조회할 수 있다.

---

### Phase 15. Document Status API
목표:
- 문서 처리 상태를 조회할 수 있게 한다.

구현 항목:
- `GET /api/v1/rag/documents/{document_id}`
- DocumentStatusResponse 구현

완료 기준:
- 문서 상태(uploaded, processing, processed, failed)를 조회할 수 있다.

---

### Phase 16. Logging / Metrics / Observability
목표:
- 운영 가능한 수준의 로그와 메트릭을 제공한다.

구현 항목:
- 업로드/인덱싱/검색/LLM 호출 로그
- 에러 로그
- correlation id 반영
- Prometheus 메트릭 노출
- latency 측정

완료 기준:
- `/actuator/health`, `/actuator/prometheus` 사용 가능
- 주요 처리 단계의 latency 추적 가능

---

### Phase 17. Testing
목표:
- 핵심 로직과 API의 안정성을 검증한다.

구현 항목:
- chunking unit test
- retrieval unit test
- session handling test
- prompt building test
- document upload API test
- chat API integration test
- Elasticsearch / Redis Testcontainers 테스트

완료 기준:
- 핵심 서비스와 API에 대한 자동화 테스트가 존재한다.

---

### Phase 18. Final..

Jenkins CI 연동은 후속 작업으로 제외
현재는 로컬 빌드 및 Docker Compose 실행 가능 상태를 완료 기준으로 한다

## Delivery Rules

- 각 Phase는 독립적으로 빌드 가능해야 한다.
- 각 Phase 완료 시 컴파일 가능한 상태를 유지해야 한다.
- 대규모 리팩토링보다 점진적 구현을 우선한다.
- 외부 API 연동은 인터페이스를 통해 추상화한다.
- 테스트 가능한 구조를 우선한다.

## Platform Integration Rules

### 1. Build / Image Convention
- 컨테이너 이미지 빌드는 Jib를 사용한다.
- Jib 설정은 기존 서버 구성을 참조하여 동일한 방식으로 적용한다.
- 이미지명, 서비스명, 태그 등 네이밍만 본 서비스 기준으로 변경한다.

### 2. Monitoring Convention
- Prometheus 및 Actuator 노출 방식은 기존 서버와 동일한 운영 표준을 따른다.
- 메트릭 endpoint, scrape 방식, 공통 태그 규칙은 기존 서버 설정을 재사용한다.

### 3. Secret Management
- 민감한 환경설정(API Key, Redis/Elasticsearch 접속 정보, 내부 토큰 등)은 Vault를 통해 주입받는다.
- 운영 환경에서는 시크릿 정보를 application.yml에 직접 저장하지 않는다.
- 로컬 개발 환경에서만 별도 로컬 설정을 허용한다.

### 4. CDC / Debezium
- CDC 기반 문서 동기화가 필요한 경우 Debezium을 사용한다.
- Debezium 커넥터 생성은 기존 운영 스크립트 또는 템플릿을 재사용한다.
- 커넥터 정의가 필요한 경우 서비스명, 토픽명, 구독 대상 테이블명만 본 서비스 기준으로 변경한다.
- CDC 사용 여부는 MVP 범위에서 명시적으로 결정한다.

### 5. etc..
- 초기 버전은 문서 업로드 API 기반 ingestion만 지원한다.
- 기존 NOTICE/FAQ 데이터의 CDC 기반 동기화는 향후 확장 항목으로 둔다.
- CDC가 필요해질 경우 Debezium과 기존 커넥터 생성 스크립트를 재사용한다.

## Coding Rules

### 1. Layered Architecture
- Controller는 요청/응답 처리만 담당하고 비즈니스 로직을 포함하지 않는다.
- Service는 비즈니스 로직을 담당한다.
- Repository는 저장소 접근만 담당한다.
- Client는 외부 API 호출만 담당한다.
- Config 클래스는 설정 정의만 담당한다.

### 2. Model Separation
- API Request/Response DTO, Domain Model, Elasticsearch Document, Redis Entity를 분리한다.
- 저장소 모델을 API 응답에 직접 사용하지 않는다.
- Elasticsearch document 모델과 Redis 저장 모델은 도메인 모델과 분리한다.

### 3. Exception Handling
- 예외는 공통 예외 체계를 사용한다.
- 비즈니스 예외는 명시적인 예외 타입으로 관리한다.
- 모든 예외 응답은 GlobalExceptionHandler를 통해 일관된 형식으로 반환한다.
- Controller 단에서 중복 try-catch를 작성하지 않는다.

### 4. Configuration Management
- 모델명, index 이름, chunk size, overlap, top_k, TTL, timeout, embedding dimension 등 주요 설정값은 하드코딩하지 않는다.
- 모든 설정값은 application.yml 또는 환경변수로 관리한다.

### 5. Prompt Management
- 프롬프트 문자열을 코드에 하드코딩하지 않는다.
- 프롬프트는 `resources/prompts` 아래 파일로 관리한다.

### 6. External Integration
- LLM 호출은 전용 Client 클래스로 분리한다.
- Embedding 호출은 전용 Client 클래스로 분리한다.
- 외부 API 연동 코드는 인터페이스 또는 교체 가능한 구조로 작성한다.

### 7. Method / Class Responsibility
- 하나의 클래스는 하나의 주요 책임만 가진다.
- 하나의 메서드는 하나의 명확한 역할만 수행하도록 작성한다.
- 하나의 Service 클래스에 과도한 책임을 몰아넣지 않는다.

### 8. Logging
- 로그는 구조화된 형태로 남긴다.
- request id, session id, document id 등 추적 가능한 식별자를 포함한다.
- 민감한 정보(API Key, 전체 문서 원문, 사용자 민감 입력)는 로그에 남기지 않는다.
- 주요 이벤트(업로드, 인덱싱, retrieval, LLM 호출 실패)는 반드시 로그를 남긴다.

### 9. Testing
- 주요 비즈니스 로직은 단위 테스트 가능하도록 작성한다.
- 외부 API 호출은 mocking 가능해야 한다.
- 정적 유틸리티와 강한 결합을 피한다.

### 10. Naming Conventions
- 클래스명은 역할이 드러나도록 작성한다.
- Controller, Service, Repository, Client, Config, Response, Request 접미사를 일관되게 사용한다.
- 축약어보다 의미가 분명한 이름을 우선한다.

### 11. Asynchronous Processing
- 문서 인덱싱과 재색인 작업은 비동기 처리로 구현한다.
- 비동기 작업 내부에서도 예외 처리와 상태 기록을 수행해야 한다.

### 12. Storage Rules
- Elasticsearch는 검색용 데이터만 저장한다.
- Redis는 세션 데이터만 저장한다.
- 저장소 간 역할을 혼합하지 않는다.

### 13. API Rules
- 모든 비즈니스 API는 `/api/v1/rag` 하위 경로를 사용한다.
- 운영용 endpoint는 Actuator 경로로 분리한다.

## Test Requirements

### 1. General Policy
- 핵심 비즈니스 로직은 자동화 테스트 대상으로 포함해야 한다.
- 단위 테스트와 통합 테스트를 함께 작성해야 한다.
- 외부 API 의존성은 mocking 가능한 구조로 분리해야 한다.
- 테스트는 빌드 파이프라인에서 반복 실행 가능해야 한다.

---

### 2. Unit Test Requirements

다음 컴포넌트는 단위 테스트를 작성해야 한다.

#### 2.1 Document Processing
- 파일 형식 검증 로직
- 텍스트 cleaning 로직
- Markdown 변환 로직
- chunking 로직
- chunk metadata 생성 로직

#### 2.2 Retrieval Logic
- retrieval query 조건 생성 로직
- hybrid search 결과 결합 로직
- context builder 로직
- source selection 로직

#### 2.3 Session Logic
- 세션 생성 로직
- 세션 메시지 append 로직
- 최근 메시지 trimming 로직
- TTL 갱신 로직

#### 2.4 Prompt / Answer Logic
- strict QA prompt 구성 로직
- query rewriting 입력 구성 로직
- answer response 조립 로직

#### 2.5 Exception / Utility
- 공통 예외 처리 로직
- 응답 변환 로직
- ID 생성 및 공통 유틸 로직

---

### 3. Integration Test Requirements

다음 시나리오는 통합 테스트로 검증해야 한다.

#### 3.1 Elasticsearch Integration
- chunk document가 Elasticsearch에 정상 저장되는지 검증해야 한다.
- lexical 검색(BM25)이 동작하는지 검증해야 한다.
- vector 검색이 동작하는지 검증해야 한다.
- active=true 조건이 retrieval에 반영되는지 검증해야 한다.

#### 3.2 Redis Integration
- 세션 데이터가 Redis에 정상 저장/조회되는지 검증해야 한다.
- TTL이 설정되는지 검증해야 한다.
- sliding expiration 동작을 검증해야 한다.

#### 3.3 Async Indexing Integration
- 문서 업로드 후 비동기 인덱싱이 시작되는지 검증해야 한다.
- 성공 시 상태가 processed로 변경되는지 검증해야 한다.
- 실패 시 상태가 failed로 기록되는지 검증해야 한다.

#### 3.4 API Layer Integration
- 문서 업로드 API가 정상 동작하는지 검증해야 한다.
- 문서 상태 조회 API가 정상 동작하는지 검증해야 한다.
- chat API가 answer + sources + session_id를 반환하는지 검증해야 한다.
- session history API가 저장된 메시지를 반환하는지 검증해야 한다.

---

### 4. External Dependency Test Policy

- Gemini Chat API와 Embedding API는 실제 호출 대신 mocking 또는 stub 방식으로 검증해야 한다.
- 외부 API 성공/실패/timeout 시나리오를 테스트해야 한다.
- 외부 API 응답 파싱 로직은 독립적으로 테스트 가능해야 한다.

---

### 5. Testcontainers Usage

- Elasticsearch 통합 테스트는 Testcontainers 기반으로 실행해야 한다.
- Redis 통합 테스트는 Testcontainers 기반으로 실행해야 한다.
- 테스트 환경은 로컬 개발 환경과 CI 환경에서 동일하게 실행 가능해야 한다.

---

### 6. Required Test Scenarios

최소한 아래 시나리오는 자동화 테스트에 포함해야 한다.

#### 6.1 Document Upload
- 허용된 파일 형식 업로드 성공
- 허용되지 않은 파일 형식 업로드 실패
- 파일 누락 요청 실패

#### 6.2 Document Processing
- PDF 파싱 성공
- DOCX 파싱 성공
- TXT 파싱 성공
- cleaning 규칙 적용 검증
- Markdown 변환 결과 검증
- 헤더 기반 chunking 검증
- chunk size / overlap 적용 검증

#### 6.3 Retrieval
- BM25 검색 결과 반환
- vector 검색 결과 반환
- hybrid 검색 결과 결합
- 검색 결과가 없을 때 빈 source 반환

#### 6.4 Chat
- session_id 없이 요청 시 새 세션 생성
- session_id 포함 요청 시 기존 세션 사용
- 후속 질문에서 query rewriting 적용
- answer에 source 포함
- context가 없을 때 fallback 응답 반환

#### 6.5 Session
- Redis 세션 저장 성공
- 최근 메시지 trimming 검증
- TTL 갱신 검증

#### 6.6 Failure Cases
- embedding API 실패
- chat API 실패
- Elasticsearch 저장 실패
- Redis 접근 실패
- 예상하지 못한 예외 발생 시 표준 오류 응답 반환

---

### 7. Non-Testable by Mock Only

다음 항목은 mocking만으로 충분하지 않으므로 실제 통합 테스트가 필요하다.

- Elasticsearch index mapping 및 retrieval 동작
- Redis TTL 및 sliding expiration
- 비동기 인덱싱 상태 전이
- Controller → Service → Repository 전체 요청 흐름

---

### 8. Coverage Priority

테스트 우선순위는 다음과 같다.

1. chat API 핵심 흐름
2. document indexing 흐름
3. retrieval 품질 관련 로직
4. session handling
5. 예외 및 실패 처리

---

### 9. Deliverable Requirement
- 핵심 서비스에 대한 단위 테스트가 포함되어야 한다.
- 주요 API에 대한 통합 테스트가 포함되어야 한다.
- Elasticsearch 및 Redis에 대한 Testcontainers 테스트가 포함되어야 한다.
- 외부 AI API는 mocking 기반 테스트가 포함되어야 한다.

### 10. Test Design Rules
- 단위 테스트는 외부 저장소 및 외부 API에 의존하지 않아야 한다.
- 통합 테스트는 실제 Elasticsearch 및 Redis 컨테이너를 사용해야 한다.
- 테스트 데이터는 각 테스트 케이스 간 독립성을 보장해야 한다.
- flaky test를 허용하지 않는다.

## Build / Env

### 1. Current Scope
- 본 프로젝트는 현재 로컬 개발 및 수동 빌드/실행 환경까지만 범위에 포함한다.
- CI 파이프라인(Jenkins)은 이번 구현 범위에 포함하지 않는다.
- Jenkins 연동은 후속 작업으로 진행한다.

### 2. Build Strategy
- 애플리케이션 빌드는 Gradle을 사용한다.
- 컨테이너 이미지는 Jib를 사용하여 생성한다.
- Jib 설정은 기존 서버 설정을 참고하되, 서비스명 및 이미지명 네이밍만 본 서비스 기준으로 변경한다.

### 3. Local Development
- 로컬 개발 환경은 Docker Compose를 사용한다.
- 최소 실행 구성은 다음을 포함한다.
  - Elasticsearch
  - Redis
  - RAG 서비스

### 4. Runtime Environments
- 본 서비스는 다음 환경을 고려한다.
  - local
  - dev
  - prod

- Spring Profile 기반으로 환경을 분리한다.
- 환경별 설정 파일:
  - application.yml
  - application-local.yml
  - application-dev.yml
  - application-prod.yml

### 5. Configuration Policy
- 모든 설정값은 외부 설정으로 관리한다.
- 하드코딩을 금지한다.
- 주요 설정값:
  - Elasticsearch 설정
  - Redis 설정
  - Gemini API 설정
  - chunk size / overlap
  - retrieval top_k
  - timeout
  - embedding dimension
  - index name

### 6. Secret Management
- 모든 민감한 값은 Vault를 통해 주입한다.
- 운영 환경에서는 application.yml에 시크릿을 저장하지 않는다.
- 로컬 개발 환경에서만 제한적으로 로컬 설정을 허용한다.

### 7. Observability
- 헬스체크는 `/actuator/health` 를 사용한다.
- 메트릭은 `/actuator/prometheus` 를 사용한다.
- Prometheus 설정 및 scrape 방식은 기존 서버와 동일한 방식으로 맞춘다.

### 8. Storage Isolation
- database per service 원칙을 준수한다.
- Elasticsearch와 Redis는 본 서비스 전용 저장소를 사용한다.
- 다른 서비스와 저장소를 공유하지 않는다.

### 9. CDC / Debezium Policy
- 초기 버전은 문서 업로드 API 기반 ingestion만 사용한다.
- CDC 기반 ingestion은 이번 범위에 포함하지 않는다.
- 향후 필요 시 Debezium을 사용하고, 커넥터 생성은 기존 스크립트를 재사용한다.

### 10. Future CI
- Jenkins 기반 CI 파이프라인은 후속 작업으로 진행한다.
- Jenkins 적용 시 기존 서버 파이프라인 구조를 참고하고, 서비스명 및 이미지명 네이밍만 본 서비스 기준으로 변경한다.

## Acceptance Criteria

## Acceptance Criteria

본 서비스는 아래 조건을 모두 만족할 경우 완료된 것으로 간주한다.

---

### 1. Application Execution
- 애플리케이션이 로컬 환경에서 정상적으로 실행되어야 한다.
- `/actuator/health` endpoint가 정상적으로 `UP` 상태를 반환해야 한다.
- Docker Compose를 통해 Elasticsearch, Redis, 애플리케이션이 함께 실행 가능해야 한다.

---

### 2. Document Upload & Indexing
- PDF, DOCX, TXT 파일 업로드가 정상적으로 수행되어야 한다.
- 업로드 요청 시 document_id가 생성되어 반환되어야 한다.
- 문서 업로드 이후 인덱싱이 비동기로 수행되어야 한다.
- 문서 상태가 `uploaded → processing → processed`로 정상 전이되어야 한다.
- 파싱 실패 시 상태가 `failed`로 기록되어야 한다.

---

### 3. Document Processing Quality
- 업로드된 문서는 Markdown 형태로 변환되어야 한다.
- 헤더 기반 chunking이 정상적으로 수행되어야 한다.
- chunk size는 설정값(기본 650 tokens)에 맞게 생성되어야 한다.
- overlap은 설정값(기본 120 tokens)을 반영해야 한다.
- 각 chunk에 metadata가 정상적으로 포함되어야 한다.

---

### 4. Embedding & Indexing
- 모든 chunk에 대해 embedding이 생성되어야 한다.
- embedding dimension은 1536으로 유지되어야 한다.
- chunk 데이터가 Elasticsearch에 정상 저장되어야 한다.
- active=true 조건의 chunk만 retrieval 대상이 되어야 한다.

---

### 5. Retrieval Functionality
- 사용자 질문에 대해 Elasticsearch 하이브리드 검색이 수행되어야 한다.
- BM25 + vector 검색이 함께 동작해야 한다.
- RRF 기반 결과 결합이 적용되어야 한다.
- top_k=5 기준으로 결과가 반환되어야 한다.

---

### 6. Chat API
- `/api/v1/rag/chat` API가 정상 동작해야 한다.
- session_id 없이 요청 시 새로운 세션이 생성되어야 한다.
- session_id 포함 시 기존 세션이 유지되어야 한다.
- 응답에는 다음이 포함되어야 한다:
  - session_id
  - answer
  - sources

---

### 7. Answer Quality
- 답변은 항상 한국어로 생성되어야 한다.
- 답변은 retrieval context 기반으로만 생성되어야 한다.
- context에 없는 정보는 포함하지 않아야 한다.
- 답을 찾지 못한 경우 명확한 fallback 응답을 반환해야 한다.
- sources는 최대 3개까지 포함되어야 한다.

---

### 8. Session Management
- 세션 데이터가 Redis에 저장되어야 한다.
- 세션 TTL은 24시간으로 적용되어야 한다.
- 세션 접근 시 TTL이 갱신되어야 한다 (sliding expiration).
- 최근 메시지 10~20개만 유지되어야 한다.

---

### 9. Re-indexing
- 동일 문서 재업로드 시 version이 증가해야 한다.
- 신규 version이 인덱싱 완료된 후 active 상태로 전환되어야 한다.
- 이전 version은 inactive 상태로 전환되어야 한다.
- retrieval은 항상 active version만 사용해야 한다.

---

### 10. Error Handling
- 잘못된 파일 형식 업로드 시 4xx 에러를 반환해야 한다.
- 필수 파라미터 누락 시 4xx 에러를 반환해야 한다.
- 내부 오류 발생 시 표준화된 에러 응답을 반환해야 한다.
- 외부 API 실패 시 서비스가 비정상 종료되지 않아야 한다.

---

### 11. Observability
- `/actuator/health` endpoint가 정상 동작해야 한다.
- `/actuator/prometheus` endpoint가 노출되어야 한다.
- 주요 이벤트(업로드, 인덱싱, retrieval, LLM 호출 실패)가 로그에 기록되어야 한다.

---

### 12. Configuration & Security
- 모든 주요 설정값은 외부 설정으로 관리되어야 한다.
- 민감한 설정값은 Vault를 통해 주입되어야 한다.
- application.yml에 시크릿 값이 하드코딩되어 있지 않아야 한다.

---

### 13. Storage Isolation
- Elasticsearch와 Redis는 본 서비스 전용 저장소를 사용해야 한다.
- 다른 서비스와 저장소를 공유하지 않아야 한다.

---

### 14. Test Coverage
- 핵심 비즈니스 로직에 대한 단위 테스트가 존재해야 한다.
- 주요 API에 대한 통합 테스트가 존재해야 한다.
- Elasticsearch 및 Redis 연동 테스트가 존재해야 한다.
- 외부 API는 mocking 기반 테스트가 포함되어야 한다.

---

### 15. Scope Constraints
- Jenkins 기반 CI 파이프라인 구성은 이번 범위에 포함하지 않는다.
- CDC 기반 ingestion(Debezium)은 이번 범위에 포함하지 않는다.
