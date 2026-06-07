# NCQ Money Management Engine (MME) — 아키텍처 문서
> **버전:** V21.3 | **최종 수정:** 2026-04-09 | **대상 파일:** `money_manager.py`, `execution_manager.py`

---

## 1. 개요

MME(Money Management Engine)는 NCQ 알고리즘 트레이딩 시스템의 포지션 사이징 및 포트폴리오 리스크 관리 핵심 모듈입니다.
V21.3에서는 기존의 고정 리스크 모델을 넘어, 시장의 변동성(ATR)에 따라 비중을 동적으로 조절하는 **'Volatility-Targeted Dynamic Sizing'**이 도입되었습니다.

---

## 2. 시스템 아키텍처 (V21.3)

### 2.1 전체 데이터 흐름

```
[전략 신호 발생]
      │
      ▼
ExecutionManager.on_timer()
  ├─ Strategy.get_signal() → atr_vol 추출
  ├─ MME.calculate_qty(atr_vol=...) 호출
  │    ├─ Volatility Scaling (V21.3 추가)
  │    │    └─ scaling = VOL_TARGET_PCT / (atr_vol/price)
  │    ├─ Heat Check (진입 전 차단)
  │    ├─ Risk-Based Sizing
  │    ├─ Cash Cap
  │    └─ Concentration Cap
  └─ TradeSignal 생성
      │
      ▼
OrderManager.send_order()
```

---

## 3. 포지션 사이징 로직 (Mode 3: Volatility-Targeted) [V21.3]

### 3.1 계산 파이프라인

V21.3 기준 수량 계산은 아래 순서로 수행됩니다.

```
① Vol Scaling    = VOL_TARGET_PCT(1.2%) / (ATR / Price)
② Scaling Cap    = min(①, 2.5)                       → 과도한 몰빵 방지 [V21.3]
③ Dynamic Risk % = Base_Risk(1.0%) × ②
④ Risk Amount    = Total_Asset × ③
⑤ Risk/Share     = Price × Effective_SL(stop_loss + 0.1%)
⑥ Raw Qty        = ④ / ⑤
⑦ Cash Cap       = Available_Deposit / Price
⑧ Concentration  = Total_Asset × 0.15 / Price
⑨ Final Qty      = min(⑥, ⑦, ⑧)
⑩ Late Heat Check: Final Qty × Price / Total_Asset × Effective_SL ≤ 잔여 Heat?
```

### 3.2 변동성 타겟팅 시나리오

| 상황 | ATR/Price (변동성) | Scaling Factor | 실질 Risk % | 의도 |
|------|-------------------|----------------|------------|------|
| **평온** | 0.6% | 2.0x | 2.0% | 낮은 변동성 구간에서 비중 확대 |
| **기준** | 1.2% | 1.0x | 1.0% | 기준 변동성 유지 |
| **과열** | 2.4% | 0.5x | 0.5% | 변동성 폭발 시 리스크 자동 축소 |
| **폭풍** | 5.0% | 0.24x | 0.24% | 극심한 변동성 장세에서 생존 최우선 |

---

## 4. Portfolio Heat Control

### 4.1 Regime별 Heat Limit (V21.3)

| Regime | Heat Limit | 전략적 의도 |
|--------|-----------|------------|
| `UPTREND` | **5.0%** | 공격적 분산 투자 허용 |
| `NEUTRAL` | **3.0%** | 기본 리스크 수준 유지 |
| `DOWNTREND` | **1.0%** | 자본 보존 및 시장 관망 |

---

## 5. 리스크 제어 레이어 (The Multi-Layer Shield)

```
┌─────────────────────────────────────────────┐
│  Layer 0: Alpha Targeting (V21.3)           │
│  — Volatility Scaling (ATR 기반 비중 조절)   │
│  — MAX Scaling Cap (2.5x 제한)              │
├─────────────────────────────────────────────┤
│  Layer 1: Heat Check (진입 심사)             │
│  — 포트폴리오 전체 리스크 합계 제한          │
│  — Regime 연동 가변 한도                     │
├─────────────────────────────────────────────┤
│  Layer 2: Cash Cap (유동성 보호)             │
│  — 가용 예수금 버퍼(0.3%) 보호               │
├─────────────────────────────────────────────┤
│  Layer 3: Concentration Cap (집중 리스크)    │
│  — 단일 종목 총자산의 15% 초과 불가          │
└─────────────────────────────────────────────┘
```

