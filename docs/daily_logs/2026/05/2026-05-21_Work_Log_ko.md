# 📅 2026-05-21 업무 일지 (Daily Work Log)

## 🎯 주요 목표 및 이슈 정의
- [x] **기가레인(049080) BUY-SELL 플립플롭 및 MFE 왜곡 오류 최종 해결**: 주문 발생 후 체결되기 전까지의 과도기 상태에서 ExitEngine이 오작동하여 즉시 SELL 신호를 보내고(플립플롭), pre-entry 감시 단계의 최고가가 MFE로 역류하는 현상을 방지하기 위해 `pending_fill` 체결 대기 가드 메커니즘을 전면 적용 완료.

---

## ✅ 상세 작업 및 패치 내역

### 1. `pending_fill` 체결 대기 가드 메커니즘 개요
* **도입 배경**: 
  - 신호 발생(BUY) 시점과 실제 증권사 체결(FILL) 시점 사이에는 약 90초 이상의 지연 시간이 존재함.
  - 기존에는 BUY 신호 발생 시점에 즉시 `sess['position']` 구조체를 생성하고 이를 "실보유 포지션"처럼 간주하여 1초 단위로 돌고 있는 ExitEngine이 즉각 청산 조건(SL 등)을 감시하게 됨. 이로 인해 체결도 안 된 상태에서 SELL 주문이 나가며 체결 취소 및 플립플롭이 발생함.
  - 또한, 체결 이전에 ExitEngine이 당일 고가를 `peak_price`로 누적하는 것을 방지하고, 체결 확인 시점에 정확한 체결 평단가와 고가 상태로 리셋되도록 변경이 필요했음.
* **구현 핵심**:
  - `sess['position']` 내부에 `pending_fill = True` 상태 값을 주입하여 체결 전까지 ExitEngine 및 틱 단위 익절선 변경(`update_tick`) 감시 경로를 모두 우회시킴.
  - 1초 단위 감시 루프(`execution_manager.py`)에서 실제 포지션 수량(`qty > 0`)이 감지되면 `pending_fill` 상태를 해제(`False`)하고, 정확한 체결 가격을 주입함.

---

### 2. 패치 코드 내역

#### ① [app/ai/_engine_entry.py](file:///c:/Users/stone/projects/nocodequant/app/ai/_engine_entry.py) 수정
* **BUY 신호 발생 시 포지션 세팅에 `pending_fill=True` 플래그 추가**
* **초기 `peak_price`를 당일 봉 고가(`curr['high']`)가 아닌 신호 당시 종가(`curr['close']`)로 하향 조정하여 고가 오염 사전 방어**
```python
sess['position'] = {
    'entry_price':        curr['close'],
    'entry_time':         curr['datetime'],
    'entry_bar_index':    len(df) - 1,
    'sl':                 initial_sl,
    'initial_sl':         initial_sl,
    'peak_price':         curr['close'],  # [V25.1 FIX] curr['high'] → curr['close'] (봉 고점 오염 방지, 체결 시 리셋)
    'tp1_hit':            False,
    'entry_regime':       current_regime,
    'strategy_type':      strategy_type_val,
    'trailing_multiplier': (
        1.5 if current_regime in ("RANGE", "CHAOS") else 3.5
    ),
    'pending_fill':       True,  # [V25.1 FIX] 체결 전 ExitEngine 차단 (플립플롭 방지)
}
```

#### ② [app/ai/_engine_exit.py](file:///c:/Users/stone/projects/nocodequant/app/ai/_engine_exit.py) 수정
* **ExitEngine 진입 시점에 `pending_fill` 상태라면 즉시 리턴하여 탈출 로직 차단**
```python
pos = sess['position']
bars_held = (len(df) - 1) - pos.get('entry_bar_index', len(df) - 1)

# [V25.1 FIX] BUY 체결 대기 중 ExitEngine 실행 차단
# _engine_entry.py에서 BUY 신호 시 pending_fill=True로 세팅.
# 실제 체결 전까지 Exit 로직을 차단하여 플립플롭(매수신호→즉시매도) 방지.
# 체결 확인은 execution_manager.on_timer()에서 수행하며 pending_fill을 해제.
if pos.get('pending_fill'):
    return result
```

