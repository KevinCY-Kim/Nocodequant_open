# 2026-02-20 AI System Upgrade Work Log

## 1. 개요 (Overview)
AI 시스템의 역할을 단순 '데이터 요약자'에서 '전략 시나리오 분석가'로 격상시키기 위한 시스템 업그레이드를 수행했습니다. 기존의 기계적인 JSON 포맷과 앵무새 같은 요약 답변을 탈피하고, 기술적 엔진의 신호를 기반으로 인간 전문가 수준의 인사이트와 구체적인 Action Plan을 제시하도록 개선했습니다.

## 2. 주요 변경 사항 (Key Changes)

### [1] AI 페르소나 및 프롬프트 고도화 (Persona & Prompt)
- **페르소나 변경**: `Fusion Strategy Expert` -> `Strategic Scenario Analyst`
- **분석 지침 강화**:
    - **요약 금지**: 이미 GUI에 표시된 수치를 단순 반복하는 행위를 금지했습니다.
    - **모순 발견**: 추세(MA)와 이격도(Gap) 간의 괴리 등 지표 간의 충돌 지점을 분석하도록 지시했습니다.
    - **Trigger Point 제시**: "향후 5~10봉 내 OOO원 돌파 시"와 같은 구체적인 가격 조건부 시나리오를 제시하도록 했습니다.
- **데이터 구조화**: 프롬프트에 `Score`, `Confidence`, `Technical Matrix(Trend, Momentum, Volatility)`를 명확히 구분하여 제공했습니다.

### [2] 데이터 파이프라인 확장 (Data Enrichment)
- **2차 가공 지표 주입**: 단순 가격 외에 AI가 상황을 더 입체적으로 판단할 수 있도록 파생 지표를 추가했습니다.
    - `RSI Trend`: 최근 5봉 간의 RSI 기울기 (상승/하락 모멘텀 가속도)
    - `MA Gap`: 이동평균선 간의 이격도 (과열/침체 판단 근거)
    - `Vol Ratio`: 20일 평균 대비 거래량 비율 (수급 강도)

### [3] 시스템 신뢰성 강화 (System Reliability)
- **Retry Logic**: `requests` 호출 시 `try-except` 블록 내에서 지수 백오프(Exponential Backoff) 로직을 적용하여 API 일시적 장애나 Rate Limit에 대응하도록 했습니다.
- **Token Limit 상향**: `Max Tokens`를 128에서 500으로 상향 조정하여, 한국어 분석 리포트가 중간에 잘리는 현상을 방지했습니다.

### [5] 주도주 건전성 강화 (Hot Stocks Quality Upgrade)
- **Search Exclusion**: 저품질 종목이 리스트를 오염시키는 문제를 해결하기 위해 엔진 레벨에서 원천 필터링을 적용했습니다.
    - **동전주 필터**: 현재가 1,000원 미만 종목 제외.
    - **초소형주 필터**: 시가총액 300억 이하 종목 제외 (Listing Count 기반 실시간 계산).
    - **저유동성 필터**: 당일 거래대금 50억 이하 종목 제외.
- **아키텍처 개선**: 랭킹 데이터 파이프라인을 6-tuple로 확장하여 거래대금 데이터를 UI까지 안정적으로 전달하도록 개선했습니다.

## 3. 기술적 상세 (Technical Details)
- **Modified Files**:
    - `app/ai/ai_utils.py`: 프롬프트 로직 전면 개편, Retry 로직 구현.
    - `app/ai/strategy.py`: `feature enrichment` (RSI Trend, MA Gap 등 계산 로직 추가), `timestamp` 직렬화 안전장치 추가.
    - `app/core/config.py`: `OPENAI_MAX_TOKENS` 기본값 500으로 상향.
    - `Main_NCQ.py`: 텍스트 파싱 로직 개선 (불릿 포인트 지원).

## 4. 향후 계획 (Next Steps)
- 실전 매매에서 AI의 시나리오 적중률 모니터링
- HOT2 지표와 AI 분석의 결합 (Heat-Aware AI Analysis)
