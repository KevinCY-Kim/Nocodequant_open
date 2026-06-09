# KevinCY-Kodex — Architecture Guideline
**Version:** 2025.11.27 (Universal Standard)
**Scope:** FastAPI + LangGraph + RAG + STT/TTS + Web Frontend
**Reference:** 본 문서는 `KevinCY-Kodex (Master Rule)`의 하위 상세 기술 문서입니다.

---

## 1. System Overview

본 시스템은 'Hybrid AI Service'를 지향하며 다음 서브시스템으로 구성됩니다.

-   **API Backend**: FastAPI (Clean Architecture)
-   **AI Engine**: LangGraph Orchestration + Hybrid LLM (SKT A.X / Ollama)
-   **Voice Pipeline**: STT (Whisper) + TTS (Generative)
-   **Web Frontend**: HTMX + Tailwind + Jinja2 (SSR/SPA Hybrid)
-   **Infra & Ops**: Docker, Conda, Local/Prod 환경 분리

---

## 2. Backend Layered Architecture

### 2.1 디렉토리 표준 구조 및 계층별 책임
프로젝트의 폴더 구조와 각 계층의 물리적 배치는 **`Folder_Standards.md`** 문서를 따릅니다.
*(Master Rule에 정의된 SRP 원칙과 Clean Architecture를 준수할 것)*

---

## 3. Request Lifecycle

### 3.1 텍스트 기반 질의 (RAG / 챗봇)

1.  **Client**: `/api/chat`으로 `POST` 요청
2.  **Router**: RequestModel 검증 후 `Service` 호출
3.  **Service**:
    -   사용자 Context(메타데이터) 조회
    -   **LangGraph Pipeline** 실행 (Retrieval → Reasoning → Generation)
    -   결과를 `ResponseModel`로 매핑
4.  **Router**: 최종 JSON 응답 반환
5.  **Logger**: Latency 및 Token Usage 기록 (JSON 구조)

### 3.2 음성 기반 질의 (STT → RAG → TTS)

1.  **Client**: `/api/voice-chat`으로 오디오 Blob 업로드
2.  **Router**: 임시 파일 저장(Temp) 후 `Service` 호출
3.  **Service**:
    -   **STT Engine**: Audio → Text 변환
    -   **Text Pipeline**: 3.1의 RAG 흐름 재사용 (Code Reusability)
    -   **TTS Engine**: 생성된 답변(Text) → Audio 변환
4.  **Router**: 텍스트 답변(JSON) + 오디오 파일 URL 반환
5.  **Cleanup**: 요청 완료 후 임시 업로드 파일 삭제 (Antigravity Safety Rule 준수)

---

## 4. AI / RAG / LangGraph Architecture

> **Note:** 상세 구현 수치(Chunk size 등)는 `Master Rule`의 설정을 우선합니다.

### 4.1 RAG Pipeline (Standard Flow)

#### Ingestion
-   Source: PDF, DOCX, HWP 등 비정형 데이터
-   Process: 텍스트 추출 → 메타데이터 태깅(Source, Page, Date)

#### Chunking & Embedding
-   **Spec:** `chunk_size = 800`, `chunk_overlap = 200` (Master Rule 동기화)
-   **Model:** `ko-sbert-nli` (Semantic Search 최적화)
-   **DB:** Vector DB (FAISS/Chroma) + Metadata Filtering

#### Retrieval (Hybrid)
-   **Algorithm:** `BM25` (Keyword) + `Dense` (Semantic)
-   **Rerank:** 후보군 추출 후 Cross-Encoder로 재정렬(Late Reranking)

#### Answer Generation
-   **Model Strategy:**
    -   **Default:** `SKT A.X-4.0` (API)
    -   **Fallback:** `Ollama` (Local)
-   **Format:** 답변 생성 시 근거(Reasoning Trace)와 출처(Citation) 명시

### 4.2 LangGraph Structure
-   **Location:** `/app/ai/graph.py`
-   **Node Design:**
    -   `router_node`: 질문 의도 분류 (General vs RAG)
    -   `retrieval_node`: 문서 검색
    -   `generation_node`: 답변 생성
    -   `grade_node`: 답변 품질 평가 (Hallucination Check)

---

## 5. Voice Pipeline (STT / TTS)

### 5.1 STT (Speech-to-Text)
-   **Engine:** `Faster-Whisper` (Local/GPU Optimized)
-   **Optimization:** `Beam Size=5`, `VAD_filter=True`
-   **Language:** 한국어(ko) 자동 감지

### 5.2 TTS (Text-to-Speech)
-   **Engine:** `gTTS` (Basic) or 상용 API
-   **Output:** `.mp3` or `.wav`
-   **Naming:** `{timestamp}_{uuid}.mp3` (충돌 방지)

