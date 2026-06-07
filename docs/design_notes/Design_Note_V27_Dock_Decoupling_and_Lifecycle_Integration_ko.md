# 📝 설계 노트: QDockWidget 독립 분리 및 엔진 라이프사이클 통합 (V27.0 / V26.2.2)

## 1. 개요 (Overview)
본 문서는 NoCodeQuant 시스템의 UI 가독성과 런타임 종료 무결성을 강화하기 위해 진행된 **주도테마 흐름 및 보유잔고 패널의 독립 QDockWidget 분리 리팩토링(V27.0)** 및 **EngineLifecycleManager 기반 라이프사이클 통합 패치(V26.2.2)**의 설계와 구현 결과를 기록합니다.

---

## 2. 주도테마 및 보유잔고 독 분리 (V27.0)

### 2.1 기존 구조의 한계
- 기존에는 단일 `QDockWidget` 내부에 `QSplitter`를 배치하여 주도테마 흐름과 보유잔고 감시 패널을 상하로 강제 결합했습니다.
- 이로 인해 창 크기가 세로로 축소될 때 두 패널 간의 크기 제약 조건이 충돌하여 화면이 깨지거나 겹치는 레이아웃 붕괴 현상이 발생했습니다.
- 또한 사용자가 개별 패널을 분리(Float)하거나 위치를 이동(Move)할 수 없어 UX의 유연성이 제한되었습니다.

### 2.2 개선 설계 및 구현
1. **독립된 QDockWidget 분리**:
   - 기존의 묶여 있던 독 구조를 해체하고, `dock_theme_flow`("🔥 실시간 주도테마 흐름")와 `dock_pos`("💼 보유 잔고 & 백그라운드 감시")를 각각 독립된 `QDockWidget`으로 리팩토링했습니다.
2. **QMainWindow Docking System 활용**:
   - [sidebar_builder.py](file:///C:/Users/stone/projects/nocodequant/app/ui/builders/sidebar_builder.py)에서 기존 vertical splitter 조립 코드를 전면 제거하고, `QMainWindow` 고유의 `addDockWidget` 및 `splitDockWidget` API를 사용하여 독을 초기 배치했습니다.
3. **사용자 정의 배치 영속화**:
   - `QMainWindow.saveState()`와 `restoreState()`가 사용자 정의 배치 상태를 완벽히 복원할 수 있도록, 각각의 독 위젯에 명시적인 `setObjectName`("dock_ranking", "dock_theme_flow", "dock_pos")을 지정했습니다.
4. **안전 및 제약 조건 제어**:
   - 사용자의 오동작으로 창이 닫혀 증발하는 것을 방지하기 위해 `DockWidgetClosable` 피처를 해제하고 `Movable | Floatable` 조합을 적용했습니다.
   - 레이아웃을 불필요하게 감싸던 `QScrollArea` 래핑 코드를 제거하여 단순화하고, `MultiSessionDashboard` 위젯의 sizePolicy를 `Preferred`로 변경하여 마우스 드래그로 극단적인 수준(최소 45px)까지 축소 가능하도록 보정했습니다.
   - 초기 세로 비율을 `[230, 310, 260]`으로 튜닝하여 6라인 가독성과 주도테마 흐름 시인성을 확보했습니다.

---

## 3. Coordinated Shutdown 및 EngineLifecycleManager (V26.2.2)

### 3.1 런타임 종료(Shutdown) 안정성 확보
- 기존의 분산된 종료 로직은 메인 윈도우 `closeEvent`에서 각 매니저 객체의 메서드를 수동 호출하는 구조였습니다.
- 비정상 크래시나 사용자 강제 종료 시 SQLite DB 락 잔존, 미체결 수동 주문 플러시 누락, 백그라운드 타이머 스레드가 정지되지 않아 좀비 프로세스가 살아남는 문제가 발생했습니다.

### 3.2 개선 설계 및 구현
1. **EngineLifecycleManager 신설**:
   - [lifecycle_manager.py](file:///C:/Users/stone/projects/nocodequant/app/services/managers/lifecycle_manager.py)를 신설하고, 시스템의 주요 매니저들을 DI(Dependency Injection) 형태로 전달받아 단일 지점에서 종료 라이프사이클을 통제하는 오케스트레이터를 구축했습니다.
2. **결정적 종료 시퀀스(Shutdown Sequence)**:
   - **Step 1 (Order)**: 미체결 수동 매도 누적기(`flush_sell_accumulators`) 즉시 플러시.
   - **Step 2 (Surveillance)**: 시장 감시 스레드 타이머(`process_timer.stop()`) 중지.
   - **Step 3 (Controller)**: 감시 컨트롤러 스레드 정지(`stop()` / `wait(2000)`).
   - **Step 4 (Ranking)**: 주도주 랭킹 연산 타이머 및 리소스 해제.
   - **Step 5 (Execution)**: 주문 집행 매니저 정지.
   - **Step 6 (Persistence)**: 메모리 내의 Flight Recorder 로그를 디스크(`flight_recorder_dump.json`)로 자동 덤프한 후, SQLite 커넥션을 안전하게 커밋 및 클로즈(`persistence_mgr.close()`).
3. **안전성 검증**:
   - `MainWindow.closeEvent` 내부에서 `lifecycle_mgr.shutdown()`을 wiring하여 런타임 크래시 직전 상태 보존과 프로세스 누수 제로를 강제했습니다.

---

## 4. Invariant Sealing & Observability (Phases 1-4)

1. **Phase 1: 읽기 락 간소화 및 DTO 스냅샷 격리**:
   - `StateManager`의 `get_position` 및 `get_all_positions`에서 무거운 포인터 참조 대신 `.copy()` DTO 스냅샷 반환 방식으로 수정하여 락 획득 시간을 최소화하고 상태 오염을 원천 격리했습니다.
2. **Phase 3: 초경량 Flight Recorder 및 미시 시장 지표 추가**:
   - 메모리 내 순환 버퍼(`deque(maxlen=2000)`)를 통해 크래시 직전 런타임 상태를 저장하고, `queue_depth`, `signal_loop_ms`, `flush_latency_ms` 등 병목 추적 텔레메트리를 `HealthMonitor`와 결합하여 30초 간격으로 로깅하도록 구성했습니다.
   - False Positive 원인 규명을 위해 호가창에서 추출한 핵심 지표 7종(`spread`, `spread_tick`, `spread_pct`, `bid_total_qty`, `ask_total_qty`, `imbalance_ratio`, `proximity_score`)을 `SignalSnapshot`에 주입해 SQLite에 자동 저장하고, 포렌식 대시보드 내 한국어 툴팁(`METRIC_GLOSSARY`)을 통해 시각화 분석을 보강했습니다.
3. **Phase 4: Determinism AST Lint 경고 적용**:
   - 시뮬레이션 및 백테스트 일관성을 방해하는 비결정성 시계 호출(`time.time()`, `datetime.now()`, `uuid.uuid4()` 등)을 탐지하기 위해 `tests/test_determinism_lint.py`를 작성하여 AST 정적 검사를 유닛 테스트 스위트에 강제 도입했습니다.
   - 예외적으로 원시 시계가 필요한 곳은 `# noqa: determinism` 또는 `allowed: system_clock`을 명시하도록 규정했습니다.

---
*Document Version: v1.0 | Created: 2026-05-25 | Aligned with V27.0 Release*
