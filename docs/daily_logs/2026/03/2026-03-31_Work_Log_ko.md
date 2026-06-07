# 2026-03-31 작업일지 (Work Log)

## [V19.8] 타임프레임 데이터 실시간 통합 및 SQLite WAL 동기화 결함 수복 (L4)
- **목표**: 전략별 타임프레임(5분, 15분 등) 정보를 성과 분석 파이프라인에 통합하고, 웹 대시보드에서 당일 데이터가 표시되지 않던 SQLite WAL 동기화 문제를 근본적으로 해결.

- **주요 변경**:
    - **타임프레임 메타데이터 전 계층 통합 (`persistence_manager.py`, `order_manager.py`)**:
        - **스키마 확장**: `trades` 테이블에 `timeframe` (INTEGER) 컬럼을 추가하고, 기존 DB를 위한 자동 마이그레이션 로직(`_migrate_trades_if_needed`) 구현.
        - **파트너 모듈 연결**: `TradeSignal` -> `pending_signal_meta` -> `PersistenceEvent`로 이어지는 timeframe 전달 경로를 완성하여, 각 거래가 어떤 분봉 전략에 의해 실행되었는지 영구 기록.
    - **웹 대시보드 데이터 가시성 결함 해결 (`strategy_dashboard.py`)**:
        - **문제**: SQLite의 WAL(Write-Ahead Logging) 모드 특성상, 최신 거래 데이터가 `-wal` 파일에 머물러 있어 웹 서버가 서빙하는 메인 `.db` 파일에는 당일 데이터가 즉시 반영되지 않는 현상 발견.
        - **조치**: 웹 서버가 DB 파일을 전송하기 직전에 `PRAGMA wal_checkpoint(TRUNCATE)`를 강제 실행하여 WAL 데이터를 메인 파일로 병합. 이를 통해 웹 대시보드에서 실시간에 가까운 당일 성과 확인 가능.
        - **캐시 가드**: HTTP 응답 헤더에 `Cache-Control: no-cache`를 추가하여 브라우저의 DB 파일 캐싱으로 인한 데이터 정체 방지.
    - **대시보드 UI 정밀 수복 및 폴리싱 (`strategy_dashboard.html`)**:
        - **KPI 계산 가드**: 데이터가 없는 경우(당일 첫 거래 전 등) `SUM` 결과가 `null`이 되어 "null승"으로 표시되던 버그를 `Number()` 및 `|| 0` 처리를 통해 해결.
        - **차트 잔상 제거**: 기간 필터 변경 시 데이터가 없으면 기존 차트 인스턴스를 명시적으로 `destroy()`하여 이전 데이터의 시각적 잔상이 남지 않도록 개선.
        - **자동 갱신 강화**: 1분 간격의 `autoFetchDB` 호출 시 필터 목록(`setupFilters`)을 함께 갱신하여 신규 전략 유형 등장 시 즉시 반영되도록 수정.
        - **MFE/MAE/Edge 실계산**: `max_price`, `min_price`를 SQL SELECT에 추가하고, JS 단에서 실제 MFE/MAE 및 Edge 값을 계산하여 표출 (기존 "---" 하드코딩 제거).
    - **로깅 시스템 최적화 (`engine.py`, `order_manager.py`)**:
        - 신호 발생 및 체결 시 발생하는 과도한 디버그 로그(`[META_DEBUG]`) 제거.
        - 키움 API의 단순 시스템 메시지(조회 완료 등) 필터링을 통해 터미널 가독성 대폭 향상.

    - **MME(Money Management Engine) 구조 개선 및 정합성 수복 (`money_manager.py`, `order_manager.py`, `main_window.py`)**:
        - **Concentration Mismatch 해결**: 종목당 비중 제한(Concentration Cap)이 적용된 후의 '실제 리스크(Actual Risk)'를 기준으로 HeatLimit을 심사하도록 로직 순서 변경. 이를 통해 상승장(UPTREND)에서 종목 진입이 조기 차단되던 버그 수정.
        - **Heat 등록 자동화**: BUY 체결 완료 시점에 실제 포지션 가치와 stop_loss_pct를 결합하여 정확한 리스크를 Heat로 등록하도록 `order_manager.py` 보완.
        - **UI 시뮬레이터 동기화 및 가시성 개선**: 
            - `main_window.py`의 MME Preview 가이드를 총자산(Total Asset) 기반으로 재설계하고, 종목당 한도를 20% -> 15%로 수정하여 백엔드 정책과 100% 일치시킴.
            - 상단 헤더(`PremiumHeaderWidget`)에 표시되던 이중 표출 "관망" 라벨을 제거하여 하단 '의사결정 허브'로 시선을 집중시키고, 레이아웃 공간을 효과적으로 확보 (AttributeError 방지 검수 완료).
    - **문서 체계 정비 및 아카이빙**:
        - `docs/changelogs/` 폴더를 신규 생성하여 버전별 변경 이력 관리 체계 구축.
        - `CHANGELOG_MME_V19.7.md`를 해당 폴더로 이동하고, `NCQ_MME_Architecture_V19.7.md`를 `docs/architecture/`에 배치 완료.

- **결과**:
    - 웹 대시보드에서 "당일" 버튼 클릭 시 즉시 최신 거래 내역과 통계가 정합성 있게 표출됨.
    - MME 엔진이 상승장에서 최대 10종목(5.0% Heat 내)까지 유연하게 포트폴리오를 확장할 수 있게 됨.
    - 프론트엔드와 백엔드 간 리스크 계산 오차가 사라져 사용자 신뢰도 향상.
    - 프로젝트 문서가 카테고리별(Architecture, Changelogs, Daily Logs)로 체계적으로 정리됨.

---
*NCQ Work Log System v1.2*
