# CHANGELOG — MME V19.7
> **적용일:** 2025 | **관련 파일:** `money_manager.py`, `order_manager.py`

---

## [V19.7] MME Heat 정합성 수정 및 파라미터 최적화

### 배경 및 문제 인식

실거래 환경 점검 중 MME의 Heat Control 로직이 **설계 의도와 다르게 동작**하고 있음을 발견.
크게 세 가지 구조적 결함이 확인되었으며, 이를 단계적으로 수정함.

---

### Bug Fix 1 — Heat 미등록 (Critical)

**파일:** `order_manager.py`

**문제:**
BUY 체결 완료 시 `money_manager.update_exposure()`가 호출되지 않아
`current_exposures`가 항상 비어 있었음.
결과적으로 `current_heat = 0`이 되어 **Heat Control이 사실상 작동하지 않는 상태**였음.

```python
# 기존: SELL 청산 시 Heat 해제만 존재
if is_position_closed:
    money_manager.update_exposure(code, 0)
# BUY 완료 시 Heat 등록 코드 없음 → current_heat 항상 0
```

**수정:**
BUY 체결 완료(`is_buy_complete`) 시점에 실제 체결가 기반으로 Heat 계산 후 등록.
`stop_loss_pct`는 OCX 콜백에서 접근 불가하므로 `send_order()` 시점에 `pending_signal_meta`에 미리 캐싱.

```python
# send_order() — stop_loss_pct 캐싱 추가
self.pending_signal_meta[code] = {
    ...
    "stop_loss_pct": getattr(signal, 'stop_loss_pct', 3.0),  # [V19.7 추가]
}

# on_fill_event() Step 5 — BUY 완료 시 Heat 등록
if is_buy_complete:
    stop_loss_pct = signal_meta_for_heat.get('stop_loss_pct', 3.0)
    effective_sl = stop_loss_pct + 0.1
    position_value = avg_fill_price * acc["cum_qty"]
    actual_risk_pct = (position_value / total_asset) * (effective_sl / 100.0) * 100.0
    money_manager.update_exposure(code, actual_risk_pct, False)
```

---

### Bug Fix 2 — Heat 조기 차단 (Early Blocking)

**파일:** `money_manager.py`

**문제:**
Heat Check가 수량 계산 파이프라인 초반(Raw Qty 기준)에서 수행되어,
실제 진입 리스크(Concentration Cap 적용 후)보다 과대한 값으로 심사되고 있었음.

```
기존 흐름:
  Heat Check (risk_pct=1.0% 기준 심사)  ← 여기서 차단
      ↓
  Raw Qty 계산 (32% 비중)
      ↓
  Concentration Cap 적용 (15%로 강제 축소)
  → 실제 리스크: 0.465%

문제: Heat 여력이 0.465% 남아 있어도 1.0%가 안 되면 차단
     UPTREND 3.0% 환경에서 실질적으로 2종목에서 조기 차단됨
```

**수정:**
모든 Cap(Cash, Concentration) 적용 완료 후 최종 수량의 실제 리스크로 재심사.

```
수정 흐름:
  Raw Qty → Cash Cap → Concentration Cap → 최종 Qty 확정
      ↓
  Late Heat Check: 최종 Qty 기준 actual_risk_pct 계산
      ↓
  current_heat + actual_risk_pct > limit? → 차단 or 통과
```

**효과:**
UPTREND 5.0% 환경에서 0.465%씩 최대 10종목까지 진입 가능.
(기존: 2종목에서 차단)

---

### 파라미터 변경

**파일:** `money_manager.py`

#### Heat Limit 상향

| Regime | 변경 전 | 변경 후 | 이유 |
|--------|--------|--------|------|
| `UPTREND` | 3.0% | **5.0%** | 상승장 분산 확대, Early Blocking 해소 |
| `NEUTRAL` | 2.0% | **3.0%** | 중립 구간 보유 여력 확보 |
| `DOWNTREND` | 1.0% | **1.0%** | 보수적 스탠스 유지 |

```python
# 변경 전
heat_limits = {"UPTREND": 3.0, "NEUTRAL": 2.0, "DOWNTREND": 1.0}

# 변경 후
heat_limits = {"UPTREND": 5.0, "NEUTRAL": 3.0, "DOWNTREND": 1.0}
```

#### Concentration Cap 하향

| 항목 | 변경 전 | 변경 후 | 이유 |
|------|--------|--------|------|
| 종목당 최대 비중 | 20% | **15%** | 종목당 리스크 축소, Heat 상향과 균형 |

```python
# 변경 전
max_pos_value = total_asset * 0.20

# 변경 후
max_pos_value = total_asset * 0.15
```

**변경 전후 비교 (1억 자산):**

| 항목 | 변경 전 | 변경 후 |
|------|--------|--------|
| 종목당 포지션 | 20,000,000원 | 15,000,000원 |
| 종목당 실제 리스크 | 0.62% | **0.465%** |
| UPTREND 최대 종목 수 | 사실상 2종목 | **~10종목** |
| DOWNTREND 최대 종목 수 | 1종목 | **~2종목** |

---

### 영향 범위

| 파일 | 변경 유형 | 위치 |
|------|---------|------|
| `order_manager.py` | Bug Fix | `send_order()` L184~196 |
| `order_manager.py` | Bug Fix | `on_fill_event()` Step 5 (L417~442) |
| `money_manager.py` | Bug Fix + Param | `calculate_qty()` Heat Check 순서 변경 |
| `money_manager.py` | Param | `heat_limits` UPTREND/NEUTRAL 상향 |
| `money_manager.py` | Param | `Concentration Cap` 20%→15% |

---

### 향후 예정 (Roadmap)

| 항목 | 조건 | 예상 버전 |
|------|------|---------|
| `is_in_profit` Heat Decay 활성화 | state_manager 평가손익 연동 | V19.8 |
| ATR 기반 동적 손절 연결 | 전략별 MAE 30건 이상 누적 | V20.x |
| 전략 유형별 stop_loss_pct 차등화 | MAE 분석 완료 후 | V20.x |