#### ③ [app/ai/strategy.py](file:///c:/Users/stone/projects/nocodequant/app/ai/strategy.py) 수정
* **틱 단위 SL 감시 및 peak_price 갱신 함수인 `update_tick()`에서 `pending_fill` 상태 검사 추가**
```python
pos = sess.get('position')
if pos is not None and not pos.get('pending_fill'):
    # [V25.1 FIX] pending_fill 중에는 peak_price 갱신 + tick SL 모두 차단
    # 체결 전 pre-entry 고점 누적 및 조기 SL 발동 방지

    # peak_price를 현재 tick 고점으로 갱신 (trailing SL 기준 최신화)
    tick_high = current_candle_dict.get('high', price)
    pos['peak_price'] = max(pos.get('peak_price', price), tick_high)

    sl_price = pos.get('sl', 0)
    ...
```

#### ④ [app/services/managers/execution_manager.py](file:///c:/Users/stone/projects/nocodequant/app/services/managers/execution_manager.py) 수정
* **매 1초 주기 타이머 루프에서 `pending_fill` 분기 로직 정교화**:
  - `pending_fill=True`일 때:
    1. 실제 계좌 포지션 수량(`qty > 0`) 확인 시: 체결 완료로 판단, `pending_fill = False` 해제 및 `entry_price`와 `peak_price`를 실제 체결 평균가 및 당시의 최고가로 정상 정렬.
    2. 주문 자체가 미체결 취소 또는 거부(`is_pending == False` 이고 `qty == 0` 일 때): 미체결 소멸로 판단하여 `sess['position'] = None` 처리하여 좀비 상태 방어.
  - `pending_fill=False`일 때:
    - 기존대로 트레일링 스탑 최고가 및 SL 양방향 동기화 처리 유지.
```python
_is_pending_fill = _sess_pos.get('pending_fill', False)

if _is_pending_fill:
    # [V25.1 FIX] BUY 체결 대기 중 — 양방향 동기화 차단
    # sess['position']은 BUY 신호 시점에 세팅되며,
    # 체결 전까지 peak_price→max_price 동기화를 차단하여
    # pre-entry 고점이 max_price에 주입되는 것을 방지.
    with self.state_manager.lock:
        _sm_pos = self.state_manager.positions.get(code)
        if _sm_pos and _sm_pos.get('qty', 0) > 0:
            # BUY 체결 확인 → pending_fill 해제 + 가격 초기화
            _fill_price = _sm_pos.get('avg_price', price)
            _sm_max = _sm_pos.get('max_price', _fill_price)
            _sm_min = _sm_pos.get('min_price', _fill_price)
            _sess_pos['pending_fill'] = False
            _sess_pos['peak_price'] = _sm_max
            _sess_pos['entry_price'] = _fill_price
            logger.info(
                f"[PendingFill] {code} BUY 체결 확인 → "
                f"entry={_fill_price:,.0f} peak={_sm_max:,.0f}"
            )
        elif not self.order_manager.is_pending(code):
            # BUY 주문 실패/취소 → 포지션 의도 철회 (좀비 방지)
            _sess['position'] = None
            logger.warning(
                f"[PendingFill] {code} BUY 주문 실패/취소 "
                f"→ sess['position'] 철회"
            )
else:
    # 기존 양방향 동기화 (pending_fill이 아닌 정상 보유 상태)
    ...
```

---

## 🔮 장중 진행 상황 모니터링 예정 사항
1. **기가레인 등 틱 변동성 높은 종목 체결 시나리오 관측**: 
   - 매수 신호가 발생하고 실제 체결될 때까지 대기하는 동안, UI 및 로그에 `[PendingFill]` 로그가 바르게 출력되고 체결 즉시 실시간 감시 모드로 전환되는지 모니터링.
2. **MFE 정상 기입 여부**:
   - 수정 패치 후 매매된 내역의 MFE(최고가 비율)가 보유 기간 외의 당일 급등 흐름을 포함하지 않고 실제 체결 이후의 고점만을 반영하는지 `trades.db` 쿼리를 통해 추적 검증.

---

## 후속 이슈 대응: SL 지연, TimeExit 미실행, Kill Switch 경로 수정

### 3. 라이브 모니터 후속 이상 징후 확인
* **현상 요약**:
  - 제주반도체(`080220`)가 Live MFE/MAE 탭에서 `peak_price=120,000`, `MFE=+12.25%` 상태인데 `현재 SL`은 114,751로 남아 `SL stale` 경고가 표시됨.
  - 하단 Exit 상세에서는 Ratchet / Hybrid Trail 보호선 계산이 이미 더 높은 가격대를 가리키고 있었지만, 실제 `sess['position']['sl']`은 거기까지 승격되지 못한 상태였음.
  - 같은 시점에 TimeExit 잔여가 `0분 (0봉)`으로 보이는데도 자동 청산이 실행되지 않았고, 터미널에는 `Global Kill Switch Active: Drawdown limit reached.` 경고가 출력됨.

