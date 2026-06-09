# Audit-Grade 운영 표준 (Audit-Grade Operational Standard)

> 출처: ArmoraAI Operational_Policies §4 + Audit_and_Compliance_Rules → NCQ 도메인 적응
> Tier: 2 (Policy)

---

## 1. 핵심 원칙 (Core Principle)

> "단순 이익 추구가 아닌, **설명 가능한 수익(Explainable Profit)**만이 유효하다."

운영 환경에서 발생하는 모든 매매 이벤트는 **사후 재현(Audit Replay)**이 가능해야 한다.

---

## 2. 재현성 요건 (Reproducibility Requirements)

### 2.1 의사결정 맥락 보존 (Decision Context Preservation)

모든 매매 결정(진입, 대기, 거절)에 대해 다음 맥락을 함께 기록해야 한다:

| 항목 | 설명 | NCQ 구현 |
|:-----|:-----|:---------|
| **Signal Snapshot** | MA, RSI, BB 지표 값 + 산출 점수 + 신뢰도 | `ExecutionManager.trigger_snapshot()` |
| **Runtime Config** | 당시 적용된 전략 파라미터 스냅샷 | `runtime_config` 참조값 기록 |
| **Market Regime** | 적용된 국면 (TREND/RANGE) | `market_regime` 기록 |
| **Rejection Reason** | 거부 시 차단 사유 코드 | Reason Code Architecture 참조 |

### 2.2 Rich Context 표준

> "로그는 그 자체로 상황을 재구성할 수 있어야 한다."

단순한 `Action(BUY/SELL)` 로그는 불충분하다. ForensicDashboard에서 **"Why(왜)"**를 즉시 시각화할 수 있어야 한다.

---

## 3. 수동 개입 기록 (Manual Intervention Audit)

사용자가 수동으로 개입(강제 청산, 엔진 정지/재개)한 경우:

- EventBus 이벤트 `type` 또는 `memo` 필드에 **`MANUAL`, `FORCE`, `CMD`** 표식이 필수.
- 자동 매매와 수동 개입을 구분할 수 없는 로그 체계는 **감사 위반**으로 간주한다.

---

## 4. 실현 손익 즉시 반영 (Immediate PnL Realization)

- 포지션 청산 즉시 실현 손익(`realized_pnl`)이 Balance에 반영되어야 한다.
- 메모리 상태와 DB 로그 간 동기화가 보장되어야 한다.

---

## 5. 선물 도메인 제거 확인

- ✅ HMM/OVS 전략 참조 없음
- ✅ GTID → NCQ EventBus 이벤트 체계로 대치
- ✅ Brain/Hand/Eye 아키텍처 참조 없음
- ✅ 수익/포지션 용어 주식 현물 기준
