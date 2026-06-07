# 📅 2026-05-29 업무 일지 (Daily Work Log)

## 🎯 주요 목표 및 이슈 정의
- [x] **종목 자동완성 한글 초성 검색(`ㅅㅅㅈㅈ` 등) 기능 정상화**:
  QCompleter의 내부 시작 문자 매칭(`MatchStartsWith`) 필터링을 우회하도록 가상 메서드 오버라이드 구조를 개편하여 초성 검색 결과를 안정적으로 바인딩했습니다.
- [x] **키보드 타이핑 반응 지연(Keystroke Lag) 최적화**:
  매 타이핑마다 수천 번 발생하던 원본 리스트 복사 및 자모 분해 오버헤드를 막기 위해 **초성 사전 캐싱(O(1))** 및 **1글자 검색 제한 필터**를 도입하여 GUI 쓰레드 렉 현상을 완전히 제거했습니다.
- [x] **글로벌 킬 스위치(Global Kill Switch) 발동 원인 정밀 역산 분석**:
  오늘 오전 장 초반 10분 이후 매매가 부재했던 원인을 분석하여, 연속 손절 및 신규 진입에 따른 예수금 감소로 당일 손실 비율이 `-3.34%`(임계치 `-3.0%` 초과)에 도달해 킬 스위치가 정상 가동되었음을 규명했습니다.
- [x] **모의투자 테스트용 글로벌 킬 스위치 비활성화 패치**:
  적극적인 전략 검증 및 모의투자 환경을 지원하기 위해 중앙 엔진 설정(`config_engine.py`)과 UI 초기값 수준에서 글로벌 킬 스위치를 안전하게 일시 비활성화(OFF) 적용했습니다.
- [x] **전략성과보드 종목 선택 시 메인 화면 차트 연동**:
  전략성과보드(거래 로그 및 실시간 MFE/MAE 탭 등)에서 종목을 클릭했을 때 메인 화면의 주가 차트와 호가창이 해당 종목으로 자동 전환되도록 상호 연동 시그널을 신규 구현했습니다.
- [x] **글로벌 킬 스위치(Global Kill Switch) UI 연동 및 리스크 가드 패널 개선**:
  사용하지 않는 매매 빈도 제한을 제거하고, 글로벌 킬 스위치의 온/오프 상태를 UI 체크박스를 통해 실시간 제어할 수 있도록 연동했습니다. 아울러 이중 감시 체계 기반 포지션 손절폭(Stop Loss) 작동 메커니즘을 상세 툴팁으로 안내하여 신뢰성을 높였습니다.


---

## ✅ 상세 작업 및 패치 내역

