# Decision Flow Architecture (의사결정 흐름 아키텍처) [V27.0]

## 1. Purpose (목적)
본 문서는 NoCodeQuant의 **지표-신호 (Indicator-to-Signal)** 의사결정 파이프라인의 **실제 구현 상태 (Implementation Truth)**를 정의합니다. 시스템은 `StrategyManager` 및 하위 모듈(`_engine_entry.py`, `_engine_exit.py`, `_engine_penalty.py`) 내부에서 지표 계산, 정규화, 확률론적 가이딩을 거쳐 최종 매매 신호를 생성하며, 생성된 신호는 전략 유형별로 분류되어 영속화됩니다.

---

## 2. Decision Pipeline (결정 파이프라인)

### Phase 1: Tiered Data Ingestion (계층형 데이터 수집)
1.  **틱 수신 (Tick Reception)**: 키움 Open API 또는 모의 엔진 (Mock Engine)으로부터 체결 데이터 수신.
2.  **티어 기반 감시 (Tiered Surveillance)**:
    -   **Tier 1 (Active)**: 보유 종목 및 승격(Promotion) 종목. 1초 루프에서 실시간 감시.
    -   **Tier 2 (Background)**: 전 종목 대상 백그라운드 스캔. 점수 > 50점 시 Tier 1으로 승격.
3.  **캔들 집계 (Candle Aggregation)**: `CandleManager`가 틱 데이터를 OHLCV 캔들로 변환. In-place Update 적용으로 메모리 부하 제로화.
4.  **웜업 가드 (Warmup Guard)**: 지표 계산을 위해 최소 **50개**의 캔들이 수집될 때까지 대기 (`ACTIVE_CONFIG.get("WARMUP_MIN_BARS", 50)`).

### Phase 2: Indicator Calculation (지표 계산)
1.  **이동평균 (MA)**: 단기/장기 이동평균선 산출 및 polyfit을 대체하는 지수 이동 평균(EMA) 기반의 Vectorized Slope 산출.
2.  **상대강도지수 (RSI)**: 가격의 상대 강도 산출.
3.  **볼린저 밴드 (BB)**: 표준편차 기반의 가격 채널 산출.
4.  **변동성 측정 (ATR)**: 시장의 평균 변동폭 산출.
5.  **ADX (Regime Detection)**: 14일 ADX 및 ADX Slope를 통한 시장 강도 산출.

### Phase 3: Probabilistic Normalization (확률론적 정규화)
1.  **자가 정규화 파이프라인 (Self-Normalizing Pipeline)**: 각 지표의 Raw 값을 최근 100개 캔들의 표준편차로 정규화.
2.  **Tanh Saturation**: 정규화된 값을 `tanh` 함수를 통해 -100 ~ 100점 사이로 포화 (Saturation).
    -   의도: 모호한 상태를 배제하고 선명한 신호 상태 (Green/Red)를 확보.

### Phase 4: Confidence & Scoring (신뢰도 및 종합 점수 산출)
1.  **강도 채널 (Strength Channel)**: 지표 점수의 크기를 통해 추세 강도 측정 (ADX 기반).
2.  **안정성 채널 (Stability Channel)**: 지표 점수의 변화율 (Stability)과 방향 전환 페널티를 계산하여 안정성 측정.
3.  **신뢰도 가중 (Confidence Weighting)**: `강도 x 안정성`을 결합하여 최종 신뢰도 산출.
4.  **최종 점수 (Final Score)**: `평균 지표 점수 x 신뢰도`를 통해 0~100점 사이의 대시보드 점수 생성 (50점 기준).
5.  **소프트 진입 점수 (Soft Entry Score)**: 기술 (40%), 수급 (20%), 가격 (20%), 리스크 (20%) 블록을 결합한 종합 진입 점수 산출 (`_calc_entry_score`).
6.  **국면 전이 (Regime Adjustments)**: 하락장(DOWNTREND) 판정 시 리스크 블록 -20점 즉각 부여.
7.  **히스테리시스 가드**: MA20/MA60 이격 **0.3%** 가드를 통해 국면 채터링 방지.

### Phase 5: Gating & Decision (제어 및 최종 결정)
1.  **진입 트리거 임계값 (Entry Threshold)**:
    -   **국면별 기본값**: TREND: 49, RANGE: 57, CHAOS: 64, DOWNTREND: 59.
    -   **전략별 기본값**: BREAKOUT: 62, VOLUME: 62, TREND: 62, REVERSAL: 62, MIXED: 62.
    -   국면별 임계값과 전략별 임계값 중 더 엄격한(큰) 임계값을 선택 적용하며, 수급 폭발 시 -5, 확신(S+) 시 -3, 확신(Low) 시 +5의 Confidence Adjustment 적용.
