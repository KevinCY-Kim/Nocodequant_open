# KevinCY-Kodex — Testing Strategy

**Version:** 2025.11.27 (Universal Standard)
**Scope:** FastAPI · AI/RAG · Clean Architecture · Pytest (Async)

---

## 1. 테스팅 철학 (Testing Philosophy)

`KevinCY-Kodex`의 테스팅 철학은 **"신뢰할 수 있는 코드를 지속적으로 제공하는 것"**입니다. 모든 코드는 테스트를 통해 그 품질을 증명해야 합니다.

-   **품질 보증**: 새로운 기능이 의도대로 동작함을 보증합니다.
-   **회귀 방지**: 코드 변경으로 인해 기존 기능이 손상되는 것을 방지합니다.
-   **자신감 있는 리팩터링**: 테스트가 안전망 역할을 하므로, 자신감을 갖고 코드 구조를 개선할 수 있습니다.
-   **살아있는 문서**: 잘 작성된 테스트 코드는 그 자체로 기능의 명세서 역할을 합니다.

---

## 2. 테스트의 종류와 범위 (Test Types and Scope)

프로젝트는 세 가지 종류의 테스트를 중심으로 품질을 관리합니다.

1.  **단위 테스트 (Unit Test)**
    -   **범위**: 단일 함수, 메서드, 또는 클래스.
    -   **특징**: 외부 의존성(DB, API, AI 모델 등)을 모두 **모킹(Mocking)**합니다. 매우 빠르고 격리된 환경에서 실행됩니다.
    -   **목표**: 로직의 정확성 검증.

2.  **통합 테스트 (Integration Test)**
    -   **범위**: 두 개 이상의 계층이 함께 동작하는 것을 테스트. (예: `Service` + `Repository`)
    -   **특징**: 실제 DB(테스트용)나 외부 시스템과의 연동을 일부 포함할 수 있습니다. 단위 테스트보다 느립니다.
    -   **목표**: 계층 간의 상호작용 및 데이터 흐름 검증.

3.  **E2E 테스트 (End-to-End Test)**
    -   **범위**: 실제 사용자의 시나리오와 동일하게 API 엔드포인트 요청부터 응답까지 전체 흐름을 테스트.
    -   **특징**: 가장 느리지만, 시스템 전체의 동작을 가장 확실하게 보증합니다.
    -   **목표**: 전체 시스템의 안정성 및 최종 사용자 관점에서의 동작 검증.

---

## 3. 계층별 테스트 전략 (Layer-by-Layer Strategy)

Clean Architecture의 각 계층은 다음과 같은 전략에 따라 테스트합니다.

### 3.1 `/app/routers`
-   **테스트 대상**:
    -   정상적인 요청에 대해 `200 OK` 상태 코드를 반환하는가?
    -   잘못된 요청(유효성 검사 실패)에 대해 `422 Unprocessable Entity`를 반환하는가?
    -   `RequestModel`이 `Service` 계층으로 올바르게 전달되는가?
    -   `Service`의 반환값이 `ResponseModel` 형식에 맞게 반환되는가?
-   **도구**: FastAPI의 `TestClient`

### 3.2 `/app/services` (Async Logic)
-   **테스트 대상**:
    -   핵심 비즈니스 로직의 모든 분기(if/else 등).
    -   **Hybrid Fallback**: API 호출 실패 시 Local LLM으로 전환되는 로직이 작동하는가?
    -   `Repository` 또는 `AI` 모듈의 함수를 올바른 인자와 함께 호출하는가?
    -   예외 상황(Exception)을 올바르게 처리하는가?
-   **도구**: `pytest`, `pytest-mock`, `pytest-asyncio` (비동기 함수 테스트용)

### 3.3 `/app/repositories`
-   **테스트 대상**:
    -   DB 쿼리가 올바르게 작성되고 실행되는가?
    -   외부 API 호출 시 올바른 엔드포인트와 파라미터로 요청하는가?
    -   DB 또는 API의 응답을 올바른 데이터 객체로 변환하는가?
-   **도구**: `pytest`, `pytest-mock`, `httpx` (외부 API 모킹용), 테스트용 DB 세션

---

## 4. AI 파이프라인 테스트 전략 (AI Pipeline Testing Strategy)

AI 모델의 비결정적인 특성을 고려하여, 결과물의 **내용**이 아닌 **구조와 형식**을 테스트하는 데 집중합니다.
*(품질 평가는 `RAG.md`의 Evaluation Rules를 따릅니다.)*

### 4.1 RAG Retrieval 테스트
-   **목표**: 특정 질문에 대해, 의도한 핵심 문서(Chunk)가 검색 결과 상위 N개에 포함되는지 검증합니다.
-   **방법**: 미리 정의된 "질문-필수 청크 ID" 쌍을 만들어두고, `retriever` 함수 실행 후 결과에 필수 청크가 포함되어 있는지 `assert`문으로 확인합니다.

### 4.2 LLM 응답 생성 테스트 (Hybrid Strategy)
-   **목표**: LLM이 생성하는 답변의 **형식(Format)**과 **Fallback 로직**을 검증합니다.
-   **방법**:
    -   **Mocking**: `SKT A.X`와 `Ollama` 클라이언트를 각각 모킹하여, 고정된 더미 텍스트를 반환하게 합니다.
    -   **Structure Check**: 답변 생성 함수가 이 더미 텍스트를 받아, 최종적으로 유효한 JSON 구조(`answer`, `sources`)를 반환하는지 검증합니다.

---

## 5. 테스트 환경 및 도구 (Test Environment & Tools)

-   **Test Runner**: `pytest`
-   **Async Plugin**: `pytest-asyncio` (FastAPI/LangGraph 비동기 테스트 필수)
-   **Mocking Library**: `pytest-mock`
-   **HTTP Client (API 테스트용)**: `TestClient` (from `fastapi.testclient`)
-   **HTTP Client (외부 API 모킹용)**: `httpx`

---

## 6. 테스트 실행 규칙 (Test Execution Rules)

1.  모든 새로운 기능 추가 및 버그 수정은 반드시 관련 테스트 코드를 포함해야 합니다.
2.  `main` 브랜치에 코드를 병합(Merge)하기 전, 모든 테스트(`unit`, `integration`)가 통과해야 합니다.
3.  테스트 코드 또한 `Code_Style.md`의 모든 규칙(Naming, Type Hint)을 준수해야 합니다.
4.  목표 테스트 커버리지는 **80% 이상**으로 유지하는 것을 권장합니다.

---

## 7. Agent Protocol (Antigravity Guide)
*에이전트가 터미널에서 테스트를 수행할 때 따르는 수칙입니다.*

1.  **Command:** 테스트 실행 시 다음 명령어를 사용하십시오.
    -   전체 테스트: `pytest`
    -   상세 로그 포함: `pytest -v`
    -   특정 파일만: `pytest tests/test_chat_service.py`
2.  **Safety First:**
    -   테스트 실행 전, `.env` 파일이 **Test Environment** 설정(예: `TEST_DB_URL`)을 가리키는지 확인하십시오.
    -   **절대 Production DB에 연결된 상태로 테스트를 실행하지 마십시오.**

End of KevinCY-Kodex Testing Strategy