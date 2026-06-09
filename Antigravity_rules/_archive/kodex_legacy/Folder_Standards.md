# KevinCY-Kodex — Folder Standards
**Version:** 2025.11.27 (Universal Standard)
**Scope:** FastAPI · LangGraph · RAG · STT/TTS · Web(HTMX+Tailwind)
**Viewer Note:** Please render tree structures in fixed-width font (Code Block).
---

## 1. Overview

이 문서는 프로젝트 파일·폴더 구조의 표준 규칙을 정의합니다.
프로젝트는 Clean Architecture 원칙에 따라 다음과 같은 핵심 계층으로 구성됩니다.

-   `core`: 핵심 설정, 로거, 보안 등
-   `routers`: API 엔드포인트
-   `services`: 비즈니스 로직
-   `repositories`: 데이터베이스, 파일, 외부 API 등 IO
-   `models`: Pydantic 스키마
-   `ai`: LLM, RAG, LangGraph 등 AI 엔진
-   `utils`: 공통 유틸리티
-   `templates` & `static`: 웹 프론트엔드

각 계층은 **단일 책임 원칙(SRP)**을 가져야 하며, 계층 간 의존 방향을 준수해야 합니다.

---

## 2. Root Folder Structure (Top-Level)

아래는 `KevinCY-Kodex` 프로젝트의 최상위 구조입니다.

```
project-root/
├── app/                  # Application Core Logic
│   ├── core/             # Settings, Security, Logger
│   ├── routers/          # API Endpoints (Controller)
│   ├── services/         # Business Logic
│   ├── repositories/     # I/O Layer (DB, File, API)
│   ├── models/           # Pydantic Schemas
│   ├── ai/               # AI Engine (LLM, RAG, Graph)
│   └── utils/            # Shared Utilities
│
├── templates/            # Jinja2 HTML Templates
│   ├── layout/           # Base Layouts
│   ├── pages/            # Page Content
│   └── partials/         # HTMX Components
│
├── static/               # Static Assets (CSS, JS, Images)
├── tests/                # Pytest Cases
├── data/                 # (Optional) Local Data Storage
│
├── .env                  # Environment Variables (Git Ignored)
├── .env.example          # Env Template
├── requirements.txt      # Python Dependencies
├── README.md             # Project Documentation
├── Makefile              # Automation Scripts
└── main.py               # App Entry Point
```

---

## 3. /app/core — Core Settings Layer

핵심 설정, 로거, 예외 처리 등 기반 시스템 구성 요소를 관리합니다.

-   **포함 파일**: `config.py`, `logger.py`, `exception_handlers.py`, `security.py` 등
-   **책임**:
    -   환경 변수 로딩
    -   공통 로거 설정
    -   보안/인증 정책
    -   FastAPI 전역 예외 처리

---

## 4. /app/routers — API Endpoint Layer

FastAPI 엔드포인트를 정의합니다. 비즈니스 로직을 절대 포함하지 않습니다.

-   **규칙**:
    -   파일명: `xxx_router.py`
    -   라우터 변수명: `router`
    -   역할: `Request` 수신 → `Service` 호출 → `Response` 반환
-   **예시**: `chat_router.py`, `voice_router.py`, `health_router.py`

---

## 5. /app/services — Business Logic Layer

모든 비즈니스 로직과 워크플로우 결합을 담당합니다.

-   **규칙**:
    -   파일명: `xxx_service.py`
    -   `Router`와 `Repository`/`AI` 계층 간의 조정자 역할
    -   외부 시스템 호출 로직은 `Repository`로 위임
-   **예시**: `chat_service.py`, `voice_service.py`, `rag_service.py`

---

## 6. /app/repositories — IO (DB/File/API) Layer

DB, 파일 시스템, 외부 API, 벡터 DB 등 모든 IO를 처리합니다.

-   **규칙**:
    -   파일명: `xxx_repository.py`
    -   가능한 한 순수 함수 형태 유지
    -   비즈니스 로직 포함 금지
-   **예시**: `document_repository.py`, `vector_repository.py`, `user_repository.py`

---

## 7. /app/models — Pydantic Models & DTOs

`Request`/`Response`/`DTO`에 사용되는 Pydantic 모델을 정의합니다.

-   **규칙**:
    -   파일명: `xxx_model.py`
    -   비즈니스 로직 포함 금지
-   **예시**: `chat_model.py`, `voice_model.py`, `rag_model.py`

---

## 8. /app/ai — LLM, RAG, LangGraph Engine Layer

