# 📅 2026-05-30 업무 일지 (Daily Work Log)

## 🎯 주요 목표 및 이슈 정의
- [x] **장초반 휩소 방지 타임 가드 부작용 보완**:
  09:00~09:05 동안 모든 신입 진입을 차단하는 'Opening Whipsaw Shield' 도입 이후, 장초반 급락했다가 강하게 반등하여 시가를 돌파하는 '살아남은 강세주'까지 통째로 놓쳐버리는 기회 손실을 막기 위해 **Opening Recovery Surveillance Engine (시초 회복 감시 엔진)**을 구축했습니다.
- [x] **모니터링 3대 시점 추적 및 로그 수집 체계 마련**:
  사용자 요청에 따라 ① 최초 감시 등록(09:00~09:05), ② 회복 후 재돌파 진입(READY → TRIGGERED), ③ 미돌파 만료(09:30 TTL)의 3가지 핵심 시점을 완벽하게 모니터링하고 독립 로그 파일로 영구 기록하는 기능을 구현했습니다.
- [x] **실시간 계좌 안전을 위한 Shadow Mode 탑재**:
  새로운 회복 판단 로직의 통계적 신뢰도가 검증될 때까지 실제 BUY 주문을 내지 않고 가상 추적 로그만 쌓는 **Shadow Mode(기본값)**를 설계하여 즉시 가동했습니다.
- [x] **Tactical HUD 실시간 감시 현황 UI 및 어댑티브 툴팁 연동**:
  오전 장초반에 어떤 종목들이 감시 풀에 들어가 있고, 어느 단계(추적, 회복중, READY)에 도달했는지 한눈에 파악할 수 있도록 메인 UI의 Tactical HUD 레이아웃을 개편하고 실시간 상태를 바인딩했습니다. 또한 마우스 호버 시 종목별 상세/컴팩트 회복 현황을 상태 우선순위로 자동 정렬하여 표시하는 **어댑티브 툴팁(Adaptive Tooltip)**을 추가 적용했습니다.

---

## ✅ 상세 작업 및 패치 내역

