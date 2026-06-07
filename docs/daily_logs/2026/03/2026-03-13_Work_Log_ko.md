# 2026-03-13 작업일지 (Work Log)

## [Phase 13] 실시간 P&L 대시보드 및 차트 복구 (L4)

### 1. 개요 (Overview)
- **목표**: 매도 체결 시 P&L 지표가 실시간으로 UI에 반영되지 않던 결함을 해결하고, 차트 동기화를 100% 보장하는 "기관급(Institutional) 피드백 시스템" 구축.
- **수정 등급**: **L4 (Architecture & Data Flow)** - UI 캐싱 및 이벤트 전달 경로 재설계.

### 2. 주요 변경 사항 (Proposed Changes)
- **포렌식 브리지 통합 (Forensic Bridge Consolidation)**:
    - 기존의 분산된 `EventBus` 구독 방식 대신, DB 영속화가 보장된 `log_persistence_to_forensic`을 UI 업데이트의 단일 진입점(SSoT Trigger)으로 일원화.
    - `ORDER_FILLED` 이벤트 감지 시 `update_performance_summary` 및 `save_pnl_snapshot` 즉시 실행.
- **차트 실시간 갱신 (Real-time Chart Sync)**:
    - `save_pnl_snapshot` 내에 `chart_widget.add_data`를 통합하여, 10초 주기 타이머가 아닌 체결 즉시 차트 애니메이션이 발생하도록 개선.
- **V18.7 지속성 및 서버 렉 방어 (Persistence & Lag Guard)**:
    - `StateManager`에 일간 통계(`daily_trades`, `win_count`, `HWM`, `cumulative_pnl`) 저장/불러오기 로직을 추가하여 앱 재시작 시 데이터 복구 보장.
    - `MainWindow.update_realized_pnl`에 서버 지연 응답(0원) 방어 로직을 구축하여 UI 리셋 현상 차단.
- **V18.8 직통 핫라인 패치 (Direct Hotline & Zero-Latency)**:
    - **UI 락 해제**: `PerformanceSummaryWidget`에서 예수금이 0원이어도 P&L과 통계는 즉시 업데이트되도록 리팩토링.
    - **직통 마샬링 구축**: DB 영속화 과정을 거치지 않고 체결 즉시 메인 스레드로 다이렉트 신호(`sig_landing_fill`)를 쏴서 지연율을 0%로 단축.
    - **비동기 배칭 병목 해결**: DB 부하로 인한 UI 지연 현상을 원천 제거.
- **아키텍처 리팩토링 (Architecture Refactoring)**:
    - **MainWindow 비대화 방지**: MDD 계산 및 통계 관리 로직을 `MainWindow`에서 `StateManager`로 완전히 이관(De-bloating).
    - **컴포넌트 독립성 강화**: P&L 대시보드 표시 로직을 `PerformanceSummaryWidget`으로 격리하여 시각화와 로직을 분리.
- **[ARCH-EDA-01] Multi-Tier 감시 체계 명문화**:
    - **Promotion (격상)**: 백그라운드 스캔 점수 > 50점 시 300초간 Tier 1 실시간 감시 대상으로 격상하는 로직 문서화.
    - **Demotion (강등)**: 5분 경과 시 자동으로 Tier 2로 복귀 및 리소스 해제 루틴 명문화.
    - **참조 업데이트**: `Signal_Gating_Rules.md` 및 `Execution_Pipeline_Architecture.md` 최신화.
- **UI 컴포넌트화 (Visual Componentization)**:
    - `main_window.py`에서 비대해진 UI 클래스(`AccountSummaryWidget`, `MultiSessionDashboard`) 약 350줄을 추출하여 `app/ui/components/position_dashboard.py`로 모듈화.
    - 메인 윈도우의 복잡도를 낮추고 유지보수성을 확보하는 'God Object' 방지 아키텍처 적용.
- **코드 안정화 및 정리 (Cleanup)**:
    - 실패한 마샬링 실험 코드 및 중복 위젯 정의를 삭제하여 `main_window.py` 파일 크기 최적화.
    - `update_performance_summary`에서 `deposit_val` 누락으로 인한 MDD 계산 오차 수정.

### 3. 영향을 받은 모듈 (Impacted Modules)
- `app/ui/main_window.py`: UI 트리거 로직 및 대시보드 함수 수정.
- `Antigravity_rules/architecture/EventBus_Persistence_Spec.md`: 데이터 흐름도 최신화.

### 4. 불변성 및 경계 점수 (Invariants & Governance)
- **Invariant Check: Pass** - `StateManager`의 데이터 원천(SSoT) 원칙을 고수함.
- **Boundary Check: Pass** - 백엔드 매매 로직과 UI 표시 로직의 계층적 분리 유지.

---

## 5. 최종 검증 (Final Verification)
- [x] **실시간 갱신**: 매도 체결 시 "실시간 운용 현황(PL)" 보드의 승률, 체결 횟수, 실현손익이 즉시 변경됨.
- [x] **차트 피드백**: 체결 발생 찰나에 하단 P&L 차트에 실시간 데이터 포인트가 추가됨.
- [x] **데이터 무결성**: 수동 계좌 조회 시 서버 데이터와 대시보드 숫자가 완벽히 일치함을 확인.

---
*NCQ Work Log System v1.2*
