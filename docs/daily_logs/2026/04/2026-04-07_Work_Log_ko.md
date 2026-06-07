# 2026-04-07 작업 일지 (Work Log)

## 0. 개요 (Overview)
- **주요 과제**: 거래 수량/수익금 중복 합산 해결 및 MFE/MAE 지표 정밀화 (Data integrity Hardening)
- **등급**: **L4** (Database & Algorithm Precision)
- **참조**: [AI 작업 헌장 v1.7](Antigravity_rules/governance/AI_Work_Constitution_and_Reference_Map.md), [구조 맵 v20.9](Antigravity_rules/architecture/Project_Structure_Map_ko.md)

---

## 1. 주요 변경 사항 (Key Changes)

### 1.1 부분 체결 수량/수익금 왜곡 해결 (1주문 1행 원칙) [V20.8.8]
- **문제**: '빛과전자(069540)' 등에서 부분 체결 시마다 중간 스냅샷이 DB에 중복 INSERT되어 수량이 8,337주(실제 3,019주)로 뻥튀기되는 현상 발생.
- **해결**: 
    - **UPSERT 도입**: `PersistenceManager`의 저장 로직을 `INSERT OR REPLACE` (order_id 기준)로 개편하여 동일 주문은 항상 단일 행으로 유지.
    - **UNIQUE 인덱스**: DB 수준에서 `idx_trades_order_id_unique`를 강제하여 물리적 중복 차단.
    - **집계 공식 교정**: 대시보드 뷰 및 KPI 쿼리에서 `SUM(qty)`을 `MAX(qty)`로 변경하여 기존 파편 데이터로부터도 정확한 최종값 추출 보장.
    - **데이터 클렌징**: `db_cleansing_v20.8.8.py` 실행으로 기존 88건의 중복 파편 데이터를 정리하고 DB 무결성 확보.

### 1.2 MFE/MAE 지표 산출 정밀화 (진입 전 오염 차단) [V20.8.9]
- **문제**: '남선알미늄(008350)' 등에서 진입 전 아침 급등 가격이 `max_price`에 포함되어, 손절 중임에도 MFE가 +7.73%로 비정상 표시되는 현상.
- **해결**: 
    - **진입 시점 강제 리셋**: `StateManager`에서 신규 `BUY` 체결 시 기존 감시(Surveillance) 데이터와 상관없이 `max_price`/`min_price`를 진입가로 즉시 초기화.
    - **타임스탬프 정밀 가드**: `RealtimeQuote.timestamp`와 `entry_time`을 직접 비교하여, 진입 시각 이전에 발생한 틱데이터가 비동기로 늦게 도착하더라도 지표 갱신을 차단하는 2중 방어선 구축.

### 1.3 당일 청산 종목 Edge 지표 정규화 (Zero-Drawdown) [V20.8.10]
- **문제**: '대한해운' 등 진입 후 손실 구간(MAE)이 0.00%인 '우상향 완벽 거래'의 경우, 0으로 나누는 수학적 예외를 피하려다가 Edge 지표가 오히려 0.00으로 표시되는 역전 현상.
- **해결**: 
    - **분모 스무딩(Smoothing)**: 개별 거래 처리(`_enrich_trade_data`) 및 통계 쿼리(`v_strategy_performance`)에서 `MAE`가 0일 경우 최소 단위(0.01%)로 치환하여 `MAX(ABS(mae), 0.01)` 분모 적용.
    - 완벽한 거래에 대해 막대한 보상치(High Edge Score)를 부여하면서 무한대 확산을 방지하는 정밀 타격 완료.

### 1.4 실시간 실현손익 SSoT 통합 (Event-to-Ledger) [V20.9]
- **문제**: 메인 화면 실시간 P/L(-224,305)과 전략성과보드 누적 수익금(-2,152,075) 간의 심각한 데이터 불일치(Event Drift) 발생.
- **해결**: 
    - **원장 기반 집계(SSoT)**: `PersistenceManager.get_daily_pnl()`을 `forensics.db`(이벤트 로그) 기반에서 `trades.db`(매매 원장) 기반으로 전면 개편.
    - **로직 단일화**: UI가 매매 신호를 받아 직접 수익금을 누적하는 구조를 폐쇄하고, 모든 UI가 DB에 기록된 '최종 진실(Final Truth)'만을 읽도록 수정.
    - **Reconciliation**: 앱 시작 시 `trades.db`의 수치를 기준으로 `StateManager`를 자동 보정하는 로직 도입 (`Delta: 0.00` 확인).

