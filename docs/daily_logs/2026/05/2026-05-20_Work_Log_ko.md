# 📅 2026-05-20 업무 일지 (Daily Work Log)

## 🎯 주요 목표 및 이슈 정의
- [x] **재기동 및 실시간 잔고 동기화 시 익절보호선(SL) 유실 버그 해결**: 포지션 보유 중 앱을 재기동하거나 키움 서버와의 잔고 동기화(`sync_positions`)가 일어날 때, 실시간으로 최고가(MFE)를 반영하여 상승해 있던 트레일링 스탑(SL) 가격이 초기화(서버 평단가 또는 최하단 안전망으로 후퇴)되는 정합성 이탈 현상을 방어하기 위해 3계층 영속화 및 폴백 아키텍처 구현 완료.

---

## ✅ 상세 작업 및 패치 내역

### 1. 트레일링 스탑(SL) 및 최고가(MFE) 유실 원인 분석
* **원인 분석**: 
  - 주성엔지니어링(036930) 등 보유 종목의 수익률이 최고 +18.43%까지 도달하여 트레일링 스탑/래칫 익절선이 14% 수준까지 견고하게 상승했으나, 앱을 재기동하거나 백그라운드 서버 동기화가 실행된 이후 익절선이 11.31% 수준(혹은 그 이하인 doomsday SL/gap floor)으로 급하강하는 버그가 식별됨.
  - **원인 ① (디스크 영속화 누락)**: `execution_manager.py`가 매 틱마다 계산된 실시간 트레일링 스탑 가격(`sl`)을 `state_manager.positions[code]['sl']` 메모리 버퍼에는 정상 반영하고 있었으나, 로컬 디스크의 `overrides.json`을 비동기로 저장하는 `_save_overrides()`에서 `sl` 필드가 직렬화 대상에서 원천 누락되어 있어 재기동 시 완전히 소멸됨.
  - **원인 ② (동기화 중 메모리 유실)**: 실시간 가동 중에도 약 30초 주기로 호출되는 키움 잔고 강제 동기화 함수인 `state_manager.py`의 `sync_positions()`에서 서버 정보로 덮어쓰는 과정 중, 기존 메모리에 올라와 있던 `sl` 값을 새로운 병합본(`merged_positions`)으로 안전하게 복사(preserve)해오는 코드가 누락되어 지속해서 지워지고 있었음.
  - **원인 ③ (재기동 시 최고가 복원 실패)**: 재기동 직후 첫 잔고 동기화 시점에는 기존 로컬 포지션 메모리(`old_pos`)가 비어 있는 상태(`{}`)가 됨. 이 때문에 복구 시 `daily_price_cache`의 최고점 기록(`209,500원`)을 참고하지 못하고 단순히 서버 평단가(`176,900원`)를 `max_price`로 강제 리셋해 버림.
  - **원인 ④ (전략 세션의 후퇴)**: 최종적으로 `strategy.py`의 `sync_position()`에서 주입받은 `_saved_sl`이 0이고 `_saved_max`가 진입 평단가 수준으로 붕괴하자, 래칫 단계 판정이 모두 무산되고 최하단 안전망(`gap_floor` = 현재가 대비 -3%) 또는 doomsday SL 수준으로 익절보호선이 강제 후퇴하게 됨.

---

### 2. 해결 방안: 3계층 방어(Layered Defense) 설계 및 적용
* **해결 방안**: 
  - **Layer 1 (디스크 영속화)**: `daily_price_cache`에 `'sl'` 필드를 공식 추가하여 `overrides.json`에 영구 저장 및 복구하고, `execution_manager.py`의 매 틱 동기화 시점에 캐시에도 동시에 기입함.
  - **Layer 2 (동기화 중 보존)**: `sync_positions()` 병합 시 메모리에 살아있던 `sl` 필드를 명시적으로 보존하여 중간 소멸 현상을 완전 차단함.
  - **Layer 3 (재기동 폴백 자가 치유)**: 재기동으로 인해 메모리가 유실되었을 때, `sync_positions()` 및 `main_window.py` 복구 시점에 `daily_price_cache`에 백업되어 있던 최고가(`max_price`)와 익절보호선(`sl`)을 2차 조회하여 자가 치유(Self-Healing) 복원함.

---

### 3. 패치 코드 내역

