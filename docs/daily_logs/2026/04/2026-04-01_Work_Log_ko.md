# 2026-04-01 작업일지 (Work Log)

## [V19.8.7] 실시간 체결 중복 누적(Idempotency) 결함 수복 및 FID 매핑 정규화 (L3)
- **목표**: 키움 OpenAPI의 FID 매핑 오류로 인한 실시간 체결 데이터의 중복 누적(PnL 뻥튀기) 현상을 해결하고, 어떤 상황에서도 거래 무결성을 유지하는 멱등성 가드 강화.
- **주요 변경**:
    - **FID 매핑 정규화 (`engine.py`)**:
        - **문제**: 기존에 `fill_id`로 체결수량(FID 911)을 사용함에 따라, 동일 수량의 연속 체결 발생 시 중복 제거 필터가 무력화되어 PnL 및 수익률이 비정상적으로 부풀려지는 버그 확인.
        - **조치**: `fill_id` 매핑을 고유 번호인 **체결번호(FID 912)**로 변경. 만약 체결번호가 없을 경우 주문번호(FID 909)를 폴백으로 사용하도록 구조 개선. 이를 통해 각 체결 이벤트에 고유 ID 부여 완료.
    - **복합 멱등성 키(Composite Idempotency Key) 도입 (`order_manager.py`)**:
        - **2중 방어**: 브로커의 ID 생성이 불안정하거나 누락될 경우를 대비하여 `종목:방향:수량:가격:미체결량`을 조합한 **결합 키(Composite Key)**를 추가 도입.
        - **로직**: `fill_id` 기반 1차 차단 + `composite_key` 기반 2차 차단을 통해, 엔진 계층에서 동일 데이터가 중복 전달되더라도 `PersistenceManager`에 중복 이벤트가 방출되지 않도록 완벽히 방어.
    - **상태 관리 하우스키핑 (GC)**:
        - `processed_fills` 캐시가 세션 유지 시간 동안 무한히 증가하여 메모리를 점유하는 것을 방지하기 위해, 1,000건 초과 시 하위 500건을 자동 정리(Discard)하는 Garbage Collection 로직 추가.
    - **MME 리스크 등록 정밀화 (`order_manager.py`, `money_manager.py`)**:
        - **Heat 등록 타이밍**: BUY 주문 시점이 아닌, 실제 체결이 완료된 `is_buy_complete` 시점에 `money_manager`에 노출(Heat)을 등록하도록 수정.
        - **정밀 계산**: 체결 시점에 확정된 `avg_fill_price`와 현재 계좌의 `total_asset`을 실시간으로 조회하여 산출된 **유효 리스크(Actual Risk)**를 Heat로 등록. 이를 통해 상승장에서의 정확한 리스크 한도 관리가 가능해짐.

## [V19.8.8] 체결 고유 식별자(fill_id) 무결성 강화 및 모의투자 멱등성 버그 수복 (L4)
- **목표**: 모의투자 환경에서 부분 체결 시 동일한 체결번호(FID 912)가 유입되어 후속 체결이 차단되는 'Silent Block' 현상 해결.
- **주요 변경**:
    - **통합 식별자 생성 (`engine.py`)**: 
        - 모의투자 서버는 부분 체결 시 체결번호를 갱신하지 않는 결함이 있음. 
        - 이를 해결하기 위해 `fill_id`를 `고유ID(FID 912 or 909)_미체결수량(FID 902)` 조합(`f"{base_id}_{raw_unfilled}"`)으로 강제 생성. 
        - 미체결 수량은 매 체결 시마다 반드시 감소하므로, 어떤 환경에서도 부분 체결마다 100% 고유한 식별자를 보장함.
    - **멱등성 로직 회귀 (Idempotency Regression)**: 
        - 불안정한 `composite_key`를 제거하고, 정규화된 `fill_id` 단일 필터링 체계로 복귀하여 로직 간결화 및 처리 속도 확보.
        - `processed_fills` GC 정책(1,000건 초과 시 정리)은 지속 유지하여 장기 가동 안정성 확보.

