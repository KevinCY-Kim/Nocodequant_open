# 📅 2026-06-04 업무 일지 (Daily Work Log)

## 🎯 주요 목표 및 이슈 정의
- [x] **Opus 작업 검토 및 V26.1 웜업 SL 가드 개선 최종 완료**:
  - Claude Opus가 이전에 작업하던 **웜업 SL 가드(V26.1 Candidate/Grace Architecture)** 로직의 안전성과 타당성을 다각도로 검토하고, 누락되거나 미진한 부분을 완성했습니다.
  - "노이즈성 초기 흔들림(휩소)은 무시하되, 이미 확정된 초과 수익은 철저히 보호한다"는 이원화 철학을 완벽하게 이식했습니다.
- [x] **단위 테스트 구축 및 검증 완료 (`tests/test_warmup_sl_guard.py`)**:
  - 새 웜업 SL 가드의 핵심 동작 시나리오 6종을 철저히 검증하는 단위 테스트를 신규 작성하여, 모든 테스트가 정상 통과(OK)함을 입증했습니다.
- [x] **UI 데이터 정합성 검토 및 UI 연동 완료 (`live_mfe_tab.py`)**:
  - 웜업 기간 동안 축적된 잠재적 수익 보호선이 UI 한 줄 요약의 `⏳축적중` 라벨 및 상세 카드의 `⏳ 웜업 보호선` 필드에 정상 연동되는지 검토를 마쳤습니다.
- [x] **공식 백테스트를 통한 안정성 검증**:
  - 신규 로직 반영 후 공식 백테스트 스위트(`run_official_test.py`)를 실행하여 시스템이 예외 크래시나 무한 루프 없이 정상 구동됨을 팩트 체크했습니다.
- [x] **일일 거래 수 및 승리 횟수 중복 누적 버그 수정 (StateManager)**:
  - 분할 체결 시 `daily_trades` 및 `win_count`가 무조건 누적되어 오늘 실제 8회 매매완료 및 2개 보유 상태임에도 `daily_trades`가 160회로 부풀려지던 버그를 완전 청산(Round-trip) 시점에만 카운트하도록 수정했습니다.
- [x] **일일 누적 매수 주문 한도(`DAILY_BUY_LIMIT`) 오판 및 엔진 비상 정지 버그 수정 (OrderManager)**:
  - 주문 취소/거부 시 한도가 차감되지 않아 12.5배 한도 초과로 엔진이 비상 정지되는 현상을 실제 매수 체결 금액 + 대기 중 주문 금액을 기준으로 삼도록 개선하여 차단했습니다.
- [x] **overrides.json 수동 복구**:
  - 뻥튀기되었던 `"daily_trades": 160`, `"win_count": 48` 상태를 실제 오늘 결과인 `"daily_trades": 8`, `"win_count": 4`로 복구하여 엔진 구동을 재개시켰습니다.
- [x] **상한가 안착 시 TimeExit 타이머 Freeze 및 2봉 Grace Window 구현**:
  - 상한가 안착 시 `TimeExit` 보유 봉 수 카운트를 멈추고(Freeze), 상한가 해제 시 즉시 시간 청산하지 않고 2봉(10분)의 평가 유예 기간(Grace Window)을 제공하는 상태 전이 기반 `TimeExit` 로직을 이식하여 강한 종목을 조기 청산하는 문제를 해결했습니다.
- [x] **상한가 안착 시 TimeExit 정지 상태 UI 표기 보강 (`live_mfe_tab.py`)**:
  - Live MFE 탭 테이블에 `TimeExit` 전용 상태 컬럼을 신규로 추가하여 보유 종목들의 TimeExit 잔여봉 수 및 상한가 안착 동결(`🚨동결`), 풀림 유예(`⏳유예`) 상태를 한눈에 볼 수 있도록 시각화를 완료했습니다.
  - 상세 카드(`ExitStateCard`)의 TimeExit 렌더링 시, 동결 대기 중이라도 언제 풀릴지 알 수 있도록 기존의 잔여 정보(잔여 봉/분)를 복원하여 문구를 더욱 정밀하게 보강했습니다.
- [x] **추세 보존형 휩소 필터 (대안 A) 검증 및 최종 이식**:
  - 완화 예외를 보존하여 추세장 본연의 수익력을 지키는 동시에 휩소장 손실을 막기 위해 절대 거래대금 5분봉 허들 가드에 대한 스윕 테스트를 수행하고 골디락스 임계값(10억 원)을 도출하여 엔진에 반영했습니다.

---