#### ① [state_manager.py](file:///c:/Users/stone/projects/nocodequant/app/services/managers/state_manager.py) 수정
* **`_save_overrides()` 및 `_load_overrides()`의 daily_price_cache 직렬화 항목에 sl 필드 추가**
* **`sync_positions()`에 sl 보존 및 재기동 시 daily_price_cache로부터 MFE/MAE/SL 강제 복원 로직 추가**
```python
# app/services/managers/state_manager.py - _save_overrides()
_serialized_price_cache[_c] = {
    'avg_price':  _v.get('avg_price', 0.0),
    'max_price':  _v.get('max_price', 0.0),
    'min_price':  _v.get('min_price', 0.0),
    'sl':         _v.get('sl', 0.0),          # [SL영속화] 트레일링 SL 보존
    'last_update': _lu.isoformat() if hasattr(_lu, 'isoformat') else str(_lu or '')
}

# app/services/managers/state_manager.py - _load_overrides()
self.daily_price_cache[_c] = {
    'avg_price': float(_v.get('avg_price', 0.0)),
    'max_price': float(_v.get('max_price', 0.0)),
    'min_price': float(_v.get('min_price', 0.0)),
    'sl':        float(_v.get('sl', 0.0)),         # [SL영속화] 트레일링 SL 복원
    'last_update': _lu
}

# app/services/managers/state_manager.py - sync_positions()
_server_avg = server_data.get('avg_price', 0.0)
_old_max = old_pos.get('max_price')
_old_min = old_pos.get('min_price')
_old_sl  = old_pos.get('sl')

# [SL영속화 Layer 2+3] 재기동 직후 old_pos가 비어있을 때 daily_price_cache에서 복원
if code in self.daily_price_cache:
    _cache = self.daily_price_cache[code]
    if _old_max is None or _old_max <= 0:
        _old_max = _cache.get('max_price')
    if _old_min is None or _old_min <= 0:
        _old_min = _cache.get('min_price')
    if _old_sl is None or _old_sl <= 0:
        _old_sl = _cache.get('sl')

merged_positions[code]['max_price'] = _old_max if (_old_max is not None and _old_max > 0) else _server_avg
merged_positions[code]['min_price'] = _old_min if (_old_min is not None and _old_min > 0) else _server_avg
merged_positions[code]['sl'] = _old_sl if (_old_sl is not None and _old_sl > 0) else 0.0
merged_positions[code]['entry_time'] = old_pos.get('entry_time')
```

#### ② [execution_manager.py](file:///c:/Users/stone/projects/nocodequant/app/services/managers/execution_manager.py) 수정
* **매 틱 타이머 동기화 시점에 daily_price_cache의 sl도 갱신하여 디스크 영속화 흐름 생성**
```python
# app/services/managers/execution_manager.py - on_timer()
_cur_sl = _sess_pos.get('sl', 0)
if _cur_sl > 0:
    _sm_sl = self.state_manager.positions[code].get('sl', 0)
    if _cur_sl > _sm_sl:  # 단방향 래칫: 올라간 SL만 저장
        self.state_manager.positions[code]['sl'] = _cur_sl
        # [SL영속화 Layer 1] daily_price_cache에도 SL 동기화 (디스크 영속화 경로)
        if code in self.state_manager.daily_price_cache:
            self.state_manager.daily_price_cache[code]['sl'] = _cur_sl
```

#### ③ [main_window.py](file:///c:/Users/stone/projects/nocodequant/app/ui/main_window.py) 수정
* **재기동 직후 메모리가 유실되었을 때 daily_price_cache의 최고가(max_price)를 세션 기동 시점에 강제 폴백 주입**
* **[V25.0 추가 패치] 잔고 동기화 시점에 3계층(positions / daily_price_cache / 기존 전략 세션) 최댓값 SL을 산출하여 동기화 데이터에 강제 주입**
```python
# app/ui/main_window.py - on_positions_sync()
if not pos_data.get('max_price'):
    _mx = float((_sm_live or {}).get('max_price') or 0)
    # [SL영속화 Layer 3] 재기동 직후 _sm_live에도 없으면 daily_price_cache에서 복원
    if _mx <= 0:
        _dpc = self.state_mgr.daily_price_cache.get(code, {})
        _mx = float(_dpc.get('max_price') or 0)
    if _mx > 0:
        pos_data['max_price'] = _mx

# [V25.0 FIX] 3계층 SL 복원 + 기존 세션 SL 보호
_best_sl = float(pos_data.get('sl') or 0)
# Layer 1: state_manager.positions
if _sm_live:
    _sm_sl = float(_sm_live.get('sl') or 0)
    _best_sl = max(_best_sl, _sm_sl)
# Layer 2: daily_price_cache (디스크 영속화 경로)
_dpc = self.state_mgr.daily_price_cache.get(code, {})
_dpc_sl = float(_dpc.get('sl') or 0)
_best_sl = max(_best_sl, _dpc_sl)
# Layer 3: 기존 strategy session의 SL (reset_state가 보존한 값)
_existing_sess = self.strategy.sessions.get(code)
if _existing_sess and _existing_sess.get('position'):
    _sess_sl = float(_existing_sess['position'].get('sl') or 0)
    _best_sl = max(_best_sl, _sess_sl)
if _best_sl > 0:
    pos_data['sl'] = _best_sl
self.strategy.sync_position(code, pos_data, current_price)
```

