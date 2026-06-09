# KevinCY-Kodex — RAG Guideline
**Version:** 2025.11.27 (Universal Standard)
**Scope:** RAG Engine · Hybrid Retrieval · Hybrid LLM (API/Local) · Document QA

---

## 1. RAG Overview

RAG(Retrieval-Augmented Generation)는 다음 4단계로 구성됩니다.

1.  **Ingestion**: 문서 수집 및 전처리 (Cleaning & Metadata Tagging)
2.  **Chunking & Embedding**: 의미 단위 분할 및 벡터화
3.  **Retrieval**: 하이브리드 검색 (BM25 + Dense + Late Rerank)
4.  **Answer Generation**: LLM 기반 응답 생성 (Reasoning Trace 포함)

모든 단계는 독립적인 모듈로 설계하며, `Service` 계층에서 `LangGraph`를 통해 오케스트레이션됩니다.

---

## 2. RAG Architecture Structure

RAG 관련 모듈의 폴더 구조는 프로젝트 전체 표준을 따릅니다. 각 단계는 **단일 책임 원칙(SRP)**을 준수해야 합니다.
> 자세한 AI 관련 폴더 구조는 `Folder_Standards.md` 문서를 참조하세요.

---

## 3. Ingestion Rules

### 3.1 지원 문서 포맷
-   **Structured:** CSV, JSON, Excel
-   **Unstructured:** PDF, DOCX, HWP(전용 파서 필수), HTML
-   **Domain Spec:** 공공기관/사내 규정 문서, 기술 매뉴얼

### 3.2 Ingestion 기본 규칙
-   **Cleaning:** 텍스트 추출 후 불필요한 노이즈(목차, 페이지 번호, 머리글/바닥글, 워터마크)를 반드시 제거합니다.
-   **Metadata Injection:** 모든 청크에 다음 메타데이터를 필수적으로 태깅합니다.
    -   `source`: 파일명 또는 출처 URL
    -   `doc_type`: 문서 유형 (예: 규정, 매뉴얼, 공지)
    -   `section`: 문서 내 섹션/목차 정보
    -   `page`: 원문 페이지 번호 (검증 용이성 확보)
    -   `date`: 문서 작성일 또는 최종 수정일

### 3.3 Domain Specific Preprocessing (범용)
-   **Context Isolation:** 현재 프로젝트의 도메인(예: 파주시, 삼성전자)과 무관한 타 기관 정보가 포함된 경우, `exclude_flag`를 설정하거나 별도 주석 처리합니다.
-   **Entity Extraction:** 전화번호, 부서명, 담당자 정보는 `meta={"contact": "...", "dept": "..."}` 형태로 별도 추출하여 필드에 저장합니다.

---

## 4. Chunking Strategy

### 4.1 기본 Spec
-   **Default Size:** `800` tokens
-   **Overlap:** `200` tokens
-   **Flexibility:** 문서의 성격(법조항 vs 매뉴얼)에 따라 탄력적으로 조정 가능.

### 4.2 분할 원칙 (Semantic Chunking)
-   단순 길이 기반 분할보다 **의미 단위 분절(문단, 표, 섹션)**을 최우선으로 합니다.
-   **Table Handling:** 표 데이터는 마크다운 형태로 변환하거나, 별도의 청크 타입(`TableChunk`)으로 관리합니다.
-   **Noise Reduction:** OCR 또는 STT 결과물의 경우, 청킹 전 스펠링 교정(Spell Check) 단계를 거칩니다.

### 4.3 Chunk Schema
-   하나의 `chunk` 객체는 다음 필드를 가집니다:
    -   `id` (UUID), `text` (Content), `embedding` (Vector), `metadata` (Dict)

---

## 5. Embedding Rules

### 5.1 기본 모델
-   **Primary:** `jhgan/ko-sbert-nli` (한국어 특화)
-   **Secondary:** `intfloat/e5-base` (범용/영문 혼용 시)
-   **Multilingual:** `intfloat/multilingual-e5-base`

### 5.2 Embedding 정책
-   모든 `chunk`는 임베딩 전 정규화(Normalization) 과정을 거칩니다.
-   **Filter:** 길이가 너무 짧거나(예: 10자 미만), 의미가 없는 청크는 임베딩 대상에서 제외합니다.

### 5.3 저장소(벡터스토어)
-   **Dev/Local:** `ChromaDB` 또는 `FAISS` (Local File)
-   **Prod:** `Milvus` 또는 `Pgvector`
-   **Management:** ID와 메타데이터 매핑은 별도 `JSONL` 또는 RDBMS로 이중 관리합니다.

---

## 6. Retrieval Rules

### 6.1 Hybrid Retrieval Strategy
1.  **Keyword Search (Sparse):** `BM25` 알고리즘으로 정확한 용어 매칭 (Recall 확보).
2.  **Vector Search (Dense):** 임베딩 기반 유사도 검색 (Semantic Context 확보).
3.  **Ensemble:** 위 두 결과를 가중치 합산(Reciprocal Rank Fusion 등)하여 상위 N개 후보군 생성.

### 6.2 Late Reranking
-   **Process:** 1차 검색된 Top-N (예: 20개) 문서에 대해 `Cross-Encoder` 모델로 정밀 재순위(Rerank)를 수행합니다.
-   **Output:** 최종적으로 품질 점수(`score`)가 높은 Top-K (3~5개) 청크만 LLM에게 전달합니다.

