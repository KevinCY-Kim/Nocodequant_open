# Execution Pipeline Architecture (실행 파이프라인 아키텍처) [V27.0]

## 1. 개요 (Overview)
`ExecutionManager`는 NoCodeQuant 시스템의 **매매 실행, 리스크 관리, 런타임 메트릭 및 포렌식 감사**를 담당하는 핵심 백엔드 컴포넌트입니다. 헌장 §1.2의 **단일 진실 공급원(SSoT)** 원칙에 따라, 모든 매매 판단과 포렌식 이벤트는 이 모듈에서만 발생합니다.

- **위치**: `app/services/managers/execution_manager.py`
- **등급**: **L3** (핵심 로직 및 실행부)
- **의존성**: `StrategyManager`, `OrderManager`, `StateManager`, `PersistenceManager`, `EventBus`, `EngineLifecycleManager`

---

## 2. 핵심 기능 및 루프 (Core Functions & Loop)

### 2.1 `on_timer` — 리스크 및 MFE/MAE 정밀 동기화 루프 [V25.1]
1초 또는 설정된 감시 루프 타이머에 의해 기동되는 메인 리스크 거버넌스 루프입니다:

1. **MFE/MAE & peak_price 정밀 동기화**: 거래 빈도가 낮은 종목도 누락 없이 추적할 수 있도록, 1초 간격으로 `StateManager` 내 positions의 `max_price`/`min_price`를 강제 갱신하고 `ExitEngine` 내부 세션의 `peak_price`와 상호 동기화시킵니다.
2. **Stop Loss / Trailing Stop Check**: 보유 포지션의 실시간 수익률 체크 → 손절 또는 Trailing Stop 반납율 도달 시 즉시 시장가 매도 주문 위임.
3. **Kill Switch Check**: 당일 누적 손실 비율 초과 시 모든 신규 진입 차단.
4. **Signal Generation**: `StrategyManager.get_signal()` 호출 → BUY/SELL/HOLD 결정.
5. **Gating & Validation**: 포지션 가드, Stamp Logic, 쿨다운 체크.
6. **Order Execution**: `OrderManager.send_order()` 호출.
7. **Event Emission**: `_trigger_event()` → EventBus 발행 → PersistenceManager 영속화.
8. **UI Callback**: UI 상태판에 실시간 결과 갱신.

### 2.2 Coordinated Shutdown [V26.2]
`EngineLifecycleManager`와의 연동을 통해 시스템 종료 시 다음 프로세스를 완벽하게 보장합니다:
1. `OrderManager` 내부의 미완료된 SELL 누적기(acc) 강제 플러시 (`flush_sell_accumulators`).
2. SQLite DB 트랜잭션의 안전한 Commit 및 백업 스레드 수집.
3. 종료 직전의 메모리 내 2000건 감사 기록을 `logs/flight_recorder_dump.json` 파일에 영속화.

### 2.3 6대 실시간 런타임 메트릭 빌더 [V26.2]
실거래 시 병목 현상을 실시간으로 관측할 수 있도록 아래 6가지 지표를 수집하고 30초 간격으로 로그와 HealthMonitor에 발행합니다:
1. `queue_depth` (DB 영속화 큐 대기열 깊이)
2. `callback_delay_ms` (키움 API 시세 수신 콜백 지연 시간)
3. `signal_loop_ms` (전략 신호 계산 루프 소요 시간)
4. `flush_latency_ms` (SQLite 디스크 쓰기 지연 시간)
5. `ranking_cycle_ms` (주도주 실시간 랭킹 순위 계산 소요 시간)
6. `replay_drift_ms` (리플레이 모드 시 시간 괴리율)

---

## 3. 리스크 관리 (Risk Management)

| Guard | Trigger | Action | 등급 |
| :--- | :--- | :--- | :--- |
| **Stop Loss** | `pnl_pct < -stop_loss%` | 즉시 시장가 매도 | L3 |
| **Kill Switch** | `daily_loss > kill_switch%` | 신규 진입 전면 차단 | L3 |
| **Position Guard** | BUY 시 이미 보유 / SELL 시 미보유 | 주문 차단 | L3 |
| **Stamp Logic** | 동일 봉 내 중복 신호 | 재진입 억제 | L3 |
| **Cooldown** | 직전 매매 후 3캔들 미만 | 재진입 억제 | L2 |

---

## 4. SSoT 경계 규칙

- ✅ `ExecutionManager` 및 연동된 백엔드 코어만이 `event_bus.publish()` 호출 허용.
- ❌ UI(`main_window.py`)에서의 `event_bus.publish()` 호출 금지.
- ❌ UI에서의 Stop Loss / Kill Switch 판단 로직 금지.
- ✅ UI Callback은 순수 시각 갱신 용도로만 허용.

---
*Document Version: v27.0 (Aligns with Engine V27.0)*
