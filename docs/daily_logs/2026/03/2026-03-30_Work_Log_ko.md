# 2026-03-30 작업일지 (Work Log)

## [V19.6] 전략 대시보드 데이터 정규화 및 메타데이터 파이프라인 결함 수복 (L4)
- **목표**: 전략별 수익률 대시보드(Strategy Dashboard)의 데이터 누락 및 시각적 오류를 전수 조사하고, 신호 발생부터 UI 표출까지의 데이터 파이프라인을 완전 복구.

- **주요 변경**:
    - **이벤트간 메타데이터 유실 해결 (`order_manager.py`)**:
        - **문제**: 키움 OpenAPI의 비동기 체결 이벤트(`sig_order_fill`) 수신 시, 원본 `TradeSignal`의 메타데이터(전략형, 진입가, 국면 등)가 유실되어 DB에 `UNDEFINED`로 기록되는 현상 발생.
        - **조치**: `pending_signal_meta`라는 단기 메모리 캐시 도입. 주문 전송 시 메타데이터를 보관하고, 체결 시점에 이를 복원하여 성과 데이터의 무결성(Integrity) 확보. 정확한 `pnl_ratio` 계산 로직을 백엔드로 이전.
    - **DB 통계 뷰 확장 (`persistence_manager.py`)**:
        - **문제**: 기존 `v_strategy_performance` 뷰에 손익비(Profit Factor) 및 평균 익/손절 수익률 컬럼이 누락되어 대시보드가 하드코딩된 값("---")을 표시함.
        - **조치**: 뷰를 재설계하여 `avg_win_ratio`, `avg_loss_ratio`, `profit_factor` 컬럼 추가. `save_fill` 레거시 브릿지에 7종의 신규 메타데이터 파라미터 대응 완료.
    - **UI 동적 데이터 바인딩 (`strategy_dashboard_window.py`)**:
        - **문제**: KPI 카드 및 요약 테이블이 정적 데이터에 의존하거나 DB 쿼리 결과와 인덱스 불일치 발생.
        - **조치**: `update_summary` 메서드가 DB에서 집계된 실제 성과 객체(`summary`)를 직접 수용하도록 구조 개선.

- **Web Dashboard (Hybrid) 결함 전수 수복 (`strategy_dashboard.html`)**:
    - **인덱스 미스매치 정정**: 뷰 컬럼 순서와 JS 배열 구조 분해 인덱스(v[4]~v[7]) 간의 불일치를 해결하여 Total PnL 자리에 손실률이 나오던 오류 수정.
    - **이중 계산 제거**: DB에서 이미 퍼센트(%) 단위로 저장된 `avg_pnl_ratio`에 UI에서 redundant하게 `* 100`을 수행하여 수익률이 100배 부풀려지던 버그 해결.
    - **보안 강화**: `renderTradeLog` 내 템플릿 리터럴 기반의 동적 쿼리 생성을 `sql.js` 파라미터 바인딩(`?` 및 params 배열) 방식으로 교체하여 SQL Injection 취약점 원천 차단.
    - **UI 일관성**: `autoFetchDB` 실패 시 `innerText` 오용으로 깨지던 DOM 구조를 `innerHTML`로 통일하여 상태 메시지 가독성 확보.

- **결과**:
    - 전략별(VOLUME, TREND, BREAKOUT 등) 실제 승률과 손익비가 Python UI 및 Web Dashboard 양쪽에서 100% 정합성 있게 표출됨.
    - 향후 발생하는 모든 체결은 `pending_signal_meta` 가드를 통해 누락 없이 추적됨.

---
*NCQ Work Log System v1.2*
