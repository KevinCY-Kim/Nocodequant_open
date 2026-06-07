# 2026-02-26 작업 기록 (V17.0 Backend Snapshot Heartbeat Architecture)

## 헌장 준수 확인 (AI Work Constitution Compliance)

### 작업 등급 식별 (Category Identification)
- **등급**: **L4** (아키텍처 및 데이터 흐름 변경)
- **근거**: Engine → EventBus → Persistence → UI 파이프라인의 발신 지점(SSoT) 변경. 기존 UI 레이어가 담당하던 포렌식 이벤트 생성 책임을 Backend(`ExecutionManager`)로 이관.

### 참조 문서 (Doc Reference)
| 문서 | 역할 |
|------|------|
| `Antigravity_rules/domains/AI_Work_Constitution_and_Reference_Map.md` | 최상위 거버넌스 |
| `docs/design_notes/Design_Note_Fix_Persistence_Interface.md` | Persistence 인터페이스 설계 |

### 영향 모듈 (Impacted Modules)
| 모듈 | 변경 유형 | 영향도 |
|------|-----------|--------|
| `app/services/managers/execution_manager.py` | L4 Core | `trigger_snapshot` → EventBus 직접 발행, `cached_results` 폴백 |
| `app/ui/main_window.py` | L3 Refactor | UI-side 포렌식 발행 제거, Decision Hub 동기화 추가 |
| `app/ui/forensic_dashboard.py` | L2 Bugfix | Aggregation 대소문자 비교 수정 |
| `app/core/events.py` | 참조 확인 | `PERSISTENCE_WHITELIST` 에 `SIGNAL_GENERATED` 포함 확인 |

### 불변성 검증 (Invariant Check)
- **Violated**: None
- **SSoT 원칙**: ✅ Backend(`ExecutionManager`)가 유일한 포렌식 이벤트 발신자로 확립
- **경계 존중**: ✅ UI 파일에서 매매 판단 로직 없음, `signal_status.update_status()`는 순수 시각화 역할만 수행
- **금지 패턴**: ✅ UI에서 `event_bus.publish()` 호출 제거 (Engine만 발행)

---

## 오늘 완료된 태스크

### 1. [L4] Backend-Driven Snapshot Heartbeat 아키텍처 전환

**목적**: 포렌식 이벤트의 단일 진실 공급원(SSoT)을 UI → Backend로 이관하여 감사 가능성과 데이터 무결성을 확보.

**변경 전 (V16.0)**:
```
UI._perform_minute_snapshot → Engine.trigger_snapshot → callback → UI.sync_execution_to_ui
  → UI에서 event_bus.publish() ← ❌ UI가 포렌식 이벤트의 발신자
  → UI에서 log_persistence_to_forensic() ← ❌ 이중 발행
```

**변경 후 (V17.0)**:
```
UI._perform_minute_snapshot → Engine.trigger_snapshot
  → Engine: event_bus.publish(SIGNAL_GENERATED) ← ✅ Engine이 유일한 SSoT
  → EventBus → PersistenceManager._handle_bus_event() → SQLite
  → on_updated_callback → forensic_bridge → ForensicDashboard
  → callback(sync_execution_to_ui) ← 순수 UI 시각 갱신만
```

**핵심 변경사항**:
- `ExecutionManager.trigger_snapshot`: `UnifiedTradeEvent(SIGNAL_GENERATED)` 직접 EventBus 발행, `is_background=True` 플래그 포함
- `MainWindow.sync_execution_to_ui`: `event_bus.publish()` 및 `log_persistence_to_forensic()` 호출 제거, 순수 UI 갱신으로 축소
- `MainWindow._perform_minute_snapshot`: 강제 callback 루프 및 더미 신호 생성 로직 제거

---

### 2. [L3] Cached Results Fallback 파이프라인

**문제**: 장 마감 후 전략 엔진에 캔들 데이터가 없어 `get_signal()` → `"데이터 수집 중"` 반환, 20/20 종목이 빈 결과로 emit.

**해결**: `trigger_snapshot(cached_results=scanner_results)` — Engine이 라이브 분석 우선 시도, 데이터 부족 시 UI의 `scanner_results` 캐시에서 마지막 유효 분석 결과를 폴백.

```python
# Engine: Live first, cached fallback
res = self.strategy_engine.get_signal(code, config_data)
if (not res or res.decision == "데이터 수집 중") and code in cached_results:
    res = cached_results[code]  # 장중 마지막 분석 결과
    cache_used += 1
```

**진단 로그**: `[SNAPSHOT-ENGINE] Completed: 20/20 emitted, 12 cached, 0 skipped`

---

