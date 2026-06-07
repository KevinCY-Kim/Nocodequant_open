# 📅 2026-06-02 업무 일지 (Daily Work Log)

## 🎯 주요 목표 및 이슈 정의
- [x] **모의투자 실시간 가상 예수금 연동 패치 (P0)**:
  - Kiwoom Mock Trading(모의투자) 시 하드코딩되었던 가짜 예수금 조건들을 제거하고, Kiwoom API에서 직접 수신한 실제 가상 예수금을 기준으로 위험 한도가 동작하도록 연동했습니다.
- [x] **하드코딩 투자 비중 제한 조건 해제 및 포지션 사이징 동적 계산 (P0)**:
  - 15%로 제한되어 있던 단일 종목 최대 투자 비중 하드코딩을 제거하고, 사용자가 입력한 % 비중 값을 동적으로 처리하도록 `money_manager.py` 및 `config_manager.py`를 개선했습니다.
- [x] **시계 동기화(Time Synchronization) & 네트워크 지연(RTT) 보정 아키텍처 도입 (P1)**:
  - UDP NTP(포트 123)와 RTT 보정 공식(`latency = rtt / 2.0`)을 적용하여 로컬 PC 시계의 정밀 오차를 감지하고, UDP 실패 시 TCP HTTPS(포트 443) 구글 Fallback 경로를 제공하도록 설계했습니다.
  - 시간 편차 허용 임계값으로 NTP는 1.5초, HTTP Fallback은 5.0초를 각각 차별 적용해 네트워크 및 캐싱 지연으로 인한 오경보를 최소화했습니다.
- [x] **MainWindow 단일 책임 원칙(SRP) 준수 및 비대화 방지 리팩토링**:
  - `main_window.py`가 불필요하게 커지는 문제를 해결하기 위해, 타임 동기화 모니터링 관련 스레드/타이머/알림창 생성 로직을 별도의 전담 매니저인 `TimeSyncGuard` 클래스로 격리하여 `app/utils/time_sync.py`로 캡슐화했습니다.
- [x] **프로그램 Graceful Shutdown 시 FlightRecorder JSON 직렬화 오류 패치**:
  - 시스템 종료 프로세스 작동 중 `FlightRecorder` 덤프 시 `EventType` Enum이나 `datetime` 객체가 직렬화되지 않아 발생하던 JSON 직렬화 장애(`Object of type EventType is not JSON serializable`)를 해결하기 위해 `persistence_manager.py`에 `custom_serializer` 인코더를 도입했습니다.
- [x] **비보존 제약 (082800) 거래 데이터 누락 분석 및 복구 (P0)**:
  - 6월 2일 장중 발생한 비보존 제약 68주 매도 데이터 누락 현상을 정밀 포렌식하여 원인을 규명하고, 누락된 68주 매도 건을 `trades` 테이블에 복구하고 1주 매도의 체결 시각을 HTS 실체결 시간과 일치하도록 정정했습니다.
  - 부분 체결 중 미체결 분이 취소되었을 때 DB 즉시 플러시 처리(`order_manager.py`) 및 부분 청산 시 포지션 추적 상태 유지(`persistence_manager.py`)를 통해 데이터 유실 재발 방지책을 최종 적용했습니다.

---

## ✅ 상세 작업 및 패치 내역