AI 관련 모든 구성 요소를 담당하는 영역입니다.

-   **권장 세부 구조**:
    ```
    /app/ai/
    ├── graph.py              # LangGraph Main Workflow
    ├── llm_client.py         # LLM Client Wrapper (API/Local)
    ├── ingest/               # Document Loading Logic
    ├── chunk/                # Text Splitter Logic
    ├── embed/                # Embedding Model Logic
    ├── retriever/            # Vector DB Search Logic
    ├── reranker/             # Re-ranking Logic
    ├── generator/            # Prompt & Generation Logic
    ├── stt/                  # Speech-to-Text Logic
    └── tts/                  # Text-to-Speech Logic
    ```
-   **역할**: LangGraph 파이프라인, LLM 호출, RAG 구성요소, 음성 처리(STT/TTS)

---

## 9. /app/utils — 공통 유틸 함수

작은 순수 함수, 공통 로직, 형식 변환, 문자열 처리 등을 담당합니다.

-   **예시**: `string_utils.py`, `time_utils.py`, `file_utils.py`

---

## 10. /templates — Jinja2 HTML Templates

HTMX와 Tailwind 기반의 UI 템플릿을 관리합니다.

-   **구조 표준**:
    ```
    /templates/
    ├── layout/               # base.html, header.html, footer.html
    ├── pages/                # index.html, chat.html, voice_chat.html
    └── partials/             # chat_message.html, loader.html
    ```

---

## 11. /static — CSS, JS, Assets

정적 리소스 저장소로 컴파일된 CSS, JS, 이미지, 오디오 파일 등을 저장합니다.

-   **구성 예시**:
    ```
    /static/
    ├── css/                  # tailwind.css (Compiled)
    ├── js/                   # main.js (HTMX Extensions)
    ├── img/                  # Images
    └── tts/                  # Generated Audio Files (Temp)
    ```

---

## 12. /tests — Test Directory

`pytest` 기반의 테스트 코드를 저장합니다.

-   **규칙**:
    -   파일명: `test_xxx.py`
    -   단위 테스트와 통합 테스트를 분리하여 구성 가능

---

## 13. Top-Level Files

-   `.env` / `.env.example`: 환경 변수 파일(API Key, DB URL 등)
-   `requirements.txt`: Python 의존성 패키지 목록
-   `Makefile`: 실행/테스트/배포 스크립트
-   `README.md`: 프로젝트 개요 및 가이드
-   `main.py`: FastAPI 앱 실행 진입점 (uvicorn.run)

---

## 14. 계층 간 의존성 규칙

의존성은 항상 바깥쪽에서 안쪽으로 향해야 합니다.

-   **허용**:
    -   `routers` → `services`
    -   `services` → `repositories` / `ai`
-   **전역 사용 가능**: `core`, `utils`, `models`
-   **금지**: 역방향 의존성 (예: `repositories` → `services`)

---

## 15. 확장용 프로젝트 구조 (Large Scale)

대규모 프로젝트에서는 기능 도메인별로 모듈을 분리하는 것을 권장합니다.

-   **예시**:
    ```
    /app/
      ├── domain_chat/
      │   ├── chat_router.py
      │   ├── chat_service.py
      │   └── chat_model.py
      │
      ├── domain_voice/
      │   ├── voice_router.py
      │   ├── voice_service.py
      │   └── voice_model.py
      │
      └── ...
    ```

---

## 16. Data Pipeline Structure (Universal)

RAG/문서 처리 프로젝트를 위한 표준 데이터 폴더로 원본 데이터와 처리된 데이터를 관리하기 위해 `data/` 폴더를 포함할 수 있습니다.

```
project-root/
  ├── data/
  │   ├── raw/        # 원문 PDF/문서
  │   ├── cleaned/    # 전처리된 텍스트
  │   ├── chunks/     # chunk 저장소
  │   └── embeddings/ # 벡터 인덱스
  └── ...
```

---

## 17. 운영·배포에서의 폴더 관리 원칙

-   모든 환경(`local`, `staging`, `prod`)은 동일한 폴더 구조를 유지합니다.
-   `data`, `embedding`, `TTS` 파일 등은 버전 규칙을 적용하여 관리합니다.
-   오래된 임시 파일, 로그, TTS 음성 파일은 주기적으로 정리합니다.
-   `.env` 파일은 절대 버전 관리 시스템(Git)에 포함하지 않습니다.

---

End of KevinCY-Kodex Folder Standards