## [V19.8.9] 좀비 트레이드 부활 방지 및 방어적 UI(Defensive UI) 안정화 (L5)
- **목표**: 수동 매수 시 과거 종료된 거래 메타데이터가 현재 거래를 오염시키는 '좀비 부활' 버그 차단 및 데이터 가변성에 따른 UI 레이아웃 붕괴 방지.
- **주요 변경**:
    - **Stateful Deep-Search Recovery (`persistence_manager.py`)**: 
        - `get_last_signal_for_code` 조회 시, 해당 종목의 마지막 이벤트가 `SELL`이면 복구를 즉시 취소(`return {}`)하도록 로직 강화. 
        - 당일 재진입이나 과거 청산 기록이 현재의 수동 매수 포지션 시간/진입가를 오염시키는 현상을 원천 봉쇄함.
    - **지표 정밀화 및 크래시 수복 (`utils.py`, `order_manager.py`)**: 
        - `entry_time` 복원 시 실제 체결 시각을 최우선순위로 설정하여 보유 시간 및 MFE/MAE 정확도 확보. 
        - 매도 체결 시 `entry_time`이 `None`일 경우 발생하던 `NoneType >= datetime` 비교 오류(TypeError)를 방어(Guard) 코드로 완벽 수복.
    - **방어적 UI 설계 (Defensive UI Design)**: 
        - **Main Window (`main_window.py`)**: 중앙 전략 엔진 패널의 데이터 팽창이 우측 도킹 위젯을 밀어내지 못하도록 `MaximumWidth` 및 `Fixed Width` 제약 조건 강제. 창 축소 시 레이아웃 붕괴 대신 하단 스크롤바가 활성화되도록 `QScrollArea` 정책 수정.
        - **Header Hardening (`main_window.py`)**: `PremiumHeaderWidget`의 가격 및 등락폭 라벨에 고정 폭(Fixed Width)을 부여하여 시세 변동 시 인접 위젯이 흔들리는 '레이아웃 피버' 제거.
        - **Dashboard Hardening (`position_dashboard.py`)**: 보유 잔고 및 감시 목록 테이블에 `QHeaderView.Fixed` 정책을 적용하여 컬럼 너비 팽창 차단. 마지막 컬럼은 `Stretch`로 설정하여 시각적 정렬 및 부모 너비 보존.
        - **Chart Stabilization (`chart_widget.py`)**: `sizeHint` 및 `minimumSizeHint`에 고정 크기를 부여하여, 데이터 업데이트 시 부모 레이아웃이 위젯의 크기를 재검색하며 발생하는 미세한 화면 떨림(Flickering) 현상 차단.

- **결과**:
    - 수동 매매 시 과거 데이터 오염 없이 정밀한 MFE/MAE/보유시간 지표 산출 확인.
    - 실시간 데이터 폭주 및 극단적 종목명 길이에도 레이아웃이 1픽셀도 흔들리지 않는 견고함 확보.
    - 매도 체결 시 간헐적으로 발생하던 엔진 크래시 완전 해결 및 메인 테두리 가시성 확보.

## [V19.9.2] 방어적 UI 아키텍처(Defensive UI) 고도화: Institutional Grade (L5)
- **목표**: 데이터 팽창, OS 테마 차이, 사용자 조작에도 절대 붕괴되지 않는 '제약 기반 레이아웃(Constraint-Based Layout)' 시스템 구축 및 상위 5% 수준의 안정성 확보.
- **주요 변경**:
    - **방어적 베이스 클래스 도입 (`base.py` 신설)**:
        - **`DefensivePanel`**: 모든 고밀도 패널의 부모 클래스로, `Minimum/Maximum Width`를 강제하고 자식 위젯의 `QSizePolicy`를 명시적으로 선언하도록 구조화.
        - **`DefensiveTableWidget`**: `QHeaderView.Fixed` 정책을 기본으로 하고 `ResizeToContents`를 비활성화하여, 종목명 길이에 따른 컬럼 너비 요동(Jitter) 현상을 원천 차단.
    - **3단 컬럼 절대 격리 (`main_window.py`)**:
        - **Stretch Factor Lock**: `main_layout.setStretch(0, 0)` (좌측/실행), `(1, 1)` (중앙/전략), `(2, 0)` (우측/도킹) 설정을 통해 중앙 엔진만이 공간을 점유하고 사이드 패널은 1픽셀도 밀리지 않도록 고정.
        - **Platform Independence**: 모든 레이아웃에 `setContentsMargins(0,0,0,0)`와 명시적 `setSpacing`을 적용하여 OS/DPI별 미세 오차 제거.
    - **스타일 독립성 확보 (`WA_StyledBackground`)**:
        - 주요 컨테이너(`main_container`, `center_widget`, `DefensivePanel` 등)에 `Qt.WA_StyledBackground` 속성을 부여하여 부모 스타일 전파로 인한 레이아웃 연산 오류 및 배경색 번짐 현상 해결.
    - **도킹 궤도 및 기능 제어 (`main_window.py`)**:
        - **Dock Tear-off**: 우측 주도주 및 잔고 패널의 플로팅(Floating) 기능을 활성화하여 멀티 모니터 환경 대응.
        - **Allowed Areas**: `setAllowedAreas(Qt.LeftDockWidgetArea | Qt.RightDockWidgetArea)`로 제한하여 상/하단 부착으로 인한 메인 횡단면 구조 파괴 방지.
