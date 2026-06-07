# 📝 차기 작업 계획: 국면(Regime) 정합성 해결 및 CHAOS 활성화

> **작성일**: 2026-04-07  
> **상태**: 계획 수립 (Pending Approval)  
> **주제**: 전략 엔진과 성과보드 간의 국면 분류 체계 동기화 및 누락된 CHAOS 로직 실전 적용

---

## 1. 현황 진단 (Current Discrepancy)

현재 NoCodeQuant 시스템에는 시장 국면을 바라보는 세 가지 시선이 충돌하고 있습니다.

| 구분 | 사용 명칭 | 문제점 |
| :--- | :--- | :--- |
| **전략 내부 (Level 1)** | `TREND`, `RANGE` | 방향성(상/하) 정보가 누락됨. |
| **외부 출력 (Level 2)** | `UPTREND`, `DOWNTREND`, `RANGE` | 엔진 내부의 Time Exit 로직과 변수명이 불일치함. |
| **설계 문서 (L3 가이드)** | `UPTREND`, `DOWNTREND`, `RANGE`, `CHAOS` | `CHAOS` 국면은 문서상에만 존재하고 실제 제어 로직에 미반영됨. |

---

## 2. 해결 목표 (Objectives)

1.  **명칭 통합**: 모든 엔진 제어 변수와 UI 표시 명칭을 `UPTREND / DOWNTREND / RANGE / CHAOS` 4대 체계로 통일.
2.  **CHAOS 제어권 복구**: ADX는 높으나 방향성이 없는 `CHAOS` 상황에서 설계 문서대로 **'진입은 엄격하게(72점)', '탈출은 빠르게(10봉)'** 로직 강제.
3.  **로직 전진 배치**: 국면 판정 세분화 로직을 매매 결정 직전으로 이동하여 모든 결정(진입/청산)이 최신 국면 정보를 즉각 반영하도록 개선.

---

## 3. 상세 실행 로직 (Planned Steps)

### Phase 1: 국면 판정 전진 배치 (`strategy.py`)
- `_get_signal_internal` 상단에서 `UPTREND`, `DOWNTREND`, `CHAOS`를 조기에 판정.
- 특히 `TREND` 국면 진입 시 `is_uptrend`와 `is_downtrend`가 모두 False인 경우를 `CHAOS`로 명확히 마킹.

### Phase 2: 제어 맵(Control Map) 동기화
- **Time Exit**: `{"CHAOS": 10, "RANGE": 20, "TREND": 40}` 맵이 세분화된 국면을 직접 인식하도록 수정.
- **Entry Threshold**: `CHAOS` 국면 시 `ENTRY_THRESHOLD_CHAOS`(72점) 필터가 실제 작동하도록 보장.

### Phase 3: 성과보드(UI) 대응
- `StrategyDashboard` 및 웹 대시보드에서 `CHAOS` 라벨링과 전용 배지(🌀) 표시 지원.

---

## 4. 기대 효과
- **자본 보호**: 불확실한 과열 장세(`CHAOS`)에서 불필요한 잦은 매매를 줄이고, 진입하더라도 짧게 치고 나옴으로써 MDD 방어.
- **인지 정합성**: 개발자(로그 확인)와 사용자(대시보드 확인)가 동일한 국면 언어를 공유.

---
*Created By Antigravity (Powered by Advanced Agentic Coding)*
