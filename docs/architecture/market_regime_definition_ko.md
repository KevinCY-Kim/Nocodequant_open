# 📊 시장 국면 판정 체계 (Market Regime Definition)

> [!CAUTION]
> **[GUARD LOGIC-01] 전략 로직 무결성 보호 구역**
> 본 문서는 시스템의 '시장 국면 판정'에 대한 최상위 설계 도안입니다.
> 본 문서의 수치를 변경할 경우, 반드시 `app/ai/strategy.py`의 `_update_regime_state` 로직과 동기화되어야 합니다.

가이드 버전: v22.1 (RSI Inversion Final)  
최종 업데이트: 2026-04-21

NoCodeQuant 엔진은 ADX 및 DMI 지표를 핵심으로 하여 시장을 4가지 국면으로 분류하며, V22.1에서는 **국면별 조화로운 RSI 점수 산출 로직**이 통합되었습니다.

---

## 1. 국면 판정 로직 (Core Logic)

### Stage 1: 추세 vs 비추세 (ADX 히스테리시스 & Slope Assist)
ADX 지표의 수치와 이평선 기울기(`ma_slope`)를 기준으로 판정합니다.

| 국면 분류 | 판정 기준 | 기술적 상태 |
| :--- | :--- | :--- |
| **TREND Family** | ADX > 25 진입 OR (ADX > 15 AND `ma_slope` > 0.2) | 에너지를 특정 방향으로 분출 |
| **RANGE Family** | ADX < 22 복귀 | 박스권 내 진동 |

*   **Slope Assist [V21.9]:** ADX 지체 현상을 보완하기 위해 이평선 기울기가 0.2 이상이면 `UPTREND`로 조기 판정합니다.

### Stage 2: 세부 국면 통합 [V22.1 Update]

#### 📈 UPTREND (상승추세)
-   **판정:** `TREND Family` 상태에서 `plus_di >= minus_di` (매수 세력 우위)
-   **V22 개선:** 기존 +5pt 격차에서 `plus_di >= minus_di`로 완화하여 추세 진입 기회비용을 최소화했습니다.

#### 📉 DOWNTREND (하락추세)
-   **판정:** `TREND Family` 상태에서 `minus_di > plus_di` 

#### 🌀 CHAOS (혼조세)
-   **판정:** `RANGE Family` 상태에서 `vol_ratio > 1.8` (방향 없는 고변동성)

#### 🧱 RANGE (횡보/안정)
-   상기 조건을 제외한 정상적인 박스권 상태.

---

## 2. [CORE-V22] 국면별 RSI 스코어 역전 (RSI Inversion)

국면에 따라 RSI를 해석하는 철학을 완전히 분리하여 역추세 팀킬을 방어합니다.

- **UPTREND:** `(RSI - 50) * 2` → **모멘텀 추종**. 강한 RSI일수록 상승 추세를 타고 있다고 판단. (80 이상 과열 시 가점 50% 제한)
- **RANGE/DOWNTREND:** `(50 - RSI) * 2` → **평균 회귀**. 과매도 구간일수록 반등 가능성을 높게 평가.

---

## 3. 국면별 적응형 제어 (Adaptive Control)

| 국면 | 진입 임계값 | Time Exit | 트레일링 배수 | RSI 해석 |
| :--- | :---: | :---: | :---: | :---: |
| **UPTREND** | **50점** | 40봉(연장) | 4.5x~ | 모멘텀(+) |
| **DOWNTREND** | **60점** | 40봉 | 1.5x | 평균회귀(-) |
| **RANGE** | **58점** | 20봉 | 1.5x | 평균회귀(-) |
| **CHAOS** | **65점** | 10봉 | 1.5x | 평균회귀(-) |

*임계값은 Exploration Mode 설정에 따라 실시간으로 변동될 수 있습니다. (기본값 설정 기준)*

---

## 4. 설계 결정 사항 (Internal Policy)

1.  **DOWNTREND 이중 패널티**: 하락 국면에서는 임계값을 높이고 리스크 점수에서 페널티(-20점)를 부여하여 낙하칼 매수를 억제합니다.
2.  **RSI Inversion Policy**: 추세장에서는 RSI가 높아야 사고, 횡보/하락장에서는 RSI가 낮아야 사는 "환경 적응형" 스코어링을 통해 지표 왜곡을 막습니다.
3.  **Panic Shield 우선순위**: 4.5x~1.5x의 비대칭 트레일링보다 Panic Shield(MA240 아래 시 1.0x)의 거부권이 기술적으로 우세하게 작동합니다.
