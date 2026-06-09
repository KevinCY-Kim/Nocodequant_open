# KevinCY-Kodex — Code Style Guide

**Version:** 2025.11  
**Scope:** Python · FastAPI · LangGraph · RAG · HTMX · Tailwind

---

## 1. General Coding Principles

-   모든 코드에 **`type hint` 100%** 적용
-   함수는 **단일 책임 원칙(SRP)** 준수
-   **Async First:** 모든 DB, API, File I/O 작업은 `async def` / `await` 패턴을 필수로 사용
-   변수명, 함수명, 파일명은 의미 기반으로 네이밍
-   매직 넘버/문자열(hard-coded values) 금지 → `settings`/`config`/상수로 분리
-   비즈니스 로직 흐름: `routers` → `services` → `repositories`/`ai`
-   사이드 이펙트 최소화 (가능하면 순수 함수 지향)

---

## 2. Python Code Style

### 2.1 네이밍 규칙

| 대상 | 규칙 | 예시 |
| :--- | :--- | :--- |
| 변수 | `snake_case` | `user_id`, `file_path` |
| 함수 | `snake_case` | `load_document()`, `run_rag()` |
| 클래스 | `PascalCase` | `DocumentParser`, `ChatService` |
| 상수 | `UPPER_SNAKE_CASE` | `DEFAULT_CHUNK_SIZE = 800` |
| 파일명 | `snake_case` | `chat_service.py`, `rag_core.py` |

### 2.2 포매팅

-   **PEP8** 기반, 공백 4칸 들여쓰기
-   최대 줄 길이: **100~120자** (초과 시 줄바꿈)
-   **`import` 순서**:
    1.  표준 라이브러리
    2.  서드파티 라이브러리
    3.  로컬 모듈
    ```python
    # 1. 표준 라이브러리
    import json
    import os
    from pathlib import Path
    
    # 2. 서드파티 라이브러리
    import numpy as np
    from fastapi import APIRouter
    
    # 3. 로컬 모듈
    from app.core.config import settings
    from app.services.chat_service import handle_chat
    ```
-   `from x import *` 사용 금지
-   사용하지 않는 `import` 및 변수는 제거
-   `try/except`는 최소화하고, 구체적인 예외 타입 지정

### 2.3 Docstring 스타일

-   Public 함수/메서드는 간단한 요약과 `Args`/`Returns` 형식을 사용합니다.
    ```python
    from pathlib import Path

    def load_document(path: Path) -> str:
        """
        주어진 파일 경로에서 텍스트를 로드한다.
    
        Args:
            path: 텍스트 파일 경로.
    
        Returns:
            파일에서 읽어온 전체 텍스트 문자열.
        """
    ```

---

## 3. FastAPI Code Style

### 3.1 Router 파일 규칙

-   **파일명**: `xxx_router.py` (예: `chat_router.py`, `voice_router.py`)
-   **Router 인스턴스 이름**: 항상 `router`
    ```python
    from fastapi import APIRouter
    
    router = APIRouter(prefix="/chat", tags=["chat"])
    ```

### 3.2 Router 내부 원칙

-   Router는 **입·출력만 담당**하며 비즈니스 로직을 포함하지 않습니다.
-   `RequestModel` → `Service` 호출 → `ResponseModel` 반환 흐름을 따릅니다.
    ```python
    from app.models.chat_model import ChatRequest, ChatResponse
    from app.services.chat_service import handle_chat
    
    @router.post("/", response_model=ChatResponse)
    async def chat(request: ChatRequest) -> ChatResponse:
        return await handle_chat(request)
    ```

### 3.3 Response 규칙

-   반환 타입은 가능한 한 Pydantic 모델을 사용합니다.
-   `dict` 직접 반환은 최소화하고, 필요한 경우 모델을 정의하여 사용합니다.

---

## 4. Pydantic Model Style

### 4.1 기본 규칙

-   **파일명**: `xxx_model.py`
-   **클래스명**: `PascalCase`
-   I/O 용 DTO 위주로 사용하며, 비즈니스 로직을 포함하지 않습니다.
    ```python
    from pydantic import BaseModel
    
    class ChatRequest(BaseModel):
        user_id: str | None = None
        query: str
    
    class ChatResponse(BaseModel):
        answer: str
        sources: list[str] | None = None
    ```

