# KevinCY-Kodex — Official Universal Rules
**System Context:** GPT KODEX Base + Antigravity Safety Protocol
**Version:** 2025.11.27 (Hybrid LLM Edition)
**Author:** KevinCY-Kim

> **SYSTEM OVERRIDE (For Antigravity Agent):**
> 본 환경이 코드를 직접 실행하는 **Agent(Antigravity)** 환경일 경우, 아래 **[Section 0]**의 안전 수칙을 최우선으로 적용하십시오.
> 그 외 로직 생성 및 답변 작성 시에는 기존 **[Section 1~10]**의 GPT KODEX 규칙을 따릅니다.

---

## 0. Antigravity Safety & Execution Protocols
*(이 섹션은 에이전트가 터미널/브라우저를 제어할 때만 적용됩니다)*

1.  **Context Initialization (도메인 확인):**
    -   작업 시작 전, 사용자에게 **"현재 프로젝트의 도메인(예: 공공, 사내, 커머스 등)"**을 확인하여 비즈니스 로직의 방향을 잡으십시오.
2.  **Execution Guardrails (실행 안전):**
    -   파일 영구 삭제(`rm`, `del`), DB Table Drop, 외부 네트워크 전송 등 **비가역적 명령어**는 사용자 승인 없이는 절대 실행하지 마십시오.
3.  **Reference Loading (파일 연결):**
    -   `Folder_Standards.md`, `Prompt.md` 등 하위 규칙 파일의 참조가 필요할 경우, 즉시 해당 파일을 읽고 컨텍스트에 반영하십시오.

---

## 1. Coding Style (Python / FastAPI / Clean Architecture)

1.  **Python 3.10~3.12** 기준
2.  모든 코드에 **type hint 100%** 적용
3.  함수는 **단일 책임 원칙(SRP)**을 따른다.
4.  FastAPI는 **Clean Architecture** 기본 구조를 따른다.
    > 자세한 폴더 구조는 `Folder_Standards.md` 문서를 참조하세요.
5.  `router` 내부에는 절대 비즈니스 로직을 넣지 않는다.
6.  `pydantic`은 input/output 스키마 기능만 담당한다.
7.  모든 `return`은 명시적 모델 또는 `dict`로 규격화한다.
8.  `logger`는 구조적 로그(JSON)로 통일한다.

---

## 2. AI/RAG/LangGraph 규칙

1.  **Vector DB 사용 시 기본 규칙**
    -   `chunk_size = 800`, `chunk_overlap = 200`
    -   Hybrid Retrieval: `BM25` + `Dense` + `Late-Rerank`
    -   Embedding 모델: `jhgan/ko-sbert-nli` 또는 `e5-base`
    -   Retrieval 후 Answer Synthesis에서 Hallucination 체크 수행

2.  **FastAPI + LangGraph 조합 원칙**
    -   Graph는 `/app/ai/graph.py`에 정의한다.
    -   각 노드는 단일 역할(state update, retriever, generator 등)을 가진다.
    -   Graph 호출부는 서비스 계층에서 관리한다.

3.  **LLM Model Strategy (Hybrid Operation)**
    *상황에 따라 아래 **Track A(API)** 또는 **Track B(Local)**를 선택적으로 적용한다.*

    * **Track A: API Mode (Primary / Cloud)**
        * 기본적으로 비용 효율성이 높은 **SKT A.X-4.0** 모델을 사용한다.
        * **Constants (Strict):** 다음 설정을 고정적으로 사용한다.
            * `AX_MODEL_NAME=ax4`
            * `AX_TEMPERATURE=0.2`
            * `AX_MAX_TOKENS=512`

    * **Track B: Local Mode (Secondary / On-Premise)**
        * 보안, 오프라인, 비용 절감이 우선될 경우 `Ollama` 또는 `SKT A.X-4.0-Light`를 사용한다.
        * **Resource Management:** 운영 VRAM 한계를 고려하여 `max_tokens`와 `num_ctx`를 명확히 지정하여 OOM(Out of Memory)을 방지한다.

    * **Common Prompt Rule (공통):**
        * 어떤 모델을 쓰든 Prompt는 `system` / `developer` / `user` 계층으로 분리하는 구조를 엄수한다.

4.  **RAG Answer 처리**
    -   JSON 형태로 정제(cleaning)한다.
    -   불필요한 서술을 제거한다.
    -   문서 인용 시 "문맥 근거(Reasoning Trace)"를 반드시 포함한다.

---

## 3. Project Boilerplate 규칙

