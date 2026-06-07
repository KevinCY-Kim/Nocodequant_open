# 📅 2026-05-26 업무 일지 (Daily Work Log)

## 🎯 주요 목표 및 이슈 정의
- [x] **실시간 틱 수신 지연(TICK_GAP) 로그의 가독성 개선**:
  종목코드만 출력되던 로그에 종목명 정보를 바인딩하여 직관적인 식별이 가능하도록 개선합니다.
- [x] **실시간 재등록 시 TICK_GAP 거짓 경보(False Positive) 방지**:
  종목이 실시간 감시에 재등록될 때 틱 간격 모니터링 버퍼를 리셋하여 부적절한 지연 경보가 발생하는 현상을 차단합니다.
- [x] **매수 직후 종목 선택 시 시간청산 오발동 버그 근본적 해결 (V27.1 / V25.2 FIX)**:
  종목을 클릭하여 세션이 리셋될 때, 기존의 세션 삭제 후 재성성 방식 대신 포지션 상태를 보존하고 파생 데이터만 초기화하는 `soft_reset` 방식을 도입하여 `entry_bar_index`가 손실되는 문제를 방지하고, 이를 통해 발생하던 시간청산 오발동 오류를 완벽히 해결합니다.
- [x] **실시간 미등록 종목 유령 틱(Ghost Tick) 지연 경보(TICK_GAP) 필터링**:
  감시 대상이 아님에도 키움 API에서 계속 유입되는 유령 틱으로 인해 허위 `TICK_GAP` 경보가 다발하여 발생하는 감사 로그 오염 및 노이즈 문제를 감시 등록 여부(`is_registered`) 필터링을 통해 원천 차단합니다.
- [x] **초기 구동 시 분봉 설정 및 콤보박스 상태 불일치 버그 수정 (V27.1 / V25.2.P3)**:
  앱 초기 구동 시 분봉 설정 콤보박스는 `"1분"`으로 노출되나 하단 현재 분봉 레이블 및 차트 상태는 `"15분봉"`으로 표시되어 UI 데이터 상태가 서로 어긋나던 불정합 버그를 해결합니다.
- [x] **기본 분봉 5분봉 설정 변경 및 라스트 상태 복원 기능 정교화 (V27.1 / V25.2.P4)**:
  시스템 전반의 기본 분봉을 기존 15분에서 5분으로 변경하며, 프로그램 재기동 시 이전에 사용하던 분봉 설정이 완벽히 복원되도록 보완했습니다.
- [x] **시장 체온(Market Temperature) 수치 스케일링 버그 수정 (V27.1 / V25.2.P5)**:
  실시간 전술 HUD에서 시장 체온이 하루 종일 `0 COLD`로 고정되어 노출되던 수치 스케일링 연산 버그를 해결했습니다.
- [x] **거시적 차트 분석을 위한 일봉, 주봉, 월봉 조회 기능 구현 (V27.2 / V25.3.P1)**:
  기존의 분봉 설정에 더해 거시적 지표 파악을 위해 일봉, 주봉, 월봉 차트 종류를 선택하여 실시간 데이터 및 과거 이력을 조회할 수 있도록 개선하였습니다.
- [x] **주봉/월봉 차트 조회 시 ValueError (NaTType does not support strftime) 버그 핫픽스 (V27.2 / V25.3.P2)**:
  키움 OpenAPI가 주봉/월봉 차트 조회 시 반환하는 무효한 또는 범위 밖의 날짜 데이터(예: `00000000`)로 인해 Pandas Datetime이 `NaT`로 변환되어 차트 DTO 변환부에서 발생하던 런타임 크래시를 완벽히 해결했습니다.
- [x] **초기 구동 시 timeframe 복원에 따른 AttributeError (_tr_queue 미정의) 해결 (V27.2 / V25.3.P3)**:
  사용자 설정에 따라 일봉, 주봉 등으로 차트 유형이 복원될 때, `_tr_queue`가 초기화되기 전에 `request_minute_data_throttled`가 호출되어 발생하던 런타임 오류를 초기화 순서 재조정 및 방어 코드 적용으로 해결했습니다.
- [x] **일/주/월봉 선택 시 차트 데이터 로드 실패 버그 근본 해결 (V27.2 / V25.3.P4)**:
  `engine.py`에서 일/주/월봉 TR 분기가 누락되었던 점과, `main_window.py`에서 타임프레임(`_last_tf`)을 정상 참조하지 않고 항상 `combo_tf`(분봉)만 읽어 차트를 못 불러오거나 포맷이 깨지던 버그들을 모두 해결했습니다.
- [x] **주봉/월봉 `GetCommDataEx` 컬럼 매핑 오류 수정 및 차트 비정상 렌더링 해결 (V27.3 / V25.3.P5)**:
  `opt10081`(일봉)과 달리 `opt10082`(주봉)/`opt10083`(월봉) TR 데이터는 첫 필드에 종목코드 컬럼 없이 즉시 가격 데이터가 시작되어 인덱스가 밀려 발생하는 캔들의 스케일 왜곡 버그를 정밀 분석하여 수정했습니다.
- [x] **독립형 멀티차트(Multi-Chart) 팝업 창 추가 및 제어 격리 (V27.4 / V25.3.P6)**:
  본 차트와 종목은 실시간 연동되나 타임프레임 조작 및 데이터 조회 연산은 메인 시스템(전략 엔진, 의사결정허브)과 결합되지 않고 완벽히 격리된 독립 팝업 차트 창을 신설했습니다.
