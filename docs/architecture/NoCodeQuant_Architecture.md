# NoCodeQuant System Architecture

## 1. Overview (개요)
NoCodeQuant는 **PyQt5** 기반의 데스크톱 애플리케이션으로, 실시간 시세 수신, 커스텀 전략 연산, 그리고 주문 실행을 단일 프로세스 아키텍처 내에서 수행합니다. V22.1에 이르러 **비대칭 출구 엔진**과 **V5.0 분석 파이프라인**이 완전히 통합되었습니다.

---

## 2. Component Architecture (컴포넌트 구조)

### 2.1 Strategy Engine (`strategy.py`) [V22.1 Master]
- **Asymmetric Exit Engine**: 모든 보유 종목에 `initial_sl`을 메타데이터로 각인하고 R-Multiple(리스크 배수)에 따라 트레일링 스탑을 4.5x → 3.0x → 2.0x → 1.5x로 동적으로 조절합니다.
- **Warmup/Doomsday Defense**: 진입 초기 2봉의 노이즈 보호(Warmup)와 -4%의 절대 파산 방어선(Doomsday)을 구축했습니다.
- **RSI Inversion Logic**: 국면에 따라 RSI를 모멘텀(추종) 또는 역추세(반전)로 다르게 해석하여 지표의 왜곡을 방지합니다.

### 2.2 Analytics Engine (V5.0) (`schema_views.py`)
- **Score Persistence**: 진입 시점의 4대 블록 점수(Core, Vol, Price, Risk) 메타데이터를 `trades.db`에 영속화하여 복기 가능하게 합니다.
- **Post-Trade Forensic**: `v_trade_performance_detailed` 등의 가상 뷰를 통해 점수와 실제 수익률 간의 통계적 상관관계를 도출합니다.

### 2.3 Persistence & State Management (`persistence_manager.py`)
- **Deep-Search Recovery**: 갑작스러운 종료 시에도 `initial_sl`, `peak_price`, `trailing_multiplier`를 복구하여 트레일링 스탑의 연속성을 확보합니다.

---

## 3. Data Flow (데이터 흐름)
1. **Sensing**: `Kiwoom API` -> `Engine Event` -> `Raw Tick`.
2. **Thinking**: `StrategyManager.get_signal` -> `Score & Confidence` -> `SignalResult`.
3. **Acting**: `SignalResult` -> `OrderManager` -> `Engine.send_order`.
4. **Analyzing (V5.0)**: `Closed Trades` -> `Score Metadata` -> `Strategy Forensic Dashboard`.

---

## 4. Security & Safety
- **Panic Shield**: 가격이 장기 이평선(MA240) 아래로 침범 시, 모든 배수를 무시하고 `1.0x`의 초타이트 스탑을 가동하여 자본을 최우선 보호합니다.
- **Doomsday Tick Guard**: 봉 완성을 기다리지 않고 실시간 틱 단위로 절대 손절선을 감시하여 블랙 스완에 대비합니다.