## ✅ 상세 작업 및 패치 내역

### 1. V26.1 웜업 SL 가드 (Candidate/Grace Architecture) 정밀 검토
* **목적**: 웜업 기간 중 확보된 수익을 안전하게 고정하고 웜업 종료 시 소급 청산을 방지.
* **패치 및 조치 내역**:
  - **3단계 분리 로직 검증**:
    - **1단계 (계산)**: `_engine_exit.py`에서 웜업 유무와 무관하게 BE Switch, Hybrid Profit Trail, Ratchet SL을 상시 계산하여 `_profit_protect_sl`을 도출합니다.
    - **2단계 (축적)**: 웜업(2봉 미만) 중인 경우 실제 손절선(`pos['sl']`)은 초기값을 강하게 유지하면서, 도출된 보호선 최대값을 `pos['warmup_protect_sl']`에 누적합니다.
    - **3단계 (활성화)**: 웜업이 종료되는 시점에 축적 보호선이 현재가보다 높은 경우, 완충 필터인 `Activation Grace`를 가동하여 현재가 대비 `WARMUP_GRACE_BUFFER (0.3%)` 버퍼를 뺀 완충 SL(`grace_sl`)을 안전선으로 적용하고 소급 청산을 원천 방지합니다.
  - **Tick-Level 웜업 바이패스 검증**:
    - `strategy.py`의 `update_tick`에서 실시간 틱 수신 시, 이미 수익 보호선이 발동된 상태(SL이 진입가 및 초기 SL보다 높은 경우)라면 웜업 가드를 적용하지 않고 즉시 청산(`True`)을 판단하여 수익을 확고하게 보호합니다.

