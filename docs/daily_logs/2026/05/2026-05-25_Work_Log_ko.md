# 📅 2026-05-25 업무 일지 (Daily Work Log)

## 🎯 주요 목표 및 이슈 정의
- [x] **5월 22일 저녁(V25.1 & V24.9) 추가 패치 내역 문서화**:
  5월 22일 16시 41분 업무 일지 작성 이후 추가로 진행된 실시간 감시 누락(SetRealReg) 제어, MFE/MAE 타이머 정밀 추정, 출처 속성(Attribution) 추적 및 스냅샷 아카이빙 등의 핵심 백엔드 업그레이드 내역을 작업일지에 반영합니다.
- [x] **`StateManager` Lock 자체 데드락 해결**:
  `get_active_surveillance_set` 메서드가 `self.lock`을 획득한 상태에서 다시 `self.surveillance_set` 프로퍼티(내부에서 다시 Lock을 획득하려고 시도)에 접근하여 발생하던 비재진입성(Non-reentrant) 자체 데드락 결함을 수정하였습니다.
- [x] **포렌식 대시보드(Forensic Dashboard) 성능 최적화 검증 및 롤백**:
  장중 로그 폭주 시 GUI 스레드가 블로킹되어 "응답 없음"이 발생하는 문제를 해결하기 위해 `QTableView` 기반 일괄(Batch) 업데이트 모델 도입 및 Active Tab 렌더링 스로틀링을 구현 및 검증했으나, 실거래 라이브 구동 시의 성능 부담으로 인해 안정성을 위해 최종적으로 이전 버전으로 안전 롤백을 집행하였습니다.

---

## ✅ 상세 작업 및 패치 내역

### 1. 5월 22일 저녁 추가 업그레이드 내역 (V25.1 & V24.9 코어 업그레이드)

#### ① OpenAPI 실시간 등록 누락(Surveillance Gap) 방어
* **소스 파일**: `app/ai/engine.py`
* **패치 내용**: 
  - 신규 종목 실시간 구독 등록 시 `SetRealReg` API의 마지막 인자(`Opt`) 값을 기존 덮어쓰기(`"0"`)에서 추가 등록(`"1"`)으로 변경하였습니다.
  - 기존 보유 종목이나 먼저 감시 중이던 종목들의 실시간 시세 구독이 무작위로 해제되는 치명적인 감시 공백 현상을 완벽하게 예방하였습니다.
  - 추가로 60초 이상 틱 수신이 지연될 경우 이를 즉시 감지하여 포렌식 로그에 기록하는 `TICK_GAP` 로직을 이식하였습니다.

#### ② `on_timer` 루프 기반 MFE/MAE 정밀 추적 및 고점(peak_price) 양방향 동기화
* **소스 파일**: `app/services/managers/execution_manager.py`
* **패치 내용**:
  - 기존에는 틱 데이터가 수신될 때만 고점/저점이 갱신되어 거래 빈도가 낮은 종목은 MFE/MAE 추적이 누락되거나 이로 인해 Ratchet SL(손절선 끌어올리기)이 작동하지 않는 결함이 있었습니다.
  - 이를 해결하기 위해 `on_timer` 타이머 루프에서 직접 가격 변화를 감지해 `state_manager` 내 positions 및 daily_price_cache의 max/min_price를 강제 갱신하도록 수정했습니다.
  - `ExitEngine` 내부 세션의 `peak_price`와 `state_manager`의 `max_price`가 서로 불일치하던 구조적 동기화 오류를 해결하기 위해 양방향 동기화 파이프라인을 추가하였습니다.
  - 매 틱 루프마다 불필요하게 `ACTIVE_CONFIG`를 import 하던 오버헤드를 파일 상단 import로 이관하여 제거하고, 메인 스레드를 블로킹하던 `print` 출력을 `logger.debug`로 전면 교체하여 Stutter(버벅임) 현상을 개선하였습니다.