2.  **수급/반등 필터 (VSA)**:
    -   **WEAK_VSA 패널티**: `volume_ratio < VSA_WARN_RATIO` (기본 0.8x)일 때 active_penalties에 등재되어 감점 요인으로 작동.
    -   **VOLUME 전략 하드 필터**: VOLUME 전략 진입 시 `volume_ratio < VOLUME_MIN_VOL_RATIO` (기본 1.5x)이면 매수 보류 (`HOLD_BLOCK`).
    -   **음봉 필터 (Soft Barrier)**: 음봉(`is_bull_candle` == False)일 경우 `price_trigger` 점수를 0.0점으로 매핑하여 전체 진입 점수(`entry_score`)가 자연스럽게 깎이도록 유도.
3.  **Hard Safety Gate (Overheat Block)**:
    -   `vol_ratio > VOL_RATIO_CAP` (기본 2.2x) 초과 시 진입 원천 차단 (`DecisionLabel.HOLD_HEAT`).
    -   **주도주 과열 완화 [V24.0]**: UPTREND/TREND 국면이고 신뢰도가 75% 이상일 때 상한선을 3.0x로 완화하여 주도주 편입 허용.

### Phase 6: Regime Adaptation (국면 가변 관리)
1.  **국면 판정 (Regime Detection)**: ADX 기반 임계값(추세 진입 25, 비추세 복귀 22) 및 0.3% 이격 가드를 통해 시장 국면 판단.
2.  **동적 출구 튜닝 (Dynamic Exit Tuning)**: 국면에 맞춰 TP/SL/Trailing 파라미터를 즉시 업데이트 (예: 추세 UPTREND 전환 시 BE SL 상향 조정).

### Phase 7: Strategy Classification (매수 전략 유형화) - [V27.0]
매수 결정 직후, 네 가지 핵심 블록의 점수 조합을 **병렬 평가 (Parallel Evaluation)**하여 전략의 성격을 정의합니다.

1.  **판정 라벨 (Labels)**:
    -   **돌파 (BREAKOUT)**: 가격 돌파 점수 >= 100 & 기술 점수 >= 60.
    -   **수급 (VOLUME)**: 수급 점수 >= 80 & 가격 점수 >= 70 & 기술 점수 >= 50.
    -   **추세 (TREND) [V24.5]**: `regime` in ('TREND', 'UPTREND') & core >= 65.0 & price >= 40.0, 그리고 **`_is_real_trend`** (이평선 실제로 우상향 `ma_slope > 0.02` 및 추세강도 `adx >= 20.0`) 충족 시에만 한정.
    -   **반전 (REVERSAL)**: `RSI < 35` & 가격 점수 >= 40 & 하락/횡보 국면.
2.  **예외 및 복합 판정**:
    -   **복합 (MIXED)**: 2개 이상의 조건 동시 충족 또는 엄격 분류 미충족 시의 fallback.
    -   **미분류 (UNDEFINED)**: 어떠한 조건도 충족하지 못할 경우의 기본 라벨.

### Phase 8: Post-Entry & Data Layer - [V27.0]
체결 이후의 성과 추적 및 데이터 영속화를 담당합니다.

1.  **MFE/MAE 정밀 추적 (Performance Tracking)**:
    -   보유 중(`qty > 0`) 최고/최저가를 실시간 추적하여 최대 수익/낙폭 기록. `on_timer` 1초 루프와 양방향 동기화되어 거래 빈도가 낮은 종목도 누락 없이 정밀 추적.
2.  **DB 이중화 및 영속성 (Dual-Decoupling)**:
    -   **`trades.db`**: 성과 요약, 개별 거래 내역 (MFE/MAE 포함).
    -   **`forensics.db`**: 모든 시그널 및 시스템 이벤트 대용량 로그.
3.  **데이터 무결성 (Integrity)**:
    -   **1-Order-1-Row**: 매도 완료 시점에만 성과를 기록하여 Ghost Row 방지.
    -   **Metadata Persistence**: 부분 매매 및 비동기 체결 시에도 `pending_signal_meta` 캐시(디스크 파일 보존)를 통해 `strategy_type`, `regime` 등 메타데이터를 100% 유지.

