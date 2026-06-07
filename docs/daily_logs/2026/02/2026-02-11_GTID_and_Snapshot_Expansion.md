# 2026-02-11 작업 일지: 감사 등급 GTID 및 데이터 스냅샷 확장

## 1. 작업 개요 (Task Overview)
*   **목표**: 감사(Audit) 가능한 GTID 체계 확립 및 종목명 영속성 보장을 위한 DB/ snapshot 확장.
*   **등급**: **L4** (아키텍처 및 데이터 흐름 변경)
*   **작업자**: Antigravity (AI)
*   **준거 문서**: [AI 작업 헌장 v1.1](file:///c:/Users/stone/projects/nocodequant/Antigravity_rules/domains/AI_Work_Constitution_and_Reference_Map.md)

---

## 2. 규정 준수 및 참조 (Compliance & References)

### 2.1 작업 등급 식별 (Category Identification)
*   본 작업은 시스템의 유일한 진실 공급원(SSoT)인 **GTID 구조 변경**과 **DB 스키마(signals 테이블) 확장**을 포함하므로 **L4 등급**으로 분류함.

### 2.2 참조 문서 (References)
*   `Antigravity_rules/domains/AI_Work_Constitution_and_Reference_Map.md` (작업 헌장)
*   `Antigravity_rules/domains/system_trading/core/Parameter_Invariants.md` (파라미터 불변성)
*   `Antigravity_rules/domains/system_trading/core/Signal_Gating_Rules.md` (신호 제어 규칙)

---

## 3. 상세 변경 사항 (Implementation Detail - L4)

### 3.1 GTID 아키텍처 개편 (Architecture Diff)
*   **기존**: 단순 시퀀스 또는 타임스탬프 기반 ID.
*   **변경**: `{YYYYMMDD}-{HHMMSS}-{STRAT}-{UUID6}` 구조의 결정론적 ID 도입.
    *   **날짜/시간**: 신호 발생 시점 즉시 파악.
    *   **전략 태그(STRAT)**: 적극(`STR`) vs 보통(`NOR`) 구분.
    *   **UUID6**: 글로벌 고유성 및 시간 순서 보장.

### 3.2 데이터 스냅샷 및 DB 확장 (Migration Strategy)
*   **DB 스키마**: `signals` 테이블에 `name` (종목명) 컬럼 추가.
*   **마이그레이션**: `PersistenceManager._init_db`에서 `PRAGMA table_info`를 통해 컬럼 존재 여부를 확인하고, 부재 시 `ALTER TABLE`을 실행하는 자동 마이그레이션 로직 구현.
*   **스냅샷 확장**: `ExecutionManager`에서 이벤트 발생 시 종목명을 `snapshot` 데이터에 포함하여 영속성 확보.

### 3.3 하위 호환성 및 안정성 (Backward Compatibility)
*   **로그 표시**: 기존의 짧은 ID 포맷 로그와 새로운 긴 ID 포맷 로그를 모두 처리할 수 있는 하이브리드 파싱 로직 적용.
*   **가드 로직**: UI(`Main_NCQ.py`)에서 종목명 로드 시 엔진 연결 상태를 확인하는 가드를 강화하여 애플리케이션 시작 시 무음 종료 방지.

---

## 4. 영향을 받는 모듈 (Impacted Modules)
1.  `app/services/managers/persistence_manager.py`: DB 스키마 및 GTID 생성 로직.
2.  `app/services/managers/execution_manager.py`: TradeEvent 생성 시 메타데이터 주입.
3.  `Main_NCQ.py`: 로그 테이블 10컬럼 확장 및 실시간 이름 맵핑.
4.  `app/ui/forensic_dashboard.py`: 포렌식 뷰 데이터 모델 및 정렬 로직.

---

## 5. 검증 결과 (Verification)
*   [x] **DB 마이그레이션**: 기존 `trades.db`에 `name` 컬럼이 성공적으로 추가됨을 확인.
*   [x] **GTID 생성**: 신규 신호 발생 시 정해진 규격의 ID가 생성되고 DB에 기록됨.
*   [x] **UI 시각화**: 메인 윈도우와 포렌식 대시보드에서 종목 코드와 이름이 분리되어 표시됨.
*   [x] **중앙 정렬**: 모든 로그 테이블의 텍스트가 중앙 정렬됨을 확인.

---

## 6. UI 개편 및 UX 최적화 (UX & UI Polish)

### 6.1 실시간 로그 자동 스크롤 및 정렬 보정
*   **기능**: 신규 로그 삽입 시 `scrollToTop()`을 호출하여 항상 최신 데이터를 최상단에 노출.
*   **정렬**: 정렬 활성화 상태에서도 `Time Descending` 정렬을 강제하여 실시간성 유지.

### 6.2 대시보드 및 윈도우 레이아웃 확장
*   **사이드바 확장**: `RankingWidget` 및 `MultiSessionDashboard`의 최소 너비를 **550px**로 대폭 상향하여 텍스트 잘림 현상 해결.
*   **윈도우 최적화**: 메인 화면은 유지하되 사이드바 위주로 공간을 재배분(Main Window Width 1650px).
*   **컬럼 인터랙티브**: 대시보드 내 모든 컬럼을 사용자가 수동으로 조절 가능하도록 전환하고 기본 너비 최적화.

### 6.3 시인성 개선 (ToolTip & Visuals)
*   **툴팁 보정**: 어두운 테마에서 보이지 않던 툴팁 배경과 글자색을 명시적으로 스타일링(#222 / #EEE).
*   **중앙 정렬**: 모든 테이블 컬럼에 대해 중앙 정렬을 일관되게 적용.

---

## 7. AI 추론 정합성 및 로직 고도화 (Logic Integrity - V13.8.3)
*   **신호-예측 동기화**: '적극 매수/매도' 결정 시 미래 힌트가 '추세 역전 대기'로 나오는 논리적 모순 해결. 
    *   신호 발생 시 **"🎯 목표 도달: 즉시 대응 권장"**으로 메시지 강제 전환.
*   **상황별 인사이트**: 쿨다운 또는 저신뢰 보류 시 그 이유를 미래 힌트에 명시적으로 결합하여 사용자 신뢰도 제고.
*   **Logic Gap 정밀화**: 진입 신호 모드에서는 분석 노이즈를 제거하고 **"신호 조건 충족"**으로 상태 클렌징.

---

## 8. 최종 검증 (Final Verification)
*   [x] **자동 스크롤**: 로그 발생 시 즉시 상단 이동 확인.
*   [x] **사이드바 가시성**: 550px 확보로 모든 데이터 및 상태 메시지 가독성 확보.
*   [x] **추론 정합성**: 포렌식 대시보드 내 결정 라벨과 인사이트 메시지 간의 논리적 일치 확인.

---

## 9. 실전 매매 기능 고도화 (Trading Core - V13.8.4 ~ V13.8.6)

### 9.1 실시간 계좌 요약 패널 (V13.8.4)
*   **기능**: HTS급 실시간 자산현황 대시보드 구현.
*   **기술적 세부사항**:
    *   `KiwoomEngine`에 `deposit` 및 `buy_amount` 파싱 로직 추가.
    *   평단가(`avg_price`) 누락 시 매입금액/보유수량을 통한 백업 계산 로직 도입.
    *   `StateManager`와의 실시간 동기화를 통해 주문 수량 계산의 기초 데이터로 활용.

### 9.2 동적 비중 조절 시스템 (V13.8.5)
*   **목표**: 자산 규모에 따른 유연한 포지션 사이징 지원.
*   **구현**:
    *   **UI**: 수량 고정 / 예수금 비중(%) / 총자산 비중(%) 하이브리드 입력 인터페이스.
    *   **Logic**: `ExecutionManager._calculate_qty`에서 실시간 자산 데이터를 참조하여 주문 직전 최적 수량 계산.
    *   **안전 가드**: 1주 미만 주문 차단 및 이미 보유 중인 종목에 대한 중복 매수 방지(`OrderManager` 가드).

### 9.3 유사 0198 시장 열기 분석 (V13.8.6)
*   **배경**: 공식 TR이 부재한 실시간 조회 순위[0198] 화면을 알고리즘으로 대체.
*   **엔진**:
    *   거래대금 상위(`opt10043`) TR 신규 연동.
    *   가중치 합산(거래대금 50%, 등락률 30%, 거래량 20%)을 통한 **Hot Index** 산출.

### 9.4 Smart Heat Index 진화 (V13.8.7 ~ V13.8.11)
*   **정규화 엔진 (V13.8.7)**: Rank-based Percentile (`1 - (rank-1)/(N-1)`) 도입으로 서로 다른 단위의 데이터를 0.0~1.0 스케일로 통합.
*   **리스크 보정 (V13.8.7)**: 3~12% 골든존 가점 및 20% 이상 과열권 선형 감점(`Momentum Guard`) 구현.
*   **데이터 안정화 (V13.8.9)**: 복잡한 `opt10043` 대신 범용적인 `opt10030`으로 TR 교체 및 **3분 주기 자동 폴링** 가동.
*   **실전형 필터링 레이어 (V13.8.10/11)**:
    *   **ETF/ETN Filter**: KODEX, TIGER 등 파생 상품군 원천 배제.
    *   **Dual-depth Lightning Filter**: -7% 미만 폭락주 하드 컷오프 및 -5%~-7% 모멘텀 패널티 존 설정을 통해 리스크 차단과 반등 포착 균형 확보.
*   **엔진 4대 사명**: 추세 추종, 모멘텀 기반, 리스크 회피, 유동성 필터라는 엔진 성격 규명.

---

## 10. 통합 검증 및 안정성 (Total Stability)
*   [x] **비중 계산**: 실전 자산(예수금) 기반의 % 주문 정상 작동 확인.
*   [x] **계좌 동기화**: 매수/매도 발생 시 즉각적인 총매입 및 예수금 업데이트 확인.
*   [x] **필터링 무결성**: ETF 및 급락주가 지수 산출 단계에서 정확히 필터링됨을 확인.
*   [x] **자동 갱신**: 3분 주기로 랭킹 데이터가 최신화되어 UI에 반영됨을 확인.

---
*Constitution Compliance: Verified (L4 Trading Core & Institutional Filtering Pipeline Documented)*