#### ③ 실시간 주도주 출처 추적(Attribution) 및 1분 스냅샷 아카이빙
* **소스 파일**: `app/services/managers/ranking_manager.py`
* **패치 내용**:
  - 종목이 최초로 발굴된 탭(HOT, PRE_HEAT, REVERSE)과 현재 소속된 탭을 분리하여 추적하는 `Attribution` 맵을 구축하였습니다. 이 정보는 주문 메타데이터에 이식되어 "이 주문이 어떤 전략적 탭에서 시작되었는가"를 사후 포렌식 분석에서 추적할 수 있게 합니다.
  - 당일 실시간 랭킹의 흐름을 백테스팅에 활용하고 검증할 수 있도록, 1분 주기로 실시간 탭 노출 내역을 `logs/ranking_snapshots_YYYYMMDD.jsonl` 파일에 백업하는 아카이버를 연동했습니다.
  - 테마 흐름 분석(`aggregate_theme_flow`) 연산 시 등수 분할 경계에 발생하던 float 정밀도 계산 공백 버그를 수정하여 등급 뱃지(`⚡`)가 공백으로 출력되는 현상을 해결하였습니다.
  - `_inferred_theme_cache` 변수가 클래스 변수로 지정되어 세션 재시작 후에도 캐시가 오염되는 현상을 해결하기 위해 인스턴스 멤버 변수로 격리 조치하였습니다.

#### ④ 동전주 필터 정책(Penny Stock Policy) 및 다중 포착 노이즈(Cross Boost) 제어
* **소스 파일**: `app/services/managers/ranking_manager.py` & `app/core/config_engine.py`
* **패치 내용**:
  - `PENNY_FILTER_MODE` 옵션을 탑재하여 1000원 미만 동전주를 원천 차단(`hard_exclude`)하거나, 진단 목적으로 점수만 감쇄(`damped`)할 수 있도록 SSoT 설정을 반영했습니다.
  - 여러 조건식(VALUE, GAIN, VOLUME)에 동시 포착될 때 적용되던 다중 포착 가산점(Cross Boost)이 선형 증폭 방식(최대 1.5x)에서 `log1p` 비선형 완화 곡선으로 변경되고 최대 1.25배(`CROSS_BOOST_MAX`)로 캡핑되어, 동전주 및 급등 잡주의 점수 오염을 방지했습니다.

---

### 2. 5월 25일 금일 작업 내역

#### ① `StateManager` 비재진입성 Lock 자체 데드락 결함 수정
* **소스 파일**: `app/services/managers/state_manager.py`
* **현상**: 
  - `get_active_surveillance_set()`이 호출되면 내부적으로 `self.lock`을 획득(Acquire)합니다.
  - 이 락이 유지되는 컨텍스트 내부에서 `self.surveillance_set` 프로퍼티를 참조하는데, `surveillance_set` 게터 메서드가 내부에 또다시 `with self.lock:` 블록을 호출하고 있었습니다.
  - Python의 기본 `threading.Lock`은 동일 스레드가 중복해서 호출할 수 없는 **비재진입성 락**이므로, 자기 자신에 의해 영구 대기 상태에 빠지는 자가 데드락(Self-Deadlock)이 발생하여 시스템이 완전히 얼어버리는 결함이 존재했습니다.
* **조치**: 
  - `get_active_surveillance_set()` 내부에서 `self.lock`을 잡은 상태에서는 외부 프로퍼티 대신 내부 프라이빗 멤버 변수인 `self._surveillance_set`을 직접 조회하도록 코드를 수정하여 데드락을 완전히 해결하였습니다.

#### ② 포렌식 대시보드(Forensic Dashboard) 성능 최적화 검증 및 롤백
* **소스 파일**: `app/ui/forensic_dashboard.py`
* **작업 내용**: 
  - 장중 실시간 로그 유입 속도가 수십~수백 건에 달할 때 GUI 스레드가 지속해서 프리즈되는 현상을 예방하기 위해 최적화 구조를 설계 및 반영했었습니다.
  - **반영되었던 구조**:
    1. QTableWidget을 가벼운 `QTableView` + `QAbstractTableModel` 모델 기반으로 교체.
    2. 열 크기를 고정하여 자동 리사이즈 Repaint 연산 방지.
    3. UI가 최소화되거나 비활성 탭에 위치할 때 연산을 스킵하는 Active Tab Guard 조건 추가.
    4. 로그 유입 시 즉시 렌더링하지 않고 250ms 타이머 기반의 일괄(Batch) 업데이트 도입.
    5. JSON Pretty 포맷팅 연산을 사용자가 마우스로 클릭할 때만 실행하도록 Lazy-load 구성.
  - **롤백 사유**: 
    - 최적화 패치 후 컴파일 자체는 성공적이었으나, 실제 실거래와 결합된 극한의 로그 입력 환경에서 여전히 GUI 부하가 관측되었고 가용 자원이 제한적인 개인 시스템 환경에서의 안정성 확보를 위해 최종적으로 안전한 이전 버전으로 롤백(Rollback)을 단행하였습니다.
    - 실시간 라이브 블랙박스 분석은 사후 `flight_recorder_dump.json` 파일을 통한 비동기 분석 방식으로 통일하고, 실시간 GUI는 핵심 요약 메트릭에만 집중하는 것이 안전하다는 판단 하에 원복 조치하였습니다.

