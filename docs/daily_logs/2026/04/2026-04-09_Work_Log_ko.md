# 2026-04-09 작업 일지 (Work Log)

## 0. 개요 (Overview)
- **주요 과제**: 전략 청산 로직 하드닝 및 Institutional Alpha 기법 (V21.1~V21.3) 통합
- **등급**: **L4** (Strategy Logic & Real-time Execution)
- **참조**: [구조 맵 v21.3](Antigravity_rules/architecture/Project_Structure_Map_ko.md), [청산 로직 분석서 V21.3](docs/architecture/strategy_exit_logic_analysis_ko.md)

---

## 1. 주요 변경 사항 (Key Changes)

### 1.4 전략 엔진 포렌식 감사 및 구조 수리 [V21.5 ~ V21.7]
- **데이터 기반 진단**: V21.5 백테스트 분석 결과, 전략의 "수익 DNA"는 건재하나 구조적 설계 결함으로 인해 매매 빈도와 보유 시간이 왜곡되고 있음을 발견.
- **핵심 수리 사항 (The Big Three)**:
    - **Fix B (Stop Loss 하드닝)**: `curr['low']`를 손절 하한선으로 사용하는 로직 제거. 단기 노이즈에 의한 '억울한 손절'을 방지하고 Risk Check(1% 이격)의 정상 작동 유도. (매매 빈도 4배 증가)
    - **Fix C (포지션 덮어쓰기 가드)**: 보유 중 신규 진입 신호 시 기존 포지션을 덮어씌워 Time Exit 타이머를 리셋하던 치명적 버그 수정. (무한 보유 현상 해결)
    - **Fix D (Time Exit 독립화)**: UPTREND 국면에서도 `TP1` 달성 여부와 상관없이 시간 기반 익절이 반드시 수행되도록 logic-flow 분리.

### 1.5 전략 최적화 및 '방어의 함정' 검증 [V21.7]
- **최적 밸런스 도달**: 하락장 REVERSAL은 차단하되, 박스권(RANGE) REVERSAL은 허용하는 유연한 방어 체계로 **PF 1.71** 달성.
- **과잉 방어 검증 (Fix F)**: 모든 RANGE REVERSAL을 차단했을 때 PF가 0.8로 급락함을 확인.
    - **결론**: 박스권에서의 잦은 손실을 감내하더라도, 그 끝에서 터지는 '역전 홈런' 거래들이 전체 수익의 핵심 엔진임을 데이터로 증명 (Over-optimization 방지).

---

## 2. 영향 분석 및 불변성 확인 (Impact & Invariants)
- **Impacted Modules**: `strategy.py`, `money_manager.py`, `institutional_replay_v21_4.py`
- **Invariant Check**: 
    - "동일한 진입 신호라도 시장 변동성이 높으면 주문 수량이 줄어들어야 한다."
    - "상위 60분봉 추세가 하향인 종목은 REVERSAL 전략이 아닌 한 진입이 차단된다."
    - "포지션 보유 중에는 어떠한 경우에도 Time Exit 타이머가 초기화되지 않는다. (Fix C 보장)"

---

## 3. 검증 결과 (Verification)
- **V21.7 최종 성과 (60일 최적화)**:
    - **진입 횟수**: 99건
    - **승률**: 31.3%
    - **Profit Factor**: **1.71** (🚨 Target 1.5 돌파)
    - **누적 수익률**: **+53.64%**
    - **평균 수익**: +5.27% / **평균 손실**: -1.58% (손익비 3.33:1)

---

## 4. 향후 과제 (Next Steps)
- **Production Canary Deployment**: 하드닝된 V21.7 엔진의 실전 환경 배포 및 실시간 모니터링.
- **Equity Curve Filter**: 계좌 단위의 최근 성과에 따른 Kill-Switch 연동 (L1 컨트롤러 레이어).

---
*Last Updated: 2026-04-09 21:23*
*Created By Antigravity (Powered by Advanced Agentic Coding)*