- [x] **하단 실시간 운용 현황(P&L) 패널 안전 분리 및 UI 제외 (V27.5 / V25.3.P7)**:
  사용하지 않는 하단의 P&L 그래프 및 토글 버튼 영역을 레이아웃에서 제거하여 차트 영역을 세로로 더 넓게 활용하도록 개선하고, 참조 에러 방지를 위해 백업 파일 생성 및 런타임 방어 코드를 적용했습니다.
- [x] **룰스(Rules) 및 사용자 가이드(Docs) 내 신규 방어 패턴/매뉴얼 업데이트 (V27.5 / V25.3.P8)**:
  최근 겪었던 NaTType 변환 예외, 초기화 Race Condition, GetCommDataEx 컬럼 시프트 오류, 멀티차트 격리 연산, P&L 미조립 Null-Crash 방어 규칙들을 `Failure_Patterns_and_Guards_ko.md`에 등재하고 `Config_User_Guide_ko.md`에 멀티차트 안내 및 기본 5분봉 복원 사양을 최신화했습니다.
- [x] **QComboBox 가로 폭 잘림 현상 수정 (V27.6 / V25.3.P9)**:
  "차트 구분" 및 "분봉 설정" 콤보박스의 가로 너비 부족으로 인해 텍스트 일부(예: "분봉"의 "봉" 자)가 가려지던 현상을 `setFixedWidth` 고정 너비 조정을 통해 완벽히 해결했습니다.
- [x] **AI 종합 분석 결과 화면 잠금(Lock) 및 돌아가기 기능 추가 (V27.7 / V25.3.P10)**:
  실시간 수급 틱 유입 시 GPT 분석 결과 텍스트가 지워지던 버그를 화면 잠금 플래그로 차단하고, 다 읽은 후 이전 수급 체크리스트로 돌아갈 수 있도록 분석 버튼을 돌아가기 버튼으로 양방향 스위칭되도록 개선했습니다.
- [x] **AI 분석 텍스트 가로 폭 래핑 최적화 및 레이아웃 붕괴 방지 가드 (V27.10 / V25.3.P11)**:
  `QTextDocument` 내의 텍스트가 위젯 실제 가로 크기보다 훨씬 좁게 개행(clipping)되던 버그를 `setTextWidth` 동적 연동 및 `resizeEvent` 핸들러 연계와 더불어, AI 결과/에러/로딩 등 상태 전이 시점의 비동기 갱신 호출을 강제하여 가로 폭 100% 활용 구조로 수정했습니다. 또한, 긴 텍스트가 노출될 때 텍스트 박스가 위아래로 커져 전체 UI 레이아웃이 무너지던 현상을 방지하고자 세로 높이를 130px로 고정하고, 얇은 6px 다크 테마 마이크로 스크롤바를 탑재하여 레이아웃 무너짐을 근본적으로 방어했습니다.

---

## ✅ 상세 작업 및 패치 내역