---

#### ③ 포렌식 대시보드 상단 요약 카드 집계 정합성 패치 (V26.6)
* **소스 파일**: `app/ui/forensic_dashboard.py`
* **현상**:
  - `TOTAL SIGNALS` 카드는 전체 DB의 총 로그 수(예: 62만 건)를 보여주지만, `ACCEPT RATE`와 `BLOCK RATE` 등의 분자(`self.stats['accept']`, `self.stats['block']`)는 최근 UI에 로드된 슬라이딩 윈도우(`events` 리스트, 최대 2000건)를 기준으로만 세고 있었습니다.
  - 이로 인해 비율 계산의 분자(최대 2000)와 분모(62만)의 척도가 크게 어긋나, 실제 승인율/차단율이 비정상적으로 왜곡(예: 0.8%, 0.0%)되어 거의 0에 수렴하는 수치로 고정되는 결함이 존재했습니다.
* **조치**:
  - `recalculate_summary_metrics()` 메서드를 신설하여, 현재 화면에 실제로 로드되어 있는 슬라이딩 윈도우 데이터셋(`self.model._data`)을 기준으로 승인율(`ACCEPT RATE`), 차단율(`BLOCK RATE`), 체결수(`FILLS`), 손절수(`STOPLOSS`)를 동적 집계하도록 개선하였습니다.
  - 실시간 이벤트 유입 및 과거 역사적 배치 로딩(`hydrate_history`)이 완료되는 즉시 재집계 로직을 트리거하여 UI 상단 카드와 하단 로그 테이블 간의 데이터 정합성을 100% 일치시켰으며, 수치 왜곡 현상을 해결하였습니다.

---

#### ④ 메인 화면 툴팁 스타일 캐스케이드(상속) 오염 결함 수정 (V26.7)
* **소스 파일**: `app/ui/widgets/signal_status_widget.py`
* **현상**:
  - `추세(MA)`, `심리(RSI)`, `변동성(BB)` 등의 QLabel 위젯에 점수에 따른 배경색(초록/빨강/노랑 등)을 칠할 때, 스타일시트를 셀렉터 없이 날것으로 지정(`setStyleSheet("background-color: ...")`)하고 있었습니다.
  - 이로 인해 해당 위젯에서 파생되는 툴팁(`QToolTip`) 객체로 스타일시트 속성이 상속 전파(Cascade Pollution)되어, 첫 번째 이미지처럼 초록색 배경에 연두색/자홍색 글씨 등으로 렌더링되어 식별이 완전히 불가능해지는 현상이 발생했습니다.
* **조치**:
  - `signal_status_widget.py` 내부에서 개별 위젯에 스타일을 적용하는 모든 `setStyleSheet` 호출부를 조사하여, 타입 셀렉터를 명시한 구조(`QLabel { background-color: ... }` 및 `QFrame { ... }`)로 전면 개편하였습니다.
  - 이를 통해 스타일 상속 오염을 원천 예방하고, 프로그램 모든 영역에서 글로벌 스타일시트로 통일된 고급 어두운 배경 디자인(두 번째 이미지)의 툴팁이 완벽히 동일하게 노출되도록 보정하였습니다.

---

