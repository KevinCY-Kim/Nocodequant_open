# Parameter Invariants (파라미터 불변 조건) [V27.0]

## 1. Purpose (목적)
본 문서는 NoCodeQuant 시스템 운용에 사용되는 핵심 파라미터들의 정의와 해당 파라미터가 코드 내에서 어떻게 강제되는지(Invariants)를 기술합니다. 본 문서는 [AI 작업 헌장 v1.8](file:///C:/Users/stone/projects/nocodequant/Antigravity_rules/governance/AI_Work_Constitution_and_Reference_Map.md)의 **L2(전략 파라미터)** 및 **L3(로직)** 거버넌스를 따르며, 코드 베이스의 `app/core/config_engine.py` (ACTIVE_CONFIG)를 유일한 진실 공급원(SSoT)으로 삼아 100% 일치하도록 강제합니다.

---

## 2. Strategy Engine Parameters (`strategy.py` & `config_engine.py`)

### 2.1 Display Score Mapping (대시보드 점수)
| Parameter | Range / Value | Reference | Description |
| :--- | :---: | :--- | :--- |
| `Neutral Base` | **50** | `strategy.py` | 매수/매도 균형점 (강력한 관망 기본값) |
| `Buy Threshold` | **65 ~ 100** | `ACTIVE_CONFIG` | 매수 신호 발생 가능 구간. (Global Gate: **60**) |
| `Sell Threshold` | **0 ~ 35** | `ACTIVE_CONFIG` | 매도 신호 발생 가능 구간 |

### 2.2 Normalization Invariants (지표 정규화)
| Parameter | Value | Constraint | Description |
| :--- | :---: | :---: | :--- |
| `RSI_NORM_WINDOW` | **100** | `Min 50` | RSI 표준편차 계산을 위한 롤링 윈도우 크기. |
| `MA_NORM_WINDOW` | **50** | `Min 30` | 이평선 표준편차 계산을 위한 롤링 윈도우 크기. |
| `BB_NORM_WINDOW` | **50** | `Min 30` | 볼린저 밴드 표준편차 계산을 위한 롤링 윈도우 크기. |
| `SOFT_CAP_SIGMA` | **10.0** | `Fixed` | 극단적 이상치 차단을 위한 10-sigma 캡핑. |
| `TANH_SENS` | **50.0** | `Fixed` | Tanh 함수의 정규화 민감도 기울기 조정값. |

### 2.3 Confidence Channels (신뢰도 채널)
| Parameter | Value | Constraint | Description |
| :--- | :---: | :---: | :--- |
| `CONF_EMA_ALPHA` | **0.2** | `Fixed` | 강도(Strength) EMA 반영 비율. |
| `STB_EMA_ALPHA` | **0.1** | `Fixed` | 안정성(Stability) EMA 반영 비율 (더 부드럽게). |
| `FLIP_PENALTY` | **1.5** | `Fixed` | 지표 방향 전환(Flip) 시 적용되는 안정성 감점 계수. |

### 2.4 Institutional Guards (기관급 제어 가드)
| Parameter | Value | Constraint | Description |
| :--- | :---: | :---: | :--- |
| `ADX_TREND_THRESHOLD` | **25** | `Min 20` | Trend 국면 진입 임계값 (설정값: 25). |
| `ADX_RANGE_THRESHOLD` | **22** | `Max 25` | Range 국면 회귀 임계값 (Hysteresis SSoT: 22). |
| `VOLUME_SPIKE_RATIO` | **2.0** | `Min 1.0` | 진짜 수급 폭발 확증을 위한 실거래량/SMA20 임계 배수. |
| `VOL_SPIKE_RATIO` (Legacy) | **1.15** | `Fixed` | 변동성/대금 보조 지표용 VSA 가중치 계수. |
| `GLOBAL_BEAR_MA` | **240** | `Fixed` | 하락장 판단을 위한 HTF 이동평균선 기간. |
| `VOL_RATIO_CAP` | **2.2** | `Min 1.5` | 과열 거래량(BB폭 비율) 진입 차단 임계값 (BB 감산 High). |
| `VOL_RATIO_WARN` | **1.8** | `< CAP` | BB 점수 경미 감산 시작 임계값 및 CHAOS 국면 진입선. |
| `VSA_WARN_RATIO` | **0.8** | `Fixed` | WEAK_VSA 패널티 적용 임계값. |
| `VOL_RATIO_PENALTY_HIGH`| **0.7** | `0.5 ~ 1.0` | VOL_RATIO_CAP 초과 시 BB 감산 멀티플라이어 계수. |
| `VOL_RATIO_PENALTY_MID` | **0.9** | `0.5 ~ 1.0` | VOL_RATIO_WARN 초과 시 BB 감산 멀티플라이어 계수. |
| `WARMUP_MIN_BARS` | **50** | `Min 30` | Warmup Gate 최소 수집 봉 수 (Signal_Gating_Rules §2.1). |
| `WARMUP_GUARD_BARS` | **2** | `Fixed` | [V20.9] 진입 초기 휩소 방지를 위한 보호 기간 (봉 단위). |

### 2.5 Soft Entry Weights & Thresholds (소프트 진입 임계값)
| Parameter | Value | Constraint | Description |
| :--- | :---: | :---: | :--- |
| `ENTRY_THRESHOLD` | **48** | `50~80` | 국면별/전략별 임계값 적용 전 최종 Fallback 기본 임계값. |
| `ENTRY_THRESHOLD_TREND`| **49** | `50~75` | TREND 국면 진입 임계값 (완화). |
| `ENTRY_THRESHOLD_DOWNTREND`| **59** | `55~80` | DOWNTREND 국면 진입 임계값. |
| `ENTRY_THRESHOLD_RANGE`| **57** | `55~80` | RANGE 국면 진입 임계값 (강화). |
| `ENTRY_THRESHOLD_CHAOS`| **64** | `60~85` | CHAOS 국면 진입 임계값. |
| `ENTRY_W_CORE` | **40** | `Sum=100` | Core Technical 가중치 (MA + RSI + BB). |
| `ENTRY_W_VOLUME` | **20** | `Sum=100` | Volume Quality 가중치. |
| `ENTRY_W_PRICE` | **20** | `Sum=100` | Price Trigger 가중치 (고가 돌파 등). |
| `ENTRY_W_RISK` | **20** | `Sum=100` | Risk Block 가중치. |
| `DOWNTREND_RISK_PENALTY`| **-20**| `Fixed` | 하락장 감지 시 리스크 블록에 즉각 부여되는 감점. |

### 2.6 Strategy-First Entry Thresholds (전략 유형별 독립 임계값) [V23.0]
> **결정 규칙**: `final_entry_th = max(ENTRY_THRESHOLD_<REGIME>, ENTRY_TH_<STRATEGY>)` (더 엄격한 기준 적용)

| Parameter | Value | Constraint | Description |
| :--- | :---: | :---: | :--- |
| `ENTRY_TH_BREAKOUT` | **55** | `≥ 45` | [V28.3 P1] Trend-Only(100) 해제 — 5/28 원값 50 대비 +5 보수 복원. |
| `ENTRY_TH_VOLUME` | **54** | `≥ 45` | [V28.4] 잠금 해제 (5/28 원값 49, +5 보수) — §2.6.2 재정의 게이트와 동시 적용. |
| `ENTRY_TH_TREND` | **60** | `≥ 50` | 추세 추종 매매전략 진입 임계 점수. (노이즈 진입 방지 복원) |
| `ENTRY_TH_REVERSAL` | **100** | `≥ 45` | [V27.9] 전략 잠금 유지 (원값 54) — 섀도 측정 후 해제 검토. |
| `ENTRY_TH_MIXED` | **65** | `≥ 50` | [V28.3 P1] Trend-Only(100) 해제 — 5/28 원값 61 대비 +4 보수 복원. |

### 2.6.1 진입 게이트 보정 파라미터 (Entry Gate Calibration) [V28.3]
> 근거: 진입게이트 비교분석 (2026-06-11) — 음성 필터 교집합의 '꼭지 수렴(역선택)' 보정.

| Parameter | Value | Constraint | Description |
| :--- | :---: | :---: | :--- |
| `ENTRY_MATURITY_PULLBACK_EXEMPT` | **True** | `Bool` | [P3] 진행봉 음봉/보합 또는 세션 고점 되돌림 시 Bar-Maturity 대기 면제. |
| `ENTRY_PULLBACK_RETRACE_PCT` | **0.01** | `0.005~0.03` | [P3] 성숙도 면제용 세션 고점 대비 최소 되돌림 비율. |
| `ENTRY_LOW_LOCATION_BONUS` | **5** | `0~10, 0=OFF` | [P4] 저점 위치(VWAP 근접/고점 되돌림) 시 entry_th 완화 폭. |
| `ENTRY_VWAP_PROXIMITY_PCT` | **0.01** | `0.005~0.02` | [P4] 세션 VWAP 근접 판정 허용 범위 (±). |
| `ENTRY_HIGH_RETRACE_PCT` | **0.02** | `0.01~0.05` | [P4] 가점용 세션 고점 대비 최소 되돌림 비율. |
| `MORNING_VOLUME_GUARD_MULT` | **1.1** | `0=OFF` | [P2] 당일 봉 3개 이상 절사평균 구간에서만 평가 (1~2개 구간 스킵 — 시초봉 분모 결함 수정). [P5] `GUARD_SHADOW_MODE` 준수. |
| `ENTRY_MIN_VALUE_5M` | **50,000,000** | `0=OFF` | [P5] `GUARD_SHADOW_MODE` 준수로 통일 (기존: 무조건 차단). |

### 2.6.2 VOLUME 전략 재정의 (지속 수급 + 비클라이맥스) [V28.4]
> **원칙**: "단일봉 스파이크 추격" → "매집 확인 후 비클라이맥스 위치 진입"으로 전략 정의 자체를 교체.
> 근거: trades.db VOLUME 428건 — 승률 26%, -0.47%/건, MFE +0.94%/MAE -0.98% 대칭(엣지 0), 65%가 0~3봉 사망.
> 분석: `docs/analysis/2026-06-11_NonTrend_Strategy_Structural_Analysis_ko.md` §3.1
> **주의**: 아래 게이트는 L3 리스크 가드가 아니라 전략 '정의'이므로 `ENTRY_TH`와 동급으로 `GUARD_SHADOW_MODE` **미적용**(무조건 차단). P5(신규 가드 섀도 통일) 위배 아님.

| Parameter | Value | Constraint | Description |
| :--- | :---: | :---: | :--- |
| `VOLUME_REDEFINE_ENABLED` | **True** | `True/False` | False 시 구정의(`VOLUME_MIN_VOL_RATIO` 1.5 가드) + TP1 면제로 복귀. `ENTRY_TH_VOLUME=100`과 함께 V28.3 완전 롤백. |
| `VOLUME_SUSTAIN_BARS` | **3** | `2~5` | 지속 수급 판정용 직전 완성봉 관찰 개수. |
| `VOLUME_SUSTAIN_MIN_RATIO` | **1.2** | `1.0~1.5` | 지속 수급 인정 vol_ratio 하한 (매집 봉 판정). |
| `VOLUME_SUSTAIN_MIN_COUNT` | **2** | `1~N` | N봉 중 최소 충족 봉 수 미달 시 차단. |
| `VOLUME_CLIMAX_RATIO` | **2.0** | `1.5~3.0` | 현재봉이 이 배수 이상 분출 중이면 클라이맥스 후보. |
| `VOLUME_CLIMAX_HIGH_PROX` | **0.998** | `Fixed` | 클라이맥스 봉이 세션 고점 × 0.998 이상에 위치하면 추격 차단. |
| `VOLUME_TP1_PCT` | **0.009** | `0=OFF` | 소수확 TP1 (+0.9% 부분익절) — MFE 분포(+0.94%) 정합. TP1 면제(주도주 취급) 해제. 잔량은 BE 버퍼/트레일 보호. |

---

## 3. Money Management & Execution Invariants (자금 및 주문 집행)

### 3.1 Money Management Engine (MME) [V14.9.2]
| Parameter | Value | Constraint | Description |
| :--- | :---: | :---: | :--- |
| `MAX_POS_EQUITY` | **20.0%** | `Hard` | 단일 종목의 최대 노출 비중 (Position Cap). |
| `MAX_TOTAL_HEAT` | **2.0%** | `Dynamic` | 포트폴리오 전체 리스크 합계 한도 (강세장 3.0%, 약세장 1.0%). |
| `HTS_TAX` | **0.20%** | `Fixed` | 키움 OpenAPI 및 HTS 실제 증권거래세 (매도 시). |
| `HTS_COMMISSION` | **0.015%** | `Fixed` | 키움 실질 위탁수수료 (양방향 각각 부과). |
| `SLIPPAGE_BUFFER` | **0.1%** | `Fixed` | 리스크 계산 시 강제 합산되는 보수적 슬리피지 버퍼. |
| `TRANSACTION_COST_BUFFER`| **0.5%**| `Fixed` | 실질 매매 연산 시 적용되는 수수료+슬리피지 종합 비용 버퍼. |

### 3.2 Framework & UI Constraints (`main_window.py`)
-   **지원 타임프레임**: 1, 5, 10, 15, 30, 60, 120 (분봉)
-   **초기화 규칙**: 타임프레임 변경 시 모든 `CandleManager`와 `StrategyManager` 상태를 **즉시 초기화**하고 `ignore_next_signal` 플래그를 작동해야 함.
-   **UI Refresh Rates**: 틱당 약 10% 확률로 UI 스캐너 제한(`UI_SCANNER_REFRESH_RATE = 0.1`).
-   **Chart Constraints**: 최대 **200개**의 캔들 데이터 포인트 유지 (메모리 제한).
-   **Docks Initial Heights**: `[230, 310, 260]` (독 위젯 세로 비율).

---

## 4. Exit Governance Invariants (청산 및 출구 전략)

### 4.1 Asymmetric Exit & R-Multiple (비대칭 출구 4계) [V20.9.1]
| Parameter | Value | Constraint | Description |
| :--- | :---: | :---: | :--- |
| `R_MIN_FLOOR_PCT` | **0.005** | `Fixed` | R-value 계산 시 분모가 0이 되어 발산하는 현상 방지 최소 하한 (0.5%). |
| `R_MULT_TRAIL_START` | **1.5** | `≥ 1.0` | 1.5R 도달 시 트레일링 스탑 추적 개시. |
| `R_MULT_BE_SWITCH` | **2.2** | `≥ 2.0` | 2.2R 도달 시 무위험 지대(BE Switch) 작동 (손절선 상향). |
| `BE_R_BUFFER` | **0.2** | `Fixed` | BE 전환 시 손절선을 `진입가 + 0.2R`로 전진 배치. |
| `TRAIL_WIDE_MULT` | **4.5** | `Fixed` | 1.5R ~ 2.2R 구간: 4.5x ATR 넓은 버퍼로 추세 보존. |
| `TRAIL_MID_MULT` | **3.0** | `Fixed` | 2.2R ~ 3.2R 구간: 3.0x ATR 중간 수익 보호. |
| `TRAIL_TIGHT_R` | **3.2** | `Fixed` | 3.2R 도달 시 타이트닝 스탑 구간 전환. |
| `TRAIL_TIGHT_MULT` | **2.0** | `Fixed` | 3.2R ~ 5.2R 구간: 2.0x ATR 타이트한 수익 잠금. |
| `TRAIL_ULTRA_TIGHT_R` | **5.2** | `Fixed` | 5.2R 도달 시 최종 수익 확보 울트라 타이트닝 전환. |
| `TRAIL_ULTRA_TIGHT_MULT`| **1.5** | `Fixed` | 5.2R+ 구간: 1.5x ATR 최종 수익 강력 잠금. |
| `TP1_R_MULT` | **2.0** | `Fixed` | 역추세/횡보 전략(`REVERSAL`/`RANGE`): 2R 도달 시 50% 분할 익절집행. |
| `SMART_TIME_MIN_R` | **0.5** | `Fixed` | 시간 연장 혜택 부여를 위한 최소 수익 조건 (0.5R 미만 시 연장 없이 청산). |
| `EARLY_CUT_R` | **-0.5** | `Fixed` | 진입 초기(2봉 이내) 손실이 -0.5R에 달하면 즉시 가설 기각 매도. |
| `STOP_LOSS_DOOMSDAY` | **0.04** | `Fixed` | 전략 신호와 관계없이 계좌 자본을 지키는 4% 절대 강제 손절선. |

### 4.2 Time-Based Exit (시간 청산) [V23.3]
| Parameter | Value (봉 수) | 시간 환산 | Description |
| :--- | :---: | :---: | :--- |
| `TIME_EXIT_BARS_TREND` | **30** | 150분 | 추세장(TREND/UPTREND) 최대 보유 시간. |
| `TIME_EXIT_BARS_RANGE` | **15** | 75분 | 횡보장(RANGE) 최대 보유 시간. |
| `TIME_EXIT_BARS_DOWNTREND`| **6** | 30분 | 하락장(DOWNTREND) 최대 보유 시간 (빠른 탈출). |
| `TIME_EXIT_BARS_CHAOS` | **6** | 30분 | 혼조세(CHAOS) 최대 보유 시간. |

### 4.3 Inactivity Guard (무풍 조기 청산) [V23.3]
| Parameter | Value | Constraint | Description |
| :--- | :---: | :---: | :--- |
| `INACTIVITY_GUARD_ENABLED` | **True** | `True/False` | 무풍 감시 및 조기 청산 기능 활성화 여부. |
| `INACTIVITY_GUARD_BARS` | **10** | `≥ 5` | 진입 후 관찰 기간 (10봉 = 50분). |
| `INACTIVITY_GUARD_THRESHOLD`| **0.005** | `Fixed` | 10봉 내 수익률이 ±0.5% 이내인 경우 모멘텀 부재로 청산. |
| `INACTIVITY_GUARD_MIN_BARS` | **3** | `Fixed` | 성급한 청산을 방지하기 위한 최소 관찰 보장 기간 (3봉). |

### 4.4 Flash Crash Protection (급락 갭 즉시 청산) [V23.3]
| Parameter | Value | Constraint | Description |
| :--- | :---: | :---: | :--- |
| `FLASH_CRASH_ENABLED` | **True** | `True/False` | 급락 갭 보호 기능 활성화 여부. |
| `FLASH_CRASH_DRAWDOWN_LIMIT`| **0.06** | `Fixed` | 고점(peak) 대비 6% 이상 급락 시 웜업가드 무관 즉시 청산. |
| `FLASH_CRASH_MIN_PEAK` | **0.03** | `Fixed` | 최소 고점 수익률이 3% 이상일 때만 노이즈 방지를 위해 발동. |

### 4.4.1 BREAKOUT Fail Cut (돌파 실패 구조 청산) [V28.4]
> **원칙**: 진입 근거(직전 봉 고가 돌파)가 무효화되면 % 손절을 기다리지 않고 즉시 청산한다.
> 근거: trades.db BREAKOUT 43건 — NORMAL +0.18% vs STOP_LOSS 12건 -2.78% (이중 분포).
> 분석: `docs/analysis/2026-06-11_NonTrend_Strategy_Structural_Analysis_ko.md` §3.3

| Parameter | Value | Constraint | Description |
| :--- | :---: | :---: | :--- |
| `BREAKOUT_FAIL_CUT_ENABLED` | **True** | `True/False` | False 시 V28.3 동작 완전 복귀 (롤백 스위치). |
| `BREAKOUT_FAIL_CUT_BUFFER` | **0.003** | `0.001 ~ 0.01` | 돌파 레벨 하회 허용 버퍼(리테스트 노이즈 흡수). 현재가 < 레벨×(1-버퍼) 시 청산. |

- 진입 시 직전 봉 고가를 **실제로 돌파한 경우에만** `breakout_level` 무장. Fallback BREAKOUT(바디 돌파/과매도 완화 경로)은 미무장 — 즉시 오발동 방지.
- WarmupGuard **무관** 발동 (Flash Crash 동급) — 표적이 0~3봉 조기 사망 구간이기 때문.

### 4.5 Hybrid Profit-Based Trailing Stop (이익 연계 트레일링) [V23.3]
> **공식**: `giveback = min(PEAK_TRAIL_<STRATEGY>, base_giveback * (profit^TRAIL_NONLINEAR_EXP))`

| Parameter | Value (반납율) | Strategy | Description |
| :--- | :---: | :--- | :--- |
| `PEAK_TRAIL_BREAKOUT` | **0.25** | `BREAKOUT` | 돌파 매매: 대시세 호흡을 위해 수익의 최대 25% 반납 허용. |
| `PEAK_TRAIL_TREND` | **0.25** | `TREND` | 추세 추종: 긴 호흡 유지를 위해 최대 25% 반납 허용. |
| `PEAK_TRAIL_VOLUME` | **0.20** | `VOLUME` | 수급 주도: 중간 호흡으로 20% 반납 허용. |
| `PEAK_TRAIL_REVERSAL` | **0.15** | `REVERSAL` | 역추세: 짧은 호흡, 빠른 이탈을 위해 15% 반납만 허용. |
| `PEAK_TRAIL_MIXED` | **0.20** | `MIXED` | 복합 조건: 20% 반납 허용. |
| `TRAIL_NONLINEAR_EXP` | **0.7** | `Fixed` | 비선형 감쇄 곡선 지수 (초반 완화, 후반 급격 조임). |
| `TRAIL_MIN_GIVEBACK` | **0.10** | `Fixed` | 수익이 아무리 확대되어도 최소 10% 반납선 보장. |
| `PROFIT_TRAIL_MIN_LOCK` | **0.015** | `Fixed` | 수익 발생 시 최소 확보 수익 절대 보장선 (1.5%). |
| `PROFIT_TRAIL_ATR_TRIGGER_MULT`|**1.5** | `Fixed` | 적응형 트리거 발동 ATR% 곱셈 계수. |
| `PROFIT_TRAIL_ATR_TRIGGER_MIN`| **0.02** | `Fixed` | ATR%가 작아도 최소 2.0% 수익 도달 시 적응형 트레일링 작동. |

### 4.6 SL Ratchet (수익 단계별 손절선 래칫) [V23.2]
| Trigger Level (수익률) | Stop Loss Line (손절선) | RATCHET_ENABLED | Description |
| :---: | :---: | :---: | :--- |
| **+1.5%** (`L1_TRIGGER`) | **본전 (0.0%)** (`L1_SL`) | **True** | 진입 후 +1.5% 도달 시 손절선을 진입가로 고정. |
| **+3.0%** (`L2_TRIGGER`) | **+1.0%** (`L2_SL`) | **True** | 진입 후 +3.0% 도달 시 손절선을 +1.0%로 상향. |
| **+5.0%** (`L3_TRIGGER`) | **+2.5%** (`L3_SL`) | **True** | 진입 후 +5.0% 도달 시 손절선을 +2.5%로 상향. |
| **+8.0%** (`L4_TRIGGER`) | **+5.0%** (`L4_SL`) | **True** | 진입 후 +8.0% 도달 시 손절선을 +5.0%로 상향. |
| **+12.0%** (`L5_TRIGGER`)| **+8.5%** (`L5_SL`) | **True** | 진입 후 +12.0% 도달 시 손절선을 +8.5%로 상향. |

---

## 5. Market Heat & Leader Filter Invariants (주도주 및 수급 필터)

### 5.1 Leader Filter Policy (동전주 및 수급 노이즈 제어) [V25.0]
| Parameter | Value | Constraint | Description |
| :--- | :---: | :---: | :--- |
| `PENNY_FILTER_MODE` | `"hard_exclude"` | `damped/allow` | 동전주 처리 정책. **운영 모드는 반드시 `hard_exclude` 고수.** |
| `PENNY_MIN_PRICE` | **1000** | `≥ 500` | 동전주 판정 기준가 하한 (원). 헌장 4조 [5]항 규정값. |
| `PENNY_SEVERE_PRICE` | **500** | `< MIN_PRICE` | `damped` 모드 전용: 2중 감쇄를 적용하는 심각 가격 하한. |
| `PENNY_DAMPING_RATIO` | **0.8** | `0.5 ~ 1.0` | `damped` 모드: 1000원 미만 점수 감쇄율. |
| `PENNY_SEVERE_RATIO` | **0.6** | `0.3 ~ 0.8` | `damped` 모드: 500원 미만 점수 감쇄율. |
| `RUNRATE_WARMUP_ABS_MIN`| **5.0억** | `≥ 1.0` | 개장 초반(20분 이내) 절대 최소 거래대금. (런레이트 예외 기간의 방어선) |
| `CROSS_BOOST_MAX` | **1.25** | `1.0 ~ 1.5` | 다중 조건식 포착 시 점수 부스트 절대 상한. |
| `CROSS_BOOST_COEF` | **0.15** | `0.05 ~ 0.25` | `log1p` 비선형 증폭 계수. |

### 5.2 Hybrid Run-Rate Gate (HOT 유동성 런레이트) [V23.6]
| Parameter | Value | Constraint | Description |
| :--- | :---: | :---: | :--- |
| `RUNRATE_WARMUP_MINUTES`| **20.0** | `Fixed` | 개장 후 런레이트 필터를 적용하지 않는 웜업 시간(분). |
| `RUNRATE_CLAMP_MINUTES` | **20.0** | `Fixed` | 개장 초반 런레이트 추정값 과대평가 방지 최소 경과분모 클램프. |
| `RUNRATE_SESSION_MINUTES`| **390.0** | `Fixed` | 정규 거래 시간 (6.5시간 × 60 = 390분). |
| `RUNRATE_TARGET_BILLION`| **300.0** | `≥ 100.0` | 당일 거래대금 최종 도달 투영 목표치 (300억). |
| `RUNRATE_ABS_MIN_BILLION`| **20.0** | `≥ 10.0` | 일반 거래시간 중 즉시 탈락을 결정하는 절대 최소 거래대금 (20억). |
| `RUNRATE_CAP_MULTIPLIER`| **10.0** | `Fixed` | 런레이트 추정값 최대 상한 캡 배수 (현재 거래대금 × 10). |
| `RUNRATE_DISCOUNT_SLOPE`| **0.5** | `Fixed` | 실시간 가속도(surge)에 의한 할인 경사 계수. |
| `RUNRATE_MAX_DISCOUNT` | **0.5** | `Fixed` | surge 최대 포화 시 적용하는 목표 거래대금 최대 할인율 (50%). |

### 5.3 Strategy-Specific Filters (전략 필터 분류 규격) [V24.5]
| Parameter | Value | Strategy | Description |
| :--- | :---: | :---: | :--- |
| `CLASSIFY_BREAKOUT_PRICE`| **100.0** | `BREAKOUT` | 돌파 전략: price_trigger 기준치 (점수). |
| `CLASSIFY_BREAKOUT_CORE` | **60.0** | `BREAKOUT` | 돌파 전략: core technical 기준치. |
| `BREAKOUT_MA20_GAP_CAP` | **0.05** | `BREAKOUT` | MA20 이격률 5% 초과 시 돌파 진입 즉각 차단. |
| `CLASSIFY_VOLUME_VOL` | **80.0** | `VOLUME` | 수급 전략: volume_ratio 기준치. |
| `CLASSIFY_VOLUME_PRICE` | **70.0** | `VOLUME` | 수급 전략: price_trigger 기준치. |
| `CLASSIFY_VOLUME_CORE` | **50.0** | `VOLUME` | 수급 전략: core technical 기준치. |
| `VOLUME_MIN_VOL_RATIO` | **1.5** | `VOLUME` | 수급 전략 진입 시 최소 volume_ratio 비율 하한. |
| `CLASSIFY_TREND_CORE` | **65.0** | `TREND` | 추세 전략: core technical 기준치 (엄격 분류). |
| `CLASSIFY_TREND_PRICE` | **40.0** | `TREND` | 추세 전략: price_trigger 기준치. |
| `CLASSIFY_TREND_MA_SLOPE`| **0.02** | `TREND` | 추세 전략: 최소 우상향 마진 이평선 기울기 하한. |
| `CLASSIFY_TREND_ADX` | **20.0** | `TREND` | 추세 전략: 최소 ADX 기준값 (추세 확인). |
| `TREND_MA20_GAP_CAP` | **0.01** | `TREND` | MA20 이격률 1% 이내일 때만 MTF GuardReward 조건부 면제. |
| `CLASSIFY_REVERSAL_RSI` | **40.0** | `REVERSAL` | 역추세 전략: RSI 기준 (완화). |
| `CLASSIFY_REVERSAL_PRICE`| **30.0** | `REVERSAL` | 역추세 전략: price_trigger 기준 (완화). |

---

## 6. Final Statement
모든 파라미터 변경은 `config.py` 또는 UI의 스핀박스를 통해서만 이루어져야 하며, 소스 코드 레벨의 리터럴(Literal) 수정은 원칙적으로 금지합니다. **L3 이상의 로직 변경은 반드시 사전 설계안(Design Note)을 동반해야 합니다.**
