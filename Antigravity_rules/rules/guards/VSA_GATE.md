# Reason Code Specification: VSA_GATE (수급 확증 필터) [V27.0]

**Category:** Execution Gate / Penalty Guard
**Logic Type:** Volume-Price Confirmation

---

## 1. 개요 (Overview)
`VSA_GATE`는 기술적 지표(Score)가 매수 진입 임계치를 넘었음에도 불구하고, 실제 '수급(Volume)'의 확증이 부족할 때 작동하는 필터링 메커니즘입니다. 현대 NoCodeQuant의 VSA 필터는 **WEAK_VSA 패널티(Penalty)**와 **VOLUME 전략형 하드 필터**의 이중 구조로 구성됩니다.

## 2. 기술적 정의 (Technical Definition) [V27.0]

### 2.1 공통 패널티 게이트: `WEAK_VSA`
*   **트리거 조건:**
    $$ \text{volume\_ratio} < \text{VSA\_WARN\_RATIO} $$
    *   `volume_ratio`: `실거래량 / 20봉 평균` (1.0이 평균).
    *   `VSA_WARN_RATIO`: `ACTIVE_CONFIG.get("VSA_WARN_RATIO", 0.8)` (기본 0.8x).
*   **동작**: 조건 만족 시 `active_penalties` 리스트에 `"WEAK_VSA"` 패널티가 등재되며, 점수 계산 및 UI 상태 표시기에 감점으로 작용합니다.

### 2.2 VOLUME 전략 전용 하드 필터
*   **트리거 조건:**
    $$ \text{volume\_ratio} < \text{VOLUME\_MIN\_VOL\_RATIO} $$
    *   `VOLUME_MIN_VOL_RATIO`: `ACTIVE_CONFIG.get("VOLUME_MIN_VOL_RATIO", 1.5)` (기본 1.5x).
*   **작동 구간**: VOLUME 전략형으로 판정된 신호 진입 시점에 활성화.
*   **Action**: `Force HOLD` (Decision: `DecisionLabel.HOLD_BLOCK`, Reason: "매수 보류: [VOLUME] 수급 부족").

### 2.3 음봉 필터 (Soft Barrier)
*   현재 캔들이 음봉(`is_bull_candle` == False)일 경우 `price_trigger` 점수를 0.0점으로 매핑하여 전체 진입 점수(`entry_score`)가 자연스럽게 깎이도록 유도합니다.

## 3. 방어 목적 (Defense Goal)
*   **허수 주문 방어**: 거래량 없이 가격만 살짝 올리는 '유인책'을 차단합니다.
*   **투매 중 반등 필터**: 급락 중 거래량 없이 발생하는 미세한 기술적 반등에서 손실을 입는 '데드 캣 바운스'를 방어합니다.
*   **수급 원칙 준수**: 가격은 일시적으로 유도할 수 있어도 거래량은 조작하기 어렵다는 시장 원리를 시스템화한 것입니다.

## 4. 운영 가이드 (Operational Guide)
*   **모니터링**: 대시보드에 수급 부족 관련 메시지가 표시되면 수급이 부족하다는 신호입니다.
*   **조정**: 너무 보수적이라고 느껴질 경우 `VSA_WARN_RATIO`를 0.7x 등으로 조정할 수 있으나, 가짜 신호 노출 확률이 높아집니다.

---
*Document Version: v27.0 (2026-05-25 V27.0 Sync)*
*Constitution Category: L3 Logic Guard*