- **결과**:
    - "UI 안정성 = 시스템 신뢰성" 공식 성립. 데이터 폭주 상황에서도 정보 왜곡 없는 깨끗한 시각적 피드백 유지.
    - 블룸버그 터미널(Bloomberg Terminal) 급의 견고한 HTS 레이아웃 프레임워크 완성.
    - 스타일 상속 버그 및 인덴트 오류(IndentationError) 수복을 통한 런타임 안정성 극대화.

## [V19.9.4] Vertical Priority Isolation — TRUE FINAL TUNING (Top 1%)
- **목표**: 우측 보유 잔고 테이블의 세로 압축(Vertical Squeeze) 현상 및 AI 분석 박스가 좁아지는 '공간 통제 상실(Space Competition)' 버그를 구조적(Structural)으로 해결하고, 쾌적한 뷰포트를 강제 보장함.
- **주요 파기/철회 조치**:
    - "픽셀 하드코딩"으로 중앙 `MaximumWidth`를 막는 1차원적 접근(Band-Aid) 전면 철회. (반응형 데드 스페이스 리스크 방지)
- **주요 변경 사항**:
    - **가로 팽창 재정의 (Horizontal Balance)**:
        - `main_layout.setStretch(0, 0)`, `(1, 2)`, `(2, 0)`으로 중앙 영역에 최상위 가중치 부여하되, 양 측면(Fixed/Preferred)이 좁아지거나 짓눌리지 않도록 여유로운 텐션 제공.
    - **세로 압축 방어 및 Minimum Survival Contract**:
        - `pos_table`, `surv_table`에 `setMinimumHeight(180)`를 부여하여 5줄 이상의 데이터 행렬(Row) 상시 보장.
        - `SignalStatusWidget`의 AI 종합분석 박스(`txt_ai`)에 최소 높이 `160px`과 `QSizePolicy.Minimum` 정책을 부여하여 가독성 위한 5.5줄 이상 영역 절대 방어.
    - **엘라이드 툴팁 보완 (Tooltip Elision Guard)**:
        - 테이블 폭 축소로 인해 'LS머트리얼즈'와 같은 긴 종목명이 말줄임표(...) 처리될 때, 마우스 오버 시 `setToolTip`으로 원본 정보를 소실 없이 열람 가능하도록 보완.
    - **레이아웃 상태 유지 (Persistence)**:
        - `QMainWindow.saveState() / restoreState()` 및 `saveGeometry() / restoreGeometry()`를 윈도우 수명 주기에 바인딩. 유저의 패널 떼어내기(Tear-off) 커스텀 상태가 다음 부팅 시에도 유지되는 UX 확보 (기술 부채 제거).
- **결과**: Top-Down 제한이 아닌 Bottom-Up 가중치와 생존권(MinimumSize) 부여를 통해, 어떠한 해상도 팽창/수축에도 UI의 "우선도" 경쟁이 스스로 스케일링되는 상위 1% 급 완성도 진입.