### 3. [L2] Forensic Dashboard Aggregation 버그 수정

**원인**: `no_agg_types` 리스트가 `"SIGNAL_GENERATED"` (대문자)인데, 실제 이벤트 타입 값은 `"signal_generated"` (소문자 enum value). **Case-sensitive 비교**로 인해 SIG 이벤트가 집계 제외 목록에 매칭되지 않아 `(x20)` 하나로 뭉침.

**수정**:
```python
evt_type_upper = str(evt_type_val).upper()
is_agg_eligible = not any(t in evt_type_upper for t in no_agg_types)
```

---

### 4. [L3] Decision Hub 동기화 복원

**문제 A**: `sync_execution_to_ui`에서 `event_bus.publish()` 제거 후, `_on_signal_generated` 핸들러가 더 이상 호출되지 않아 `signal_status.update_status()` 경로 단절.

**수정**: `sync_execution_to_ui`에서 `is_active` 종목일 때 직접 `signal_status.update_status(result)` 호출 추가.

**문제 B**: `_on_signal_generated`가 EventBus의 모든 `SIGNAL_GENERATED` 이벤트(백그라운드 20개 포함)를 수신하여 활성 종목의 Decision Hub를 덮어씀.

**수정**: `_on_signal_generated`에 2단계 가드 추가 — (1) `is_background` 스킵, (2) `event_code != current` 스킵.

**문제 C**: 종목 선택 시 `_on_indicators_ready` (히스토리 로드 완료 콜백)가 차트만 업데이트하고 Decision Hub 미갱신.

**수정**: `_on_indicators_ready`에 `signal_status.update_status(signal)` 및 `scanner_results[code] = signal` 추가.

---

### 5. [L1] 디버그 코드 정리

- `log_persistence_to_forensic`: `PersistenceEvent`에 존재하지 않는 `user_friendly_msg` 접근 디버그 print 제거 (`AttributeError` 수정)
- `_perform_minute_snapshot`: V16.0 디버그 주석 및 강제 루프 코드 제거

---

## 아키텍처 다이어그램 (V17.0 Final)

```
┌─────────────────────────────────────────────────────────────┐
│  UI Layer (MainWindow)                                       │
│  ┌─────────────────────┐   ┌────────────────────────┐       │
│  │ _perform_minute_     │   │ sync_execution_to_ui   │       │
│  │ snapshot (Orchestr.) │   │ (Pure UI Refresh)      │       │
│  │ - warm_up            │   │ - signal_status.update │       │
│  │ - trigger call       │   │ - scanner cache        │       │
│  └──────────┬──────────┘   └───────────▲────────────┘       │
│             │                          │ callback            │
├─────────────┼──────────────────────────┼────────────────────┤
│  Engine Layer (ExecutionManager)       │                     │
│  ┌──────────▼──────────────────────────┼──────────┐         │
│  │ trigger_snapshot(config, heat_map,  │          │         │
│  │                  callback, cached)  │          │         │
│  │   1. get_signal(code) [Live]        │          │         │
│  │   2. cached_results[code] [Fallback]│          │         │
│  │   3. event_bus.publish(SIG) ← SSoT  │          │         │
│  │   4. callback(code, signal) ────────┘          │         │
│  └──────────┬─────────────────────────────────────┘         │
│             │ EventBus                                       │
├─────────────┼───────────────────────────────────────────────┤
│  Persistence Layer                                           │
│  ┌──────────▼──────────┐   ┌────────────────────────┐       │
│  │ PersistenceManager  │   │ ForensicDashboard      │       │
│  │ _handle_bus_event   │──▶│ sig_log_emitted        │       │
│  │ → SQLite INSERT     │   │ (Individual rows)      │       │
│  │ → on_updated_cb     │   │                        │       │
│  └─────────────────────┘   └────────────────────────┘       │
└─────────────────────────────────────────────────────────────┘
```

---

## 향후 주요 목표 (Roadmap)
- **Main_NCQ SoS (Separation of Services)**: `main_window.py` 4,200+라인에서 타이머/백그라운드 로직을 `SurveillanceCoordinator`로 분리
- **Strategy Session 통합**: Engine snapshot과 UI stock-selection이 동일한 Strategy Session 데이터를 사용하도록 통합 (현재 두 경로가 서로 다른 데이터로 분석)
- **실시간 드리프트 경고**: POE 건강 점수 급격 하락 시 시각/청각 경고 가드 추가

---
**Status: ✅ L4 Architecture Refactor Complete | ✅ Engine SSoT Established | ✅ Forensic Pipeline Verified | ✅ Decision Hub Synced | 🛡️ Constitution v1.1 Compliant**