#### ④ [strategy.py](file:///c:/Users/stone/projects/nocodequant/app/ai/strategy.py) 수정 (추가 작업)
* **`reset_state()` 세션 초기화 시 보유 포지션이 존재하면 SL/MFE 등의 핵심 상태값을 백업했다가 즉시 복원하는 보호 가드 추가**
* **`sync_position()`에서 최종 포지션 복구 시, 계산된 SL이 외부에서 주입된 최고 SL(`pos_data['sl']`)보다 낮아지는 현상 방지**
```python
# app/ai/strategy.py - reset_state()
def reset_state(self, code: str = None):
    """[V25.0 FIX] 보유 포지션의 position 정보(SL, peak_price 등)를 보존.
    세션 삭제 시 position dict가 함께 소멸하여 sync_position 재생성 때
    낮은 SL로 리셋되는 치명적 버그 방지.
    """
    if code:
        _preserved_position = None
        if code in self.sessions:
            _pos = self.sessions[code].get('position')
            if _pos is not None:
                _preserved_position = _pos.copy()
            del self.sessions[code]
        if code in self.cache:
            del self.cache[code]
        if _preserved_position is not None:
            new_sess = self._get_session(code)
            new_sess['position'] = _preserved_position
    else:
        _preserved = {}
        for c, s in self.sessions.items():
            _pos = s.get('position')
            if _pos is not None:
                _preserved[c] = _pos.copy()
        self.sessions = {}
        self.cache = {}
        for c, pos in _preserved.items():
            new_sess = self._get_session(c)
            new_sess['position'] = pos

# app/ai/strategy.py - sync_position()
# [V25.0 FIX] SL 하락 방지 최종 안전장치:
# 호출부(main_window.on_positions_sync)에서 3계층 max SL을 pos_data['sl']에 주입.
# 여기서는 _restored_sl이 pos_data['sl']보다 낮아지지 않도록 최종 확인.
_injected_sl = float(pos_data.get('sl') or 0)
if _injected_sl > _restored_sl:
    _restored_sl = _injected_sl
```

---

## 🔮 장중 진행 상황 모니터링 예정 사항
1. **코드 변경/감시 변경 시 SL 무결성 확인**: 사용자가 UI에서 종목 코드를 변경하거나 감시 해제/강등 이벤트가 발생할 때, 기존 보유 종목의 세션 리셋에 의해 트레일링 스탑(SL)선이 소멸되거나 낮아지지 않고 `[SL_GUARD]` 로깅과 함께 견고하게 보존되는지 검증.
2. **동기화 중 익절보호선 변동성 확인**: 동기화 신호 수신 시 래칫 SL이 다시 doomsday SL로 찰나의 순간이라도 흔들리는 현상(Whipsaw)이 완벽히 방지되었는지 실시간 로그 검사.

---

## 🔍 [소급 반영] 기가레인(049080) 플립플롭 및 MFE 왜곡 버그 원인 진단
* **현상**: 기가레인 종목 진입 시 1초 내외로 매수/매도가 무한 반복(플립플롭)되고, 실제 보유 시간이 극히 짧음에도 MFE가 당일 최고점 상승 폭(10% 이상)으로 오염되어 기록되는 오류 발생.
* **원인 식별**:
  - **원인 ① (플립플롭)**: `_engine_entry.py`에서 BUY **신호 발생 즉시** `sess['position']`을 미리 생성함. 실제 주문 체결(FILL)은 약 90초 후인데, 주문 중인 대기 상태에서 `_engine_exit.py`의 ExitEngine이 포지션을 보유한 것으로 판단하고 즉시 매도(SELL) 신호를 뿜어냄. 이 매도 신호는 쿨다운 우회 로직을 타고 들어가 1초 만에 매수 취소 및 강제 청산 루프를 반복함.
  - **원인 ② (MFE 왜곡)**: 감시 단계(pre-entry)에서 ExitEngine이 당일 캔들 고점을 `peak_price`로 계속 누적하고 있었음. 실제 진입 시 `state_manager` 상의 `max_price`는 리셋되나, 1초마다 수행되는 `execution_manager.py`의 양방향 동기화 로직에 의해 세션의 감시 단계 고점(`peak_price`)이 `max_price`로 역류하여 오염됨.
* **조치 계획 수립**:
  - BUY 신호와 실제 체결 시점을 구분하기 위한 `pending_fill` 가드 필터 도입 결정 및 수정 설계 승인 획득.

