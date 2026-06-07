# 💡 NoCodeQuant 생성형 AI 결합 및 성능 향상 제안서

본 제안서는 **개인 퀸트의 한정된 하드웨어 및 비용 자원** 내에서 NoCodeQuant(NCQ)의 기존 정량적/확률론적 강점을 그대로 보존하면서, 생성형 AI(LLM 및 Multi-Agent)를 결합해 시너지를 극대화할 수 있는 실전 아키텍처 설계안입니다.

---

## 1. 핵심 철학: 하이브리드 시너지 (Hybrid Synergy)

개인 퀸트 환경에서 **"모든 틱마다 실시간으로 LLM을 호출하는 것"**은 API 비용 폭탄, 네트워크 지연(Latency), 32비트 키움 싱글 스레드 환경의 연산 부하로 인해 **기술적으로 불가능하며 부적절**합니다. 

따라서 정량(수치)과 정성(텍스트) 데이터의 결합은 다음과 같은 **하이브리드 분리 원칙**을 따라야 합니다.
*   **실시간 거래 영역 (Real-time & Quantitative)**: 틱 체결 수집, 5분봉 지표 연산, 트레일링 스탑, Doomsday 스탑 등은 기존과 같이 100% C-language 급 속도의 **수학적 알고리즘**([strategy.py](file:///c:/Users/stone/projects/nocodequant/app/ai/strategy.py))이 담당합니다.
*   **전략적 감시 및 조율 영역 (Asynchronous & Qualitative)**: 거시 경제 판정, 일일 뉴스 및 공시 분석, 마켓 테마 추적, 장 마감 후 매매 복기 등은 **비동기 스레드 및 오프라인 LLM/Multi-Agent 파이프라인**이 담당합니다.

---

## 2. 개인 퀸트 맞춤형 4대 AI 융합 방안 (Actionable Architectures)

```mermaid
graph TD
    subgraph Asynchronous AI Core (정성적/비동기)
        A[뉴스/공시/거시 데이터 수집] --> B[LLM / Multi-Agent 분석]
        B --> C1[정성적 감성 팩터 -1 ~ +1]
        B --> C2[거시/테마 파라미터 조율]
        B --> C3[오프라인 매매 복기 보고서]
    end

    subgraph Real-Time NCQ Engine (정량적/실시간)
        C1 -->|로컬 DB/캐시 동기화| D[StrategyManager.get_signal]
        C2 -->|ncq_config.json 자동 업데이트| D
        E[실시간 시세/기술적 지표] --> D
        D -->|확률론적 스코어 0~100| F[비대칭 트레일링 스탑 & 체결 가드]
        F -->|주문 실행| G[키움 API 연동]
        G -->|체결 데이터 기록| H[(trades.db)]
    end

    H -->|장 마감 후 피드백| B
```

### ① 비동기 뉴스/공시 이벤트 기반 정성 데이터의 팩터화
*   **개념**: 관심 종목([favorites](file:///c:/Users/stone/projects/nocodequant/app/ai/engine.py#L284))에 대해 주요 공시(DART API)나 언론 보도가 발생하면, 비동기 백그라운드 워커가 작동하여 LLM API(GPT-4o-mini 또는 DeepSeek)를 호출합니다. 뉴스의 파급력을 `-1.0(매우 악재)`에서 `+1.0(매우 호재)` 사이의 수치(Sentiment Score)로 신속히 정량화합니다.
*   **NCQ 연동 방식**:
    *   이 점수를 인메모리 세션 캐시나 로컬 SQLite DB에 저장합니다.
    *   [strategy.py](file:///c:/Users/stone/projects/nocodequant/app/ai/strategy.py)의 `_get_signal_internal`에서 4대 블록 스코어링 시, **Risk/Core 블록의 점수 보정 팩터(최대 ±10점)**로 결합합니다.
*   **기대 효과**: 재무 수치나 기술 지표에 아직 반영되지 않은 **갑작스러운 공시 악재(배임/횡령, 대규모 블록딜 등) 발생 시 매수 진입을 원천 차단**하거나, **강력한 호재 발생 시 가점**을 주어 돌파 진입 시점을 앞당길 수 있습니다.

### ② 멀티 에이전트 오프라인 매매 복기 및 자율 피드백 (Self-Correction)
*   **개념**: 장 중 실시간 연산에 LLM을 쓰는 대신, 장이 끝난 오후 4시 이후 로컬 [trades.db](file:///c:/Users/stone/projects/nocodequant/data/trades.db)의 당일 체결 기록을 분석하는 오프라인 Multi-Agent 시스템을 구동합니다.
*   **역할 분담**:
    *   **데이터 분석 에이전트(Analyst Agent)**: 당일 진입 시점의 Core, Vol, Price, Risk 점수와 실제 익절/손절 여부를 매칭하여 통계 분석을 수행합니다.
    *   **포렌식 에이전트(Forensic Agent)**: 특히 손절 처리된 거래(예: 5/28 Polaris Office 손절 건)의 진입/청산 타점을 추적하고, "슬리피지가 컸는지", "엇박자 고점 매수였는지"를 규명합니다.
    *   **최적화 에이전트(Optimizer Agent)**: 분석 결과를 바탕으로, [ncq_config.json](file:///c:/Users/stone/projects/nocodequant/ncq_config.json)의 런타임 매개변수(예: MIXED 전략의 임계값 조정, UPTREND 트레일링 배수 완화 등)를 수정하는 최적의 가이드를 작성합니다.
*   **기대 효과**: 개인 개발자가 일일이 엑셀로 복기하고 코드를 고칠 필요 없이, 매일 저녁 시스템이 **자율 진단서와 파라미터 튜닝 제안서**를 UI에 띄워줍니다. 사용자는 클릭 한 번으로 [Tuning User Guide](file:///c:/Users/stone/projects/nocodequant/docs/AI_Tuning_User_Guide_ko.md) 규칙에 따라 파라미터를 안전하게 적용(APPLIED/QUEUED) 및 롤백할 수 있습니다.

### ③ RAG 기반 실시간 주도 테마 및 모멘텀 매핑
*   **개념**: 개인 퀸트의 돌파/추세 전략에서 가장 빈번히 발생하는 실패는 **"거래대금이 없는 소외주를 샀다가 휩소(Whipsaw)에 물리거나 호가 스프레드가 넓어 슬리피지 폭탄을 맞는 경우"**입니다. RAG(검색 증강 생성)를 통해 당일 오전 배포된 증권사 데일리 리포트, 경제 뉴스에서 '오늘의 주도 테마(예: AI 전력망, 우주항공 등)' 핵심 키워드와 관련 종목 맵을 추출합니다.
*   **NCQ 연동 방식**:
    *   [engine.py](file:///c:/Users/stone/projects/nocodequant/app/ai/engine.py#L270)의 `graph_mgr` (종목-업종-테마 그래프)과 연동하여 당일 강세 주도주 테마군에 포함된 종목들의 `heat_weight` 가중치를 자동으로 상승시킵니다.
    *   주도 테마 외 종목의 돌파(BREAKOUT) 신호는 임계값을 상향 조정하여 진입을 까다롭게 제한합니다.
*   **기대 효과**: 돈과 거래량이 몰리는 **'진짜 주도주' 중심의 변동성 돌파 전략이 수행**되어, 5/28 일지에서 언급된 "급등/초단기 탭의 타점 밀림 및 슬리피지"를 근본적으로 방지합니다.

### ④ LLM 가이드식 거시 국면(Macro Regime) 리스크 가드
*   **개념**: 현재 NCQ의 국면 판정([market_regime_definition_ko.md](file:///c:/Users/stone/projects/nocodequant/docs/architecture/market_regime_definition_ko.md))은 개별 종목의 ADX/DMI/이평선에만 의존합니다. 하지만 매크로 변수(FOMC 금리 결정, CPI 발표, 환율 변동성 등)가 시장 전체를 지배할 때는 개별 종목의 기술적 분석이 무력화됩니다.
*   **NCQ 연동 방식**:
    *   하루 1회 또는 주요 매크로 지표 발표 직후, LLM이 글로벌 거시 환경 지수(CPI, 환율, Yield Curve 등)를 읽고 현재 거시 리스크 등급(Low / Medium / High)을 판정합니다.
    *   거시 리스크가 **High**로 지정되면, NCQ 엔진은 글로벌 안전 가드([Panic Shield](file:///c:/Users/stone/projects/nocodequant/docs/architecture/NoCodeQuant_Architecture.md#L32))를 선제 가동하고, 1회 진입 비중(Position Size)을 평소의 50%로 자동 제한하며, 진입 임계값(Threshold)을 전반적으로 5~10점 상향합니다.
*   **기대 효과**: 거시적 폭락장이 오기 전 자산 배분 비중과 리스크 임계치를 선제적으로 수축시켜, 개인의 소중한 원금을 방어합니다.

---

## 3. 당면 과제 해결을 위한 구체적인 정량 + 정성 결합 방안

5월 28일 자 업무 일지 및 성과 보고서에서 나타난 구체적인 페인 포인트(Pain Point)에 대한 즉각적인 개선 조합 예시입니다.

### 📌 이슈 A: MIXED(혼합형) 전략 진입 타점 지체 및 고점 물림 (-94,112원 기록)
*   **정량적 해결책 (즉시 적용)**:
    *   **체결강도(Volume Power) 가드**: `Volume Power` (최근 5분 체결 매수 계약 수 / 매도 계약 수 * 100) 절대 하한선 가드(예: 110% 이상)를 진입 조건에 강제 바인딩합니다. 호가창에 허수 주문만 쌓이고 실매수 강도가 약한 상태에서의 허수 돌파를 원천 차단합니다.
*   **정성적 해결책 (AI 결합)**:
    *   **테마 연관도 가점 필터**: RAG를 통해 해당 종목이 '당일 거래대금 상위 3대 테마'에 매핑되어 있는 상태인지 검증합니다. 테마 중심부에 있는 종목이 아닐 경우 진입 점수에서 페널티(-10점)를 부여하여 개인의 뇌동/추격 매수를 억제합니다.

### 📌 이슈 B: 급등 탭 종목의 초단기 슬리피지 및 윗꼬리 저항 (-23,022원 기록)
*   **정량적 해결책**:
    *   **호가 잔량 불균형(Imbalance Ratio) 가드**: 체결 시점의 총매수잔량/총매도잔량이 극단적으로 높거나 낮아 호가 공백이 클 경우 매수를 보류하는 스프레드 가드를 활성화합니다.
*   **정성적 해결책 (AI 결합)**:
    *   **뉴스 임팩트 검증**: 해당 급등이 '지속 가능한 뉴스(예: 대기업과의 공급 계약)'인지 '단발성 테마(예: 막연한 기대감)'인지를 LLM이 문맥적으로 즉시 판별하여, 단발성 찌라시인 경우 초단기 타임 아웃(Time Exit)을 10봉 이하로 타이트하게 리셋하여 약익절/본청 기회를 넓힙니다.

---

## 4. 제언: 로드맵 및 개인 퀸트 생산성 극대화

이 모든 로직을 한번에 개발하려면 리소스 오버헤드가 발생하므로, 아래와 같은 순차적 접근을 권장합니다.

1.  **Phase 1 (정량 가드 보강 - 비용 0원)**: 당일 일지에 제언된 `MIXED 전략 내 체결강도 가드` 및 `호가 스프레드/공백 제어 가드`를 [strategy.py](file:///c:/Users/stone/projects/nocodequant/app/ai/strategy.py)에 우선 패치합니다.
2.  **Phase 2 (비동기 RAG/테마 매핑 - 난이도 하)**: 장 시작 전(08:30)에 주도 테마 리포트를 읽고 해당 종목코드 리스트를 로컬 JSON으로 생성한 뒤, NCQ 실행 시 로딩하여 `heat_weight` 가중치를 더해주는 오프라인 배치 스크립트를 작성합니다.
3.  **Phase 3 (마감 후 Multi-Agent 매매 복기 - 난이도 중)**: 하루의 매매가 끝난 후 `trades.db`를 읽어 AI가 분석해 주는 독립 스크립트를 작성하여 GUI의 `[AI Tuning]` 피드백 카드와 연결합니다.
