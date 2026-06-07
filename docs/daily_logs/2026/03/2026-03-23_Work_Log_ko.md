# 2026-03-23 작업일지 (Work Log)

## [Phase 19.1-8] 전략 데이터 수집 및 영속화 레이어 무결성 확보 (L3/L4)
- **목표**: `order_manager`와 `persistence_manager` 간의 시그널 데이터 유실 해결 및 정밀한 성과 집계 환경 구축.
- **주요 변경**:
    - `order_manager.py`: `pending_signal_meta` 캐싱 시스템을 도입하여 OCX 체결(`FILL`) 이벤트 발생 시 유실되던 원본 시그널의 핵심 메타데이터(`strategy_type`, `regime`, `entry_price`) 복원 로직 구현.
    - `pnl_ratio` 정교화: 체결 시점에서 진입가(Entry Price)와 현재 체결가를 비교하여 `pnl_ratio`를 즉시 계산하고, 이를 `PersistenceEvent` 페이로드에 포함하도록 개선.
    - `persistence_manager.py`: `save_fill` 메서드의 인터페이스를 확장하여 7개의 신규 메타데이터 필드 영속화 지원 및 `trades` 테이블 스키마 대응.
    - **SQL View 전면 재설계**: `v_strategy_performance` 뷰를 KPI 중심(`avg_win_ratio`, `avg_loss_ratio`, `profit_factor`)으로 재구축하여 대시보드 데이터 공급원 최적화.
- **결과**: 기존 `UNDEFINED`로 기록되던 기술적 한계를 극복하고, 전향적(Forward) 관점에서 100% 신뢰 가능한 성과 분석 데이터 축적 개시.

## [UI/UX] 하이브리드 전략 대시보드 렌더링 및 보안 강화 (V19.1 Patch) (L4)
- **목표**: PyQt Native 및 웹 기반 하이브리드 대시보드의 데이터 정합성 해결 및 보안 취약점 차단.
- **주요 변경**:
    - **KPI 자동 연동**: PyQt Native UI(`strategy_dashboard.py`)의 상단 KPI 카드들이 DB 요약 집계 데이터를 실시간으로 반영하여 전략별 평균 승률 및 Profit Factor를 가시화하도록 수정.
    - **데이터 매핑 동기화**: SQL 뷰의 컬럼 인덱스와 JS 데이터 바인딩 로직(`renderOverview`, `renderTuning`)을 재정렬하여 PnL 및 수익률 컬럼이 섞여 출력되던 오류 해결.
    - **보안 강화 (SQL Injection)**: 웹 대시보드(`strategy_dashboard.html`) 내 `renderTradeLog` 함수의 동적 쿼리 구문을 Parameterized Query(`?`) 방식으로 전면 전환하여 보안성 확보.
    - **렌더링 버그 수정**: `avg_pnl_ratio`의 중복 백분율 계산(redundant *100) 오류 제거 및 DOM 조작 방식(`innerHTML`) 통일로 UI 깨짐 현상 해결.
- **결과**: 시각적 무결성이 확보된 고품질의 전략 분석 환경 구축 및 시스템 보안성 강화.

## [중요 참고사항] V19.1 데이터 소급 관련
- 이번 패치는 데이터 **수집 시점의 로직**을 개선한 것으로, 이미 `trades.db`에 `UNDEFINED`로 기록된 과거 데이터에 대해서는 별도의 마이그레이션을 수행하지 않음 (향후 발생하는 데이터부터 정상 기록됨).

---
*NCQ Work Log System v1.2*