### 6.3 Retrieval 품질 규칙
-   **Domain Filter:** 타 도메인 문서가 섞이지 않도록 메타데이터 필터링을 우선 적용합니다.
-   **Deduplication:** 내용이 95% 이상 유사한 중복 청크는 하나만 남기고 제거합니다.

---

## 7. Answer Generation Rules (The "LLM" Phase)

### 7.1 Prompt 정책 (Hybrid Strategy)
*`Code_Style.md` 및 `rules.md`의 설정을 준수합니다.*
-   **API Mode (Primary):** `SKT A.X-4.0` (Env: `AX_MODEL_NAME`) 사용. 복잡한 추론과 고품질 작문에 적합.
-   **Local Mode (Fallback):** `Ollama` (Llama 3, Mistral) 사용. 단순 요약이나 보안 데이터 처리에 적합.

### 7.2 응답 생성 원칙 & Constraints
**Role:** System Prompt에 "문서 기반 답변 전문가" 역할을 부여합니다.
-   **Constraints:**
    -   *"주어진 컨텍스트 내에서만 답변하라."* (Hallucination 방지)
    -   *"답변의 근거가 되는 문장을 반드시 인용하라."* (Citation)
    -   *"모르면 모른다고 답하라."* (Honesty)

### 7.3 Output Schema (JSON)
```json
  {
    "answer": "생성된 최종 답변 텍스트",
    "sources": ["doc_id_1#chunk_2", "doc_id_3#chunk_5"],
    "reasoning": "답변을 도출한 논리적 근거 (Internal use)",
    "followup": ["관련된 추가 질문 1", "관련된 추가 질문 2"]
  }
```

---

## 8. Post-Processing Rules

### 8.1 텍스트 정제
-   Text Cleaning: 중복된 문장, 불필요한 접속사, 기계적인 어투를 자연스럽게 다듬습니다.
-   Safety Filter: 비속어, 개인정보(PII) 포함 여부를 최종 검사합니다.

---

## 9. LangGraph Workflow Rules

### 9.1 RAG Pipeline Graph 구성
-   **필수 노드**: `preprocess_node`, `retrieval_node`, `rerank_node`, `generation_node`, `postprocess_node`
-   **흐름 제어**: `router_node`를 통해 RAG가 불필요한 일반 대화는 바로 LLM으로 연결합니다.

### 9.2 State 구조
-   **Core State**:
    -   `query`: 사용자 질문 (str)
    -   `user_meta`: 사용자 정보 (dict, optional)
    -   `retrieved_docs`: 1차 검색 결과 (list[Document])
    -   `reranked_docs`: Rerank 후 최종 문서 (list[Document])
    -   `answer`: 생성된 답변 (str)
    -   `sources`: 출처 ID 목록 (list[str])

### 9.3 에러 처리
-   **Retrieval Failure**: 검색 실패 시 에러를 내지 않고 `빈 리스트([])`를 반환하여 로직이 끊기지 않게 합니다.
-   **No Context**: 유효한 `chunk`가 없는 경우, LLM 호출을 생략하고 "문서 내 근거 없음" 메시지를 즉시 반환합니다.
-   **System Error**: 파이프라인 내부 오류 발생 시, 사전에 정의된 `fallback` 응답을 생성합니다.

---

## 10. Evaluation Rules

### 10.1 품질 지표 (Metrics)
-   `Recall@K`, `Precision@K`: 검색 정확도 측정
-   `Rerank Score`: Reranking 모델의 정렬 성능
-   `Answer Faithfulness`: 답변이 문서 내용에 기반했는지 측정 (Hallucination 감지)
-   `Hallucination Score`: 팩트와 다른 내용 포함 여부

### 10.2 자동 테스트 (CI/CD)
-   **Golden Set**: `Known-answer`(정답이 있는 질문셋) 베이스라인과 주기적으로 비교합니다.
-   **Coverage**: `Critical keyword`(필수 키워드)가 검색 결과에 포함되는지 검사합니다.
-   **Alignment**: 검색된 `Chunk`와 생성된 `Answer` 간의 정합성을 검증합니다.

---

## 11. 운영(Production) 규칙

### 11.1 Index Versioning & Safety
-   매 `ingest` 작업마다 `index_version=YYYYMMDD_vX` 태그를 적용하여 관리합니다.
-   **Deletion Guard**: 오래된 인덱스는 삭제하되, 에이전트는 인덱스 삭제(drop) 작업 전 반드시 사용자 승인을 받아야 합니다.
-   문서 변경 시 해당 문서는 전부 재임베딩(Full Re-indexing) 하는 것을 원칙으로 합니다.

### 11.2 Monitoring
-   **Retrieval hit ratio**: 검색 성공률
-   **LLM latency**: 답변 생성 소요 시간
-   **Pipeline latency**: 전체 파이프라인(STT/TTS 포함) 지연 시간
-   **Traffic**: 사용자 섹션별/시간대별 질문량 분석

### 11.3 데이터 보안 (Data Security)
-   **Masking**: 전화번호, 주민번호, 여권번호 등 민감 정보(PII)는 임베딩 전에 반드시 **마스킹(Masking) 처리**하여 저장합니다.
    -   *Example:* `010-1234-5678` -> `010-****-****`

End of KevinCY-Kodex RAG Guideline