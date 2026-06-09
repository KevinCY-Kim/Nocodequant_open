# [SSoT-01] Financial Ledger Integrity Rule (금융 원장 무결성 원칙)

**최종 수정일**: 2026-04-07
**적용 범위**: PersistenceManager, StateManager, MainWindow PnL 산출 로직

---

## 1. 핵심 원칙: "진실은 오직 원장(Ledger)에만 존재한다"

NoCodeQuant의 모든 실현손익(PnL) 및 성과 지표 산출은 다음의 계층 구조를 엄격히 준수해야 한다.

1.  **Single Source of Truth (SSoT)**: `trades.db` 내의 `trades` 테이블만이 유일한 법적 원장이다.
2.  **No In-Memory Accumulation**: UI나 메모리(StateManager)에서 이벤트를 받아 직접 수익금을 누적(`+=`)하는 행위를 절대 금지한다.
3.  **Aggregation over Summation**: 수익금은 항상 `order_id` 또는 `gtid`로 그룹화된 '확정된 매매 기록'의 집계값이어야 한다.

---

## 2. 금지 사항 (Anti-Patterns)

다음과 같은 구현이 코드에 침투하지 않도록 AI 및 작업자는 상시 감시해야 한다.

*   ❌ **이벤트 스냅샷 합산 금지**: `forensics.db`의 신호 로그(signals)에 포함된 `realized_pnl` 스냅샷을 단순 합산하여 당일 수익을 계산하는 로직. (이중 집계 및 드리프트의 주원인)
*   ❌ **UI 상태값 의존 금지**: `state_mgr.cumulative_pnl` 값을 UI가 직접 수정하거나, 이 값을 최상위 진실로 믿고 렌더링하는 행위.
*   ❌ **로직 파편화 금지**: 메인 화면의 PnL 산출 쿼리와 전략성과보드의 KPI 산출 쿼리가 서로 다른 DB나 테이블을 참조하는 행위.

---

## 3. 구현 가이드라인 (Implementation Guardrails)

### 3.1 PersistenceManager의 역할
*   `get_daily_pnl()`은 반드시 `get_performance_kpi(today)`와 동일한 로직(trades.db 기반)을 공유해야 한다.
*   로직 수정 시, 한 곳의 수정이 시스템 전체 수치에 동일하게 반영되도록 **'함수 재사용'**을 원칙으로 한다.

### 3.2 MainWindow 및 UI의 역할
*   UI는 **Calculation Engine이 아니라 View**이다. 
*   체결 이벤트 발생 시, UI는 "데이터가 변경되었음"을 인지하고 `persistence_mgr`에 "최신 SSoT 수치를 달라"고 요청(Trigger & Fetch)하는 방식으로만 동작해야 한다.

### 3.3 StateManager의 역할
*   `state_mgr` 내의 PnL 데이터는 오직 **'캐시(Cache)' 및 '리스크 가드(MDD/HWM 계산)'** 용도로만 제한한다.
*   시스템 시작 시 반드시 `trades.db`의 실제 수치와 동기화(Reconciliation) 과정을 거쳐야 한다.

---

## 4. 위반 시 복구 절차

만약 `trades.db` 기반 집계 로직이 유실되거나 이벤트 기반으로 회귀한 것이 발견될 경우:
1.  `app/services/managers/persistence_manager.py`의 `get_daily_pnl()`을 확인하여 `trades.db` 참조 여부를 체크한다.
2.  `app/ui/main_window.py`의 `update_performance_summary()`가 직접 누적 방식인지, DB 조회 방식인지 확인한다.
3.  본 문서 [SSoT-01]에 기술된 **'목표 아키텍처'**로 즉시 롤백한다.

---

> [!IMPORTANT]
> 이 규칙은 NoCodeQuant 아키텍처의 최상위 가이드라인 중 하나이다. 성능 최적화나 실시간성 확보를 이유로 데이터의 무결성(Integrity)을 희생하는 수정은 허용되지 않는다.