---

## 6. 파라미터 레퍼런스 (V21.3)

| 파라미터 | 위치 | 현재값 | 비고 |
|---------|------|-------|------|
| `VOL_TARGET_PCT` | `config_engine.py` | 1.2% | [V21.3] Scaling의 중심점 |
| `MAX_SCALING` | `money_manager.py` | 2.5x | [V21.3] 최대 비중 확대 제한 |
| `order_qty` | `ncq_config.json` | 1.0% | Base Risk % |
| `concentration_cap` | `money_manager.py` | 15% | 종목당 자산 점유 한도 |
| Heat UPTREND | `money_manager.py` | 5.0% | 상승장 리스크 한도 |
| Heat DOWNTREND | `money_manager.py` | 1.0% | 하락장 리스크 한도 |

---
**기록:** V19.7 로드맵에 있던 ATR 기반 손절/비중 조절이 V21.3에서 'Volatility-Targeted Dynamic Sizing'으로 공식 구현 완료되었습니다.
│  — order_manager.py 담당 (MME 외부)         │
└─────────────────────────────────────────────┘
```

---

## 6. 향후 로드맵

### 6.1 ATR 기반 동적 손절 (중기)

현재 `money_manager.py`에 `atr_vol` 파라미터와 로드맵 주석이 이미 준비되어 있습니다.

```python
# 현재 (미구현 상태)
atr_vol: float = 0.0
# risk = base_risk * (target_vol / current_vol)  ← 주석 처리됨
```

**구현 조건:** 전략 유형별 MAE 데이터 30건 이상 누적 후 진행.
**목표 산식:**
```
target_vol = 전략별 기준 변동성 (예: 1.5%)
dynamic_sl = stop_loss_pct × (target_vol / atr_vol)
```
변동성이 낮을 때 손절을 타이트하게, 높을 때 넓게 조정하여
불필요한 손절 빈도를 줄이고 수익 구간 보유 시간을 늘립니다.

### 6.2 is_in_profit Heat Decay 활성화 (단기)

```python
# update_exposure의 Decay 로직은 구현되어 있으나 호출부 미연결
actual_risk = risk_pct * 0.5 if is_in_profit else risk_pct
```

보유 포지션이 수익권 진입 시 Heat를 50%로 감쇄하면
UPTREND에서 추가 진입 여력이 확대됩니다.

**구현 방법:** `state_manager`의 평가손익 체크 후 주기적으로 `update_exposure` 재호출.

### 6.3 전략 유형별 손절 차등화 (장기)

현재 모든 전략이 동일한 `stop_loss_pct = 3.0%`를 사용합니다.
MAE 분석 데이터가 전략 유형별 30건 이상 누적되면:

| 전략 유형 | 현재 | 개선 방향 |
|---------|------|---------|
| 단기 모멘텀 | 3.0% | MAE 중앙값 기반 타이트 조정 |
| 추세 추종 | 3.0% | ATR × 1.5 동적 손절 |
| 역추세 | 3.0% | 별도 손절 파라미터 분리 |

---

## 7. 파라미터 레퍼런스

| 파라미터 | 위치 | 현재값 | 비고 |
|---------|------|-------|------|
| `order_qty` (risk_pct) | `ncq_config.json` | 1.0% | 거래당 허용 손실 |
| `stop_loss_pct` | 전략 프리셋 | 3.0% | 진입가 대비 손절 |
| `slippage_buffer` | `money_manager.py` | 0.1% | effective_sl에 가산 |
| `transaction_buffer` | `money_manager.py` | 0.3% | Cash Cap 계산 시 차감 |
| `concentration_cap` | `money_manager.py` | 15% | V19.7: 20%→15% |
| Heat UPTREND | `money_manager.py` | 5.0% | V19.7: 3%→5% |
| Heat NEUTRAL | `money_manager.py` | 3.0% | V19.7: 2%→3% |
| Heat DOWNTREND | `money_manager.py` | 1.0% | 변경 없음 |