### 1. Opening Recovery Surveillance Engine 설계 및 구현
* **소스 파일**:
  - [app/services/managers/opening_recovery_manager.py](file:///c:/Users/stone/projects/nocodequant/app/services/managers/opening_recovery_manager.py) (신규 생성)
* **주요 설계**:
  - **독립 클래스 구조**: `execution_manager.py`에 불필요하게 비대해지는 책임을 분산하기 위해 완벽히 독립된 전용 감시 엔진 클래스를 설계했습니다.
  - **5단계 상태 머신**: `WATCHING` (09:00~09:05 감시 등록 및 고저점 추적) ➔ `RECOVERING` (09:05 이후 가격 회복 관찰) ➔ `READY` (3대 재진입 조건 충족) ➔ `TRIGGERED` (진입 완료 및 감시 종료) ➔ `EXPIRED` (09:30 시간 만료로 자동 소멸)의 유기적인 상태 전이도를 구축했습니다.
  - **3대 재진입 판단 조건 동시 충족 검증**:
    1. **시가 안착**: `Price >= Day Open`
    2. **5분 봉 종가 고가 재돌파**: `Price >= opening_close_high * 0.997` (슬리피지 0.3% 허용 버퍼)
    3. **낙폭 회복 비율 70% 이상**: `Recovery Ratio >= 70%` (09:00~09:05 고가-저가 대비 현재가 회복 비율)
  - **이벤트 추적 로깅**: 비동기 및 파일 입출력 예외 방지 처리가 완벽히 가미된 JSONL 방식 로그 파일([logs/opening_recovery_log.jsonl](file:///c:/Users/stone/projects/nocodequant/logs/opening_recovery_log.jsonl)) 자동 저장을 구현했습니다.

---

### 2. 타임 가드 연동 및 수식 정밀화 패치
* **소스 파일**:
  - [app/ai/_engine_entry.py](file:///c:/Users/stone/projects/nocodequant/app/ai/_engine_entry.py)
* **조치**:
  - 09:00~09:05 사이 타임 가드 동작으로 인해 `DecisionLabel.HOLD_WAIT`가 발생하는 모든 종목을 무시하거나 폐기하지 않고, `OpeningRecoveryManager.register()`를 호출하여 실시간 회복 감시 대상(Opening Watch Pool)에 정식 등록하게 변경했습니다.
  - `_safe_float` 헬퍼 함수가 단일 인자(`v`, `default`)만 지원함에도 불구하고 `_safe(curr, 'open', 0)` 형식으로 다중 인자를 보냈던 기존의 잠재적 버그를 발견하여, `_safe(curr.get('open', 0))` 형태로 정정 조치했습니다. (Pandas Series인 `curr`에서 안전하게 값을 빼낸 뒤 단일 인자로 전달)

---

### 3. 백엔드 오케스트레이터 바인딩 및 KST 시각 동기화
* **소스 파일**:
  - [app/services/managers/execution_manager.py](file:///c:/Users/stone/projects/nocodequant/app/services/managers/execution_manager.py)
* **조치**:
  - `ExecutionManager` 초기화 단계에서 `OpeningRecoveryManager(shadow_mode=True)` 인스턴스를 정상 바인딩하고, 전역 설정인 `ACTIVE_CONFIG["_opening_recovery_mgr"]`에 등재하여 인덱싱 엔진들이 접근할 수 있도록 연동했습니다.
  - 백엔드가 UTC 시각을 기본으로 처리하여 09:05 및 09:30 시간대 판정이 무력화되던 현상을 해결하기 위해, `tick_time`을 KST(한국 표준시)로 정확히 오프셋 변환(`timedelta(hours=9)`)하여 만료(expire_stale) 및 회복 판정(evaluate_recovery)을 수행하도록 로직을 정밀화했습니다.
  - `OP_RECOVERY` 전용의 연산 슬롯 예약 한도(`PROMOTED_SLOTS_LIMIT["OP_RECOVERY"] = 5`)를 할당하여, 시스템 과부하를 막고 주요 종목만 선별 추적하도록 자원 배분을 최적화했습니다.

---

### 4. Tactical HUD UI 컴포넌트 추가 및 어댑티브 툴팁/우선순위 정렬 연동
* **소스 파일**:
  - [app/ui/widgets/tactical_hud_widget.py](file:///c:/Users/stone/projects/nocodequant/app/ui/widgets/tactical_hud_widget.py)
  - [app/ui/managers/hud_manager.py](file:///c:/Users/stone/projects/nocodequant/app/ui/managers/hud_manager.py)
* **조치**:
  - Tactical HUD의 5번째 행으로 **"시초 감시 (Opening Recovery)"** 전용 인포메이션 라인을 신설했습니다.
  - 감시 중인 종목 현황을 실시간 파싱하여 `🔍 [감시건수] | 추적 [W건]  회복중 [R건]  ⚡READY [Ready건]` 포맷으로 렌더링하고, 상태 변화에 따라 다음과 같이 동적 컬러링을 적용하여 시각적 직관성을 향상시켰습니다:
    - **하늘색 (`#5ac8fa`)**: 단순 고저점 추적 단계 (`WATCHING`)
    - **주황색 (`#ff9500`)**: 낙폭 극복 시도 단계 (`RECOVERING`)
    - **녹색 (`#4caf50`)**: 모든 조건 충족 후 돌파 대기 단계 (`READY`)
    - **어두운 회색 (`#3a4a5a`)**: 비거래시간 비활성 단계 (`EXPIRED`)
  - **어댑티브 툴팁 (Adaptive Tooltip) 및 우선순위 정렬** 추가 구현:
    - **상태별 고정 색상 시스템**: `WATCHING` (회색, `#8a9aab`), `RECOVERING` (하늘색, `#5ac8fa`), `READY` (초록색, `#4caf50`), `EXPIRED` (어두운 회색, `#5a6a7a`) 적용.
    - **상태 우선순위 정렬**: 툴팁 내 종목 목록을 `READY` ➔ `RECOVERING` ➔ `WATCHING` ➔ `EXPIRED` 순서로 정렬하여 실시간 매매 대상에 신속히 반응할 수 있도록 개선.
    - **어댑티브 모드 레이아웃**: 감시 종목이 3개 이하일 때는 **상세 모드** (상태, 회복률, 시가대비, 고가돌파 대기/완료 여부, 발동시각 표시)로 구성하고, 4개 이상으로 많아지면 화면을 가리지 않도록 **컴팩트 모드** (상태별 요약 한 줄 목록 형태)로 자동 전환되도록 구현.
  - 신규 UI 정보 라인 추가로 인해 기존 텍스트들이 찌그러지거나 잘리지 않도록, **HUD 고정 높이를 216px ➔ 236px**로 상향하고 **Top Panel의 높이를 168px ➔ 188px**로 20px씩 정교하게 늘려 완벽한 비주얼 밸런스를 복원했습니다.

---

## 🔮 향후 대응 및 실거래 운영 안정화 방안

1. **Shadow Mode 통계 데이터 수집 및 백테스트 검증**:
   - 다가오는 거래일 동안 `logs/opening_recovery_log.jsonl`에 기록된 로그를 확인하여, 실제로 타임 가드에 차단되었다가 09:05 이후 회복 조건을 통과(`TRIGGERED`)한 종목들의 주가 흐름이 당일 우상향 마감되었는지 승률(Win-rate)과 손익비(Profit Factor)를 역산할 예정입니다.
2. **실시간 GUI 쓰레드 모니터링**:
   - 09:00~09:05의 변동성 집중 구간에서 최대 10개의 감시 대상 종목이 등록되고 틱 갱신이 빈번해질 때, HUD 실시간 리드로우 연산이 메인 스레드에 레이턴시(Keystroke Lag 등)를 재유발하는지 체크하고 필요시 드로틀링(Throttling)을 보완합니다.

---

## 🚀 TO-BE 검토 워크 (시초가 휩소 방지 고도화 로드맵)

> [!IMPORTANT]
> **전략적 핵심 철학**
> "진입 기회를 강제로 더 확장하려 애쓰는 것보다, 통계적으로 기대값이 극히 낮고 자본이 훼손되는 죽는 구간(오전 9:00~9:05 유동성 함정)을 우선적으로 제거함으로써 생존력을 얻는 것이 고도화의 핵심입니다."

### 📅 Phase 1. 시초 회복 감시 Shadow Mode 관찰 (현재 적용 완료)
* **운영 기준**: 최소 **3~5 거래일** 동안 실거래 주문은 차단한 채 Shadow Mode 로그만 축적하며 감시 필터가 작동하는 모습을 지켜봅니다.

### 📅 Phase 2. 구조적 보완 필터 및 실거래(Live Mode) 적용
* **Live Mode 전환**: Shadow Mode 통계 수집 결과 9시 5분 이후 진입의 기대값이 뚜렷이 입증되면, `execution_manager.py`의 `shadow_mode` 설정을 `False`로 전환해 실제 매수 오더가 발행되도록 활성화합니다.
* **진입 슬롯 우선순위 큐(Priority Queue) 도입**: `OP_RECOVERY` 슬롯(최대 5개) 초과 등록 시, `entry_score` 순으로 정렬하여 점수가 가장 높은 최고의 종목만 슬롯을 선점하고 나머지는 예비 후보로 대기하는 알고리즘 고도화를 논의합니다.