#### ⑤ Invariant Sealing & Observability 실행 계획 (Phases 1-4) 구현 및 검증 완료
* **패치 내용**:
  - **Phase 1 (읽기 락 간소화 및 DTO 스냅샷 격리)**: `StateManager`의 `get_position` 및 `get_all_positions` Getter에서 무거운 포인터 스왑 대신 `positions.copy()`를 통해 락 보유 시간을 최소화하고 상태 오염을 원천 차단하는 스냅샷 격리 구조를 검증 및 확정하였습니다.
  - **Phase 2 (EngineLifecycleManager 통합)**: MainWindow 종료 이벤트(`closeEvent`)에서 스레드 및 DB 커밋/정리를 대행하는 오케스트레이터 `EngineLifecycleManager`를 완벽히 wiring하고, lifecycle shutdown을 보장하도록 구현하였습니다.
  - **Phase 3 (초경량 Flight Recorder 및 런타임 메트릭 빌더 통합)**:
    - `PersistenceManager` 내부에 메모리 내 순환 버퍼(`deque(maxlen=2000)`)를 연동하여 실거래 crash 발생 직전의 시스템 메모리 상태를 보존하고, Coordinated Shutdown 수행 시 `flight_recorder_dump.json`으로 디스크에 자동 덤프하도록 이식하였습니다.
    - `queue_depth` (DB 병목), `callback_delay_ms` (키움/API 지연), `signal_loop_ms` (전략 연산 지연), `flush_latency_ms` (SQLite commit latency), `ranking_cycle_ms` (랭킹 loop 지연), `replay_drift_ms` (리플레이 시간 괴리) 등 실전 병목을 추적하는 초경량 실시간 메트릭 수집기 구조를 설계하여 각 Manager에 탑재하였습니다.
    - 수집된 런타임 메트릭을 `HealthMonitor`와 연동하여 30초 간격으로 published payload 및 가독성 높은 시스템 로그 메시지에 함께 바인딩하여 출력하도록 강화하였습니다.
    - **[가짜 진입 분석용 미시 시장 지표 추가 및 UI 툴팁 연동]**: 대규모 리팩토링이나 핫패스 성능 저하 없이 False Positive 원인을 역추적할 수 있도록, 이미 백엔드 단에서 실시간 계산 중이거나 즉시 추출 가능한 핵심 지표 7종(`spread`, `spread_tick`, `spread_pct`, `bid_total_qty`, `ask_total_qty`, `imbalance_ratio`, `proximity_score`)을 호가 이벤트 및 랭킹 캐시에서 수집해 `SignalSnapshot` DTO에 주입하고 자동 영속화하도록 기능을 확장하였습니다. 아울러 포렌식 대시보드 UI(`forensic_dashboard.py`) 내의 메트릭 설명 사전(`METRIC_GLOSSARY`)에도 해당 7종 지표의 한국어 툴팁 정의를 추가하여, 사후 분석 시 사용자가 마우스 오버만으로 각 지표의 의미를 직관적으로 파악할 수 있도록 편의성을 개선했습니다.
  - **Phase 4 (Determinism AST Lint 경고 적용 & 테스트 스위트 정리)**:
    - 핵심 엔진 코어 디렉토리(`app/core`, `app/services/managers`) 내부의 Wall Clock 누출을 전수 차단하기 위해 `tests/test_determinism_lint.py` AST 린트 유닛 테스트를 구현하였습니다.
    - `time.time()`, `datetime.now()`, `time.sleep()`, `uuid.uuid4()` 등 비결정적(Non-deterministic) 호출을 자동으로 검출하며, 기존 레거시 코드에 대한 호환성 보장을 위해 `(file, call_type)` 기반의 라인 독립적 baseline 기법 및 `# noqa: determinism` 인라인 제외 처리를 통합 구축하여 테스트 무결성을 통과시켰습니다.
    - `unittest discover` 실행 시 v12 레거시 모듈(`strategy`) 임포트 실패를 유발하던 `tests/test_scoring_v12_1.py`를 `tests/deprecated_test_scoring_v12_1.py`로 리네임하여 테스트 수집 오류를 해소하고, 전체 11개 유닛 테스트를 성공적으로 통과(OK)시켰습니다.

---

## 🔮 향후 대응 및 실거래 운영 안정화 방안
1. **사후 분석 중심의 포렌식**:
   - 실시간으로 대량의 로그 데이터를 PyQt GUI 인프라가 감당하는 구조를 지양하고, 장중에는 DB 쓰기 및 가벼운 메모리 서큘러 버퍼(Flight Recorder)에만 기록한 뒤, 장 마감 후 또는 크래시 시점에 덤프된 JSON 파일을 오프라인 툴로 시각화 분석하는 아키텍처로 선회합니다.