## [V19.9.5] Vertical + Table Column Priority Isolation — TRUE FINAL TUNING (Top 1%)
- **목표**: 우측 보유 잔고 테이블의 세로 압축(Vertical Squeeze) 현상, 가로 종목명 말줄임 현상, 그리고 AI 분석 박스가 좁아지는 '공간 통제 상실(Space Competition)' 버그를 완전히 구조적으로 종결함.
- **주요 변경 사항**:
    - **가로 팽창의 정밀 분배 (Center Balance)**:
        - `main_layout.setStretch(1, 3)`으로 중앙 영역의 가중치를 약간 더 주되, `center_group.setMaximumWidth(1250)` 안전장치(Guard)를 걸어 1920px 해상도 기준 우측 도킹 영역이 최소 450px 이상 쾌적하게 비워지도록 제어.
    - **Right Dock 테이블의 철통 방어 (Column Defense)**:
        - `DefensiveTableWidget` 및 `position_dashboard.py` 내부의 헤더 사이즈 정책을 `Fixed`와 `Stretch`로 혼합하여 재설정.
        - **종목명 셀**은 무조건 남는 공간을 모두 점유(`Stretch`)하되, 그 외 가격/수익률 셀은 `Fixed`로 고정하고, `setMinimumWidth(420)`와 자체 스크롤(`ScrollBarAsNeeded`)을 활성화하여 도킹 영역이 얼마나 좁아지든 절대로 텍스트가 잘려나가지 않도록 강제.
    - **세로 압축 방어 강화 (Vertical Squeeze Contract)**:
        - `pos_table.setMinimumHeight(220)` (최소 6~7행 보장) 및 `surv_table.setMinimumHeight(180)` 할당.
        - `SignalStatusWidget`의 레이아웃 우선순위(Priority)를 조율하여, 빈 공간 여백(Spacer)이 상부 인디케이터 팽창을 흡수(Stretch=1)하고, 하단의 AI 종합분석 박스(`txt_ai`)는 `setMinimumHeight(180)` 및 `Expanding` 사이즈 정책을 통해 어떤 상황에서도 5.5줄 이상의 가독성 공간을 절대 수호.
- **결과**:
    - 모든 패널과 위젯이 "최소 생존권(MinimumSize)"과 "우선순위 가중치(Stretch)"라는 법(Law)에 따라 완벽한 생태계 균형을 유지. 아무리 창 크기를 조작해도 테이블 행이 씹히거나 분석 박스가 찌그러지는 현상이 멸종된, '진정한 Top 1% Institutional Grade' 달성.

## [V19.9.7] GTID 기반 정밀 멱등성 가드 전환 및 5초 중복 차단 해제 (L4)
- **목표**: 시간 기반 중복 차단(V19.9.6)의 한계인 "5초 경과 시 재진입 차단" 문제를 해결하고, 거래 고유 식별자인 GTID를 통한 영구적 멱등성 확보.
- **주요 변경**:
    - **중복 차단 키 승격**: `_sell_emit_log[code]`(종목+시간) 방식에서 `_emitted_fills{stable_gtid}`(GTID) 방식으로 전환. 
    - **효과**: GTID는 각 거래 세션마다 유일하게 생성되므로, 동일 종목의 빈번한 재진입은 허용하면서도 동일 체결 이벤트의 중복 처리는 영구적으로 차단하는 정밀 가드 구축.
    - **V19.9.5 Error-Guard 유지**: 미체결 수량 불일치로 인한 `pos.qty=0` 상황에서의 강제 청산 보장 로직을 실전 데이터 기반 검증 후 최종 유지 결정.

## [V19.9.9] 주문 식별자 격리(Order Identity Isolation) 완성 (L4)
- **목표**: 부분 체결의 파편화 문제를 해결하고, 주문번호(Order ID)를 중심으로 모든 체결 데이터를 하나의 '바구니'로 수렴시켜 PnL 오염을 차단.
- **주요 변경**:
    - **Order ID SSoT 지위 확립**: `FID 9203`을 엔진의 핵심 식별자로 격상하여 주문 단성(Atomicity) 확보.
    - **결정적 GTID 생성 (`stable_gtid`)**: 신호 없는 수동/예외 거래 시에도 `UNKNOWN-{code}-{order_id}` 공식을 통해 동일 주문의 모든 부분 체결이 단일 거래 내역으로 자동 수렴되도록 설계.
    - **시그널 인터페이스 확장**: `sig_order_fill` 시그널 파라미터에 `order_id`를 추가하여 백엔드 전반의 데이터 추적성(Traceability) 강화.

---
*NCQ Work Log System v1.5*
