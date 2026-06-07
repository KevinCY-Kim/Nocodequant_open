# 2026-03-25 작업일지 (Work Log)

## [Phase 19.4] Strategy Dashboard 강력한 분석 지표 도입 및 정합성 패치 (L3)
- **목표**: MFE, MAE, Holding Time, Edge Ratio 등 퀀트 레벨의 고급 분석 지표를 대시보드에 도입하고, 메타데이터 유실 및 로그 중복 문제를 원천 차단.
- **주요 변경**:
    - **분석 지표 백엔드 이식 (`state_manager.py`, `order_manager.py`)**:
        - `update_quote` 수신 시 오픈 포지션 한정(`qty > 0`)으로 `max_price`, `min_price`를 지속 추적하도록 추가. 포지션 청산 후 늦게 들어오는 틱이 MAE를 왜곡하는 현상 차단.
        - 매수 진입 시 최초 오더 콜백으로부터 `entry_time`을 확정해 Holding Time의 SSoT(Single Source of Truth) 마련.
    - **안전한 DB 핫-마이그레이션 (`persistence_manager.py`)**:
        - `ALTER TABLE trades ADD COLUMN` 방식으로 기존 DB 락/손실 없이 `max_price`, `min_price` 컬럼을 동적 추가(`_migrate_trades_if_needed`).
        - 과거 미기록 데이터(값 없음)의 경우 `entry_price`로 안전하게 리트리빙 되게 폴백 락아웃 추가.
    - **1주문 1행 원칙 도입 (Option A)**:
        - `order_manager.py`의 `on_fill_event` 함수 내에서 이벤트를 쏘는 조건을 `if is_position_closed` (매도 완료)로 단일화하여 BUY 트리거 시 발생하던 수익금 0원 Ghost Row 소멸 조치 완료.
    - **메타데이터 크랙 영구 복원 (`state_manager.py`, `order_manager.py`)**:
        - 부분 매도 시 남은 포지션 딕셔너리에 기존 `strategy_type`, `regime`이 그대로 상속되도록 병합 패치.
        - 매수 진입 시점에 `pos.get('avg_price')`에 의존하던 방식을 개선해, `set_position_meta`에 초기 시점부터 `entry_price`, `entry_time` 직접 주입 적용.
    - **UI 표출 및 성능 시각화 (`strategy_dashboard.py`)**:
        - 조회단에서 보유시간(HH:MM:SS) 포맷팅, MFE/MAE(%) 계산 및 `Edge Ratio (MFE / |MAE|)` 공식(`abs(mae) > 1e-6` 필터링) 반영.
        - `StrategyTradeTableModel` 헤더 확장 및 매핑(`보유봉` 제거 ➡️ `보유시간`, `MFE`, `MAE`, `Edge` 추가).
        - 전용 Color Highlight 로직(익절/손절/Edge 1.2 이상 등)으로 직관적 피드백 제공.
- **결과**:
    - 데이터 이중 기록(Double log) 원천 제거 및 깔끔하게 '1거래 1결과' 도출. 감에 의존하던 부분 체결 및 전략 검증이 명확한 수치(MFE, MAE, Edge Ratio)로 전환됨으로써 AI와 유저가 전략 튜닝을 할 수 있는 강력한 데이터 토대 조성 완료.

---
*NCQ Work Log System v1.2*
