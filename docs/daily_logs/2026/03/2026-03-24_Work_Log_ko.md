# 2026-03-24 작업일지 (Work Log)

## [Phase 19.3] Dashboard 데이터 정합성 강화 및 DB 클린업 (L3/L4)
- **목표**: 대시보드의 부정확한 승률/거래 통계 원인을 분석하고, DB 내 데이터 오염 및 집계 로직 오류 해결.
- **주요 변경**:
    - **DB 클린업 (L4)**: `trades` 테이블에 잘못 혼입된 `side='BUY'` 레코드 706건을 일괄 삭제하여 데이터 단일 책임 원칙(청산 데이터만 보존) 확립.
    - **기간 조회 기능 (`strategy_dashboard.py`)**: 사용자 편의를 위해 대시보드 상단에 "📅 기간 필터" `QComboBox` (전체/당일/1주/1달) 추가. 
    - **쿼리 동적화 (`persistence_manager.py`)**: `get_strategy_performance_summary` 및 `get_recent_trades`에 `start_date` 파라미터 적용하여 기간별 실시간 필터링 구현.
    - **집계 무결성 복구**: 상단 KPI 카드 데이터를 200건으로 제한된 UI 리스트가 아닌, DB 전수 데이터를 Aggregate(합산/카운트)한 Summary 결과를 사용하도록 변경하여 데이터 왜곡 교정.
- **결과**: 누적 수천 건의 거래가 발생하더라도 수익률과 승률이 정확히 표시되는 신뢰할 수 있는 대시보드 환경 구축.

---
*NCQ Work Log System v1.2*