### 1. 모의투자 가상 예수금 및 동적 포지션 사이징 개선 (P0)
* **목적**: 키움 모의투자 접속 시 하드코딩되어 있던 계좌 위험 거버넌스 가드를 제거하고, 실제 계좌 잔고 상태에 맞춤형 가드를 수립.
* **패치 내역**:
  - **가짜 예수금 캡 제거**: 키움 Mock API 모드에서 가상 예수금을 강제 지정하거나 예외 처리하던 코드를 제거하여, API 상의 실시간 예수금을 SSoT(단일 진실 공급원)로 반영했습니다.
  - **비중 한도 캡 해제**: [money_manager.py](file:///c:/Users/stone/projects/nocodequant/app/core/money_manager.py) 및 [config_manager.py](file:///c:/Users/stone/projects/nocodequant/app/ui/managers/config_manager.py)에서 기존 15% 비중으로 묶여 있던 제한 조건을 지우고, 사용자가 HTS UI를 통해 설정한 % 비율(예: 25%, 50% 등)에 따라 동적으로 주문 한도 및 예상 주문 수량이 계산되도록 개선했습니다.

---

### 2. 표준 시계 동기화 모니터링 수립 (P1)
* **목적**: 현지 PC 시계의 미세 시간 편차(Drift)로 인해 발생할 수 있는 키움 API 주문 거절 및 분봉 왜곡 현상을 사전에 방지.
* **설계 및 상세 구현**:
  - **비동기 처리**: UI 스레드 프리징을 막기 위해 `QThread` 기반 `TimeSyncWorker`를 작성하여 백그라운드에서 표준 시각을 조회하도록 했습니다.
  - **RTT 보정**: NTP 서버 쿼리 시 송수신 왕복 지연 시간을 계산하여 `corrected_time = ntp_time + (rtt / 2.0)` 공식을 통해 오차 정밀도를 높였습니다.
  - **이중화 인프라**: UDP 포트 123이 차단된 방화벽 환경에서도 작동 가능하도록 HTTPS TCP(포트 443) HEAD 요청을 통해 구글 서버 시각을 파싱하는 Fallback 메커니즘을 내장했습니다.
  - **차별화된 오차 임계값**: 정밀도가 높은 NTP는 **1.5초**, 프록시나 CDN 캐싱 지연이 수반될 수 있는 HTTP Fallback은 **5.0초**의 편차 임계치를 각각 지정해 잘못된 경보를 예방했습니다.
  - **알림 최소화 정책**: 트레이딩 도중 오동작을 줄이기 위해, 모달 팝업 경고창(`QMessageBox.warning`)은 **오직 앱 초기 실행 시점(엔진 비활성화 상태)**에만 표시되도록 제한하고, 구동 중인 주기적(30분 주기) 검사 단계에서는 백그라운드 상태 로그로만 기록되도록 보호 장치를 마련했습니다.

---

### 3. MainWindow SRP 준수 리팩토링
* **목적**: 120KB가 넘는 [main_window.py](file:///c:/Users/stone/projects/nocodequant/app/ui/main_window.py)의 가독성을 확보하고 불필요한 기능 집중에 의한 기술 부채를 예방.
* **리팩토링 내역**:
  - 기존 메인 윈도우에 통합 구현하려던 시계 동기화 타이머 설정, 주기적 검사 트리거, 결과 분석 콜백 슬롯 등 70여 줄의 중복 코드를 전부 덜어냈습니다.
  - [time_sync.py](file:///c:/Users/stone/projects/nocodequant/app/utils/time_sync.py)에 전담 컴포넌트인 [TimeSyncGuard](file:///c:/Users/stone/projects/nocodequant/app/utils/time_sync.py#L95-L181)를 선언하여 제어 책임을 위임했습니다.
  - `MainWindow` 내에서는 `TimeSyncGuard(self)`를 생성하고 `.start()`를 호출하는 단 2줄의 코드로 모든 동기화 절차가 초기화되도록 최적화했습니다.

---

### 4. FlightRecorder JSON 직렬화 장애 패치
* **목적**: 시스템 안전 종료 과정에서 FlightRecorder의 디버깅 스냅샷 덤프 실패 및 예외 누수 결함을 해결.
* **패치 내역**:
  - [persistence_manager.py](file:///c:/Users/stone/projects/nocodequant/app/services/managers/persistence_manager.py)의 [dump_flight_recorder](file:///c:/Users/stone/projects/nocodequant/app/services/managers/persistence_manager.py#L918-L933) 내부에서 `json.dump` 호출 시 `EventType` Enum 및 `datetime` 객체가 직렬화되지 않아 프로세스가 중단되는 문제를 확인했습니다.
  - 직렬화 도중 직렬화가 불가능한 객체들을 문자열이나 Enum 원시 값으로 변환하는 `custom_serializer` 로컬 인코더 헬퍼를 추가 및 주입하여 에러를 완전 차단했습니다.
  - 조치 후 FlightRecorder의 이벤트들이 정상적으로 `data\flight_recorder_dump.json`에 기록되고 지연 없이 Graceful Shutdown이 완료되는 것을 최종 검증했습니다.

---

### 5. 비보존 제약 (082800) 거래 데이터 누락 복구 및 부분 체결 보완 (P0)
* **목적**: 장중 부분 체결 및 미체결 취소 시 발생하는 포지션 정보 유실 현상(Orphan Sell)을 분석하고 실거래 DB의 거래 정합성을 복구.
* **패치 및 조치 내역**:
  - **거래 데이터 복구**: HTS 실체결 내역(15:15:06 68주 매도, 15:25:09 1주 매도)과 로컬 DB의 불일치를 해결하기 위해 복구 스크립트([recover_vivozone_68.py](file:///c:/Users/stone/projects/nocodequant/scratch/recover_vivozone_68.py))를 작성하여 68주 매도 건(`order_id: 0149897`)을 복구(INSERT)하고, 1주 매도 건의 exit_time을 `15:25:09`로 정정 완료했습니다.
  - **부분 체결 취소 시 DB 즉시 플러시 구현**: [order_manager.py](file:///c:/Users/stone/projects/nocodequant/app/services/managers/order_manager.py)의 `on_order_event`를 개선하여 주문이 취소/거부되더라도 이미 부분 체결된 수량(`cum_qty > 0`)이 있을 경우 강제 플러시를 수행해 DB 누락을 원천 차단했습니다.
  - **부분 청산 시 포지션 추적 상태 유지**: [persistence_manager.py](file:///c:/Users/stone/projects/nocodequant/app/services/managers/persistence_manager.py)의 포지션 라이프사이클을 개선하여 부분 청산 시 status를 `CLOSED`가 아닌 `PARTIAL`로 유지함으로써 잔여 수량이 지속 추적되도록 보완했습니다.

---

## 🔮 향후 대응 및 시스템 모니터링 방안
1. **시계 동기화 상태 지속 관찰**:
   - 30분 간격으로 출력되는 시계 오차(Forensic Log)를 모니터링하여, 실거래 중 지속적인 Drift 발생 시 시스템 차원의 알림이나 지수 가드 등의 연계 방안을 검토합니다.
2. **비중 설정 범위 테스트**:
   - 동적 비중 계산이 정상 적용됨에 따라 15% 초과(예: 25%, 50%) 설정 시 슬롯 간 거래대금 간섭이나 예수금 부족 예외가 매끄럽게 처리되는지 지속 모니터링합니다.
3. **부분 체결 거래 추적 모니터링**:
   - 부분 체결 발생 후 주문 취소/정정 또는 분할 매도 완결 시, DB의 `orders_tracked` 상태 변환(`PARTIAL` ↔ `CLOSED`)과 `trades` 테이블의 분할 삽입 상태가 수량 오차 없이 정확하게 추적되는지 실시간 로그를 통해 밀착 모니터링합니다.
