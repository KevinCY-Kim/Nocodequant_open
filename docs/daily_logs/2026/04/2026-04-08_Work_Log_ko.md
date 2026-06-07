# 2026-04-08 작업 일지 (Work Log)

## 0. 개요 (Overview)
- **주요 과제**: 분할 매도 및 부분 체결로 인한 거래 중복 집계 근본 해결 (3-Layer Trade Integrity Hardening)
- **등급**: **L4** (Execution Logic & Data Integrity)
- **참조**: [SSoT-01 금융 원장 무결성](Antigravity_rules/governance/SSoT-01_Financial_Ledger_Integrity_ko.md), [구조 맵 v20.9](Antigravity_rules/architecture/Project_Structure_Map_ko.md)

---

## 1. 주요 변경 사항 (Key Changes)

### 1.1 분할 매도 주문 Identity 통합 로직 (Execution Layer) [V20.9.5]
- **문제**: 한 포지션을 여러 번에 나눠 매도할 경우(Partial Sell Orders), 첫 번째 매도 완료 시점에 GTID와 누적기(Acc)가 삭제되어 이후 매도 건들이 `GTID_UNKNOWN`으로 분리 기록되는 버그 발견.
- **해결**:
    - **GC 지연 (Persistence Preservation)**: `OrderManager`에서 `pending_gtid_map` 및 `fill_accumulator` 제거 조건을 `is_sell_complete`(주문 완료)에서 `is_position_closed`(포지션 완전 청산)로 변경.
    - **Acc 재활용 (Logic Fallback)**: `GTID_UNKNOWN` 상태의 SELL 체결 발생 시, 메모리 내 동일 종목의 기존 SELL 누적기가 있으면 이를 강제 연결하여 하나의 거래(GTID)로 수렴시킴.
    - **기록 시점 최적화**: 매도 기록 발행(`emit`)을 포지션이 0이 되는 최종 시점으로 단일화하여 DB에 파편 데이터가 쌓이는 것을 원천 차단.

### 1.2 DB 레벨 중복 카운트 및 PnL 왜곡 수정 (Persistence Layer) [V20.9.6]
- **문제**: DB에 이미 기록된 과거 분할 매매 데이터로 인해 대시보드 KPI(승률, 총거래수) 및 자산 곡선(Equity Curve)이 과대계상되는 문제.
- **해결**:
    - **순거래 단위 집계 (Deduplicated Query)**: `get_recent_trades`, `get_performance_kpi` 등 모든 조회 쿼리에 `GROUP BY COALESCE(order_id, gtid)` 적용.
    - **물리적 카운트 보정**: 상태바의 총 거래수 계산 시 `COUNT(*)` 대신 `COUNT(DISTINCT COALESCE(order_id, gtid))`를 사용하여 UI와 통계 수치 100% 일치시킴.
    - **자산 곡선 무결성**: `get_equity_curve_data`에서 동일 거래의 PnL이 중복 누적되지 않도록 그룹화 집계 후 누적합 계산 로직 적용.

### 1.3 UI 레이어 2단계 중복 제거 가드 구축 (Presentation Layer) [V20.9.7]
- **문제**: DB와 엔진 레시피 수정만으로는 과거 오염된 데이터나 예외적인 기록 누락을 완벽히 방어하기 어려움.
- **해결**:
    - **Composite Key Dedup**: `TradeLogTabWidget`에 `code + entry_price + mfe + mae + edge` 조합의 2단계 키 생성 로직 도입.
    - **지능형 통합**: 표시 데이터를 렌더링하기 직전 동일 거래로 판별되는 행들을 합산(수량, 수익금 누적)하여 사용자에게 "순거래" 단위로 압축 표시.
    - **실시간 보정**: 대시보드 KPI 카드 업데이트 시, 데둡된 리스트를 기준으로 `total_trades`, `wins`를 재계산하여 표시 신뢰도 극대화.

---

## 2. 영향 분석 및 불변성 확인 (Impact & Invariants)
- **Impacted Modules**: `order_manager.py`, `persistence_manager.py`, `trade_log_tab.py`, `strategy_dashboard.py`
- **Invariant Check**: 
    - "동일한 진입 포지션에서 발생한 모든 매도 체결은 단 하나의 거래 행(Row)으로 집계된다."
    - "분할 청산 시 수익률은 합산 수익금과 진입 원금 총액을 기준으로 재계산된다."
    - "앱 재시작 전 세션 내의 분할 매도는 항상 하나의 GTID를 공유한다."

---

## 3. 검증 결과 (Verification)
- **시뮬레이션 테스트**: 6건의 분할 체결 레코드가 포함된 데이터를 로드한 결과, UI에서 정확히 3건의 순거래로 합산됨을 확인.
- **상태바 정합성**: "분할체결 N건 통합 → 순거래 M건" 메시지 출력을 통해 데이터 정화 과정을 투명하게 공개.
- **PnL 합산 검증**: 개별 체결의 pnl_amount 합계와 통합된 거래의 pnl_amount가 일치함 확인.

---

## 4. 향후 과제 (Next Steps)

---
*Last Updated: 2026-04-08 17:55*
*Created By Antigravity (Powered by Advanced Agentic Coding)*