2. **SSoT 검증 테스트 정기 가동**:
   - PnL 정합성 테스트(`tests/test_pnl_ssot_consistency.py`) 및 감시/우선순위 스케줄러 기능 검증 테스트(`tests/test_logic_improvements.py`), Determinism AST Lint 테스트(`tests/test_determinism_lint.py`)를 운영 개시 전 정기적으로 기동하여 코어 무결성을 감시합니다.

---

#### ⑥ 주도테마 흐름 및 보유잔고 감시 패널의 독립 QDockWidget 분리 리팩토링 (V27.0)
* **소스 파일**: `app/ui/widgets/theme_flow_widget.py`, `app/ui/builders/sidebar_builder.py`, `app/ui/main_window.py`
* **현상**:
  - 기존 단일 `QDockWidget` 내부에 `QSplitter`를 배치하여 주도테마 흐름과 보유 잔고 감시 패널을 상하로 묶은 구조는 서로 다른 라이프사이클과 제약조건으로 인해 세로 크기가 좁아질 때 화면이 깨지고 겹치며, 개별 유연성(Movable/Floatable)을 제한하는 구조적 한계가 있었습니다.
* **조치**:
  - **독립 QDockWidget 분리**: 기존의 묶여 있던 독 구조를 분해하여 `dock_theme_flow`("🔥 실시간 주도테마 흐름")와 `dock_pos`("💼 보유 잔고 & 백그라운드 감시") 두 개의 독립된 `QDockWidget`으로 리팩토링하였습니다.
  - **ObjectName 명시**: 각 독 위젯에 명시적인 `setObjectName`("dock_ranking", "dock_theme_flow", "dock_pos")을 지정하여 사용자정의 독 배치 상태가 `saveState`/`restoreState`를 통해 안정적으로 영속성(Persistence)을 갖도록 처리했습니다.
  - **QSplitter 제거 및 QMainWindow Docking System 활용**: `sidebar_builder.py`에서 vertical splitter를 제거하고 `QMainWindow` 고유의 `addDockWidget` 및 `splitDockWidget`을 사용하여 상/중/하 스플릿 형태로 자연스럽게 초기 정렬 배치했습니다.
  - **안전 제약 조건 보장**: 닫기 버튼으로 인해 실수로 창이 증발하는 현상을 예방하기 위해 `DockWidgetClosable` 피처를 제외하고 `DockWidgetMovable | DockWidgetFloatable` 조합을 지정했습니다.
  - **레이아웃 단순화 롤백**: 복잡한 `QScrollArea` 래핑 코드를 걷어내고 원래의 단순하고 직관적인 fixed-height 아키텍처로 환원함으로써 기술 부채를 소멸시키고 겹침 현상을 원천 해결했습니다.
  - **보유잔고 감시 패널의 세로 축소 제약 해제**: [position_dashboard.py](file:///c:/Users/stone/projects/nocodequant/app/ui/components/position_dashboard.py) 내 `MultiSessionDashboard` 위젯 자체의 sizePolicy를 `MinimumExpanding`에서 **`Preferred`**로 완정 전환하고, `pos_table` 및 `surv_table`의 최소 높이(`setMinimumHeight`) 설정을 기존 `100px`/`80px`에서 **`45px`**로 크게 낮추어, 독 창 크기를 마우스 드래그로 극단적 수준(1라인 이하)까지 축소 가능하도록 유연성을 확보했습니다.
  - **초기 독(Dock) 세로 비율 정밀 보정**: [main_window.py](file:///c:/Users/stone/projects/nocodequant/app/ui/main_window.py)의 `_apply_dock_initial_width()`에서 세 독립 독의 초기 세로 힌트 사이즈를 기존 `[260, 360, 310]`에서 **`[230, 310, 260]`**으로 최종 조율하여, 주도주 6라인 보장, 주도테마 공백 최소화, 보유잔고 영역의 시인성 확보를 완전하게 구현했습니다.

---

#### ⑦ 거버넌스 및 아키텍처(Architecture)·독스(Docs) 전수 동기화 최신화 (V27.0)
* **대상 파일**:
  - `Antigravity_rules/GOVERNANCE_MANIFEST.md`
  - `Antigravity_rules/architecture/Project_Structure_Map_ko.md`
  - `Antigravity_rules/architecture/L4_System_Relationship_Map_ko.md`
  - `Antigravity_rules/architecture/Decision_Flow_Architecture.md`
  - `Antigravity_rules/architecture/Execution_Pipeline_Architecture.md`
  - `Antigravity_rules/rules/Failure_Patterns_and_Guards_ko.md`
  - `Antigravity_rules/rules/Signal_Gating_Rules.md`
  - `docs/design_notes/Design_Note_V27_Dock_Decoupling_and_Lifecycle_Integration_ko.md` (신규 생성)
* **작업 내용**:
  - **거버넌스 매니페스트 (`GOVERNANCE_MANIFEST.md`)**: 신규 생성된 설계 문서 및 프로젝트 구조 맵을 등록하고 버전을 **v27.0**으로 상향 동기화했습니다. 또한 Phase 4의 AST 결정성 린트 테스트(`test_determinism_lint.py`) 완성 상태를 마킹했습니다.
  - **L4 시스템 관계 지도 (`L4_System_Relationship_Map_ko.md`)**: V27.0의 핵심 아키텍처 변경사항인 QDockWidget 독립 분리, `EngineLifecycleManager`를 통한 Coordinated Shutdown 및 Flight Recorder 메트릭 로직 등을 Theme 5/6 및 UI 데이터 연동 설명에 반영하여 **v27.0**으로 최신화했습니다.
  - **의사결정 흐름 아키텍처 (`Decision_Flow_Architecture.md`)**: 실제 코드와 불일치하던 가변 진입 임계치 기본값(TREND:49, RANGE:57 등), ADX 국면 전환 경계값(25/22), `_is_real_trend` (ma_slope/adx) 전략 분류 필터 및 `WEAK_VSA` 패널티 사양을 실제 실행 코드(SSoT) 수치와 완전히 일치하도록 정정하여 **v27.0**으로 최신화했습니다.
  - **실행 파이프라인 아키텍처 (`Execution_Pipeline_Architecture.md`)**: `on_timer` 루프의 1초 간격 MFE/MAE 정밀 동기화, `EngineLifecycleManager`를 통한 Coordinated Shutdown 흐름, 그리고 6대 실시간 런타임 지연 메트릭 수집 및 HealthMonitor 바인딩을 설명에 추가하여 **v27.0**으로 최신화했습니다.
  - AI 작업 헌장 v1.8에 따라 최신 코드 패치 내역(동전주 필터, 다중 포착 노이즈 완화, 테마 흐름 Attribution, StateManager 데드락 해결, QDockWidget 독립 분리, EngineLifecycleManager 도입 및 AST determinism lint 테스트 스위트 등)을 규칙 및 실패 패턴 문서에 전면 업데이트하였습니다.
  - 전역 프로젝트 구조 맵(`Project_Structure_Map_ko.md`)에 신규 파일(`lifecycle_manager.py`, `theme_flow_widget.py`) 및 `tests/` 디렉토리 하위 유닛 테스트 목록을 동기화하고 버전을 v27.0으로 갱신하였습니다.
  - QDockWidget 독립 분리 및 EngineLifecycleManager 통합 배경과 세부 인프라 설계를 상세히 기록하는 신규 설계 노트(`Design_Note_V27_...md`)를 성공적으로 작성 완료했습니다.


#### ⑧ 잔여 규칙(Rules) 및 하위 가드(Guards) 5종 전수 교차 검증 및 최신화 (V27.0)
* **대상 파일**:
  - `Antigravity_rules/rules/Order_Lifecycle_Spec.md`
  - `Antigravity_rules/rules/Order_Identity_and_Idempotency_Standards_ko.md`
  - `Antigravity_rules/rules/guards/LOW_CONFIDENCE.md`
  - `Antigravity_rules/rules/guards/STARTUP_GUARD.md`
  - `Antigravity_rules/rules/guards/VSA_GATE.md`
* **작업 내용**:
  - **주문 생명주기 사양 최신화 (`Order_Lifecycle_Spec.md`)**: `order_manager.py` 실제 구현에 맞춰 `30초 Pending Timeout Guard`, `pending_signal_meta` 디스크 파일 영속화(`data/pending_meta.json`), 분할 매도 시 parent_gtid 무관하게 동일 종목 기존 acc 무조건 재활용 규칙, `VWAP Sanity Guard` (진입가 대비 50% 이상 괴리 시 보정) 및 앱 정상 종료 시 미완료 SELL 누적기 강제 플러시(`flush_sell_accumulators`)를 사양서에 등재하고 버전을 **v27.0**으로 갱신했습니다.
  - **주문 식별자 및 멱등성 표준 정렬 (`Order_Identity_and_Idempotency_Standards_ko.md`)**: `kiwoom_constants.py` 내 `sanitize_order_id` 함수에서 실제 필터링 중인 블랙리스트 대상(`""`, `"JJ"`, `"None"`)을 실제 실행 코드(SSoT)와 100% 일치하도록 규격화하고 버전을 **v27.0**으로 갱신했습니다.
  - **저신뢰 차단 가드 정형화 (`LOW_CONFIDENCE.md`)**: 계산식 `Risk Block Score = 0.5 * Confidence + Adjustments`가 코드와 완벽히 일치함을 검증하고, 진입 예비 상태(`DecisionLabel.HOLD_READY`)가 UI의 "전략 준비 중"으로 나타남을 보강 기술하여 버전을 **v27.0**으로 갱신했습니다.
  - **데이터 수집 대기 가드 연동 (`STARTUP_GUARD.md`)**: 트리거 조건인 최소 확정 캔들 개수(`min_bars`)가 `ACTIVE_CONFIG.get("WARMUP_MIN_BARS", 50)`을 통해 런타임에 동적으로 주입됨을 명시하고 버전을 **v27.0**으로 갱신했습니다.
  - **수급 확증 필터 개편 (`VSA_GATE.md`)**: 기존 레거시 OR 조건 대신, 현대 NoCodeQuant 엔진에서 `volume_ratio < 0.8x` 조건 시 발동되는 `WEAK_VSA` 패널티 게이트와, VOLUME 전략형 진입 시 `volume_ratio < 1.5x`일 때 발동되는 하드 필터 이중 구조로 구성됨을 실체에 맞게 개편하고 버전을 **v27.0**으로 갱신했습니다.

#### ⑨ 사용자 가이드(User Guide) 문서 4종 전수 정합성 검증 및 최신화 (V27.0)
* **대상 파일**:
  - `docs/Config_User_Guide_ko.md`
  - `docs/Trading_Logic_Guide_ko.md`
  - `docs/AI_Tuning_User_Guide_ko.md`
  - `docs/POE_User_Guide_ko.md`
* **작업 내용**:
  - **설정 가이드 (`Config_User_Guide_ko.md`)**: 가이딩 엔진의 임계값 적용 방식(국면 vs 전략 2중 검증), ADX 히스테리시스 수치(25/22), `WEAK_VSA` 패널티 및 VOLUME 전략형 수급 필터 구조와 `Pending Timeout Guard`/`VWAP Sanity Guard` 리스크 가드 추가 사양을 정합화하고 버전을 **v27.0**으로 갱신했습니다.
  - **자동매매 로직 가이드 (`Trading_Logic_Guide_ko.md`)**: 실시간 진입 임계값 수치, 이중 VSA 수급 확증 필터 체계, 국면별 변속 Trailing Stop 배수 갱신, 그리고 `Pending Timeout Guard` 및 `VWAP Sanity Guard` 등의 신규 정책을 가이드에 추가하고 버전을 **v27.0**으로 갱신했습니다.
  - **AI 전략 튜닝 가이드 (`AI_Tuning_User_Guide_ko.md`)**: AI 헌장 v1.8 표준에 부합하도록 롤백 및 거버넌스 감사 로깅 사양을 기술하고 버전을 **v27.0**으로 갱신했습니다.
  - **POE 가이드 (`POE_User_Guide_ko.md`)**: 수익 최적화 엔진의 v27.0 사양 동기화 및 `EngineLifecycleManager`를 통한 Coordinated Shutdown과의 라이프사이클 관찰 연동 내역을 설명에 추가하고 버전을 **v27.0**으로 갱신했습니다.





