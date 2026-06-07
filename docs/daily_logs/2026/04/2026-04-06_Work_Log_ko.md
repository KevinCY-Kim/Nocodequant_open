# 2026-04-06 작업 일지 (Work Log)

## 0. 개요 (Overview)
- **주요 과제**: 전략 성과 대시보드 데이터 정합성 해결 및 PnL SSoT 하드닝
- **등급**: **L4** (Architecture & Data Flow)
- **참조**: [AI 작업 헌장 v1.7](Antigravity_rules/governance/AI_Work_Constitution_and_Reference_Map.md), [구조 맵 v20.9](Antigravity_rules/architecture/Project_Structure_Map_ko.md)

---

## 1. 주요 변경 사항 (Key Changes)

### 1.1 거래 건수 불일치 해소 (9건 vs 5건)
- **문제**: 부분 체결이 발생할 경우 요약 카드는 매도 row 수를 기반으로 9건으로 표시하고, 로그 테이블은 주문 기반으로 5건으로 표시하여 정합성 붕괴.
- **해결**: `PersistenceManager`에 `v_trades_aggregated` DB 뷰를 도입하여 `order_id` (또는 `gtid`) 기반으로 거래를 단일 진실 공급원(SSoT)으로 정의.

### 1.2 수익금/수익률 계산 정밀화 (HTS 동기화)
- **가중평균(VWAP) 도입**: 단순 평균이나 MIN/MAX 단가가 아닌 거래량 가중평균 방식으로 진입/청산가 및 수익률 산출 로직 개선.
- **세율 및 수수료 현실화**: 
    - `config.py`에 `HISTORICAL_FEE_RATES` 도입 (2026년 기준 0.26%).
    - `StateManager`에서 하드코딩된 0.23% 수식을 제거하고 설정값을 창조하도록 수정.
- **데이터 소급 적용 (Option B)**: `trades` 테이블의 기존 191건 데이터를 신규 세율 및 가중평균 로직으로 일괄 업데이트(Migration) 수행.

### 1.3 데이터 무결성 가드 (NULLIF)
- **Division by Zero 방어**: DB 뷰 내 수익률 및 MFE/MAE 계산 시 `NULLIF`를 적용하여 수량이나 진입가가 0인 예외 상황에서도 시스템 안정성 확보.

### 1.4 AI 튜닝 탭 인터페이스 안정화 (UI Hardening)
- **문제**: AI 튜닝 제안을 "적용"할 때, 비동기 데이터 로딩이나 분석 엔진 내부 오류로 인해 화면이 통째로 검게 사라지는(Blackout) 현상 발생.
- **해결**: 
    - **방어적 렌더링(Defensive Rendering)**: `update_data` 및 `_render_diagnostic_card` 로직을 `try-except` 블록으로 보호하여 개별 카드 오류가 전체 화면 증발로 이어지지 않도록 격리.
    - **Null Guard**: `impact_data`가 비어있거나 불완전할 경우를 대비한 방어 코드 추가로 런타임 Crash 방지.

### 1.5 프로젝트 구조 맵(File Map) 정밀화
- **내용**: `app/ui/dialogs`, `widgets`, `components` 등 실제 파일 시스템에는 존재하나 구조 맵에서 누락되었던 12개 이상의 핵심 UI 파일을 전수 조사하여 맵에 명시.
- **결과**: `Project_Structure_Map_ko.md` 버전을 **v20.9**로 상향하여 AI 탐색 효율 극대화.

### 1.6 고스트 바(99/-99/0) 현상 완전 제압 [V20.8.5]
- **문제**: 캔들 인덱스(bar_index) 유실이나 역전 현상으로 인해 보유 봉수가 -99, 99, 0 등으로 비정상 출력되는 문제 발생.
- **해결**: 
    - **듀얼 게이트(Dual-Gate) 검증**: `OrderManager`에서 인덱스 기반 계산과 시간 기반 계산을 교차 검증하여 괴리 발생 시 시간 기반으로 강제 전환.
    - **최소 1봉 원칙**: 1초라도 거래가 발생했다면 무조건 1봉으로 인정하도록 `utils.py:calculate_holding_bars` 로직 강화.
    - **최후의 방어선**: `PersistenceManager`의 이벤트 발행 단계에서 비정상적인 `bars_held` 값을 0으로 클램핑(Clamp)하여 DB 오염 방어.

---

## 2. 영향 분석 및 불변성 확인 (Impact & Invariants)
- **Impacted Modules**: `persistence_manager.py`, `state_manager.py`, `config.py`, `ai_tuning_tab.py`, `strategy_dashboard.py`
- **Invariant Check**: 
    - "거래 데이터는 유실되지 않으며 집계 방식만 통일한다" 원칙 준수 확인.
    - `trades.db` 및 `forensics.db` 백업본 생성 후 마이그레이션 진행하여 안전성 확보.
    - UI는 어떠한 데이터 예외 상황에서도 빈 화면을 보여주지 않고 최소한의 상태 정보를 유지한다.

---

## 3. 검증 결과 (Verification)
- **PnL 정합성**: 피노(033790) 수익률 3.94% 등 HTS와 99.5% 이상 일치 확인.
- **UI 복원력**: 고의로 데이터 오류를 유입시켜도 AI 튜닝 탭이 검게 변하지 않고 정상적인 헤더와 에러 메시지를 출력함.
- **네비게이션**: 신규 등록된 UI 파일들에 대해 구조 맵 기반 탐색이 정상 작동함.

---

## 4. 향후 과제 (Next Steps)
- 장중 재시작 시 `daily_pnl`이 DB 뷰를 통해 실시간으로 자동 복구되는지 모니터링.
- `v_trades_aggregated` 뷰의 쿼리 성능이 데이터 증가에 따라 저하되지 않는지 확인 필요.
- 튜닝 적용 시 "처리 중..." 로딩 레이어를 추가하여 사용자 경험(UX) 고도화 검토.

---
*Created By Antigravity (Powered by Deepmind)*