### 1.5 전략 로직 및 금융 원장 무결성 방어막 구축 (3-Layer Shield)
- **목적**: 향후 작업이나 AI에 의한 핵심 알고리즘 및 데이터 정합성 훼손 방지.
- **해결**: 
    - **거버넌스 확립**: `SSoT-01_Financial_Ledger_Integrity_ko.md`를 생성하여 "진실은 오직 원장에만 존재한다"는 시스템 헌법 선포.
    - **문서 불변성 가드**: `market_regime_definition_ko.md` 및 `strategy_classification_ko.md`에 `[GUARD LOGIC-01, 02]` 헤더를 삽입하여 설계 도안의 임의 수정 차단.
    - **코드 불변성 가드**: `persistence_manager.py`, `main_window.py`, `strategy.py`의 핵심 로직부에 `[GUARD]` 주석을 삽입하여 구현부와 설계 문서의 결속 강화.
    - **거버넌스 맵핑**: `Project_Structure_Map_ko.md`에 해당 구역을 **[L4_최상위보호구역]**으로 격상 등록.

### 1.6 전략 엔진 매도(Exit) 로직 정밀 수선 및 가드 구축 [V20.9.1]
- **문제**: Exit 로직 내 3가지 버그(Dead Code, 손절 우선순위 역전, 보유봉수 계산 오류) 발견.
- **해결**: 
    - **손절 우선순위 정상화**: `SL_TOUCH` 체크를 익절보다 먼저 수행하도록 로직 순서 재배치 (자본보호 최우선).
    - **봉 지수(Bar Index) 도입**: `bars_held` 계산 시 타임스탬프 대신 인덱스 차이값을 사용하여 장기 보유 종목의 오작동 차단.
    - **매직 넘버 제거**: 하드코딩된 목표가 배수(1.15)를 `ACTIVE_CONFIG` 설정 값으로 이관.
    - **방어막 삽입**: 수선된 로직 상단에 `[GUARD LOGIC-03]`을 삽입하여 아키텍처적 일관성 확보.

---

## 2. 영향 분석 및 불변성 확인 (Impact & Invariants)
- **Impacted Modules**: `persistence_manager.py`, `main_window.py`, `strategy.py`, `composition_root.py`, `config_engine.py`
- **Invariant Check**: 
    - "UI의 모든 수익금 표시는 오직 trades.db의 집계 결과와 1원 단위까지 일치한다."
    - "시장 국면 판정, 전략 분류, 매도 로직은 설계 문서(docs/architecture)와 100% 일치한다."
    - "자본 보호(SL)는 항상 익절(TP)보다 높은 우선순위를 갖는다."

---
*Last Updated: 2026-04-07 18:20*
*Created By Antigravity (Powered by Advanced Agentic Coding)*

## 3. 검증 결과 (Verification)
- **SSoT 정합성**: `tests/test_pnl_ssot_consistency.py` 실행 결과, 메인 표시부와 KPI 대시보드 간의 오차 **0.00** 확인.
- **구문 무결성**: 가드 주석 삽입 후 모든 핵심 모듈(`strategy.py` 등)의 Syntax Check 통과.
- **Reconciliation**: `-224,518 -> -3,052,050` (로그 확인) - 오염된 초기 상태값이 시스템 원장에 의해 자동 복구됨을 확인.

---

## 4. 향후 과제 (Next Steps)
- 장중 실시간 틱 수신 시 `max_price` 업데이트 부하 모니터링 ($O(1)$ 이므로 안정적).
- **Regime-Adaptive Sizing**: 현재 3.5x/1.5x로 분리된 트레일링 스탑과 연동하여, 국면별 위험 노출도(Risk Exposure)를 자동 조절하는 로직 검토.
- **Zero-Trust Logic**: 중요 로직 수정 시 `GUARD` 표식이 있는 경우 자동으로 변경 내역의 임팩트를 분석하는 AI 도슨트 기능 고도화.

---
*Last Updated: 2026-04-07 15:25*
*Created By Antigravity (Powered by Advanced Agentic Coding)*