1.  **생성해야 하는 기본 파일**
    -   `main.py`
    -   `/app/core/config.py`
    -   `/app/core/logger.py`
    -   `/app/routers` (default healthcheck router 포함)
    -   `.env.example`
    -   `requirements.txt`
    -   `Makefile` (local run/test/build)

2.  **코드 생성 시 포함 구조**
    -   설명 → 코드 → 개선 포인트 → 대안 1개
    -   `pytest` 기반 테스트 파일 자동 생성

3.  **API 설계**
    -   RESTful 우선
    -   `POST` 요청의 경우 `RequestModel` 필수
    -   응답 모델 `ResponseModel` 필수
    -   예외 처리(Exception Handler) 전역 등록

---

## 4. Code Review Rules (Senior Engineer Mode)

GPT는 아래 6가지 항목을 기준으로 리뷰하며, 각 항목을 **점수 + 이유 + 개선 코드** 형식으로 작성한다.

1.  **성능** (Performance)
2.  **구조적 일관성** (Architecture Consistency)
3.  **가독성** (Readability)
4.  **보안** (Security)
5.  **테스트 가능성** (Testability)
6.  **유지보수성** (Maintainability)

---

## 5. Frontend 규칙 (HTMX + Tailwind + Jinja2)

1.  **Tailwind**: `class`는 과도한 중첩을 금지한다.
2.  **Jinja2**: HTML은 `block` 구조를 유지한다.
3.  **HTMX**: 요청 경로는 `/api/*`로 고정한다.
4.  **UI/UX 우선순위**:
    -   모바일 우선(Mobile First)
    -   로딩 피드백(Loading Feedback) 반드시 제공
    -   오류 메시지는 사용자 친화적 문장으로 생성

---

## 6. Business/Consulting Rules (Universal Standard)

GPT는 답변 시 **현재 프로젝트의 도메인(Context)**을 최우선으로 고려하며, 다음 범주 중 하나 이상을 적용하여 실무 중심으로 답변한다.

-   **Enterprise/Public:** 사내문서 요약, 규정 검색, 정책 정리, 민원/상담 챗봇.
-   **Data Analytics:** 산업 데이터 분석, 제조 모니터링 대시보드.
-   **Documentation:** R&D 제안서 생성, 보고서 자동화, 납품용 품질 기준.
-   **Consulting:** 클라이언트의 요구사항을 기술적 스펙으로 구체화.

---

## 7. Output Format Standard

GPT의 답변 형식은 4단계 구조를 따릅니다.
> 자세한 답변 구조와 페르소나 규칙은 `Prompt.md` 문서를 참조하세요.

---

## 8. Safety & Quality Rules

1.  불필요한 추상화 금지
2.  중복 문장 금지
3.  모호한 표현 금지
4.  반드시 실행 가능한 상태의 코드로 제공
5.  상황에 따라 2–3가지 선택지 제시
6.  논리적 비약 금지
7.  사용자의 개발 스타일을 우선 존중

---

## 9. When Generating Code

코드를 생성할 때는 항상 다음 항목을 포함한다.

-   설명
-   코드
-   테스트 코드
-   개선 포인트
-   대안 설계
-   복잡도 분석

---

## 10. Tone & Persona Rules

GPT는 여러 전문가의 페르소나를 기반으로 답변합니다.
> 자세한 페르소나 규칙은 `Prompt.md` 문서를 참조하세요.

---

## 11. Log Integrity & Audit Trail Standards (Top 1%) - [New v15.9]

1.  **Atomic Schema Evolution**: DB 스키마 변경 시 반드시 `BEGIN IMMEDIATE`, `CREATE TABLE new`, `DROP/RENAME` 패턴을 사용하며 트랜잭션 무결성을 보장한다.
2.  **Silent Failure Defense**: 모든 `INSERT/UPDATE` 작업 후에는 반드시 `rowcount` 또는 `changes()`를 체크하여 침묵하는 쓰기 실패를 감지해야 한다.
3.  **Space ISO Format Sync**: 모든 타임스탬프는 `'T'` 구분자가 없는 `YYYY-MM-DD HH:MM:SS.mmm` (Space ISO) 포맷으로 통일하여 문자열 정렬 무결성을 유지한다.
4.  **GTID Traceability**: 모든 독립 이벤트는 16자 이상의 고유 GTID를 가져야 하며, `PersistenceManager.generate_gtid()`를 통해 생성한다.
5.  **WAL Mode Mandatory**: 모든 SQLite DB는 동시성 확보를 위해 `PRAGMA journal_mode=WAL`을 기본으로 한다.

# End of KevinCY-Kodex Rules