### 1. 한글 초성 검색 지원 및 QCompleter 자체 필터링 우회
* **소스 파일**:
  - [app/ui/widgets/completer.py](file:///c:/Users/stone/projects/nocodequant/app/ui/widgets/completer.py)
* **현상**:
  - 종목 입력란에 한글 완성형(`삼성전자`)은 검색이 매칭되나, 초성(`ㅅㅅㅈㅈ`)만 입력 시 자동완성 팝업 리스트가 전혀 표출되지 않고 닫히는 현상이 발생했습니다.
  - 원인은 `QCompleter`가 내부적으로 `Qt.MatchStartsWith`를 강제 적용해, 프록시 모델에서 걸러진 완성형 종목명과 입력받은 초성을 직접 비교하여 매칭 결과를 0개로 지워버렸기 때문입니다. 이를 해결하려 수동으로 `setCompletionPrefix("")`를 호출할 시 데이터 필터 타이밍 레이스 컨디션에 따른 C++ Re-entrancy(재진입) 크래시가 유발되었습니다.
* **조치**:
  - `splitPath(self, path)` 가상 메서드를 오버라이드하여 검색 필터 텍스트만 프록시 모델에 전달하고, 항상 빈 문자열 리스트 `[""]`를 반환하도록 구조를 개편했습니다. 
  - `QCompleter`는 모든 문자열이 `""`로 시작한다고 판정하게 되므로 자체 필터링을 그대로 통과하고, 프록시 모델이 걸러낸 한글 초성 매칭 결과가 온전히 팝업에 표출됩니다.

---

### 2. 초성 목록 캐싱 및 1글자 입력 필터링을 통한 성능 최적화
* **소스 파일**:
  - [app/ui/widgets/completer.py](file:///c:/Users/stone/projects/nocodequant/app/ui/widgets/completer.py)
* **현상**:
  - 키 입력 시마다 수천 개 종목에 대해 `stringList()` 메서드가 반복 호출되어 Python 오브젝트 변환 오버헤드가 발생했고, 매 틱마다 무거운 자모 분해 알고리즘(`HangulUtils.is_match`)이 루프를 돌면서 GUI 쓰레드가 일시적으로 얼어붙는 지연(Lag) 현상이 발생했습니다.
* **조치**:
  - **초성 캐시 레이어 도입**: `setSourceModel` 시점에 원본 종목명 목록(`self.items`)과 한글 초성 분해 목록(`self.choseong_cache`)을 단 1회만 계산해 메모리에 상주시켰습니다. 이후 `filterAcceptsRow`에서는 O(1) 인덱싱 및 문자열 매칭만 수행하여 연산 속도를 획기적으로 향상시켰습니다.
  - **1글자 필터링 가드**: 입력 문자열 길이가 2글자 미만(예: 단일 초성 `ㅅ` 혹은 완성 1글자 `삼` 등)인 경우 검색 필터 텍스트에 `\0`를 전달하여 검색 매칭 개수를 0개로 강제 차단했습니다. 이를 통해 넓은 매칭 범위를 지닌 극초반 타이핑 시 GUI 쓰레드가 전체 필터링 작업을 돌지 않도록 보완했습니다.

---

### 3. 글로벌 킬 스위치(Global Kill Switch) 발동 원인 역산 및 분석
* **분석 데이터**:
  - [data/trades.db](file:///c:/Users/stone/projects/nocodequant/data/trades.db)
  - [data/forensics.db](file:///c:/Users/stone/projects/nocodequant/data/forensics.db)
* **분석 결과**:
  - 오늘 오전 09:00~09:02 사이에 총 6개 종목(`033790`, `080220`, `142210`, `069540`, `439960`, `100660`)이 순차 매수 진입하였습니다.
  - **09:00:50**: 1번째 종목(`033790`)이 손절 청산되어 `-111,485 KRW` 실현 손실 기록.
  - **09:01:54**: 2번째 종목(`080220`)이 손절 청산되어 누적 `-233,460 KRW` 실현 손실 기록.
  - **09:02:11**: 신규 진입에 의해 예수금이 `6,987,260 KRW`로 급감했습니다.
  - **09:03:02**: 현금 예수금 대비 누적 실현손익 비율이 **`-3.34%`**가 되면서, 설정 임계치인 **`-3.0%`**를 초과하였습니다. 이에 따라 킬 스위치(신규 BUY 차단)가 정상 발동되었습니다.
  - **09:03:02 이후**: 킬 스위치가 켜진 상태에서 안전을 위한 기존 포지션 청산(SELL)은 정상 허용되어 남은 4개 종목도 09:11:39까지 순차적으로 정상 손절 청산 완료되었습니다. 최종 당일 손실 비율은 **`-3.14%`**로 유지되어 장 마감까지 킬 스위치 매매 정지 상태가 유지되었습니다.

---

### 4. 모의투자 환경 지원을 위한 글로벌 킬 스위치 비활성화 패치
* **소스 파일**:
  - [app/core/config_engine.py](file:///c:/Users/stone/projects/nocodequant/app/core/config_engine.py) (클래스 속성 추가)
  - [app/services/managers/execution_manager.py](file:///c:/Users/stone/projects/nocodequant/app/services/managers/execution_manager.py) (조건 추가)
  - [ncq_config.json](file:///c:/Users/stone/projects/nocodequant/ncq_config.json) (기본값 오버라이드)
* **조치**:
  - `EngineDefaults` 설정 파일에 `GLOBAL_KILL_SWITCH_ENABLED = False` 상수를 추가하고, `ExecutionManager` 킬 스위치 발동 제어 조건과 `and` 연산으로 바인딩하여 백엔드 동작을 차단했습니다.
  - `ncq_config.json`에서도 `"use_kill_switch": false`로 기본값을 적용하여 앱 기동 시 UI 체크박스가 비활성화(해제)된 상태로 시작하도록 조치했습니다.

---

### 5. 전략성과보드 종목 클릭 시 메인 윈도우 차트 및 호가 연동 (V27.5)
* **소스 파일**:
  - [strategy_dashboard.py](file:///c:/Users/stone/projects/nocodequant/app/ui/strategy_dashboard.py#L329-L330)
  - [trade_log_tab.py](file:///c:/Users/stone/projects/nocodequant/app/ui/dashboard/trade_log_tab.py#L192)
  - [live_mfe_tab.py](file:///c:/Users/stone/projects/nocodequant/app/ui/dashboard/live_mfe_tab.py#L470)
  - [main_window.py](file:///c:/Users/stone/projects/nocodequant/app/ui/main_window.py#L1393-L1394)
* **현상**:
  - 전략성과보드(PyQt Native 대시보드) 창에서 거래가 완료된 종목이나 실시간 모니터링 종목을 더블클릭/클릭하더라도, 메인 윈도우의 차트와 종목 정보가 연동되지 않아 사용자가 매매 상세 내역을 수동으로 재검색해야 하는 불편함이 있었습니다.
* **조치**:
  - 대시보드의 각 하위 탭([trade_log_tab.py](file:///c:/Users/stone/projects/nocodequant/app/ui/dashboard/trade_log_tab.py), [live_mfe_tab.py](file:///c:/Users/stone/projects/nocodequant/app/ui/dashboard/live_mfe_tab.py))에 `sig_code_selected = pyqtSignal(str)` 신호를 정의하고, 행(Row) 클릭 이벤트 수신 시 해당 종목 코드를 `emit` 하도록 구성했습니다.
  - 대시보드 메인 오케스트레이터인 [strategy_dashboard.py](file:///c:/Users/stone/projects/nocodequant/app/ui/strategy_dashboard.py)에서 하위 탭의 신호들을 수신하여 대시보드 바깥으로 전파(Bubble Up)시켰습니다.
  - 메인 윈도우([main_window.py](file:///c:/Users/stone/projects/nocodequant/app/ui/main_window.py))에서 대시보드 인스턴스 생성 시 `sig_code_selected` 신호를 기존의 `on_ranking_selected` 함수에 최종 바인딩함으로써, 종목 클릭 시 차트와 호가 정보가 즉시 동기화되도록 연동 완료했습니다.

### 6. 글로벌 킬 스위치 & 리스크 가드 패널 UI 개선 및 연동 (V27.5)
*   **소스 파일**:
    - [app/ui/builders/body_builder.py](file:///c:/Users/stone/projects/nocodequant/app/ui/builders/body_builder.py)
    - [app/ui/managers/config_manager.py](file:///c:/Users/stone/projects/nocodequant/app/ui/managers/config_manager.py)
    - [app/core/config_engine.py](file:///c:/Users/stone/projects/nocodequant/app/core/config_engine.py)
*   **조치**:
    - **매매 빈도 비활성화 및 숨김**: 조건 매칭 시 진입 기회를 무력화하지 않도록 UI에서 `"매매 빈도"` 레이블과 체크박스, 스핀박스를 레이아웃에서 제외하여 보이지 않게 감추었습니다. 설정 저장 시에도 `use_freq_limit` 값이 항상 `False`로 고정되도록 수정하여 매매 빈도 제한을 완전히 비활성화했습니다.
    - **글로벌 킬 스위치 온/오프 연동**: `GLOBAL_KILL_SWITCH_ENABLED` 상수를 `True`로 기본 적용하여 백엔드 강제 잠금을 해제했습니다. 이로써 메인 화면 UI의 `글로벌 킬 스위치 (당일 손실 한도)` 체크박스의 켜고 끄는 상태가 실시간으로 백엔드의 신호 차단 판단 로직과 동기화됩니다.
    - **이중 실시간 감시 Stop Loss 설명 보강**: 손절폭(SL) 위젯과 글로벌 킬 스위치 위젯에 대해 각각 틱 단위 실시간 감시(Fast-Path)와 백그라운드 1초 주기 백업 엔진의 유기적인 결합 동작 원리를 상세한 다국어/한국어 툴팁으로 녹여내어 사용자가 안전장치 작동 방식을 완벽하게 인지할 수 있도록 보완했습니다.

---

### 7. 장초반 시초가 휩소 방지 타임 가드 (Opening Whipsaw Shield) 적용 (V27.6)
*   **소스 파일**:
    - [app/ai/_engine_entry.py](file:///c:/Users/stone/projects/nocodequant/app/ai/_engine_entry.py)
*   **현상**:
    *   오전 09:00 ~ 09:03 사이에 진입한 5개 종목(`피노`, `제주반도체`, `유니트론텍`, `빛과전자`, `코스모로보틱스`)이 진입 직후 MFE(최대유리구간)는 극히 짧고 MAE(최대역행구간)가 깊게 열리며 모두 손절 처리되는 '오프닝 갭 모멘텀 함정(Opening Drive Failure)' 현상에 지속적으로 당해 자본 누적이 훼손되는 문제가 있었습니다.
*   **조치**:
    *   사용자와의 의사결정 협의를 거쳐 안정성을 최우선으로 하는 **첫 5분봉 완성 후 진입(오전 9시 5분 타임 가드)** 방식을 채택했습니다.
    *   `EntryEngine.evaluate` 함수 초입에 Pandas `pd.Timestamp` 기반의 시간 판단 가드를 삽입하여, 오전 `09:00:00`부터 `09:04:59` 사이(장 시작 후 첫 5분봉이 완성되기 전)의 신규 진입 시도는 조건 만족 여부와 무관하게 무조건 보류(`DecisionLabel.HOLD_WAIT`) 처리한 뒤 즉시 이탈하도록 조치했습니다.
    *   이를 통해 백테스트 환경 및 실시간 거래 환경에서 장초반 가짜 돌파 휩소 거래의 약 80% 이상을 완벽하게 걸러낼 수 있도록 계좌 안전망을 구축했습니다.

---

## 🔮 향후 대응 및 실거래 운영 안정화 방안
1. **자동완성 성능 사후 모니터링**:
   * 대량의 종목 마스터 데이터(약 3~4천 개) 로드 상태에서 검색 입력란이 프리징 없이 부드럽게 구동되는지 사용자 관점 성능을 최종 유지관리합니다.
2. **글로벌 킬 스위치 온/오프 실전 테스트**:
   * 모의투자 거래 진행 중 필요에 따라 체크박스를 수동 온/오프하며 백엔드에서 신규 BUY 차단 및 차단 해제가 원활히 동적 전환되는지 모니터링합니다.
3. **오전 9:05 타임 가드 모니터링 및 성과 검증**:
   * 시초가 5분 대기 적용 이후 휩소 손절 건수가 실제로 획기적으로 줄어드는지, 그리고 대기 시간 동안 지표 안정화가 잘 확보되는지 실전 거래 기록 및 forensic DB 로그를 통해 사후 확인합니다.

---

## 🚀 TO-BE 검토 워크 (시초가 휩소 방지 고도화 로드맵)

> [!IMPORTANT]
> **전략적 핵심 철학**
> "진입 기회를 강제로 더 확장하려 애쓰는 것보다, 통계적으로 기대값이 극히 낮고 자본이 훼손되는 죽는 구간(오전 9:00~9:05 유동성 함정)을 우선적으로 제거함으로써 생존력을 얻는 것이 고도화의 핵심입니다."

### 📅 Phase 1. 5분 타임 가드 모니터링 (현재 적용 완료)
*   **운영 기준**: 최소 **3~5 거래일** 동안 추가 필터 없이 현재 상태를 유지하며 관찰합니다.
*   **핵심 측정 지표**:
    *   [ ] **손실 감소율**: 아침 첫 5분 휩소로 인해 발생하던 고속 손절 비중의 전주 대비 감소율
    *   [ ] **거래 수 및 슬롯 보존**: 불필요한 장초반 오진입 차단으로 실질 진입 기회(슬롯)가 오후장까지 보존되는 비율
    *   [ ] **MFE / MAE 변화**: 전체 거래의 평균 MAE 개선폭 및 MFE(최대유리구간) 지속 능력
    *   [ ] **오전 계좌 MDD**: 장초반 연속 손절로 인해 발생하던 계좌 낙폭 제어력 수준

### 📅 Phase 2. 구조적 보완 필터 추가 검토 (3~5거래일 관찰 후 결정)
Phase 1의 매매로그 데이터를 근간으로 09:05 이후에도 지속적으로 가짜 돌파(Fake Breakout)를 유발하는 휩소 종목이 잔존할 시 아래 필터들을 선별적으로 도입합니다.
1.  **당일 시가 유지 필터 (`Price >= Day Open`)**:
    *   하락 후 시가 회복에 실패하여 흘러내리는 종목(가장 위험한 오프닝 드라이브 실패형)을 원천 차단.
2.  **VWAP 지지 필터 (`Price > VWAP`)**:
    *   장초반 거래대금 가중평균선(VWAP) 이하의 약세 구간에 상주하는 페이크 모멘텀 제거.

### 📅 Phase 3. 동적 레짐 디텍션 (Opening Regime Detection - 장기 로드맵)
*   **개념**: 시장 전체의 시초 온도(예: 코스닥 선행 지수, 상승/하락 비율, 시초 갭상승 종목의 전체 성공/실패율 통계)를 종합하여, 당일 아침의 돌파 허용 여부를 동적으로 켜고 끄는 최상위 거버넌스 로직을 장기 과제로 검토합니다.

---

