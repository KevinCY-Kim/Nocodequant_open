# 📅 2026-06-11 업무 일지 (Daily Work Log)

> **[거버넌스 준수]**
> * **작업 등급**: L3 (핵심 실행부 결함 수정)
> * **참조 문서**: 
>   * [AI_Work_Constitution_and_Reference_Map.md](file:///c:/Users/stone/projects/nocodequant/Antigravity_rules/governance/AI_Work_Constitution_and_Reference_Map.md)
>   * [order_manager.py](file:///c:/Users/stone/projects/nocodequant/app/services/managers/order_manager.py)
>   * [state_manager.py](file:///c:/Users/stone/projects/nocodequant/app/services/managers/state_manager.py)
>   * [execution_manager.py](file:///c:/Users/stone/projects/nocodequant/app/services/managers/execution_manager.py)
>   * [engine.py](file:///c:/Users/stone/projects/nocodequant/app/ai/engine.py)
>   * [async_runtime.py](file:///c:/Users/stone/projects/nocodequant/app/core/async_runtime.py)
>   * [persistence_manager.py](file:///c:/Users/stone/projects/nocodequant/app/services/managers/persistence_manager.py)
>   * [main_window.py](file:///c:/Users/stone/projects/nocodequant/app/ui/main_window.py)
> * **영향 받는 모듈 (Impacted Modules)**: `engine.py`, `order_manager.py`, `state_manager.py`, `execution_manager.py`, `async_runtime.py`, `persistence_manager.py`, `main_window.py`
> * **불변성 위반 여부 (Invariant Check)**: 위반 없음 (Violated: None)
> * **경계 준수 선언**: 비동기 감사 결과 도출된 핵심 Race Condition 및 상태 붕괴 결함 수정만을 집중적으로 반영하였으며, 불필요한 리팩토링이나 외부 모듈 오염을 차단했습니다.

---

## 🎯 주요 목표 및 이슈 정의

1. **비동기 Race Condition 2차 감사 및 핵심 결함 패치**:
   * **C1**: 분할 매도(2차 주문) 최종 기록이 `_emitted_fills` 중복 방어막에 의해 영구 차단되는 버그 해결.
   * **C2**: 첫 부분 체결 시 pending 조기 해제로 인해 중복 매도(oversell) 주문이 발주될 수 있는 취약점 해결.
   * **C3**: `cancel_pending_orders`가 실제 브로커 취소 없이 로컬 pending만 강제 해제하여 EMERGENCY 시 중복 주문이 가능했던 문제 해결.
   * **C4**: 30초 지정가 미체결 시 PENDING_TIMEOUT 강제 해제로 인해 중복 발주가 가능했던 문제 해결.
   * **C5**: `sync_positions` 전체 교체 시 stale 계좌 스냅샷이 실시간 체결된 로컬 포지션을 덮어써서 포지션 추적이 유실(SL 미발동)되던 stale snapshot race 해결.
   * **C6**: 부분 체결 후 취소 시 stale acc가 정리되지 않아 다음 주문에 오염된 데이터를 상속시키던 GC 키 불일치 문제 해결.
   * **H1**: `processed_fills` 및 `_emitted_fills` GC가 set of 해시 순서로 임의 제거되어 최신 fill_id가 증발 및 이중 반영될 수 있었던 결함 해결.
   * **H2**: `_save_pending_meta`에서 dictionary size change RuntimeError가 발생해 메타데이터 영속화가 누락되던 무락 동시 변이 경합 해결.
   * **H4**: `promoted_pool`을 메인 스레드와 워커 스레드가 동시 변이하여 발생하는 RuntimeError 및 이로 인한 해당 틱 분석 누락(손절 감시 포함) 문제 해결.
   * **H5**: 워커 스레드에서 직접 Qt GUI 위젯(`update_status`, `input_code.text()`)에 동기 접근하여 간헐적 UI 크래시 및 렌더 desync를 일으키던 크로스 스레딩 결함 해결.
   * **M1**: 종료(shutdown) 단계에서 `loop.close()` 예외 전파로 이후 persistence close 등이 중단되던 문제와, in-flight 틱이 persistence가 닫힌 뒤에 도착하여 누적 acc가 유실되던 시퀀스 결함 해결.
   * **M2**: `persistence_manager` writer 스레드가 최초 DB 연결(connect) 실패 시 NameError로 스레드가 영구 사망하고, 이후 대기 이벤트들이 조용히 유실되던 취약점 해결 및 DLQ 복구 체계 마련.
   * **M3**: replay 모드 사용 중 active_symbol 동기화가 라이브 엔진의 Tier-1 분석 대상을 오염시키는 현상 해결.
   * **M4**: rollover 시 `daily_price_cache`가 비워진 직후 도착하는 전일 지연 체결에 의한 실현 PnL 왜곡 해결.
   * **보너스 (재진입 데드락)**: `on_order_event` (취소 flush)가 `order_locks[code]`를 획득한 채 `on_fill_event`를 호출할 때, 기존 non-reentrant Lock 사용으로 메인 스레드가 영구 동결되던 데드락 결함 해결.

---

## ✅ 상세 작업 및 패치 내역

### 1. 부분 체결 시 pending 조기 해제 차단 및 종결 분기 처리 (C2)
* **원인 분석**:
  * 모든 체결 이벤트마다 발화되는 `sig_order_confirmed` 경로에서 `clear_pending`이 호출되어, 부분 체결 시에도 pending 상태와 dedup hash가 즉시 풀렸음. 이로 인해 잔량이 거래소에 살아있는 동안 cooldown이 통과되면 중복 매도가 나갈 수 있는 oversell 백터 존재.
* **패치 내역**:
  * [order_manager.py](file:///c:/Users/stone/projects/nocodequant/app/services/managers/order_manager.py#L131): `sig_order_confirmed` 경로의 `clear_pending` 제거.
  * [order_manager.py](file:///c:/Users/stone/projects/nocodequant/app/services/managers/order_manager.py#L949): SELL 주문 종결(`unfilled_qty == 0`) + 포지션 잔존 분기(`is_sell_complete`) 및 `is_order_completed`인 시점에만 pending을 정상 해제하여 잔여 포지션의 후속 청산이 즉시 가능하도록 조치.

### 2. 분할 매도 최종 기록 누락 방지 (C1)
* **원인 분석**:
  * 1차 체결 후 acc를 유지한 상태에서 2차 분할 매도 발주 시, 동일 `GTID`를 그대로 상속하여 체결되는데, `_emitted_fills` 중복 차단 가드가 GTID 단독으로 처리되고 있어서 2차 체결에 대한 최종 emit이 차단되어 DB에 최종 집계가 유실되었음.
* **패치 내역**:
  * [order_manager.py](file:///c:/Users/stone/projects/nocodequant/app/services/managers/order_manager.py#L851): emit guard 키를 `GTID` 단독에서 `{GTID}:{cum_qty}` 복합키로 교체. 이를 통해 동일 GTID일지라도 수량이 누적 증가한 재-emit은 trades.db의 `INSERT OR REPLACE`를 통해 누적값으로 정상 갱신되도록 하되, 동일 상태의 중복만 차단.
  * [order_manager.py](file:///c:/Users/stone/projects/nocodequant/app/services/managers/order_manager.py#L973): shutdown flush 시에도 중간 emit 이후 추가 체결이 누적된 acc가 스킵되지 않고 정상 flush되도록 `{GTID}:{cum_qty}` 복합키로 가드 구조 개선.

### 3. 브로커 취소 주문 구현 및 pending 수명주기 고도화 (C3, C4)
* **원인 분석**:
  * 로컬 pending 강제 해제 타임아웃(30초) 및 EMERGENCY 상황 시 실제 브로커 취소가 나가지 않고 로컬 상태만 해제되어, 거래소에 미체결 잔량이 생존해 있음에도 중복 주문이 나가던 구조적 문제점 존재.
* **패치 내역**:
  * [engine.py](file:///c:/Users/stone/projects/nocodequant/app/ai/engine.py#L905): 키움 `SendOrder` API의 주문유형 3(매수취소)/4(매도취소)를 사용해 원주문번호 기준 취소 주문을 발송하는 `cancel_order` 함수 신규 구현.
  * [order_manager.py](file:///c:/Users/stone/projects/nocodequant/app/services/managers/order_manager.py#L490): chejan '접수' 및 체결 이벤트에서 주문번호를 `pending_order_ids`에 추적하여 취소 발주의 전제조건 확보.
  * [order_manager.py](file:///c:/Users/stone/projects/nocodequant/app/services/managers/order_manager.py#L189): `cancel_pending_orders` 시 로컬 강제 해제 대신 실제 브로커 취소를 발송하고 취소 완료 시(`status == '취소'`) pending 해제. 10초의 취소 스로틀(`CANCEL_RETRY_SEC`) 적용.
  * [order_manager.py](file:///c:/Users/stone/projects/nocodequant/app/services/managers/order_manager.py#L325): 30초 지정가 미체결 타임아웃 발생 시 강제 해제 대신 브로커 취소를 발주하고 신규 주문 차단은 유지. 주문번호를 받지 못한 극단적인 경우에만 `PENDING_HARD_TIMEOUT_SEC` (90초) 이후 최후 수단으로 해제.
  * [execution_manager.py](file:///c:/Users/stone/projects/nocodequant/app/services/managers/execution_manager.py#L626): 긴급 손절(`is_stop_loss_triggered`) 중 pending 주문이 존재하면 취소 발주를 통해 즉시 취소 후 다음 틱에 손절이 집행되도록 개선.

### 4. stale 계좌 스냅샷 덮어쓰기 방지 (C5)
* **원인 분석**:
  * TR 계좌 조회는 요청 시점의 스냅샷이라 요청~응답 사이에 발생한 실시간 체결이 반영되어 있지 않음. `sync_positions`가 이를 강제 덮어쓰기하면서 방금 체결된 로컬 포지션을 삭제(SL 추적 중단)하거나 방금 청산한 포지션을 부활시키는 현상 발생.
* **패치 내역**:
  * [state_manager.py](file:///c:/Users/stone/projects/nocodequant/app/services/managers/state_manager.py#L414): 모든 체결 시각을 `recent_fill_ts`에 기록하고, `sync_positions`에 10초 grace window(`SYNC_FILL_GRACE_SEC`) 적용.
  * [state_manager.py](file:///c:/Users/stone/projects/nocodequant/app/services/managers/state_manager.py#L938): grace 내 신규 체결 포지션이 스냅샷에 없어도 삭제하지 않고 보존하며, 부분 청산 시에는 로컬 수량을 우선하고, 방금 전량 청산한 종목은 유령 부활을 방지함. grace 만료 후에는 기존처럼 서버를 SSoT로 유지.

### 5. GC 키 불일치 수정 및 RLock 재진입 데드락 방지 (C6, 데드락)
* **원인 분석**:
  * `on_order_event` (취소 flush)가 `order_locks[code]`를 획득한 채 `on_fill_event`를 재호출 시 non-reentrant Lock에 의해 메인 스레드가 영구 동결되는 데드락이 잠재했음. 또한 exact-key pop과 실제 키 형식 불일치로 Unknown acc가 무한 누적되어 이전 정보를 상속받는 오염이 발생했음.
* **패치 내역**:
  * [order_manager.py](file:///c:/Users/stone/projects/nocodequant/app/services/managers/order_manager.py#L381): exact-key pop 대신 prefix 기반 스캔 제거로 교체하여 stale Unknown acc를 제거하고, 부분 체결 acc는 보존하여 분할 매도 집계(취소를 가로지르는 경우 포함)를 정상적으로 보존.
  * [order_manager.py](file:///c:/Users/stone/projects/nocodequant/app/services/managers/order_manager.py#L110): `order_locks`의 Lock 유형을 재진입 가능한 `threading.RLock`으로 교체하여 메인 스레드 프리즈 현상 차단.

### 6. GC 무작위성 제거 (H1)
* **원인 분석**:
  * `processed_fills`와 `_emitted_fills`가 set으로 동작해, 1000건 초과 시 `list(set)[:500]` 형태로 GC할 때 삽입 순서가 아니라 해시 순서(무작위)로 제거되었음. 이 때문에 최신 체결 정보가 먼저 제거되어 키움 중복 데이터 수신 시 이중 체결 처리 위험 존재.
* **패치 내역**:
  * [state_manager.py](file:///c:/Users/stone/projects/nocodequant/app/services/managers/state_manager.py#L110) 및 [order_manager.py](file:///c:/Users/stone/projects/nocodequant/app/services/managers/order_manager.py#L490): set을 삽입 순서가 보존되는 dict로 변환. GC 발생 시 가장 오래된 항목부터 선입선출(FIFO) 형태로 제거하도록 안전하게 교체 완료.

### 7. pending_signal_meta 직렬화 경합 해결 (H2)
* **원인 분석**:
  * `send_order`를 수행하는 워커 스레드가 `pending_signal_meta`에 쓰기를 함과 동시에 메인 스레드가 `clear_pending(clear_meta=True)`를 실행하여 pop하면, `_save_pending_meta()`의 `json.dump` 호출 루프 중 `RuntimeError: dictionary changed size during iteration`이 발생해 메타 저장에 실패했음.
* **패치 내역**:
  * [order_manager.py](file:///c:/Users/stone/projects/nocodequant/app/services/managers/order_manager.py#L416): 직렬화 저장 시 `dict(self.pending_signal_meta)` 형태로 원자적 로컬 스냅샷을 먼저 복사한 뒤, 이를 대상으로 `json.dump`를 수행하도록 변경하여 경합 차단.

### 8. promoted_pool 동시 변이 및 틱 루프 중단 예외 해결 (H4)
* **원인 분석**:
  * 메인 스레드(`add_to_promoted_pool` 승격 큐 흡수)와 틱 분석 워커 스레드(`_cleanup_promoted_pool` 만료 처리)가 `promoted_pool` 등의 내부 세 컬렉션을 동시 변이할 경우 `RuntimeError`가 유발되었음. 광역 except 블록이 이를 삼켜버리면서 해당 틱의 나머지 종목 분석 및 손절 평가가 통째로 묵살되는 심각한 오류 발생.
* **패치 내역**:
  * [execution_manager.py](file:///c:/Users/stone/projects/nocodequant/app/services/managers/execution_manager.py#L78): 신규 재진입 가능 락 `self._promoted_lock = threading.RLock()` 선언.
  * pool 변이 메서드(`add_to_promoted_pool`, `_cleanup_promoted_pool`, `_enforce_slot_limit` 등) 전체에 락 가드 블록 추가.
  * 락을 쥔 상태에서 EventBus로 이벤트를 발행하면 구독자 콜백들이 동일 스레드 상에서 락을 쥔 채 실행되는 위생 문제가 있으므로, 이벤트 발행 로직(`_publish_pool_event`)을 락의 임계 구역 바깥에서 실행되도록 구조 개선.

### 9. 워커 스레드의 Qt GUI 직접 접근에 의한 크래시 가드 (H5)
* **원인 분석**:
  * 백그라운드 워커 스레드에서 생성된 시그널 콜백 핸들러인 `_on_signal_generated` 및 `_on_surveillance_opportunity`가 메인 스레드(Qt GUI) 전용 컴포넌트인 `signal_status.update_status()`, `input_code.text()`, `candle_managers` 등에 직접 동기식으로 교차 접근하여 크래시 및 렌더링 동기화 실패를 야기했음.
* **패치 내역**:
  * [main_window.py](file:///c:/Users/stone/projects/nocodequant/app/ui/main_window.py#L148): 스레드 마샬링용 PyQT 시그널 `sig_landing_signal_evt` 및 `sig_landing_surv_evt` 선언.
  * [main_window.py](file:///c:/Users/stone/projects/nocodequant/app/ui/main_window.py#L641): 핸들러 진입 시 호출 스레드가 메인 GUI 스레드가 아닐 경우, 데이터를 시그널로 emit하고 즉시 리턴(마샬링 우회)시킴으로써 Qt queued connection이 안전하게 메인 GUI 스레드에서 핸들러를 실행하도록 가드.

### 10. AsyncRuntime 종료 안정화 및 틱 quiesce 도입 (M1)
* **원인 분석**:
  * (a) `AsyncRuntime.shutdown` 시 태스크 `join(timeout=5)` 후에도 완고하게 떠 있는 루프에 강제로 `loop.close()`를 가하면 `RuntimeError: Cannot close a running event loop`가 발생해 persistence close 등 다음 프로세스가 정지했음.
  * (b) shutdown 도중에 루프를 정지하면 틱 루프에서 닫힌 루프에 run_coroutine_threadsafe를 호출하여 `RuntimeError`가 전파되었음.
  * (c) `flush_sell_accumulators` 호출 시점에 실행 중인 in-flight 틱이 계속 진행되어, flush 직후 완료된 체결이 acc에 다시 쌓이나 persistence close 직후여서 기록이 유실되는 창이 존재했음.
* **패치 내역**:
  * [async_runtime.py](file:///c:/Users/stone/projects/nocodequant/app/core/async_runtime.py#L149): join timeout 시 loop close를 안전하게 바이패스하도록 우회 처리 (daemon 스레드이므로 프로세스 종료 시 자동 회수됨). 닫힌 loop 스케줄링 시 RuntimeError를 try-except로 흡수하고 `None`을 반환하여 틱 루프 중단을 완전 격리 차단.
  * [execution_manager.py](file:///c:/Users/stone/projects/nocodequant/app/services/managers/execution_manager.py#L112): `quiesce()` 메서드 신규 구현. 새로운 틱 진입 플래그(`_is_shutting_down`)를 차단하고, 이미 실행 중인 in-flight 틱(`is_processing == True`)이 모두 안전하게 종료될 때까지 대기(최대 3초)한 뒤 shutdown이 진행되도록 라이프사이클 동기화 흐름 수정.

### 11. Persistence Manager Writer 스레드 사망 방지 및 DLQ 적용 (M2)
* **원인 분석**:
  * sqlite3 DB 연결(connect) 시 디스크 잠금 등으로 일시적인 예외가 터지면, except/finally 블록이 try 내부에서 생성되기도 전인 `events` 변수를 참조해 `NameError`로 스레드가 즉시 사망했음. 이로 인해 누적 큐가 5000건에서 버려지는 현상 발생.
* **패치 내역**:
  * [persistence_manager.py](file:///c:/Users/stone/projects/nocodequant/app/services/managers/persistence_manager.py#L176): `events = []` 선언을 try문 시작 이전에 배치하여 변수 미정의 참조(NameError) 가능성을 제거.
  * except 구역 내 `rollback` 및 `close` 호출에 개별 try-except 감싸기를 적용하여 2차 예외로 스레드가 동반 사망하지 않도록 락 위생 강화.
  * 큐 유실을 예방하기 위해 큐 포화 시 이벤트를 폐기하지 않고 데드레터큐 파일(`persistence_dlq.jsonl`)에 안전하게 보존하도록 예외적 방어선 구축.

### 12. Replay에 의한 Live 엔진 오염 차단 (M3)
* **원인 분석**:
  * `on_tick_timer_callback` 내에서 활성화된 active_symbol의 UI 갱신 동기화 코드가 replay 모드인지 여부를 판단하지 않았음. 이로 인해 Replay에서 과거 종목을 열람할 때 해당 종목이 input_code를 통해 실시간 라이브 엔진의 Tier-1 분석 대상을 강제로 밀어내고 승격되어 실거래 분석 대상을 역오염 시키는 심각한 구멍이 존재했음.
* **패치 내역**:
  * [main_window.py](file:///c:/Users/stone/projects/nocodequant/app/ui/main_window.py#L2062): active_symbol 동기화 호출 부근에 `not self.replay_context.is_replay` 가드를 추가하여, Replay 진입 중에는 과거 종목 조회에 따른 라이브 감시 대상의 강제 변경을 차단하고, Replay 진입 이전 상태가 라이브 백그라운드 엔진에서 계속 감시되도록 분리 격리.

### 13. Rollover 직후 지연 체결에 의한 realized PnL 왜곡 해결 (M4)
* **원인 분석**:
  * `check_daily_rollover`가 `daily_price_cache`를 완전히 비운 직후, 전일에 미처 체결되지 않고 지연 전송된 체결 이벤트가 도착하면 `update_position_on_fill`에서 Cache Miss가 발생함. 이때 로컬 평단가가 없으므로 지연 체결 시점의 현재가를 평단가로 강제 임치하여 gross PnL이 0에 가깝게 왜곡되고, trades.db/승률 등 성과 평가 전체가 오염되던 현상 발생.
* **패치 내역**:
  * [state_manager.py](file:///c:/Users/stone/projects/nocodequant/app/services/managers/state_manager.py#L484): `daily_price_cache`가 비어있을 때 즉시 현재가로 덮어쓰는 대신, `overrides.json`에 영속화되는 로컬 `position_meta_cache`에서 해당 종목의 `entry_price`를 탐색하는 2차 복원 fallback 레이어를 추가하여 가격 정합성 복구.

---

## 🧪 검증 및 테스트 결과

* **신규 회귀 테스트 작성**: [test_order_lifecycle_races.py](file:///c:/Users/stone/projects/nocodequant/tests/test_order_lifecycle_races.py)
  * **T1**: 부분 체결 시 pending 유지 및 중복 SELL 차단 검증 (C2)
  * **T2**: 분할 매도 최종 집계 emit 정상 발행 및 동일 GTID 누적 갱신 검증 (C1)
  * **T3**: flush가 누적 상태 변화분 기록 검증 (C1)
  * **T4**: 브로커 취소 발주 + 취소 확인까지 pending 유지 검증 (C3)
  * **T5**: 타임아웃 시 취소 발주 및 중복 주문 차단 유지 검증 (C4)
  * **T6**: 주문번호 미수신 pending의 HARD TIMEOUT 최후 해제 검증
  * **T7**: `sync_positions` grace guard 검증 (보존/로컬우선/유령차단/만료 후 서버 SSoT) (C5)
  * **T8**: 취소 flush 데드락 검증 및 acc 보존을 통한 분할 집계 유지 검증 (C6 + RLock)
  * **T9**: GC가 삽입순서 기반으로 오래된 항목부터 정확히 제거함을 검증 (H1)
  * **T10**: 복수 Promoter 스레드와 만료 Cleaner 스레드의 2초 극한 경합 스트레스 하에서 예외 0건 및 3종 자료구조의 엄격한 싱크 정합성 확인 (H4)
  * **T11**: sqlite3 connect 실패(OperationalError) 모킹 주입 하에 writer 스레드가 안전하게 대기/생존하고, 연결 재활성화 즉시 이벤트를 forensics.db에 정상 기록해 내는 자가 복구 검증 (M2)
  * **T12**: 강제 timeout stubborn 코루틴 shutdown 시 loop close RuntimeError 누수 없이 안전 퇴출하고, 닫힌 루프에 schedule 시 None 반환으로 예외 고립 검증 (M1(a))
  * **T13**: in-flight 틱 drain 완료 및 quiesce 진입 차단 흐름 검증 (M1(b))
  * **T14**: BUY → 전량 SELL → daily_price_cache 초기화(rollover 상황) 후 잔여 3주 체결 시, meta 진입가를 정상 복원해 실현 PnL이 정확히 계산됨을 검증 (M4)

* **테스트 및 커밋 결과**:
  * `conda run -n trade_exe_v2 python tests/test_order_lifecycle_races.py` 실행 결과 **T1~T14 전체 정상 통과 완료**.
  * 수정된 모든 모듈(`main_window.py`, `lifecycle_manager.py` 등)의 python 구문 검사(`py_compile`) 완료.
  * 기존 `test_pnl_ssot_consistency` 회귀 테스트 영향 없음 확인.
  * **커밋 완료**: 커밋 해시 **`7df4631`** (총 8개 파일 수정 및 1개 신규 테스트 추가, +1,073/-155줄)

---

## 🔮 향후 계획

- **실거래 장중 검증 체크리스트 (5대 항목)**:
  1. **취소 흐름 (가장 중요)**: 지정가 주문 30초 방치 시 로그에 브로커 취소 발주 → chejan 취소 확인 → `Pending Cleared`가 순차적으로 정확히 찍히는지 검증 (단위 테스트가 불가능한 OCX 호출 영역).
  2. **부분 체결**: `Pending Cleared`가 부분 체결 이후 주문이 완전히 종결(unfilled=0)되기 전에 발생하여 oversell 여지를 주는 일이 없는지 확인 (C2).
  3. **UI 안정성**: GUI 시그널 마샬링 도입 후(H5 패치), 장중 신호/감시 패널 갱신 및 UI 위젯 렌더링에 안정성 보장 여부 모니터링.
  4. **Replay 격리**: Replay 모드 진입 및 이탈 시 라이브 신호 및 active_symbol 역오염 여부 교차 검증 (M3).
  5. **종료 흐름**: 앱 종료 시 `[Execution] Quiesced` 로그가 찍히고 in-flight 틱이 안전하게 배출된 뒤 `flush` 및 DB close가 진행되는지 최종 확인 (M1).

- **남은 잔여 항목 해결**:
  - **H3 (get_signal 동시 호출)**: ✅ **당일 후속 세션에서 해결 완료** — 하단 "H3 해결: 전략 세션 동시성 직렬화" 섹션 참조.

---

## 📝 V28.0 적용 및 튜닝 파라미터 백테스트 비교 분석

### 1. 개요 및 병행 비교 수행
* **목적**: 어제(06-10) 작업일지에 반영된 V28.0 개선안(L0 조기 수익 잠금 래칫, 봉 성숙도 진입 게이트, 시간대 보수 가중)의 실효성 검증을 위해, 2가지 독립 데이터셋(`260507`, `260609`)을 대상으로 Baseline 대비 비교 백테스트를 수행함.
* **비교 대상 데이터셋**:
  1. `data/ohlcv/260507` (총 67개 종목 - 완만한 노이즈 장세)
  2. `data/ohlcv/260609` (총 281개 종목 - 강한 주도주 추세 장세)

### 2. 백테스트 결과 및 요인별 분석 (Factor Analysis)

#### 2-1. `260507` 데이터셋 결과 (완만한 노이즈 장세)
* **Baseline (적용 전)**: 거래 278회, 승률 34.53%, 누적 수익률 **+94.53%**, 평균 보유 8.8봉
* **V28.0 (적용 후)**: 거래 270회, 승률 35.56%, 누적 수익률 **+92.71%**, 평균 보유 8.0봉
* **분석**: 거래 8회가 줄고 승률이 +1.02%p 개선되었으나, 누적 수익률은 -1.82%p 미세 하락하여 전반적으로 휩소 방어를 통한 **안정성 향상 효과**를 입증함.

#### 2-2. `260609` 데이터셋 및 6개 튜닝 시나리오 비교 결과 (강한 주도주 추세 장세)
주도주 쏠림과 추세 분출이 강했던 `260609` 장세에서는 V28.0 개선안이 오히려 성능을 크게 악화시키는 모순(아이러니)이 발견되어, 이를 해결하기 위해 파라미터 스윕 튜닝을 병행하여 분석함.

| 시나리오 (Scenario) | 거래 횟수 | 승률 | 누적 수익률 (Total PnL) | 평균 수익률 | 평균 보유 기간 |
| :--- | :---: | :---: | :---: | :---: | :---: |
| **1. Baseline (이전)** | 1,123회 | 34.28% | **+26.82%** | +0.02% | 6.3봉 |
| **2. V28.0 (기존 개선안)** | 1,052회 | 34.03% | **-13.43%** | -0.01% | 5.9봉 |
| **3. Tuning A (180s / 1.5%-0.5%)** | 1,050회 | 34.76% | **-7.11%** | -0.01% | 6.1봉 |
| **4. Tuning B (150s / 1.5%-0.5%)** | 1,050회 | 34.76% | **-7.11%** | -0.01% | 6.1봉 |
| **5. Tuning C (180s / 1.0%-0.3%)** | 1,052회 | 34.03% | **-13.43%** | -0.01% | 5.9봉 |
| **6. Tuning D (240s / 1.5%-0.5%)** | 1,050회 | 34.76% | **-7.11%** | -0.01% | 6.1봉 |

* **아이러니의 원인 분석 (Why Baseline Wins?)**:
  1. **L0 래칫 조기 컷의 역설 (Choking Winners)**: 고점 +1.0% 터치 후 스톱로스를 +0.3%로 올리는 타이트한 규칙으로 인해, 눌림목을 거쳐 +3% ~ +5% 이상 크게 갈 주도주들이 미세 조정 구간에서 조기 청산되어 수익 폭이 강제로 제약됨. (1.5%/0.5%로 완화 시 수익률이 -13.43% ➔ -7.11%로 6.32%p 개선되는 것이 검증됨)
  2. **진입 가드의 우량 진입 필터링 (Over-filtering)**: 개별 진입들의 평균손익이 나쁘다는 이유로 10시·12시 및 봉 초반 진입을 가로막자, 손실 거래를 차단한 이득보다 소수의 초대형 대박 거래(Outlier) 기회를 잃은 타격이 훨씬 더 커서 누적 수익률이 급감함.
  3. **백테스트 상의 봉 성숙도 제한**: 백테스트는 완성된 과거 분봉을 사용하므로 경과 시간 `_elapsed` 계산이 300초를 초과하여, 성숙도 게이트(150s, 180s, 240s) 조건이 어차피 전부 자동 통과(바이패스)로 작동하지 않음. 즉, 이 게이트는 실전에만 작동하는 Live 전용 가드임.

---

### 3. 향후 계획: 1주일 라이브 테스트 및 병행 비교 수집 돌입
* 백테스팅 환경과 실전(Live) 체결의 괴리가 뚜렷하고 트레이드오프가 매우 크므로, 모의투자에 적용된 V28.0 상태 그대로 **1주일간 실전 라이브 테스트**를 수행하여 최종 결정을 내리기로 합의함.
* **병행 비교 도구 배치**:
  * [analyze_live_gating_performance.py](file:///c:/Users/stone/projects/nocodequant/scratch/analyze_live_gating_performance.py) 작성 완료.
  * 1주일 뒤 이 스크립트를 가동하면 `data/forensics.db`의 `SIGNAL_BLOCKED` 로그와 당일 생성된 분봉 데이터를 기반으로 **"가드에 차단당하지 않았다면 났을 실제 손익률"** 및 **"래칫에 걸리지 않았다면 났을 최종 청산 손익률"**을 역산하여 Baseline vs V28.0의 정량적 실전 비교 보고서를 자동으로 생성함.

---

## 🔒 H3 해결: 전략 세션 동시성 직렬화 (비동기 감사 15/15 완료)

### 1. 문제 구조 (분석 결과)
* `StrategyManager.sessions[code]` (EMA/std_cache/regime/position)에 **4개 스레드가 무락 접근**:
  1. **on_timer 워커** (NCQ_Compute): `get_signal()` — ExitEngine/EntryEngine이 `sess['position']` 직접 변이
  2. **분 스냅샷 워커** (NCQ_Compute): `analyze_snapshot_pure()` → 동일 `get_signal()` 호출
  3. **UI 워커** (QThread `StrategyWorker`): `_get_signal_internal()` — V25.X 핵은 position만 원복 (EMA/std_cache 오염 잔존)
  4. **메인 스레드**: `update_tick`(df in-place + peak_price)/`update_data`/`sync_position`
* **핵심 사고 경로**: 분 스냅샷 워커의 SELL 판정 시 ExitEngine이 주문 발주 없이 `sess['position']=None` 세팅 → on_timer의 트레일링/래칫 청산 영구 누락. BUY 판정 시 유령 `pending_fill` 포지션 생성 → 실주문 없이 실진입 차단.

### 2. 적용 수정 (메모리 접근안 2가지 동시 적용)
* **종목 단위 세션 락** ([strategy.py](file:///c:/Users/stone/projects/nocodequant/app/ai/strategy.py)):
  * `session_lock(code)` (RLock) 신설 — `get_signal`/`update_tick`/`update_data`/`sync_position`/`reset_state`/`soft_reset`/UI 워커 `calculate_async` 전부 직렬화.
  * `_get_session` 생성 경합 가드 (동시 생성 시 한쪽 세션 유실 방지).
  * **락 순서 고정: `session_lock` → `state_manager.lock`** (역순 금지).
* **분 스냅샷 읽기 전용 분리** ([execution_manager.py](file:///c:/Users/stone/projects/nocodequant/app/services/managers/execution_manager.py)):
  * `get_signal(..., readonly=True)` 신설 — 세션 상태 백업→분석→원복으로 부수효과 0 보장 (`analyze_snapshot_pure` 적용).
  * UI 워커의 V25.X position-만-원복 핵을 전체 상태 원복으로 교체.
  * execution_manager의 직접 세션 변이 지점(peak↔max 양방향 동기화, kill-switch position 철회)도 동일 세션 락으로 보호.

### 3. 검증
* 신규 회귀 테스트 [test_strategy_session_races.py](file:///c:/Users/stone/projects/nocodequant/tests/test_strategy_session_races.py) **T1~T5 ALL PASS**:
  * T1 대조군(일반 호출은 세션 진화) / T2 readonly 변이 0 / T3 Flash Crash SELL 판정에서 readonly는 position 보존·mutable만 소거 / T4 3-스레드 × 120회 동시 스트레스(예외 0, std_cache 윈도우·EMA 불변식) / T5 락 동일성·RLock 재진입.
* 기존 [test_order_lifecycle_races.py](file:///c:/Users/stone/projects/nocodequant/tests/test_order_lifecycle_races.py) **T1~T14 ALL PASS** (회귀 없음).
* 이로써 2026-06-10 비동기 감사 식별 항목 **15건 전부 수정 완료** (C1~C6, H1~H5, M1~M4 + H3).

---

## 🚀 비TREND 전략(BREAKOUT, VOLUME) 구조적 결함 패치 및 튜닝 (V28.4)

### 1. 1단계: BREAKOUT 돌파 실패 컷 구현 완료 (V28.4 1단계)
* **결함 정의**: 
  * 기존 돌파 매매는 NORMAL 청산 평균 +0.18%에도 불구하고, 손절 12건 평균 -2.78%로 인해 손실이 가중됨. 돌파 레벨 이탈 후 지지 받지 못하고 붕괴하는 종목을 웜업 기간(2봉) 내에서도 조기에 자르지 못해 손실폭이 자기 노이즈 밴드보다 커지는 문제점 존재.
* **패치 내역**:
  * [config_engine.py](file:///c:/Users/stone/projects/nocodequant/app/core/config_engine.py): `BREAKOUT_FAIL_CUT_ENABLED = True`, `BREAKOUT_FAIL_CUT_BUFFER = 0.003` (0.3% 리테스트 노이즈 방어 버퍼) 파라미터 신설.
  * [_engine_entry.py](file:///c:/Users/stone/projects/nocodequant/app/ai/_engine_entry.py): 돌파 성공 시 직전 봉 고가(`breakout_level`)를 포지션 정보에 영속화.
  * [_engine_exit.py](file:///c:/Users/stone/projects/nocodequant/app/ai/_engine_exit.py): 현재가가 `breakout_level * (1 - buffer)` 하회 시 웜업 가드를 무력화하고 `STOP_LOSS`로 조기 청산하는 로직 이식.
  * **검증**: `test_breakout_fail_cut_260611.py` 작성 및 7/7 테스트 통과.

### 2. 2단계: VOLUME 전략 수급 재정의 및 소수확 TP1 구현 완료 (V28.4 2단계)
* **결함 정의**:
  * 1개 분출봉 추격 매수(MFE +0.94% / MAE -0.98%)로 인해 짧은 시세 분출 후 즉시 꺾이는 "분출 한복판 매수" 휩소에 극도로 취약했음. 또한 기대수익이 낮음에도 2R 익절 한도를 공유해 익절 기회를 놓치던 상태.
* **패치 내역**:
  * [config_engine.py](file:///c:/Users/stone/projects/nocodequant/app/core/config_engine.py): `VOLUME_REDEFINE_ENABLED = True` 및 지속 수급 수집 분봉 수(`SUSTAIN_BARS=3`), 수급 강도(`SUSTAIN_MIN_RATIO=1.2`), 최소 수급 캔들 수(`SUSTAIN_MIN_COUNT=2`), 클라이맥스 제한(`CLIMAX_RATIO=2.0`), 고점 근접 제한(`CLIMAX_HIGH_PROX=0.998`), TP1 목표수익률(`VOLUME_TP1_PCT=0.009`) 파라미터 신설.
  * [_engine_entry.py](file:///c:/Users/stone/projects/nocodequant/app/ai/_engine_entry.py):
    * **지속 수급 검증**: 직전 3개 완성봉 중 2봉 이상 `vol_ratio >= 1.2`인 경우만 진입 허용.
    * **클라이맥스 차단**: 2배 이상의 거래량 급증이면서 당일 세션 고점 부근(`high_prox` 이상)에 바짝 붙은 양봉 진입 차단.
    * 수급 재정의 필터 이식과 함께 `ENTRY_TH_VOLUME` 임계값을 100 ➔ 54로 완화/해제.
  * [_engine_exit.py](file:///c:/Users/stone/projects/nocodequant/app/ai/_engine_exit.py): VOLUME 전략 진입 건에 대해 +0.9% 수익률 도달 시 50% 분할 익절(`PARTIAL_TP`)을 강제 집행하도록 분기 설계.
  * **검증**: `test_volume_redefine_260611.py` 작성 및 8/8 테스트 통과. 기존 BREAKOUT 테스트 7/7도 회귀 없이 동시 통과 완료.
  * **문서 동기화**: [Parameter_Invariants.md](file:///c:/Users/stone/projects/nocodequant/Antigravity_rules/rules/core/Parameter_Invariants.md) 최신화 완료.

---

## 🔎 실거래 사후 분석 및 이상 현상 진단

### 1. 예수금 비동기 지연으로 인한 MME 총자산 이중 계산 (Double Counting) 결함
* **발생 시나리오 (14:04)**:
  * `14:04:05`: 원익IPS(`240810`) 매수 주문 체결 (420만 원 사용).
  * `14:04:06`: 스피어(`347700`) 매수 신호 계산.
* **이상 현상**: 스피어 매입 금액(530만 원)이 원익IPS 체결 전의 예수금(1,701만 원) 기준 25% 비율을 훨씬 초과해 예수금 대비 **31.2%**가 매수됨.
* **원인 분석**:
  * 키움 OpenAPI의 추정예수금(`opw00001`) 갱신은 거래소 원장 반영까지 약 1~2초의 **비동기 지연**이 존재하여 캐시된 예수금은 여전히 1,701만 원을 고수하고 있었음.
  * 반면 로컬 `chejan` 체결 수신은 즉시 처리되어 원익IPS의 주식 평가액(420만 원)이 포지션 정보에 가산됨.
  * 이에 따라 수량 계산식([money_manager.py](file:///c:/Users/stone/projects/nocodequant/app/core/money_manager.py#L73))의 `Total Asset`은 `예수금 (1,701만) + 주식 평가액 (420만) = 2,121만 원`으로 **원익IPS의 420만 원이 양쪽에 중복 합산(Double Counting)되는 심각한 논리적 오류**가 발생함.
  * 예전에는 15% 포지션 집중 제한 룰(Concentration Cap)이 가동되어 최종 매수가 15% 수준으로 통제되었기 때문에 이 결함이 드러나지 않았으나, `V27.9.2` 패치에서 해당 룰이 주석 처리되면서 이중 계산의 수치가 수량에 직격되었습니다. MME Heat Control은 손절 리스크 한도(5.0%)만을 관리하므로 이를 방어하지 못했습니다.

### 2. 갭하락 양봉장 내 "진입 기각 및 꼭지 매수" 현상
* **이상 현상**: 오늘처럼 갭하락 후 V자로 강력하게 상승하는 장세에서 전체 매매가 단 3건에 그쳤으며, 그마저도 상한가 부근, 20% 급등 지점, 그리고 SK하이닉스를 고점 부근에서 추격 매수(꼭지 매수)하여 컷당함.
* **원인 분석**:
  * **장초반 5분 진입 보류 (Opening Whipsaw Shield - `V27.6`)**: 9시 정각 이후 가장 저렴하게 살 수 있었던 최초 5분간의 눌림목 기회를 원천 차단함.
  * **아침 휩소 방지 허들 상향 (`OP_ENTRY_MORNING_TH_ADD`) 및 아침 동적 수급 가드 (`MORNING_VOLUME_GUARD_MULT`)**: 9시 30분 이전에는 진입 점수 임계치(`entry_th`)를 강제로 상향하고 평균 대비 비정상적인 대금 폭발만 허용하므로, 웬만한 주도주는 다 거르고 **거래량이 꼭대기까지 솟구치며 과열 상태에 도달해 허들을 뚫은 꼭지 종목만 진입시키는 부작용**을 유발함.
  * **봉 성숙도 대기 가드 (Bar-Maturity Gate - `V28.0`)**: 5분봉 중 240초 경과 후에만 진입을 승인하여, 캔들 중간의 눌림 기회를 모두 날리고 양봉의 꼬리가 다 차오른 고점에서 진입이 확정됨.
  * **기대수익 이격 제한 필터**: 하이닉스(`000660`)의 경우 급등 중일 때는 기대수익 부족으로 차단되다가, 고점 부근 횡보로 이평선(MA20)이 캔들 근처까지 따라온 시점에 '이격 3% 이내 만족'으로 진입이 승인되어 꼭지에서 매수 처리된 후 컷당하는 현상이 발생함.
* **대응**: 자본 방어를 위한 제어 장치들이 V자 반등 장세에서는 오히려 역선택 필터로 오작동함을 규명했으며, 아침 가드 해제 시각 단축이나 수급 가중치 하향 튜닝 등의 국면 적응형 완화 전략(Regime-Adaptive Relaxation)을 검토할 예정입니다.
