# EventBus & Persistence Specification (이벤트 버스 및 영속화 사양서) [V17.0]

## 1. 개요 (Overview)
EventBus와 PersistenceManager는 NoCodeQuant의 **감사 추적 인프라(Audit Infrastructure)**를 구성합니다. 모든 매매 이벤트는 이 파이프라인을 통해 영속화되며, [Forensic & DB Immutability Rule](file:///c:/Users/stone/projects/nocodequant/Antigravity_rules/domains/Forensic%20%26%20DB%20Immutability%20Rule.md)에 의해 과거 데이터의 불변성이 보장됩니다.

---

## 2. EventBus (`app/core/event_bus.py`)

### 2.1 아키텍처
- **패턴**: Pub/Sub + In-Memory History (최대 10,000건, 초과 시 8,000건으로 트림)
- **구독**: `subscribe(event_type, callback)` — EventType별 콜백 등록
- **발행**: `publish(event)` — 모든 구독자에게 이벤트 전달 + 히스토리 저장
- **안전성**: 콜백 실행 시 `try/except`로 개별 오류 격리 (하나의 구독자 실패가 다른 구독자에 영향 없음)

### 2.2 발행 권한 규칙 (SSoT)
| 발행자 | 허용 여부 | 비고 |
| :--- | :--- | :--- |
| `ExecutionManager` | ✅ 허용 | 유일한 SSoT 발신자 |
| `OrderManager` | ✅ 허용 | FILL 이벤트 발행 (PersistenceManager 직접 호출) |
| `MainWindow` (UI) | ❌ 금지 | 헌장 §1.3 경계 존중 |

---

## 3. PersistenceManager (`app/services/managers/persistence_manager.py`)

### 3.1 역할
- **SQLite 영속화**: WAL 모드, Foreign Key 제약 조건
- **GTID 생성**: `YYMMDDHHMMSS-{TAG}-{HEX8}` 형식의 고유 감사 ID
- **EventBus 자동 구독**: `subscribe_to_bus(event_bus)` → `PERSISTENCE_WHITELIST` 내 이벤트 타입 자동 구독
- **콜백 전파**: INSERT 완료 시 `on_updated_callback` 호출 → UI ForensicDashboard 갱신

### 3.2 PERSISTENCE_WHITELIST
```python
PERSISTENCE_WHITELIST = {
    EventType.SIGNAL_GENERATED,
    EventType.ORDER_FILLED,
    EventType.ORDER_REJECTED,
    EventType.ERROR,
    EventType.SURV_OPPORTUNITY,
    # HEARTBEAT는 DB 비대화 방지를 위해 의도적 배제
}
```

### 3.3 이벤트 DTO 구조 (`PersistenceEvent`)
```python
@dataclass
class PersistenceEvent:
    gtid: str           # Global Trade ID
    event_type: str     # "SIGNAL_GENERATED", "FILL" 등
    payload: dict       # {symbol, name, side, score, confidence, regime, reason, ...}
    version: str        # DTO 버전 (현재 "1.0")
    timestamp: datetime # 이벤트 발생 시각
```

---

## 4. 데이터 흐름 (End-to-End Pipeline)

```
ExecutionManager._trigger_event()
    ↓
PersistenceEvent DTO 생성 (GTID 포함)
    ↓
event_bus.publish(event)
    ↓
PersistenceManager._handle_bus_event(event)
    ↓
persistence_manager.emit_event(event_type, event)
    ↓
SQLite INSERT (WAL Mode, Rowcount Check)
    ↓
on_updated_callback(event)
    ↓
MainWindow.sig_persistence_updated → log_persistence_to_forensic
    ↓
[V18.6] UI Sync Trigger (If ORDER_FILLED) → 1. update_performance_summary
                                           2. save_pnl_snapshot (Chart Sync)
    ↓
ForensicDashboard._append_event(event)
```

---

## 5. 방어 로직 (Guards)

| Guard | 위치 | 설명 |
| :--- | :--- | :--- |
| **Rowcount Check** | `PersistenceManager` | INSERT 후 `cursor.rowcount == 1` 검증 |
| **WAL Mode** | SQLite PRAGMA | 읽기/쓰기 비차단 처리 |
| **Busy Timeout** | `busy_timeout=20000` | 20초 대기 후 재시도 |
| **History Trim** | `EventBus` | 10,000건 초과 시 자동 트림 (8,000건) |
| **Callback Isolation** | `EventBus.publish` | 구독자별 `try/except` 격리 |

---
*Document Version: v1.0 (Aligns with Engine V17.0)*
