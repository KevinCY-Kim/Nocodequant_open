# Reason Code Specification: LOW_CONFIDENCE (저신뢰 차단) [V27.0]

**Category:** Decision Gate
**Logic Type:** Probabilistic Filter

---

## 1. 개요 (Overview)
`LOW_CONFIDENCE`는 지표의 방향성은 뚜렷하나(Score가 높거나 낮음), 그 신호의 질적 신뢰도가 낮아 진입을 유보하는 상태입니다.

## 2. 기술적 정의 (Technical Definition) [V27.0]
*   **소프트 웨이트 (Soft Weight)**: 신뢰도(`Confidence`)는 더 이상 하드 게이트(`MIN_CONFIDENCE`)로 작동하지 않으며, **리스크 블록(Risk Block, 20%)**의 점수를 결정하는 핵심 성분으로 통합되었습니다.
*   **계산 방식**: `Risk Block Score = 0.5 * Confidence + (Regime/Trend Adjustments)`
*   **Confidence 산출**: `강도(Strength) x 안정성(Stability)`의 결합.

## 3. 발생 사유 (Root Causes)
1.  **시장 노이즈(Choppiness)**: 가격이 급격히 방향을 틀며 Stability 점수가 깎인 경우.
2.  **지표 간 모순(Conflicts)**: MA는 상승인데 RSI가 과매도 상태로 급락하는 등 지표들이 서로 다른 말을 할 때 신뢰도가 급감합니다.
3.  **에너지 부족**: 방향은 정해졌으나 거래량이나 변동성 강도가 임계치에 도달하지 못함.

## 4. 대응 가이드 (Response Guide)
*   **전략 준비 중 (Strategy Warming Up)**: 신뢰도가 낮아 `entry_score`가 기준(예: 62점)을 넘지 못할 경우, 시스템은 `DecisionLabel.HOLD_READY` ("전략 준비 중" / "진입 예비") 상태를 유지하며 수급 확장을 기다립니다.
*   **동적 리스크 조절**: 신뢰도가 낮으면 자동으로 `risk_block` 점수가 깎여, 전체 진입 임계값을 넘기 어려워지는 **부드러운 장벽(Soft Barrier)** 효과를 냅니다.

> **💡 핵심 요약 (Key Insight):**
> *"확신 없는 베팅은 도박입니다. 엔진은 통계적으로 뒷받침되지 않는 모든 '우연한 신호'를 여기서 걸러냅니다."*

---
*Document Version: v27.0 (2026-05-25 V27.0 Sync)*
