# Entry Points (진입점) [V17.0]

## 1. System Components

### 1.1 `app/main.py` (The Entry Point)
-   **Type**: Desktop Application (PyQt5)
-   **Run**: `python app/main.py`
-   **Environment**: Windows 10/11, Python 32-bit (Kiwoom API 필수 조건).
-   **Role**: `CompositionRoot`를 통한 의존성 주입(DI) 및 어플리케이션 부트스트래핑.

### 1.2 `app/ui/main_window.py` (The UI Controller)
-   **Role**: UI 관리, 위젯 배치, 사용자 인터랙션 처리. **매매 로직 금지** (헌장 §1.3).
-   **Size**: ~4,200+ 라인 (SoS 분리 대상).

### 1.3 `app/core/composition_root.py` (The DI Container)
-   **Role**: 모든 서비스 매니저(`ExecutionManager`, `PersistenceManager`, `RankingManager` 등)의 생성 및 의존성 연결.

### 1.4 `app/core/config.py` / `app/core/runtime_config.py` (The Configuration)
-   **Role**: 시스템의 모든 하드 상수를 관리하며, 런타임에 UI로부터 변경된 설정을 주입받습니다.

---

## 2. Boot Sequence (부팅 순서) [V17.0]

1.  **Composition Root**: `CompositionRoot.build()` → 모든 서비스 매니저 DI 생성.
2.  **Initialize Engine**: `KiwoomEngine` (또는 Mock) 생성 및 API 로그인 시도.
3.  **EventBus Wiring**: `persistence_mgr.subscribe_to_bus(event_bus)` → 이벤트 자동 구독.
4.  **Setup UI**: `MainWindow(context)` → 도킹 윈도우, 차트 서버, 대시보드 위젯 배치.
5.  **Load Config**: 로컬 `settings.json`에서 이전 설정 로드.
6.  **Connect Signals**: 위젯 간, 엔진-매니저 간 신호 연결.
7.  **Event Loop**: `QApplication.exec_()` 실행을 통한 실시간 시세 처리 시작.

---

## 3. Key Service Managers [V17.0]

| Manager | Module | Role |
| :--- | :--- | :--- |
| `ExecutionManager` | `app/services/managers/execution_manager.py` | 매매 실행, 리스크 관리, 포렌식 이벤트 발행 (SSoT) |
| `PersistenceManager` | `app/services/managers/persistence_manager.py` | SQLite 영속화, EventBus 구독, GTID 생성 |
| `RankingManager` | `app/services/managers/ranking_manager.py` | Smart Heat Index 산출, 주도주 발굴 |
| `StateManager` | `app/services/managers/state_manager.py` | 포지션/설정 상태 관리 (SSoT) |

---

## 4. Operational Integrity (운영 무결성)

### 4.1 32-bit Environment
키움증권 Open API는 OCX 컨트롤 기반이므로 반드시 **Python 32-bit** 환경에서 실행되어야 합니다. (64-bit 환경에서는 `QAxWidget` 생성 시 `QAxBase::setControl` 오류가 발생합니다.)

### 4.2 Mock Mode vs Real Mode
-   **Mock Mode**: 가상의 체결 정보를 생성하며, 실제 증권사 서버로 주문을 보내지 않습니다. 테스트 및 전략 검증용입니다.
-   **Real Mode**: 키움증권 서버와 직접 통신하며 실제 자산을 운용합니다. 체크박스를 통해 수동으로 전환해야 합니다.

---
*Document Version: v2.0 (Aligns with Engine V17.0)*