---

## 5. Services Layer Style

### 5.1 규칙

-   **파일명**: `xxx_service.py`
-   **역할**: 비즈니스 로직과 워크플로우 조합
-   **주요 책임**:
    -   입력 검증 (필요시 추가)
    -   여러 `repository`/`ai` 호출 조합
    -   예외 처리 및 로깅
-   Router에서 서비스 함수를 직접 호출하는 구조를 유지합니다.
    ```python
    from app.models.chat_model import ChatRequest, ChatResponse
    from app.ai.graph import run_chat_graph
    
    async def handle_chat(request: ChatRequest) -> ChatResponse:
        """채팅 요청을 처리하는 서비스 레이어 진입점."""
        state = {"query": request.query, "user_id": request.user_id}
        result = await run_chat_graph(state)
        return ChatResponse(answer=result["answer"], sources=result.get("sources"))
    ```

---

## 6. Repositories Style

### 6.1 규칙

-   **파일명**: `xxx_repository.py`
-   **책임**: DB, 파일, Vector DB, 외부 API 등 **IO 전담**
-   가능한 한 순수 함수 형태로 작성하며, 비즈니스 규칙은 포함하지 않습니다.
-   저장/조회/검색과 같은 역할만 수행합니다.
    ```python
    from typing import Sequence
    from app.models.chunk_model import Chunk

    async def load_chunks(document_id: str) -> Sequence[Chunk]:
        """문서 ID에 해당하는 청크 목록을 비동기로 로드한다."""
        # ... DB 또는 파일 시스템 로직 ...
        pass
    ```

---

## 7. AI / LangGraph Code Style

### 7.1 폴더 구조

AI 및 LangGraph 관련 폴더 구조는 프로젝트 전체 표준을 따릅니다.
> 자세한 AI 관련 폴더 구조는 `Folder_Standards.md` 문서를 참조하세요.

### 7.2 LangGraph 노드 함수 스타일

-   모든 노드는 `state: dict -> dict` 형태를 유지합니다.
    ```python
    async def retrieval_node(state: dict) -> dict:
        query: str = state["query"]
        docs = retrieve_documents(query)
        return {**state, "retrieved_docs": docs}
    ```

### 7.3 LLM 호출 스타일 (Hybrid Config 준수)

-   Master Rule에 정의된 `AX_MODEL_NAME` 등의 환경 변수를 활용합니다.
    ```python
    from app.core.config import settings

    async def generate_answer(context: str, question: str) -> str:
        prompt = f"""
        [규칙] 근거 기반 답변, 모르면 모른다고 하기.
        [문맥] {context}
        [질문] {question}
        """
        # SKT A.X or Ollama 호출 (설정에 따름)
        return await llm_client.chat(
            model=settings.AX_MODEL_NAME,
            messages=[{"role": "user", "content": prompt}],
            temperature=settings.AX_TEMPERATURE
        )
    ```

---

## 8. RAG Code Style

### 8.1 Chunking

-   **기본 설정**: `chunk_size = 800`, `chunk_overlap = 200`
    ```python
    def chunk_text(content: str, size: int = 800, overlap: int = 200) -> list[str]:
        chunks: list[str] = []
        start = 0
        length = len(content)

        while start < length:
            end = min(start + size, length)
            chunk = content[start:end]
            chunks.append(chunk)
            start += size - overlap

        return chunks
    ```

### 8.2 Retrieval

-   **Hybrid**: 키워드(`BM25`) + 벡터 검색 조합
-   `Rerank`는 별도 함수/모듈로 분리합니다.
    ```python
    async def hybrid_search(query: str, k: int = 10) -> list[dict]:
        keyword_docs = await bm25_search(query, k=k)
        vector_docs = await dense_search(query, k=k)
        merged = merge_results(keyword_docs, vector_docs)
        return await rerank(query, merged)
    ```

### 8.3 Answer Generation

-   가능하면 구조화된 출력(JSON 형태의 `dict`)을 사용합니다.
    ```python
    def build_answer_payload(answer: str, sources: list[str]) -> dict:
        return {
            "answer": answer,
            "sources": sources,
        }
    ```

