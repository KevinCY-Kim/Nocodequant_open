# 2026-05-01 작업 로그 (Work Log)

## 📋 작업 요약
*   **주제**: MFE/MAE 지표 정합성 개선 및 수익률 계산 엔진 하드닝 (Full-Stack Patch)
*   **등급**: **L3 (핵심 로직 및 UI 표현 수정)**
*   **상태**: ✅ 완료

---

## 🔍 문제 진단 (Forensic Analysis)
1.  **MFE 음수 발생**: `OrderManager`가 청산 시점에 `max_price`를 신호가(`entry_price`)와만 비교하면서, 실제 체결가가 신호가보다 낮은 경우(유리한 슬리피지) MFE가 음수로 기록되는 논리적 결함 발견.
2.  **수익률 과대계상**: `PersistenceManager` SQL 집계 시 분모를 `MIN(entry_price) * SUM(qty)`로 계산하여, 분할 매수 시 평균 단가가 높아졌음에도 낮은 초기 진입가 기준으로 수익률이 부풀려지는 현상 확인.
3.  **데이터 오염**: `StateManager.sync_positions`에서 `or` 연산자 사용 시 `max_price=0`인 경우를 누락된 데이터로 오판하여 서버의 평균 단가로 덮어씌우는 버그 확인.
4.  **UI 가독성 저하**: 다크 테마 배경에서 무채색(` #eee`) 처리가 배경과 구분되지 않아 지표가 소실된 것처럼 보이는 시각적 문제 확인.

---

## 🛠️ 수정 내역 (Changes)

### 1. `app/services/managers/order_manager.py` (DB 기록 레이어)
*   SELL 분기 시작 시점에 `state_manager.get_position()`을 호출하여 **실시간 추적된 `max_price/min_price`를 직접 참조**하도록 수정.
*   `position_meta_cache`에 의존하던 기존 방식(MFE 음수의 근본 원인) 제거.

### 2. `app/services/managers/state_manager.py` (실시간 추적 레이어)
*   `sync_positions` 내 극값 보존 로직 수정.
*   `is not None and > 0` 조건을 통해 명시적 분기 처리를 하여 실시간 추적 데이터가 서버 평균가에 의해 초기화되는 것 방지.

### 3. `app/services/managers/persistence_manager.py` (조회 및 분석 레이어)
*   **Python 가드**: `_enrich_trade_data`에 최후 방어선 클램핑 추가 (`mx = max(mx, ep)`, `mn = min(mn, ep)`).
*   **SQL 가드**: 성과 요약 및 최근 거래 내역 쿼리에 MFE 음수 방지용 `CASE` 문 적용.
*   **계산 엔진 교정**: `pnl_ratio` 계산 분모를 `SUM(entry_price * qty)`로 변경하여 **가중 평균 기반의 실투자금 대비 수익률** 산출로 정정.

### 4. `app/ui/dashboard/trade_log_tab.py` (표현 레이어)
*   **출력 보정**: DisplayRole에서 MFE($\ge 0$), MAE($\le 0$) 불변성 강제 클램핑 적용.
*   **색상 체계 개선**: 3단계 분기 로직 도입.
    *   **이상값(Anomaly)**: `#888`(회색)으로 표시하여 데이터 이상 유무 식별 가능하게 함.
    *   **중립(Neutral)**: `#555`(어두운 회색)를 적용하여 배경과 구분하면서도 '수익 구간 미도달' 상태 명시.
    *   **강조(Action)**: 수익 구간 도달 시 **`#ff5252`(빨강)**, 손실 구간 발생 시 **`#44aaff`(파랑)** 가독성 강화.

---

## 🧪 결과 확인 (Validation)
*   MFE가 음수로 표시되던 종목들이 모두 0.00% 이상의 정상 수치로 복구됨.
*   UI에서 지표가 소실되어 보이던 현상이 해결되고, 프리미엄 다크 테마에 맞는 색상 대비가 확보됨.
*   분할 매수된 거래의 수익률이 실제 투자 원금 대비 정확하게 계산됨을 검증 완료.

---
*Created By: Antigravity AI*