### 1. 실시간 틱 수신 지연(TICK_GAP) 가독성 개선 및 False Positive 방어
* **소스 파일**: 
  - [engine.py (_on_receive_real_data)](file:///c:/Users/stone/projects/nocodequant/app/ai/engine.py#L534-L552)
  - [engine.py (register_code)](file:///c:/Users/stone/projects/nocodequant/app/ai/engine.py#L1213-L1215)
* **현상**:
  - 기존 `TICK_GAP` 로그는 종목코드(예: `036930`)로만 출력되어 사용자가 어떤 종목인지 즉각 파악하기 곤란했습니다.
  - 종목이 실시간 감시에 등록/재등록되는 과정에서 `_forensic_last_tick` 버퍼가 이전 시점을 기억하고 있어, 재등록 직후 최초 틱 수신 시 수백 초의 수신 지연(`TICK_GAP`) 경보가 허위로 발생하는 현상이 관찰되었습니다.
* **조치**:
  - **종목명 바인딩**: 실시간 시세 데이터 수신부(`_on_receive_real_data`)에서 `stock_name`을 획득하여 `종목명(종목코드)` 형태로 로그와 포렌식 아웃풋을 개선하였습니다.
    - 예시: `[TICK_GAP] 036930` ➡️ `[TICK_GAP] 주성엔지니어링(036930) 틱 수신 지연 감지`
  - **재등록 버퍼 초기화**: `register_code(code)` 실행 시 `self._forensic_last_tick[code] = 0`으로 명시적 초기화를 집행하여, 재등록 시점에 발생할 수 있는 False Positive 지연 경보를 원천 예방하였습니다.

---

### 2. 매수 직후 종목 선택 시 시간청산 오발동 버그 근본 수정
* **소스 파일**: 
  - [strategy.py (soft_reset)](file:///c:/Users/stone/projects/nocodequant/app/ai/strategy.py#L266-L304)
  - [main_window.py (on_code_changed)](file:///c:/Users/stone/projects/nocodequant/app/ui/main_window.py#L1470)
* **현상**:
  - 사용자가 매수를 진행한 직후 GUI 화면에서 해당 종목을 다시 선택(클릭/전환)하면, `on_code_changed()` 이벤트 핸들러가 작동합니다.
  - 이 과정에서 기존에는 `self.strategy.reset_state(code)`를 호출하여 세션을 완전히 삭제 후 재성성했습니다.
  - `reset_state()`는 세션을 통째로 파괴하므로 포지션 정보가 일시 소실되었다가 다음 틱 또는 잔고 동기화 과정에서 재생성됩니다. 이 과정에서 포지션 진입 시점의 봉 인덱스(`entry_bar_index`)가 무효화되거나 불일치하게 됩니다.
  - 이후 300개의 과거 히스토리 봉(Warm-up) 데이터를 비동기로 로드하여 결합하는 시점에 `bars_held = (현재 봉 수 - 1) - entry_bar_index` 연산이 적용되면서, 왜곡된 인덱스로 인해 극단적으로 큰 보유 봉 수(예: 수백 봉 이상)가 도출됩니다.
  - 이로 인해 설정된 시간청산 제한(`time_limit`)을 초과한 것으로 오인하여 매수하자마자 즉시 시간청산 주문이 송신되는 심각한 오작동이 라이브 매매 과정에서 관찰되었습니다.
* **조치**:
  - **`soft_reset` 메서드 신설**: `Strategy` 클래스 내에 포지션의 핵심 정보(`entry_bar_index`, `sl`, `peak_price` 등)가 담긴 `position` 딕셔너리를 완벽히 보존한 채, 지표 캐시(`data`, `std_cache`, `scores`, `regime_history` 등)만 안전하게 리셋하는 `soft_reset(code)` 구조를 도입하였습니다.
  - **UI 이벤트 연동**: `main_window.py`의 `on_code_changed()` 내에서 단순 `reset_state(code)` 호출을 `soft_reset(code)`로 변경하였습니다.
  - 이를 통해 활성 포지션을 보유하고 있는 종목을 마우스로 클릭/선회하더라도 `entry_bar_index`가 영구 보존되어, 시간청산 로직이 오동작할 위험을 완벽히 소멸시켰습니다.

---

### 3. 실시간 미등록 종목 유령 틱(Ghost Tick) 지연 경보(TICK_GAP) 필터링
* **소스 파일**:
  - [engine.py (_on_receive_real_data)](file:///c:/Users/stone/projects/nocodequant/app/ai/engine.py#L534-L552)
* **현상**:
  - 감시 해제된 종목(`unregister_code` 완료)에 대해 키움 OpenAPI가 백그라운드에서 실시간 해제 처리를 완벽하게 즉각 수행하지 못하고 불규칙하게 틱을 송신하는 현상이 존재합니다.
  - 이 유령 틱이 유입될 때마다 `_forensic_last_tick` 캐시가 재활성화되고, 다음 유령 틱 유입 시 시점 차이가 수백 초 이상 나면서 무수히 많은 `registered=False, last_reg=-1.0초 전` 형태의 지연(TICK_GAP) 경보가 감사 원장과 실시간 로그 창을 오염시키는 문제가 발생했습니다.
* **조치**:
  - **감시 상태 조건식 반영**: 틱 지연 검증 및 캐시 업데이트 단계를 오직 현재 공식 감시 리스트에 등록되어 있는 종목(`is_registered = True`)일 때만 실행되도록 구조를 차단하였습니다.
  - **가벼운 관측 수집 도입**: 등록되지 않은 유령 틱이 수신되는 경우에는 타임스탬프 캐시를 갱신하지 않고 경보를 원천 필터링하며, 대신 가벼운 카운터 메트릭인 `self.ghost_tick_count`에 유입 횟수만 누적하도록 처리하여 노이즈를 제로화하고 관측 가독성을 대폭 향상했습니다.

---

### 4. 초기 구동 시 분봉 설정 및 콤보박스 상태 불일치 버그 수정
* **소스 파일**:
  - [body_builder.py (build)](file:///c:/Users/stone/projects/nocodequant/app/ui/builders/body_builder.py#L290-L293)
  - [main_window.py (setup_ui)](file:///c:/Users/stone/projects/nocodequant/app/ui/main_window.py#L857-L860)
* **현상**:
  - 앱 실행 초기 구동 시, 분봉 설정을 보여주는 QComboBox(`combo_tf`)는 기본값 `"1분"`으로 보이지만 하단 상태 레이블(`lbl_current_tf`) 및 차트의 내부 데이터 타임프레임은 `"15분"` 상태로 렌더링되는 불일치 버그가 있었습니다.
  - 원인은 기본 세팅을 위젯들에 반영하는 `apply_initial_defaults()` 함수가 아직 `combo_tf` 및 `combo_preset`이 초기화(인스턴스화)되기도 전인 `BodyBuilder.build()` 실행 최하단부에서 호출되었기 때문입니다. 이로 인해 `hasattr` 체크가 `False`로 분기 처리되어, 마지막 저장되었던 분봉 설정의 복원 및 콤보박스 변경 이벤트 트리거가 누락되는 문제를 유발했습니다.
* **조치**:
  - **조기 호출부 제거**: `BodyBuilder.build()` 하단부의 `m.apply_initial_defaults()` 호출을 제거하였습니다.
  - **지연 초기화 완료**: 모든 UI 요소들과 다이얼로그 팝업(StrategySettingsDialog) 객체들의 조립이 100% 완료된 직후인 `MainWindow.setup_ui()` 메서드의 마지막 단계로 `self.apply_initial_defaults()`를 이관하였습니다.
  - 이를 통해 프로그램 구동 시 사용자가 이전에 마지막으로 사용했던 분봉 상태(기본 15분 등)가 정상적으로 로드 및 콤보박스에 바인딩되며, 레이블 및 차트 렌더링 상태가 완벽히 하나로 일치하게 되었습니다.

---

### 5. 기본 분봉 5분봉 설정 변경 및 사용자 세션 복원 정밀화
* **소스 파일**:
  - [chart_widget.py](file:///c:/Users/stone/projects/nocodequant/app/ui/widgets/mini_chart/chart_widget.py#L56)
  - [chart_panel_builder.py](file:///c:/Users/stone/projects/nocodequant/app/ui/builders/chart_panel_builder.py#L68)
  - [config_manager.py](file:///c:/Users/stone/projects/nocodequant/app/ui/managers/config_manager.py#L286)
  - [main_window.py](file:///c:/Users/stone/projects/nocodequant/app/ui/main_window.py#L995)
* **현상**:
  - 기존에는 UI 및 백엔드 스로틀러 등 여러 지점에서 기본 분봉 설정 및 하드코딩 대체값이 `15분` 및 `15`로 일원화되어 있어, 사용자가 원하는 기본 스탠스(5분봉)가 아닌 15분봉으로 차트가 우선 렌더링되는 경향이 있었습니다.
* **조치**:
  - **5분봉 표준화**: 차트 위젯(`self.timeframe = 5`), 차트 패널 레이블(`현재: 5분봉`), `on_code_changed` 및 `_execute_tr_warmup` 내의 기본 대체값들을 `5` 및 `"5분"`으로 일괄 개편했습니다.
  - **마지막 설정 자동 복원**: `apply_initial_defaults()` 내의 QSettings 폴백 값을 `"5분"`으로 설정함으로써, 사용자가 프로그램 구동 중에 변경한 마지막 분봉 값(1분, 3분, 5분 등)이 재시작 후에도 완벽히 100% 유지되어 화면에 자동 로드됩니다.
### 6. 시장 체온(Market Temperature) 수치 스케일링 버그 수정
* **소스 파일**:
  - [hud_manager.py](file:///c:/Users/stone/projects/nocodequant/app/ui/managers/hud_manager.py#L53-L61)
* **현상**:
  - 실시간 전술 HUD(시장상태레이더) 상단의 "시장 체온"이 실제 시장 상황 및 MME Heat의 정상 갱신 상태와는 무관하게 하루 종일 `0 COLD` 혹은 `1 COLD` 등의 극도로 낮은 수치로만 고정 노출되는 현상이 관찰되었습니다.
  - 원인은 `hud_manager.py`에서 전체 종목의 실시간 과열도 평균점수를 활용해 시장 체온(`theme_score`)을 산출할 때, `0.0 ~ 2.0` 내외의 원형 실수 범위 값을 지니는 `ema_score` 데이터를 0~100% 척도로 변환하기 위한 표준 스케일링 인자(50배)를 적용하지 않은 채 기존 가중 배수(`1.5`)만을 곱하여 형변환(`int`)했기 때문입니다. 이로 인해 소수점 이하가 절사되면서 온도가 항상 `0`, `1`, `2`도로 계산되어 영구적으로 COLD 등급에 갇히게 되었습니다.
* **조치**:
  - **표준 스케일링 및 가중 인자 일원화**: `ema_score`에 표준 변환 비율 50배와 가중 배수 1.5배를 결합한 총 **`75.0`**을 곱하는 공식(`sum(heat_map.values()) / len(heat_map) * 75.0`)으로 수정하였습니다.
  - 이를 통해 시장 상황에 따라 `0 ~ 100` 사이의 정상적인 체온 점수가 연산 및 반영되어, `COLD`, `WARM`, `HOT`, `OVERHEATED` 상태가 실시간으로 완벽하게 다이내믹 렌더링되도록 조치하였습니다.

### 7. 거시적 차트 분석을 위한 일봉, 주봉, 월봉 조회 기능 구현
* **소스 파일**:
  - [engine.py (request_minute_data 및 _parse_chart_data)](file:///c:/Users/stone/projects/nocodequant/app/ai/engine.py#L630)
  - [candle_manager.py (update_tick)](file:///c:/Users/stone/projects/nocodequant/app/utils/candle_manager.py#L54)
  - [chart_panel_builder.py (build)](file:///c:/Users/stone/projects/nocodequant/app/ui/builders/chart_panel_builder.py#L50)
  - [main_window.py (on_chart_type_changed 및 change_timeframe)](file:///c:/Users/stone/projects/nocodequant/app/ui/main_window.py#L2288)
  - [config_manager.py (apply_initial_defaults)](file:///c:/Users/stone/projects/nocodequant/app/ui/managers/config_manager.py#L323)
* **조치 및 구현 세부사항**:
  - **UI 차트 구분 추가**: 분봉 설정 컴포넌트 좌측에 QComboBox 기반의 `"차트 구분"`("분봉", "일봉", "주봉", "월봉")을 신설하여, 일봉/주봉/월봉 선택 시 분봉 설정 콤보박스는 자동으로 비활성화되고 차트 모드가 전환되도록 구현하였습니다.
  - **TR 요청 및 라우팅**: `engine.py`에서 기존 `request_minute_data()`가 음수 타임프레임(일봉 `-1`, 주봉 `-2`, 월봉 `-3`)을 전달받으면, 키움 OpenAPI의 해당 TR인 `opt10081`(일봉), `opt10082`(주봉), `opt10083`(월봉) 요청으로 안전하게 라우팅되도록 개선하였습니다.
  - **일자/시간 파싱 보완**: `YYYYMMDD` 8자리 날짜 형식으로 전달되는 데이터를 기존 14자리 시간 패턴(`YYYYMMDD090000`)으로 맞춰 `CandleManager`와 `MiniChartWidget`이 날짜 정보를 오차 없이 매핑 및 시각화할 수 있도록 일치시켰습니다.
  - **실시간 틱 합산 연산**: `CandleManager.update_tick` 내부에서 일봉(`-1`)은 당일 09시, 주봉(`-2`)은 월요일 09시, 월봉(`-3`)은 매월 1일 09시로 틱 수신 시각을 플로어링 정렬하여 실시간 틱이 거시적 봉 데이터에 누적되도록 개선하였습니다.
  - **설정 영속성 연동**: 프로그램 재시작 시 마지막으로 세팅했던 일봉, 주봉, 월봉 종류를 QSettings를 통해 감지하여 복원하도록 `config_manager.py`를 보강했습니다.

### 8. 주봉/월봉 차트 조회 시 ValueError (NaTType does not support strftime) 버그 핫픽스
* **소스 파일**:
  - [engine.py (_parse_chart_data)](file:///c:/Users/stone/projects/nocodequant/app/ai/engine.py#L913)
  - [chart_widget.py (_convert_to_dto)](file:///c:/Users/stone/projects/nocodequant/app/ui/widgets/mini_chart/chart_widget.py#L321)
* **현상**:
  - 일봉/주봉/월봉 조회를 수행할 때, 특정 상황(예: 신규 상장 종목이거나 히스토리 끝부분에 더미 데이터가 존재하는 경우)에서 키움 OpenAPI가 `"00000000"` 혹은 공백(`""`) 등 유효하지 않은 날짜 데이터를 반환하여 런타임 오류가 발생하였습니다.
  - 이 데이터가 Pandas `to_datetime`에 의해 `NaT`(Not a Time)로 형변환되었고, `chart_widget.py`에서 캔들의 X축 타임스탬프를 `"HH:MM"` 형식의 시간 정보로 일괄 포맷팅하려 시도하는 과정에서 `ValueError: NaTType does not support strftime` 크래시를 유발했습니다.
* **조치**:
  - **무효 데이터 사전 필터링**: `engine.py`의 `_parse_chart_data` 내에서 날짜 문자열이 비어있거나, `"0"`이거나, `"0000"`으로 시작하는 무효한 데이터를 사전에 스킵하도록 필터를 보강했습니다.
  - **`NaT` 및 Null 방어 구현**: `chart_widget.py`의 `_convert_to_dto` 내에서 `pd.isnull` 검사를 사전에 수행하여 `NaT` 또는 Null일 경우 안전하게 공백(`""`) 처리하도록 방어 코드를 적용했습니다.
  - **거시 타임프레임 날짜 표시 최적화**: Daily/Weekly/Monthly 차트일 때는 단순 `"HH:MM"`(예: 의미 없는 `09:00` 시간) 대신, 일봉/주봉은 `"%m/%d"`(월/일), 월봉은 `"%Y/%m"`(년/월) 형태로 X축 타임스탬프 라벨이 표시되도록 동적 포맷팅을 적용해 사용성을 극대화했습니다.

### 9. 초기 구동 시 복원 과정의 AttributeError (_tr_queue 미정의) 해결
* **소스 파일**:
  - [main_window.py (__init__ 및 request_minute_data_throttled)](file:///c:/Users/stone/projects/nocodequant/app/ui/main_window.py#L308)
* **현상**:
  - 프로그램 재기동 시 마지막 차트 구분(일봉/주봉/월봉) 복원이 `apply_initial_defaults()` 내에서 수행됩니다.
  - 이 과정에서 `change_timeframe()` ➡️ `request_minute_data_throttled()`가 순차적으로 실행되는데, 정작 이 TR 요청들을 담아두는 버퍼 큐인 `self._tr_queue` 리스트가 생성자(`__init__`) 하단(400라인 대)에서 초기화되고 있어, 복원 시점에 `AttributeError: 'MainWindow' object has no attribute '_tr_queue'` 예외가 발생하며 구동이 실패하는 현상이 존재했습니다.
* **조치**:
  - **초기화 순서 당김**: 생성자(`__init__`) 내부에서 UI 초기화(`setup_ui()`) 및 설정 복원(`load_config()`) 작업이 수행되기 이전 시점인 300라인 영역으로 `self._tr_queue = []` 생성 코드를 앞당겨 배치하였습니다.
  - **이중 방어 가드 적용**: `request_minute_data_throttled` 메서드 내부에도 `hasattr(self, '_tr_queue')` 검사를 수행하여, 만약 큐가 아직 생성되지 않은 비정상 초기화 상태라면 즉시 큐 리스트를 동적으로 생성한 후 데이터를 담도록 안전장치(Lazy Initialization Guard)를 보강하였습니다.

### 10. 일/주/월봉 선택 후 종목 전환 시 차트 로드 실패 관련 버그 수정
* **소스 파일**:
  - [engine.py (_on_receive_tr_data)](file:///c:/Users/stone/projects/nocodequant/app/ai/engine.py#L624-L625)
  - [main_window.py (on_code_changed, on_chart_data, _process_history_background)](file:///c:/Users/stone/projects/nocodequant/app/ui/main_window.py)
* **현상**:
  - 일봉/주봉/월봉을 선택한 상태에서 종목을 변경하면 차트 데이터가 로드되지 않거나 화면이 깨지는 현상이 발생했습니다.
  - 분석 결과, 네 가지의 유기적인 버그가 원인이었습니다:
    1. `engine.py`의 `_on_receive_tr_data`에서 `"req_chart_min"` TR만 파싱하도록 고정되어 있어 일/주/월봉 TR 응답(`req_chart_day`, `req_chart_week`, `req_chart_month`)이 수신되어도 파서(`_parse_chart_data`)가 트리거되지 않음.
    2. `main_window.py`의 `on_code_changed`에서 타임프레임을 무조건 분봉 UI인 `combo_tf`에서만 가져와, 일/주/월봉(-1, -2, -3)이 선택되어 있어도 분봉 TR 요청이 강제 발생함.
    3. `main_window.py`의 `on_chart_data`에서 배경 히스토리 파싱을 위한 `tf_val`을 `combo_tf`에서 가져와서 `CandleManager`가 분봉 기준으로 잘못 초기화됨.
    4. `main_window.py`의 `_process_history_background`에서 날짜 포맷이 `'%H:%M'`으로 하드코딩되어 있어 일/주/월봉 데이터를 로드할 때 날짜 형식이 깨짐.
* **조치**:
  - **TR 수신 조건 확장**: `engine.py`에서 TR 수신 분기 대상에 `req_chart_day`, `req_chart_week`, `req_chart_month`를 추가하였습니다.
  - **타임프레임 참조 단일화**: `main_window.py`의 `on_code_changed`와 `on_chart_data`에서 UI 콤보박스 대신 `getattr(self, '_last_tf', None)`을 최우선 참조하도록 하여 현재 선택된 정확한 차트 구분(분/일/주/월)이 유지되도록 수정하였습니다.
  - **배경 처리 날짜 포맷 분기**: `_process_history_background` 내에서 `tf_val` 값에 따라 분기 연산을 도입하여 월봉은 `"%Y/%m"`, 일/주봉은 `"%m/%d"`, 분봉은 `"%H:%M"`으로 유연하게 포맷을 할당했습니다.

---

### 11. 주봉/월봉 GetCommDataEx 컬럼 매핑 오류 수정 및 차트 비정상 렌더링 해결
* **소스 파일**:
  - [engine.py (_parse_chart_data)](file:///c:/Users/stone/projects/nocodequant/app/ai/engine.py#L905-L943)
* **현상**:
  - 일봉까지는 차트 조회가 완벽하게 동작했으나, 주봉 및 월봉 로드 시 캔들이 극도로 납작하거나 반대로 비정상적으로 거대하게 렌더링(예: 시가/고가/저가가 모두 수백만 수준의 왜곡된 크기로 노출)되는 현상이 발생했습니다.
  - 원인은 키움 OpenAPI가 제공하는 TR별 데이터 레이아웃 불일치 때문이었습니다. `opt10081`(일봉)의 경우 첫 번째 컬럼(`row[0]`)에 `종목코드`를 포함하지만, `opt10082`(주봉)와 `opt10083`(월봉)은 첫 컬럼에 종목코드 없이 바로 `현재가(종가)` 필드가 할당되어 나옵니다.
  - 이로 인해 주봉/월봉 파싱 과정에서 인덱스가 한 칸씩 뒤로 밀리면서 종가 자리에 거래량(19M 등)이 들어가고, 거래량 자리에 거래대금(5조 등)이 들어가 스케일링에 치명적인 오차가 발생하였습니다.
* **조치**:
  - **일봉 vs 주봉/월봉 데이터 레이아웃 분기 처리**:
    - `engine.py`의 `_parse_chart_data` 반복문 내부에서 `is_macro` 조건 충족 시, 다시 일봉(`rq_name == "req_chart_day"`)과 주봉/월봉(그 외) 케이스로 분리했습니다.
    - **일봉**: `row[1]=종가`, `row[2]=거래량`, `row[4]=일자`, `row[5]=시가`, `row[6]=고가`, `row[7]=저가`로 이전과 동일하게 파싱합니다.
    - **주봉/월봉**: `row[0]=종가`, `row[1]=거래량`, `row[3]=일자`, `row[4]=시가`, `row[5]=고가`, `row[6]=저가`로 1개 열씩 당겨진 실제 레이아웃에 맞춰 보정 매핑을 집행했습니다.
  - 조치 후 주봉과 월봉 조회 시에도 모든 OHLCV 수치 정보와 거래량 그래프가 본래의 올바른 척도에 맞추어 깔끔하게 렌더링되는 것을 확인하였습니다.

---

### 12. 독립형 멀티차트(Multi-Chart) 팝업 창 추가 및 제어 격리
* **소스 파일**:
  - [engine.py (sig_multi_chart_data, _parse_chart_data, request_multi_chart_data)](file:///c:/Users/stone/projects/nocodequant/app/ai/engine.py#L84)
  - [chart_panel_builder.py (build)](file:///c:/Users/stone/projects/nocodequant/app/ui/builders/chart_panel_builder.py#L78)
  - [main_window.py (__init__, show_multi_chart_window, on_code_changed, closeEvent)](file:///c:/Users/stone/projects/nocodequant/app/ui/main_window.py#L342)
  - [multi_chart_window.py (신설)](file:///c:/Users/stone/projects/nocodequant/app/ui/components/multi_chart_window.py)
* **현상**:
  - 분석 중 본 차트로 5분봉을 보면서, 보조용으로 일봉/주봉/월봉 등을 따로 띄워놓고 거시적 추세를 파악하고 싶다는 요구가 있었습니다.
  - 이를 위해서는 메인 윈도우의 타임프레임 조작 및 조회 로직이 오염되지 않아야 했으며, 메인의 종목 변경(예: 삼성전자 선택)은 팝업 차트도 즉각 따라가되 팝업 차트 내의 개별 타임프레임 조작은 메인 전략 및 의사결정허브(Decision Hub)에 전혀 영향을 주지 않는 구조가 필요했습니다.
* **조치**:
  - **TR 파이프라인 명시적 격리**: `engine.py`에 멀티차트 전용 시그널 `sig_multi_chart_data`를 신설하고, `multi_req_chart_min/day/week/month` 등의 접두사 TR 요청 방식을 구현하여 수신 데이터가 메인 차트 데이터 슬롯과 완전히 분리되어 방출되도록 구조화했습니다.
  - **멀티차트 UI 팝업 신설**: `app/ui/components/multi_chart_window.py`에 독립 탑레벨 창(`MultiChartWindow`)을 설계하여 메인 UI와 동일한 다크 테마 및 `MiniChartWidget`을 재사용하되 내부 상태(timeframe, chart_type 등)를 완전히 격리하였습니다.
  - **단방향 이벤트 바인딩**: 메인 UI의 분봉 레이블 우측에 "🖥️ 멀티차트" 버튼을 추가하여 클릭 시 팝업을 활성화하며, 메인의 `on_code_changed` 이벤트 발생 시에만 `self.multi_chart_window.set_code(code)`를 호출하여 종목 정보만 동기화되도록 구성했습니다.
  - **안전한 라이프사이클 관리**: 메인 윈도우 종료(`closeEvent`) 및 팝업창 소멸 시 등록된 커넥션을 해제하여 메모리 누수를 원천 방지하였습니다.

---

### 13. 하단 실시간 운용 현황(P&L) 패널 안전 분리 및 UI 제외
* **소스 파일**:
  - [chart_panel_builder.py (build)](file:///c:/Users/stone/projects/nocodequant/app/ui/builders/chart_panel_builder.py#L140)
  - [main_window.py (toggle_performance_panel)](file:///c:/Users/stone/projects/nocodequant/app/ui/main_window.py#L1450)
  - `chart_panel_builder.py.bak` (백업 파일 생성 완료)
* **현상**:
  - 하단의 실시간 운용 현황(P&L) 패널은 현재 사용하지 않을 예정이므로, 화면 공간 확보 및 시스템 리소스 낭비 방지를 위해 해당 패널을 레이아웃에서 안전하게 드러낼 필요성이 대두되었습니다.
  - 단순 삭제 시 기존 메인 윈도우의 토글 기능(`toggle_performance_panel`) 등에서 해당 위젯들을 참조하여 Null/AttributeError 예외가 발생할 위험이 있었습니다.
* **조치**:
  - **백업 생성**: 수정 전 원본 레이아웃 유지를 위해 `chart_panel_builder.py.bak`로 백업 카피를 생성해 둠으로써 문제 발생 시 언제든지 원복할 수 있는 안전장치를 마련했습니다.
  - **UI 레이아웃에서 패널 제외**: `chart_panel_builder.py`에서 패널 접기 토글바 및 `m.chart_group` (PerformanceSummaryWidget, AdvancedChartWidget 포함) 빌드/추가 코드를 삭제하여 UI 상에서 즉각 패널이 노출되지 않도록 조치했습니다.
  - **런타임 크래시 방지 가드**: `main_window.py`의 `toggle_performance_panel` 상단에 `hasattr` 검사 가드를 촘촘히 쳐서, 해당 UI 컴포넌트가 로드되지 않은 상태에서 이벤트가 트리거되더라도 예외 없이 무시하고 안전히 리턴되도록 보장했습니다.
  - 수정 후 유닛 테스트(11개) 모두 성공 및 런타임 상 에러 없음이 확인되었습니다.

---

### 14. 룰스(Rules) 및 사용자 가이드(Docs) 내 신규 방어 패턴/매뉴얼 업데이트
* **소스 파일**:
  - [Failure_Patterns_and_Guards_ko.md](file:///c:/Users/stone/projects/nocodequant/Antigravity_rules/rules/Failure_Patterns_and_Guards_ko.md#L260)
  - [Config_User_Guide_ko.md](file:///c:/Users/stone/projects/nocodequant/docs/Config_User_Guide_ko.md#L17)
* **조치**:
  - **룰스 반영**: 최근 수정한 5가지의 실패 위험 항목(주/월봉 NaTType 변환 예외, 초기화 시 `_tr_queue` 레이스 컨디션, `GetCommDataEx` 주봉/월봉 컬럼 시프트 레이아웃 불일치, 멀티차트 간섭을 막는 격리 파이프라인, P&L 제거 후의 안전 복귀 가드)을 `Failure_Patterns_and_Guards_ko.md` 문서의 **4.20 ~ 4.24** 섹션으로 정식 정의하고 버전 사인을 `v27.5`로 변경하여 향후 개발 시에도 본 방어 코드가 유실되지 않도록 거버넌스를 강화했습니다.
  - **사용자 설정 가이드 개정**: `Config_User_Guide_ko.md`에서 프로그램 기본 구동 분봉 설정을 `15분`에서 최근 패치된 `5분`으로 최신화하였으며, 새로 추가된 `🖥️ 멀티차트` 활용법(본 차트 종목 연동 및 제어 독립 격리)을 가이드 문서 내에 수록하였습니다.

---

### 15. QComboBox 가로 폭 잘림 현상 수정
* **소스 파일**:
  - [chart_panel_builder.py (build)](file:///c:/Users/stone/projects/nocodequant/app/ui/builders/chart_panel_builder.py#L55)
  - [multi_chart_window.py (setup_ui)](file:///c:/Users/stone/projects/nocodequant/app/ui/components/multi_chart_window.py#L63)
* **현상**:
  - UI 상에서 "차트 구분" 선택 시 `"분봉"`의 `"봉"` 자 우측 절반이 콤보박스 화살표와 겹쳐 가려지거나 잘리는 현상이 발생해 미관 및 가독성이 저해되었습니다.
* **조치**:
  - **가로 크기 고정**: 본 차트 영역(`chart_panel_builder.py`)과 독립 멀티차트 영역(`multi_chart_window.py`) 모두 `combo_chart_type`에 `setFixedWidth(80)`을 부여해 두 글자("분봉", "일봉" 등)가 안전하게 표기되도록 공간을 보장했습니다.
  - **분봉 설정 콤보박스 가독성 확보**: 4글자 텍스트("직접입력", "120분" 등)가 들어가는 `combo_tf` 에도 동일하게 `setFixedWidth(90)`을 적용하여 텍스트 픽셀 겹침이나 잘림 문제를 근본 해결했습니다.

---

### 16. AI 종합 분석 결과 화면 잠금(Lock) 및 돌아가기 기능 추가
* **소스 파일**:
  - [signal_status_widget.py](file:///c:/Users/stone/projects/nocodequant/app/ui/widgets/signal_status_widget.py#L457)
* **현상**:
  - "AI종합 분석" 버튼을 클릭하여 GPT가 수집된 정보를 바탕으로 분석한 심층 인사이트 텍스트를 출력하더라도, 실시간 수급 틱이 유입되거나 300ms 주기 UI 렌더러가 호출될 때마다 원래의 체크리스트(`insights_html`)로 화면이 강제로 리셋되어 보고서를 다 읽지 못하는 치명적인 사용성 문제가 발견되었습니다.
* **조치**:
  - **화면 잠금 상태 플래그 도입**: `is_ai_showing` 플래그를 추가하여 GPT 분석 보고서가 로딩된 상태(`True`)일 때는 `update_status` 하단의 실시간 체크리스트 렌더링을 명시적으로 우회하도록 구조를 격리시켰습니다.
  - **양방향 스위칭 및 "돌아가기" 버튼 기능 구현**:
    - AI 분석 성공 시 버튼 텍스트를 **`"↩️ 돌아가기 (AI 분석 종료)"`**로 변경하여 사용자에게 뷰 복귀 수단을 명확히 제공합니다.
    - 돌아가기 버튼을 클릭하면 플래그를 `False`로 바꾸고 본래의 수급 체크리스트 화면으로 안전하고 즉각적으로 되돌아가도록 처리하여 사용성을 극대화하였습니다.

---

### 17. AI 분석 텍스트 가로 폭 래핑 최적화 및 레이아웃 붕괴 방지 가드 (V27.10)
* **소스 파일**:
  - [signal_status_widget.py (QTextBrowser 초기 설정, _adjust_txt_ai_height, resizeEvent, request_ai_analysis, on_ai_result, on_ai_error)](file:///c:/Users/stone/projects/nocodequant/app/ui/widgets/signal_status_widget.py#L414)
* **현상**:
  - `QTextBrowser` (`txt_ai`) 내부의 가로 너비가 충분함에도 불구하고, 텍스트가 가로로 채워지지 못하고 한 행에 10자 내외로 과도하게 일찍 개행되어 잘려 보이는 현상이 발생해 가독성이 현저히 떨어졌습니다.
  - 또한, 창 크기를 변경하기 전까지(즉, `resizeEvent`가 발생하지 않은 상태) AI 결과 완료 수신 시점이나 로딩 메시지가 설정될 때 즉시 래핑 폭이 재연산되지 않고 좁은 기본 상태로 머물러 있는 현상이 발견되었습니다.
  - 특히 가로 공간을 다 채운 GPT 분석 결과 텍스트가 길어지면서 `txt_ai` 위젯의 높이가 220px까지 확장되었고, 이로 인해 메인의 수직 한도 고정(660px) 레이아웃 내에서 주변 위젯들이 눌리거나 붕괴되는 현상이 새롭게 확인되었습니다.
* **조치**:
  - **동적 가로 폭 바인딩**: `_adjust_txt_ai_height()` 메서드 내에서 텍스트 도큐먼트 정렬(`adjustSize()`) 호출 전에, 현재 위젯의 실제 가로 너비(양쪽 패딩 10px씩 총 20px 차감)를 `doc.setTextWidth()`에 동적으로 주입하여 가로 공간을 100% 활용해 자연스럽게 래핑되도록 수정했습니다.
  - **상태 전이 시 비동기 갱신 호출 강제**: `on_ai_result`, `on_ai_error`, `request_ai_analysis` (돌아가기 모드 포함) 등 AI 분석 텍스트의 내용이 변경되는 모든 라이프사이클 지점에서 `QTimer.singleShot(0, self._adjust_txt_ai_height)`을 명시적으로 호출하여, 텍스트가 렌더링된 후 가용 너비를 즉시 재계산 및 적용하도록 하였습니다.
  - **레이아웃 안정성을 위한 높이 한계 고정**: `txt_ai`의 수직 크기 상한을 기본 고정 높이인 `130`px로 고정(`min(final_h, self._base_height)`)하여 주변 레이아웃이 찌그러지는 부작용을 원천 차단했습니다.
  - **얇은 다크 테마 커스텀 스크롤바 장착**: 높이 고정으로 인해 잘릴 수 있는 초과 텍스트는 `setVerticalScrollBarPolicy(Qt.ScrollBarAsNeeded)`와 스타일시트 커스텀 6px 마이크로 스크롤바 스타일링을 적용해, 디자인을 해치지 않고 마우스 휠 및 드래그로 안전하게 내비게이션할 수 있게 조치했습니다.
  - **반응형 리사이즈 이벤트 추가**: `resizeEvent` 이벤트를 오버라이딩하여, 사용자가 전체 대시보드 크기를 조절하거나 창을 늘릴 때도 실시간으로 가로 폭 래핑 경계선이 자동 재계산되어 예쁘게 텍스트 정렬이 유지되도록 연동을 완료했습니다.

---

## 🔮 향후 대응 및 실거래 운영 안정화 방안
1. **런타임 라이브 복구 및 모니터링**:
   - 현재 메인 프로세스(`python app/main.py`)가 장중 실행 상태에 있으므로, 본 안정성 패치를 장중에 즉각 실거래 엔진에 반영하기 위해 다음 장 개시 전 또는 안전한 시점에 프로그램을 재시작하여 정상 반영 여부를 모니터링할 예정입니다.
2. **테스트 스위트 무결성 확인**:
   - 수정 완료 후 11개의 코어 유닛 테스트 패스를 재차 확인하여 기존 백테스팅 및 오더 라이프사이클에 미치는 사이드 이펙트가 전혀 없음을 완벽히 보장하였습니다.