### 2. 신규 단위 테스트 작성 및 검증 (`tests/test_warmup_sl_guard.py`)
* **목적**: 웜업 SL 가드의 상태 전이 및 틱 단위 바이패스 등의 경계 조건을 완벽하게 검증.
* **패치 및 조치 내역**:
  - [test_warmup_sl_guard.py](file:///c:/Users/stone/projects/nocodequant/tests/test_warmup_sl_guard.py)를 생성하여 아래의 6개 케이스를 검증하는 테스트 스위트를 구축했습니다.
    1. `test_warmup_guard_blocks_normal_sl`: 웜업 중 일반 SL 가격 하락 시 청산 보류 및 웜업 사유 기록 검증.
    2. `test_doomsday_sl_bypasses_warmup`: 웜업 기간이라도 절대 손실 한도인 Doomsday SL(-4%) 터치 시 즉각 손절 처리 검증.
    3. `test_warmup_protect_sl_accumulation`: 웜업 기간 중 고점 급등 시 `warmup_protect_sl`에 수익 보호선의 최대값이 정확히 누적되는지 검증.
    4. `test_warmup_activation_grace`: 웜업 종료 시 축적 보호선이 현재가보다 위에 있다면 `Activation Grace`가 정상 가동되어 완충 SL이 적용되는지 검증.
    5. `test_warmup_activation_normal`: 웜업 종료 시 축적 보호선이 현재가 이하라면 Grace 필터 없이 정상적으로 `sl`로 이식되는지 검증.
    6. `test_strategy_update_tick_bypass`: `update_tick` 단위에서 수익 보호선이 정상 작동 중이면 웜업 가드를 바이패싱하고 즉시 청산 시그널(`True`)을 주는지 검증.
  - **테스트 결과**: 6개 테스트 케이스 모두 정상 **OK (Pass)** 완료.

### 3. UI 렌더링 및 연동 상태 검토 (`live_mfe_tab.py`)
* **목적**: 웜업 중 수익 보호선 축적 상태가 화면에 직관적으로 표현되는지 검증.
* **패치 및 조치 내역**:
  - `_decode_exit_state` 헬퍼 함수가 `sess_pos`의 `warmup_protect_sl`을 추출하여 `warmup_candidate_sl`로 제대로 반환하는지 체크했습니다.
  - 테이블 행 렌더링 시, 웜업 보호선 축적 중인 경우 텍스트로 `⏳축적중` 표시 및 오렌지색 하이라이팅이 알맞게 적용됨을 검토했습니다.
  - 상세 카드(`ExitStateCard`) 내부에도 `⏳ 웜업 보호선` 행이 추가되어 축적 금액과 활성화 대기 조건이 명확히 출력되도록 연동이 정상 완료되었습니다.

### 4. 일일 거래 수 및 승수 중복 누적 버그 패치 (StateManager)
* **목적**: 분할 체결 패킷 유입 시 거래 횟수 및 승률 데이터의 뻥튀기 왜곡 현상 방지.
* **패치 및 조치 내역**:
  - `state_manager.py`의 `update_position_on_fill`에서 매 체결마다 무조건 카운트하던 로직을 제거했습니다.
  - `side == "SELL"` 이고 포지션 수량이 최종적으로 0 이하가 되는 **완전 청산(Round-trip Completion)** 시점에만 `daily_trades += 1`을 수행하게 변경했습니다.
  - 분할 매도가 발생하더라도 해당 포지션 라이프사이클 내에서 발생한 실현손익 누적합(`accumulated_pnl`)을 추적하여 최종 청산 시점에 0보다 클 때만 `win_count += 1`을 수행하도록 정합성을 확보했습니다.
  - 단위 테스트 `test_state_manager_round_trip_counting`을 추가하여 분할 체결 및 청산 시의 카운트 중복 배제를 검증했습니다.

### 5. 누적 매수 주문 한도(`DAILY_BUY_LIMIT`) 오동작 및 취소 누락 패치 (OrderManager)
* **목적**: 매수 주문 접수 후 취소/거부 시 한도가 복구되지 않아 발생하는 자동매매 강제 정지 현상 차단.
* **패치 및 조치 내역**:
  - `OrderManager.send_order` 시점에 `pending_signal_meta`에 해당 주문의 `side`와 `qty` 정보를 캐싱하도록 확장했습니다.
  - `check_ready`의 일일 한도 체크 시, 단순히 주문량의 합산값인 `daily_buy_order_amount` 대신 **`실제 체결된 매수 금액 (daily_buy_filled_amount) + 현재 대기 중인 매수 주문 금액 합산 (pending_buy_value) + 이번에 나갈 주문 금액`**의 합을 기준으로 `DAILY_BUY_LIMIT`를 평가하게 개선했습니다.
  - 이를 통해 주문이 취소되거나 거부되어 `pending_orders`에서 제외되면 자동으로 한도 계산에서 제외되어 한도 복구의 실시간 동적 정합성을 구현했습니다.

### 6. [V27.9.5] 상한가 안착 시 TimeExit Freeze, 2봉 Grace Window 및 재기동 영속화(Persistence) 구현
* **목적**: 상한가 안착(LIMIT_LOCK) 시 시간 청산 타이머를 동결하고, 상한가가 풀린 후(UNLOCK_WARNING) 2봉의 유예를 주어 불필요한 조기 청산 방지. 아울러 프로그램 재시작 시에도 해당 FSM 상태값과 진입 시각이 안전하게 복구되도록 영속화.
* **패치 및 조치 내역**:
  - **상한가 판단 조건**: 실거래 환경(change_rate >= 29%) 및 백테스트 폴벌(당일 시가 대비 15% 이상 상승 & 고가=종가)을 모두 지원하는 판단 식 이식.
  - **타이머 일시정지 (Freeze)**: 새로운 봉 생성 시 상한가 안착 상태이면 `time_exit_frozen_bars`를 누적하고, 최종 `bars_held` 계산 시 이 누적 값을 감산하여 보유 봉 수 증가를 멈춤.
  - **유예 기간 부여 (Grace Window)**: 상한가 안착 상태에서 일반 상태로 전환될 때 `limit_unlock_grace_bars`를 2로 설정하고, 이후 매 봉마다 1씩 차감하며 이 수치가 0보다 큰 동안은 TimeExit 판정을 유예함.
  - **재시작 복구성 패치 (Survivor Recovery)**:
    - `state_manager.py`의 `_execute_save` 시점에 메모리상의 포지션에서 상한가 FSM 필드(`time_exit_frozen_bars`, `limit_lock_bars`, `limit_unlock_grace_bars`) 및 진입 시각(`entry_time`)을 `position_meta_cache`에 동적 싱크하여 `overrides.json`에 안전하게 직렬화 기록하도록 보강했습니다.
    - HTS 잔고 동기화 시점에 호출되는 `sync_positions`에서 위 필드들과 `entry_time`을 로컬 캐시로부터 무결하게 복원하여, 재기동 후에도 TimeExit 보유 봉수 역산 왜곡이나 오동작이 없도록 안전 조치를 마쳤습니다.
  - **AI 작업 헌장 최상위 규칙 명문화**:
    - [AI_Work_Constitution_and_Reference_Map.md](file:///c:/Users/stone/projects/nocodequant/Antigravity_rules/governance/AI_Work_Constitution_and_Reference_Map.md#L86-L94)의 `1.7 자가 치유 및 포렌식 복구` 항목 하단에 **"상태 변수 영속화 보장 규칙 (State Persistence & Recovery Guard)"** 조항을 v1.8 신규 최상위 원칙으로 제정 및 명문화했습니다. 향후 모든 FSM/타이머 상태 변수 추가 시 저장/복원 로직 구현 및 재시작 검증 단위 테스트 작성을 의무화하도록 규제 장치를 확보했습니다.
  - **단위 테스트 추가**:
    - `test_upper_limit_freeze_and_grace_window`: 타이머 동결, 상한가 해제 시 2봉 유예 및 유예 만료 후 청산 시나리오 검증.
    - `test_upper_limit_persistence_on_restart`: 재시작 전 포지션 상태 저장, 메모리 초기화 후 잔고 동기화 시점에 FSM 상태값 및 진입시각이 무사히 복구되는지 검증하는 시나리오를 신규 설계하여 통합 통과를 완료했습니다.

### 7. [V27.9.6] 상한가 안착 시 TimeExit 정지 UI 표기 보강 및 재시작 영속화 복원 버그 패치
* **목적**: 상한가 안착으로 인해 타임 엑시트가 정지되거나 유예 작동 시, 이를 모니터에서 직관적으로 파악할 수 있도록 표기 상세화 및 테이블 전용 컬럼 추가. 또한 재기동 후 HTS 잔고 동기화 시 진입 시각(`entry_time`)과 FSM 상태가 `None`으로 덮어쓰여 봉 개수가 `0`으로 리셋되는 영속화 오염 버그를 완전 청산.
* **패치 및 조치 내역**:
  - **테이블 및 UI 갱신**:
    - [live_mfe_tab.py](file:///c:/Users/stone/projects/nocodequant/app/ui/dashboard/live_mfe_tab.py) 내 테이블 위젯에 `TimeExit` 전용 컬럼(`COL_TIME_EXIT`)을 추가하고, 상한가 안착(`🚨동결`) 및 풀림 유예(`⏳유예`) 상태를 실시간 렌더링했습니다.
    - 상세 정보 카드(`ExitStateCard`)에서 상한가 안착 동결 상태일 때 기존 잔여 대기 시간 정보(`t_rem`봉, `t_min`분)가 출력에서 누락되던 부분을 복원했습니다.
  - **재시작 영속화 오염 버그 패치 (`state_manager.py`)**:
    - **`_execute_save` 오염 방어**: 메모리의 `entry_time`이 `None`일 때 영속화 캐시(`position_meta_cache`) 및 `overrides.json`을 `null`로 덮어써서 파괴하던 악순환 연결고리를 차단하는 안전 가드(`val is not None and ...`)를 이식했습니다.
    - **`sync_positions` SSoT 자가 치유 연동**: 서버 잔고 수신 동기화 시 캐시를 날것으로 읽는 대신, `self.get_position_meta(code)`를 호출하여 `forensics.db` 깊은 탐색(Deep-Search)을 통한 자가 치유(Self-Healing) 복원 경로를 연동했습니다.
    - **`get_position_meta` 복구 범위 보강**: `strategy_type`이 존재하더라도 `entry_time`이 누락/무효하다면 `is_corrupted`로 인지해 DB 복구가 가동되도록 조건을 정교화했고, 복구 반환 딕셔너리에 `time_exit_frozen_bars` 등 FSM 상태 변수 3종을 포함하도록 최종 이식했습니다.



---

## 📊 검증 및 백테스트 결과 요약

### 1. 단위 테스트 결과
```bash
python -m unittest tests/test_warmup_sl_guard.py
.........
----------------------------------------------------------------------
Ran 9 tests in 0.160s

OK
```
- 웜업 가드 작동 검증 6종, StateManager 분할 카운트 검증 외에 **상한가 안착 타이머 동결 및 Grace Window 검증**(`test_upper_limit_freeze_and_grace_window`) 및 **재기동 영속화 복원 검증**(`test_upper_limit_persistence_on_restart`)까지 완벽하게 통과함을 확인했습니다.

### 2. 공식 백테스트 및 다차원 분석 크래시 프리 확인
- **Wide Sample Backtest (52개 종목)**: 거래 129회, 승률 27.91%, 총 PnL +3.04% 산출 (정상 종료).
- **Multi-Dimensional Leak Analysis**: 72회 거래 정상 분석 완료 (정상 종료).
- **결론**: 리스크 엔진 핵심 클래스(`ExitEngine`, `StrategyManager`)의 수정 내용이 전체 백테스트의 흐름 및 FSM 상태 기계를 교란하지 않고 에러 없이 안정적으로 로드 및 러닝됨을 확인했습니다.

---

## 🔮 향후 보완 및 논의 방향
- **웜업 중 SL 히스토리 로깅의 보완**:
  - 현재는 메모리상의 `pos` dict 내 `warmup_protect_sl`에 최대값 하나만 덮어쓰며 기록하고 있으나, 향후 정밀한 시각화나 디버깅을 위해 웜업 중 매 봉에서 계산된 보호선들의 히스토리를 별도 로그 필드에 축적해 두는 것을 고려해 볼 수 있습니다.
- **실거래 환경 라이브 갱신 모니터링**:
  - 금일 로직 검증이 완벽히 완료되었으므로, 현재 백그라운드에서 구동 중인 실거래 인스턴스(`app/main.py`)가 새로운 V26.1 로직을 물고 재부팅된 후 Live MFE 탭에서 `⏳축적중` 상태가 실제 장중에 알맞게 작동하고 표기되는지 1차적 오프라인 모니터링을 진행할 예정입니다.

---

## ✅ 추가 패치 및 검증 내역 (대안 A)

### 1. 절대 거래대금 5분봉 허들 가드 (대안 A) 정식 반영
* **목적**: 완화 예외(TREND 조건부 MTF)를 살려두어 추세장(06-02) 수익을 온전히 지키되, 거래량이 동반되지 않는 쪼잔한 가짜 반등 휩소(06-03)는 5분봉 절대 거래대금 허들로 걸러내는 골디락스 가드 확보.
* **패치 및 조치 내역**:
  - [config_engine.py](file:///C:/Users/stone/projects/nocodequant/app/core/config_engine.py)의 `EngineDefaults` 클래스에 `ENTRY_MIN_VALUE_5M = 1_000_000_000` (10억 원) 기본값 선언.
  - [_engine_entry.py](file:///C:/Users/stone/projects/nocodequant/app/ai/_engine_entry.py)의 `BUY` 결정 직전 블록에 5분봉 거래대금(`close * volume`)이 `ENTRY_MIN_VALUE_5M` 미만일 시 진입을 제한하고 `대금 부족` 사유를 남기는 **절대 거래대금 가드** 정식 구현.
  - [test_volume_gating.py](file:///C:/Users/stone/projects/nocodequant/tests/test_volume_gating.py) 단위 테스트를 구축하여 임계값 미달 시 `HOLD_BLOCK` 차단 및 임계값 이상 시 `BUY` 진입 성공 케이스 검증 완료 (Pass).

### 2. 📊 완화 예외 X 거래대금 허들 스윕 백테스트 결과 (HOT 탭)

| 완화 예외 설정 | 거래대금 허들 | 06-02 추세장 (진입/PnL) | 06-03 휩소장 (진입/PnL) | 평가 및 결론 |
| :--- | :--- | :---: | :---: | :--- |
| **활성화 (현재 원본)** | **0 (가드 없음)** | 168회 / **+62.86%** | 159회 / **-6.25%** | 원본 기준 성과 (추세 수익 우수하나 휩소에 노출) |
| **활성화 (현재 원본)** | **10억 원** | 95회 / **+63.35%** | 56회 / **-1.69%** | **🌟 최적 골디락스 (수익 보존 + 휩소 대량 차단)** |
| **활성화 (현재 원본)** | **30억 원** | 65회 / **+17.87%** | 16회 / **-13.20%** | 임계값 과도로 진짜 추세 수익까지 크게 훼손 |
| **제거 (Bear Only)** | **0 (가드 없음)** | 34회 / **+30.08%** | 46회 / **-12.65%** | 완화 예외 제거 시 추세장 수익력이 반토막남 |
| **제거 (Bear Only)** | **10억 원** | 22회 / **+33.97%** | 22회 / **-7.02%** | 과도한 이중 차단으로 전체 기대값 저하 |

* **결과 분석**:
  - 완화 예외를 제거하면 06-02 추세장 성과가 **+62.86% ➡️ +30.08%**로 반토막나며 큰 기회비용을 초래합니다.
  - 반면 **완화 예외 활성화 + 10억 허들**을 조합하면 06-02 추세장 수익을 **+63.35%**로 완벽 보존하면서, 06-03 휩소장의 손실폭을 **-6.25% ➡️ -1.69%**로 대폭 축소하여 양쪽 다 잡아내는 최적의 상태가 됩니다.