### 5.3 Pipeline 규칙
-   STT와 TTS는 `Service` 계층에서 별도 모듈로 분리하여 관리.
-   Router는 비즈니스 로직 없이 파일 I/O만 처리.

---

## 6. Frontend Architecture

### 6.1 템플릿 구조

```
/templates/
  ├── layout/
  │   ├── base.html
  │   ├── header.html
  │   └── footer.html
  ├── pages/
  │   ├── index.html
  │   ├── chat.html
  │   └── voice_chat.html
  └── partials/
      ├── chat_message.html
      └── loader.html
```

### 6.2 HTMX Integration 규칙
-   **Namespace:** 모든 `hx-post`, `hx-get` 요청은 `/api/` 프리픽스를 사용합니다.
-   **Implementation Pattern:**
    -   **Form 예시:** `<form hx-post="/api/chat" hx-target="#chat-window" hx-swap="beforeend">`
    -   **Loading:** `hx-indicator`를 사용하여 AI 처리 중 스피너/로딩 바를 반드시 표시합니다.

### 6.3 UI/UX Design Principles
-   **Mobile First:** Tailwind의 `md:`, `lg:` prefix를 활용하여 모바일 화면을 최우선으로 설계합니다.
-   **Card UI:** 콘텐츠는 여백(Whitespace)이 충분한 **카드형 UI**로 구성하여 가독성을 높입니다.
-   **Feedback:** 4xx/5xx 오류 발생 시, 시스템 로그 노출 대신 **사용자 친화적인 문장**으로 Toast 또는 Modal 알림을 제공합니다.

---

## 7. Infra / Environment / Deployment

### 7.1 환경 분리 및 설정
-   **Environment:** `local`, `staging`, `prod` 환경을 명확히 분리합니다.
-   **Config:** 각 환경별 설정은 `.env` 파일로 관리하며, `AX_MODEL_NAME` 등 Master Rule의 필수 변수를 포함합니다.

### 7.2 Dependency & Resource Management
-   **Package:** `conda` 환경, `requirements.txt`, 또는 `env.yml`을 통해 의존성 및 Python 버전을 고정합니다.
-   **Isolation:** STT/LLM 등 GPU 자원이 필요한 엔진은 웹 서버와 프로세스/컨테이너 레벨에서 분리하여 운영 안정성을 확보합니다.

### 7.3 Logging & Monitoring
-   **Metrics:** 요청 수(RPS), 응답 시간(Latency), 오류 비율(Error Rate)을 핵심 지표로 모니터링합니다.
-   **Log Fields:** 모든 로그에는 `path`, `method`, `status_code`, `latency`, `user_id`를 포함하여 추적 가능하게 합니다.

---

## 8. Cross-Cutting Concerns

### 8.1 Global Error Handling
-   **Location:** 전역 예외 처리기는 `/app/core/exception_handlers.py`에 정의합니다.
-   **Response Format (Standard):**
    ```json
    {
      "error_code": "E400",
      "message": "입력 값이 올바르지 않습니다.",
      "detail": "Field 'query' is missing."
    }
    ```

### 8.2 Security
-   **CORS:** 최소한의 Origin만 허용하는 화이트리스트 정책을 적용합니다.
-   **Privacy:** 로그 저장 시 개인정보(PII)나 API Key 등 민감 정보는 반드시 마스킹 처리합니다.

### 8.3 Configuration Management
-   **Library:** `/app/core/config.py`에서 `Pydantic`의 `BaseSettings`를 사용하여 타입 안전(Type-safe)하게 설정을 로드합니다.

---

## 9. Architectural Manifesto (Core Principles)
*에이전트는 코드 설계 시 다음 5대 원칙을 최종 점검 기준(Checklist)으로 삼으십시오.*

1.  **Clean Architecture First:** 모든 코드는 계층 간 의존성 규칙을 최우선으로 준수한다. (Controller → Service → Repository/AI)
2.  **Thin Router, Fat Service:** `Router`는 요청 검증과 반환만 담당하며, 모든 비즈니스 로직은 **`Service`** 계층에 집중한다.
3.  **Decoupled AI Engine:** AI/LLM 파트(LangGraph, Prompt)는 비즈니스 로직과 분리하여, 모델이 변경되어도 서비스 코드는 영향받지 않도록 설계한다.
4.  **Universal Extensibility:** 특정 도메인(예: 파주시)에 종속되지 않고, 타 도메인으로 즉시 확장/복제 가능한 모듈 구조를 지향한다.
5.  **Reusable Pipeline:** STT, TTS, RAG 파이프라인은 일회용 함수가 아닌, 재사용 가능한 **클래스 또는 모듈** 단위로 구현한다.

---

# End of Architecture Guideline
