# 2026-03-18 작업일지 (Work Log)

## [Phase 18.05] TREND vs RANGE 진입 임계값 분리 (V18.5 - L3)
- **목표**: 단일 임계값(62)의 한계를 극복하고 시장 국면에 따른 유연한 진입 전략 수립.
- **주요 변경**:
    - `config_engine.py`: `ENTRY_THRESHOLD_TREND=58`, `ENTRY_THRESHOLD_RANGE=65`, `ENTRY_THRESHOLD_CHAOS=72` 분리.
    - `strategy.py`: `_get_signal_internal`에서 국면(Regime)별 임계값 자동 선택 로직 구현.
    - `main_window.py`: Decision Hub 툴팁에 국면별 동적 임계값 및 번역어 반영.
- **결과**: 추세장 조기 탑승 및 박스권 가짜돌파 억제력 동시 확보.

## [Phase 18.10] 로스컷(Stop-Loss) 작동 불능 긴급 수술 (L3)
- **목표**: 엔진의 긴급 로스컷 신호가 주문으로 이어지지 않는 결함 해결 및 앱 재시작 시 상태 복구.
- **주요 변경**:
    - `schema.py`: `STOP_EXIT` → `ORDER_SUBMITTED` 전이 경로 신설.
    - `execution_manager.py`: 주문 실행 단계 유연화 (동적 이벤트 타입 참조).
    - `strategy.py`: `sync_position` 메서드를 통한 `high_watermark` 및 트레일링 스탑 기준선 복원.
- **결과**: 재시작 후에도 안정적인 로스컷 추적 및 즉각적인 시장가 청산 보장.

## [Phase 18.11] 로스컷 안정성 개선 및 중복 방지 (L3)
- **목표**: 분할 체결 시 발생하는 로스컷 주문 루프 차단 및 수기 주문과의 충돌 방지.
- **주요 변경**:
    - `order_manager.py`: `order_hash` (종목_수량_방향) 기반 30초 강력 데드락 차단기(Strong Dedup) 도입.
    - `execution_manager.py`: `STOP_EXIT` 평가 시 즉시 `cancel_pending_orders` 호출하여 인터럽트 구현.
- **결과**: 로스컷 도배 원천 봉쇄 및 긴급 상황 시 기존 지정가 주문 즉각 취소 토대 마련.

## [Phase 18.12] 매도 오더 MME 우회 및 ExitType 도입 (L3)
- **목표**: SELL 오더가 MME 리스크 한도에 막혀 0주로 삭감되는 결상 교정 및 전략적 청산 타입 명확화.
- **주요 변경**:
    - `execution_manager.py`: `SELL` 신호 시 MME 우회 및 `held_qty` 기반 수량 산출 강제.
    - `schema.py`: `ExitType` Enum (`PARTIAL_TP`, `FULL_TP`, `STOP_LOSS`, `TIME_EXIT`) 전면 도입.
    - `order_manager.py`: 주문 시점이 아닌 실제 체결 시점(`on_fill_event`)에 리스크 노출(`exposure`) 안전 해제.
- **결과**: 보성파워텍 등 매도 누락 케이스 해결 및 부분 익절(TP1) 정합성 확보.

## [Phase 18.45] ADX 방향성 SSoT 확립 및 포렌식 필터 정교화 (L3/L4)
- **목표**: ADX 방향 맹점으로 인한 하락장 매수 오작동 차단 및 포렌식 데이터 무결성 확보.
- **주요 변경**:
    - **ADX SSoT**: `strategy.py` 내 `is_downtrend` 계산 단일화 및 하위 주입. 하락장 감지 시 리스크 -20점 즉각 부여.
    - **히스테리시스**: MA20/MA60 이격 0.3% 히스테리시스 가드로 국면 채터링 방지.
    - **포렌식 정비**: `main_window.py` 중복 시그널 제거 및 `PersistenceEvent` 페이로드 필터링 오류 수정. 
- **결과**: "떨어지는 칼날" 매수 차단 및 포렌식 대시보드 1줄 출력(Clean logs) 복구.

---
*NCQ Work Log System v1.2*
