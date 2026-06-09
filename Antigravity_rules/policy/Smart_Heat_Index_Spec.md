# Smart Heat Index Technical Specification (V14.0)

본 문서는 키움 [0198] 실시간 조회 순위를 넘어서기 위한 **Smart Heat Index(구 Pseudo-0198)**의 기술적 사양과 랭킹 로직을 정의합니다. (Top 5% Institutional Grade)

## 1. 개요 (Overview)
Smart Heat Index는 시장의 자금 흐름(거래대금), 가격 탄력성(변동률), 체결 강도를 실시간으로 가중합산하여 **현재 시장을 주도하는 진짜 종목**을 발굴하는 엔진입니다. V13.9부터는 단순 열기(Heat)를 넘어 **관심의 가속도(Acceleration)**를 포착합니다.

## 2. 엔진 4대 사명 (Engine Mission)
1.  **추세 추종 (Trend Following)**: 검증된 방향성에 탑승.
2.  **모멘텀 기반 (Momentum Driven)**: 가속도가 붙는 임계점 포착.
3.  **리스크 회피 (Risk Averse)**: 과열권 및 폭락주 자동 필터링.
4.  **유동성 필터 (Liquidity Focused)**: 실질적 자본 유입 종목 선별.

## 3. 핵심 아키텍처 (Core Architecture)

### 3.1 Rank-based Normalization (정규화)
$$Score_{norm} = 1 - \frac{Rank - 1}{N - 1}$$

### 3.2 통합 가중치 모델 (Weighted Model)
$$HeatIndex = \alpha \cdot Val + \beta \cdot Mom + \gamma \cdot Acc + \delta \cdot Pow$$
- **거래대금 ($\alpha=0.4$)** / **모멘텀 ($\beta=0.3$)** / **가속도 ($\gamma=0.2$)** / **체결강도 ($\delta=0.1$)**

### 3.3 Institutional Filtering & Adaptive Overlays (V14.0)
1.  **Strict Derivative Filter**: `KODEX`, `TIGER`, `ETN`, `선물`, `인버스`, `레버리지`, `스팩` 등 원천 배제.
2.  **Hard Lightning Cutoff**: 당일 등락률 **-7.0%** 미만 종목은 즉각 퇴출.
3.  **Exponential Momentum Damping**: 음봉 영역 $Multiplier = e^{chg / 5}$ 적용으로 리스크 관리.
4.  **Adaptive Quality Overlays**:
    - **Liquidity Overlay**: 당일 거래대금에 따른 가중치 부여 (300억 미만 시 $Score \times \frac{Val}{300\text{억}}$).
    - **Penny Stock Damping**: 1,000원 미만 종목의 변동성 노이즈 억제 (0.6x~0.8x 감쇄).

## 4. Heat Acceleration (2nd Derivative) 로직
관심도가 급증하는 '임계점 종목' 포착을 위해 2차 미분 로직을 적용합니다.

### 4.1 Discrete Second Derivative
$$a = h_{now} - 2h_{prev} + h_{prev2}$$
- $h$: 합산된 Raw Heat Index.
- **Persistence Gate**: 가속($a > 0$)과 방향($v > 0$)이 모두 양수일 때만 활성화.

### 4.2 Liquidity-Weighted Accel Score (HOT2)
$$HOT2\_Score = rank\_normalize(EMA(a)) \times Val\_Norm$$
- **Liquidity Cutoff**: $Val\_Norm < 0.2$ (하위 20% 유동성) 종목은 가속도 분석에서 제외하여 노이즈 차단.

## 5. UI 가독성 및 하이라이트 표준
1.  **60s Time-based Persistent Highlight**: 신규 기회 포착 시 60초간 🔥 유지.
2.  **EMA Decoupling**: 
    - UI 수치는 **EMA(0.3)**로 부드럽게 표시.
    - 하이라이트(🔥)는 **Raw Spike** 감지로 즉각 반응.

---
*Last Updated: 2026-02-19 | Version: 13.9.0*