---

## 9. Error Handling Style

### 9.1 규칙

-   FastAPI 전역 예외 처리기는 `/app/core/exception_handlers.py`에서 관리합니다.
-   `Service` 레이어에서 예외를 적절한 `HTTPException`으로 변환합니다.
    ```python
    from fastapi import HTTPException, status
    
    def ensure_document_exists(doc_id: str) -> None:
        if not document_exists(doc_id):
            raise HTTPException(
                status_code=status.HTTP_404_NOT_FOUND,
                detail="해당 문서를 찾을 수 없습니다.",
            )
    ```
-   로그에는 민감 정보(주민번호, API 토큰 등)를 기록하지 않습니다.

---

## 10. Logging Style

-   구조화된 로그(JSON 형태)를 사용합니다.
    ```python
    import logging
    
    logger = logging.getLogger(__name__)
    
    def log_chat_request(user_id: str | None, query: str) -> None:
        logger.info({
            "event": "chat_request",
            "user_id": user_id,
            "query": query[:100],  # 민감 정보 및 길이 제한 고려
        })
    ```

---

## 11. HTMX Code Style

### 11.1 기본 원칙

-   HTMX 요청 경로는 항상 `/api/*` 네임스페이스를 사용합니다.
-   `hx-target`, `hx-swap`, `hx-indicator`를 명시적으로 사용합니다.
    ```html
    <form
      hx-post="/api/chat"
      hx-target="#chat-window"
      hx-swap="beforeend"
      hx-indicator="#chat-loading"
    >
      <input type="text" name="query" />
      <button type="submit">보내기</button>
    </form>
    ```

---

## 12. Tailwind Code Style

-   **클래스 순서**는 다음 그룹 순서를 유지합니다:
    1.  레이아웃 (`flex`, `grid` 등)
    2.  정렬/간격 (`items-center`, `gap`, `p`/`m`)
    3.  타이포그래피 (`text-sm`, `font-medium`)
    4.  색상/배경 (`bg-white`)
    5.  기타 (`rounded-lg`, `shadow`, `transition`)
    ```html
    <div class="flex items-center gap-2 px-4 py-2 text-sm font-medium bg-white rounded-lg shadow">
      ...
    </div>
    ```
-   의미 없는 커스텀 색상보다 Tailwind 기본 팔레트 사용을 우선합니다.

---

## 13. File Naming Rules

| 파일 종류 | 네이밍 예시 |
| :--- | :--- |
| 라우터 | `chat_router.py` |
| 서비스 | `chat_service.py` |
| 리포지토리 | `chat_repository.py` |
| 모델 | `chat_model.py` |
| 유틸 | `string_utils.py` |
| AI 모듈 | `retriever.py` |
| 테스트 코드 | `test_chat_service.py` |

---

## 14. Test Code Style (pytest)

### 14.1 규칙

-   **파일명**: `test_xxx.py`
-   **테스트 함수명**: `test_` 접두사 사용
-   **Given–When–Then** 패턴을 권장합니다.
    ```python
    import pytest
    from app.models.chat_model import ChatRequest
    from app.services.chat_service import handle_chat
    
    @pytest.mark.asyncio
    async def test_handle_chat_returns_answer():
        # Given
        request = ChatRequest(query="안녕하세요")
    
        # When
        response = await handle_chat(request)
    
        # Then
        assert response.answer
        assert isinstance(response.answer, str)
    ```

---

## 15. Git Commit Message Style

-   영어 기반 `prefix`를 사용합니다.
    -   `feat`: 새로운 기능 추가 (e.g., `feat: add voice chat STT pipeline`)
    -   `fix`: 버그 수정 (e.g., `fix: handle empty query in chat service`)
    -   `refactor`: 코드 리팩터링 (e.g., `refactor: simplify rag retrieval flow`)
    -   `docs`: 문서 수정 (e.g., `docs: update architecture and code-style`)
    -   `chore`: 빌드, 의존성 등 기타 변경 (e.g., `chore: bump dependency versions`)

---

End of KevinCY-Kodex Code Style Guide