---

## 3. Implementation Status (구현 현황) [V27.0]

| Component | Status | Code Location | Note |
| :--- | :--- | :--- | :--- |
| Strategy Labeling | ✅ Active | `app/ai/strategy.py` | V24.5 TREND 세분화 및 MIXED 대응 |
| Dual-DB Layer | ✅ Active | `app/services/managers/persistence_manager.py` | [V16.0] Batch Copy & Decoupling |
| MFE/MAE Tracking | ✅ Active | `app/services/managers/state_manager.py` | [V25.1] on_timer 및 양방향 동기화 |
| PnL SSoT Sync | ✅ Active | `app/ai/engine.py` | [V14.1] opt10077 기반 정합성 확보 |
| Dashboard Data Sync| ✅ Active | `app/ui/dashboard/` | [V26.6] KPI 정규화 및 수치 정합성 패치 |
| Regime Detection | ✅ Active | `app/ai/strategy.py` | ADX (25/22) + 0.3% 히스테리시스 |

---

## 4. Final Doctrine (최종 원칙)
*   **신호는 감정보다 통계에 근거해야 한다.**
*   **모든 아키텍처 변경은 AI 작업 헌장에 따른 정합성 검증을 거쳐야 한다.**

---
*Document Version: v27.0 (Aligns with Engine V27.0)*미분류 (UNDEFINED)**: 어떠한 조건도 충족하지 못할 경우의 기본 라벨.
3.  **성과 연동**: 판정된 유형은 UI 배지로 표시되며, `v_strategy_performance` 뷰를 통해 전략별 승률, 익/손절 평균 수익률, 손익비(Profit Factor) 분석의 핵심 지표가 됨. [V19.6]

### Phase 8: Post-Entry & Data Layer - [V19.4]
체결 이후의 성과 추적 및 데이터 영속화를 담당합니다.

1.  **성과 지표 추적 (Performance Tracking)**:
    -   **MFE/MAE**: 보유 중(`qty > 0`) 최고/최저가를 실시간 추적하여 최대 수익/낙폭 기록.
    -   **Edge Ratio**: `MFE / |MAE|`를 통해 전략의 효율성(Edge) 평가.
    -   **Holding Time**: 최초 체결 시점(`entry_time`)부터 청산 시점까지의 시간 기록.
2.  **DB 이중화 (Dual-Decoupling)**:
    -   **`trades.db`**: 성과 요약, 개별 거래 내역 (MFE/MAE 포함, 수십 KB).
    -   **`forensics.db`**: 모든 시그널 및 시스템 이벤트 대용량 로그.
3.  **데이터 무결성 (Integrity) [V19.6]**:
    -   **1-Order-1-Row**: 매도 완료 시점에만 성과를 기록하여 Ghost Row 방지.
    -   **Metadata Persistence**: 부분 매매 및 비동기 체결 시에도 `pending_signal_meta` 캐시를 통해 `strategy_type`, `regime` 등 메타데이터를 100% 유지.
4.  **마이그레이션**: 500행 단위 Batch Record Copy로 DB Lock 방지.

---

## 3. Implementation Status (구현 현황) [V19.6]

| Component | Status | Code Location | Note |
| :--- | :--- | :--- | :--- |
| Strategy Labeling | ✅ Active | `app/ai/strategy.py` | V19.0 4대 유형 및 MIXED 대응 |
| Dual-DB Layer | ✅ Active | `app/services/managers/persistence_manager.py` | [V16.0] Batch Copy & Decoupling |
| MFE/MAE Tracking | ✅ Active | `app/services/managers/state_manager.py` | [V19.4] 실시간 최고/최저가 추적 |
| PnL SSoT Sync | ✅ Active | `app/ai/engine.py` | [V14.1] opt10077 기반 정합성 확보 |
| Dashboard Data Sync| ✅ Active | `app/ui/dashboard/` | [V19.6] KPI 정규화 및 보안 강화 |
| Regime Detection | ✅ Active | `app/ai/strategy.py` | ADX (22) + 0.3% 히스테리시스 |

---

## 4. Final Doctrine (최종 원칙)
*   **신호는 감정보다 통계에 근거해야 한다.**
*   **모든 아키텍처 변경은 AI 작업 헌장에 따른 정합성 검증을 거쳐야 한다.**

---
*Document Version: v3.0 (Aligns with Engine V19.4 & Stability Patch V16.0)*