### 4. 근본 원인 분석
* **UI 해석 기준 불일치**:
  - `live_mfe_tab.py`는 Ratchet 단계를 `peak_price` 기반으로 해석하고 있었음.
  - 반면 실제 ExitEngine의 Ratchet 승격은 `peak_price`가 아니라 **현재 수익률(`curr['close']`) 기준**으로 동작하고 있었음.
  - 이 차이 때문에 하단 설명은 “+8.5% 잠금 가능”처럼 보이는데 표의 실제 SL은 아직 그 수준이 아닌, 혼란스러운 상태가 만들어졌음.
* **실제 미작동 원인**:
  - `execution_manager.py`의 Global Kill Switch가 조건 충족 시 `on_timer()` 루프를 즉시 `return`하고 있었음.
  - 이 때문에 신규 BUY만 막힌 것이 아니라, **보유 종목의 ExitEngine 실행 자체가 중단**되어 SL 승격, TimeExit, 보호 청산이 모두 멈췄음.
  - 결과적으로 `SL stale`와 `TimeExit 미실행`은 별개 버그가 아니라 같은 차단 경로에서 같이 발생한 증상으로 정리됨.

### 5. 적용한 수정

#### [app/ai/strategy.py](file:///c:/Users/stone/projects/nocodequant/app/ai/strategy.py) 수정
* 과열 게이트(`HOLD_HEAT`)를 **신규 진입 차단 전용**으로 한정.
* `sess['position'] is not None`인 보유 포지션은 과열 상태여도 ExitEngine까지 계속 진입하도록 변경.
* 목적: Ratchet / Hybrid Trail / TimeExit / 보호 청산 경로가 volume 과열 때문에 멈추지 않도록 보장.

#### [app/ui/dashboard/live_mfe_tab.py](file:///c:/Users/stone/projects/nocodequant/app/ui/dashboard/live_mfe_tab.py) 수정
* Ratchet 단계 계산 기준을 `peak_price`가 아닌 **현재 수익률(`curr_pct`) 기준**으로 ExitEngine과 일치시킴.
* `현재 SL` 표시를 `손실구간` / `본전보호` / `익절보호` / `SL지연`으로 분리해 실제 상태를 더 정확히 드러내도록 수정.
* 계산상 보호선(`Ratchet`, `Hybrid Trail`, `BE Switch`)보다 실제 `sl`이 낮을 때 `SL stale` 경고를 표시하도록 추가.

#### [app/services/managers/execution_manager.py](file:///c:/Users/stone/projects/nocodequant/app/services/managers/execution_manager.py) 수정
* Global Kill Switch 동작을 **루프 전체 중단**에서 **신규 BUY 차단**으로 변경.
* 보유 포지션의 SELL / TimeExit / SL 동기화 / pending_fill 해제는 Kill Switch 상태에서도 계속 실행되도록 보정.
* `portfolio.balance > 0` 조건을 추가하여 예수금 초기화 전 `balance=0` 상태에서 Kill Switch가 허위 발동하는 경로를 방어.
* Kill Switch가 BUY를 막는 경우, 남아 있던 `pending_fill` 세션은 즉시 철회하여 좀비 포지션 의도 상태가 남지 않도록 처리.

### 6. 검증 결과
* `python -m py_compile`
  - `app/services/managers/execution_manager.py`
  - `app/ai/strategy.py`
  - `app/ai/_engine_exit.py`
  - `app/ui/dashboard/live_mfe_tab.py`
  - 모두 통과.
* `ExitEngine().evaluate(...)` 단위 검증으로 TREND 보유 + `bars_held >= time_limit` 조건에서
  - `SELL`
  - `ExitType.TIME_EXIT`
  - `"45봉 시간제한 종료(TREND)"`
  가 반환되는 것을 확인.
* 정리:
  - TimeExit 로직 자체가 고장난 것이 아니라, Kill Switch가 ExitEngine 호출을 통째로 막고 있었던 구조적 문제였음.

### 7. 후속 모니터링 포인트
1. Kill Switch 활성 상태에서도 보유 종목의 `SL`, `TimeExit`, `SELL` 경로가 계속 동작하는지 실전 로그로 재확인.
2. Live MFE/MAE 탭에서 `SL stale` 경고가 실시간으로 해소되는지, 혹은 실제 보호선 미반영 상황을 정확히 드러내는지 점검.
3. `pending_fill` 도입 이후 발생한 2차 파생 이슈를 이번 수정으로 모두 흡수했는지, 다음 장중 세션에서 재관찰.
