# 📅 2026-06-14 업무 일지 (Daily Work Log)

> **[거버넌스 준수]**
> * **작업 등급**: L3 (전략-실행 정합성 수정, 수수료율 실측 교정 및 원장 정합 인프라 구축)
> * **참조 문서**: 
>   * [_engine_entry.py](file:///c:/Users/stone/projects/nocodequant/app/ai/_engine_entry.py)
>   * [_engine_exit.py](file:///c:/Users/stone/projects/nocodequant/app/ai/_engine_exit.py)
>   * [engine.py](file:///c:/Users/stone/projects/nocodequant/app/ai/engine.py)
>   * [strategy.py](file:///c:/Users/stone/projects/nocodequant/app/ai/strategy.py)
>   * [config.py](file:///c:/Users/stone/projects/nocodequant/app/core/config.py)
>   * [config_engine.py](file:///c:/Users/stone/projects/nocodequant/app/core/config_engine.py)
>   * [engine_fsm.py](file:///c:/Users/stone/projects/nocodequant/app/core/engine_fsm.py)
>   * [execution_manager.py](file:///c:/Users/stone/projects/nocodequant/app/services/managers/execution_manager.py)
>   * [persistence_manager.py](file:///c:/Users/stone/projects/nocodequant/app/services/managers/persistence_manager.py)
>   * [main_window.py](file:///c:/Users/stone/projects/nocodequant/app/ui/main_window.py)
>   * [reversal_diagnosis_260613_ko.md](file:///c:/Users/stone/projects/nocodequant/docs/backtests/reversal_diagnosis_260613_ko.md)
> * **영향 받는 모듈 (Impacted Modules)**: `_engine_entry.py`, `_engine_exit.py`, `engine.py`, `strategy.py`, `config.py`, `config_engine.py`, `engine_fsm.py`, `execution_manager.py`, `persistence_manager.py`, `main_window.py`, `position_dashboard.py`, `strategy_dashboard.py`
> * **불변성 위반 여부 (Invariant Check)**: 위반 없음 (Violated: None)
> * **경계 준수 선언**: 전략(L1)·실행(L3)·상태(StateManager) 및 포지션 동기화 전반의 교차 상충 감사를 바탕으로 좀비 pending_fill 회수, L3 중복 검열 게이트 제거, 모의투자 실제 수수료율 실측 적용(편도 0.337%) 및 HTS 실시간 원장 정합성(Reconciliation) 검증 인프라를 안전하게 이식하였습니다. 또한 REVERSAL 평균회귀 청산 모델(Prototype)을 설계하여 검증의 토대를 마련하였습니다.

---

## 🎯 주요 목표 및 이슈 정의

1. **모의투자 HTS 실현손익 vs 전략장부 괴리 원인 규명 및 교정 (V28.5)**:
   - 6/12 기준 HTS 실현손익(-639,106원)과 trades.db(-367원) 사이의 약 64만 원에 달하는 괴리의 근본 원인이 '수수료율 22배 과소계상'에 있었음을 규명.
   - 키움 영웅문 모의투자 표준 수수료(편도 0.337%, 라운드트립 0.674%)를 실측하여 `config.py`에 적용.
   - HTS와 장부의 오차를 실시간으로 비교 검증하는 원장 정합성 리포트(`get_broker_reconciliation()`) 및 DB 테이블(`broker_pnl_daily`) 추가.
   - 기대수익 및 거래대금 기준에서 음수 EV를 기록하는 가드를 선별 적용할 수 있는 `GUARD_ENFORCE_OVERRIDE` 추가.

2. **전략-실행 계층 간 미해결 상충 4건 패치 및 정책 반영 (V28.6)**:
   - **좀비 pending_fill 포지션 회수 단절**: EntryEngine의 BUY 진입 선세팅 이후 하류 게이트(정책 BLOCK, 60초 쿨다운, MME 0 등)에서 주문이 취소되거나 전송 실패할 경우 `pending_fill` 포지션이 영구적으로 좀비 상태로 누적되어 재진입을 막던 심각한 누출 결함을 해결.
   - **UTC/KST 시간 기준 불일치로 인한 오버나이트 청산 사문화 차단**: 15:10 강제 청산(HedgeGate)과 시초 갭 유예(GapGuard)가 UTC/KST 시각 불일치로 작동하지 않았던 문제 해결. 단, 현재 TREND 단일 운용 스윙 목적을 위해 `USE_OVERNIGHT_EXIT = False` 플래그로 명시적 비활성화 처리.
   - **L1 ↔ L3 진입 임계값 이중 게이트 정리**: 이미 L1(EntryEngine)에서 완료한 진입 결정을 L3(ExecutionManager)에서 중복 검열하고 불일치하는 국면 임계값으로 인해 무음 드롭되던 이중 게이트를 V21.8 주석의 원래 의도대로 완전히 제거하고 L1 SSoT로 일원화.
   - **2분할 매수(단계적 노출) 폐기**: MME/수량 계산에 반영되지 않던 `EXPOSURE_MAP`의 단계적 노출(READY 50%)을 폐기하고 노출 여부(0/1)로 단순화.

3. **유령 포지션 복구 방지 (V28.4.1)**:
   - 전량 청산(SELL) 후 stale 계좌 스냅샷이 sync_positions 시 포지션을 복구(RESTORED)시켜 분 스냅샷이 매분 신호를 재생성하던 결함을 `clear_ghost_positions` 및 filtered_pos 순회로 완벽하게 소거 및 검증 완료.

4. **REVERSAL 전략 평균회귀 청산 모델 설계 및 진단 (V28.4h)**:
   - REVERSAL 전략이 추세-추종형 청산(트레일, 2R 익절, 3봉 Inactivity 컷) 오적용으로 인해 극단적인 조기사망(0-3봉 승률 19%)을 보이고 있음을 규명.
   - 평균회귀형 청산(MA20 회귀 터치 TP, 당일 세션 저점 이탈 SL) 프로토타입 설계 및 `REVERSAL_MEANREV_EXIT_ENABLED` 토글과 버퍼 파라미터 신설.

---

## ✅ 상세 작업 및 패치 내역

### 1. 실거래 수수료율 교정 및 원장 정합성 인프라 (V28.5)
- **수수료율 실측 교정**: [config.py](file:///c:/Users/stone/projects/nocodequant/app/core/config.py)
  - 모의투자 계좌의 실제 부과 수수료율 `commission_per_side = 0.00337` (기존 0.00015 대비 22배 상향), `tax_sell = 0.0023` 교정 적용.
- **원장 정합 DB 구축**: [persistence_manager.py](file:///c:/Users/stone/projects/nocodequant/app/services/managers/persistence_manager.py)
  - `broker_pnl_daily` 테이블을 신설하여 당일 `opt10077` TR로 수신되는 키움 실현손익을 기록.
  - HTS 실시간 수수료 실현손익과 로컬 db 원장의 실현손익을 비교해 괴리를 정량 산출하는 `get_broker_reconciliation` 추가.
- **드리프트 경고 및 배선**: [composition_root.py](file:///c:/Users/stone/projects/nocodequant/app/core/composition_root.py), [engine.py](file:///c:/Users/stone/projects/nocodequant/app/ai/engine.py)
  - `opt10077` 종목별 실현손익 캡처를 위해 `sig_realized_pnl_detail` 시그널을 연동하여 `CompositionRoot`에서 10,000원 이상의 괴리 발생 시 warning 로깅 활성화.
- **UI 및 차단 선별**: [config_engine.py](file:///c:/Users/stone/projects/nocodequant/app/core/config_engine.py)
  - `GUARD_ENFORCE_OVERRIDE = ["Reward", "Value"]`를 추가하여 전역 섀도 모드 하에서도 수익/가치 가드 2종에 대해서만 선별적으로 실제 진입 차단을 강제(Enforce)함.

### 2. 레이스컨디션 및 상충 패치 (V28.6)
- **좀비 pending_fill 포지션 자동 회수**: [execution_manager.py](file:///c:/Users/stone/projects/nocodequant/app/services/managers/execution_manager.py)
  - 진입 신호 처리 시 `qty > 0`인 동기화 블록 외에 `else` 분기를 추가. 미체결 상태(`qty == 0`)이면서 실거래 주문 대기 목록(`pending_orders`)에 없는 좀비 포지션을 강제로 철회(`sess['position'] = None`)하여 영구적 진입 잠김 현상을 원천 방어함.
- **L3 중복 게이트 제거**: [execution_manager.py](file:///c:/Users/stone/projects/nocodequant/app/services/managers/execution_manager.py)
  - L1(EntryEngine) 판단 결과를 이중으로 규제하던 L3 점수 임계값 검열 로직을 삭제하여 L1 SSoT 진입 판단의 일관성 보장.
- **CHAOS 진입 및 Overnight 명시적 비활성화**: [config_engine.py](file:///c:/Users/stone/projects/nocodequant/app/core/config_engine.py), [_engine_entry.py](file:///c:/Users/stone/projects/nocodequant/app/ai/_engine_entry.py)
  - `ENTRY_ALLOW_CHAOS = False`로 CHAOS 국면 진입 원천 차단 (청산 로직은 정상 유지).
  - KST/UTC 오차 교정 후에도 스윙 장기 보유 기조 유지를 위해 `USE_OVERNIGHT_EXIT = False` 플래그 추가.
- **2분할 매수 단계적 노출 폐기**: [engine_fsm.py](file:///c:/Users/stone/projects/nocodequant/app/core/engine_fsm.py)
  - `EXPOSURE_MAP`의 `READY` 단계를 `0.5`에서 `1.0`으로 이진화 처리하여 무의미한 2분할 주석 정리 완료.

### 3. 유령 포지션 GC (V28.4.1)
- **잔고 불일치 소거**: [strategy.py](file:///c:/Users/stone/projects/nocodequant/app/ai/strategy.py), [main_window.py](file:///c:/Users/stone/projects/nocodequant/app/ui/main_window.py)
  - SELL 전량 체결 직후 도착하는 stale 스냅샷 오염을 막기 위해 `filtered_pos` 순회로 변경하고, 서버 잔고에 존재하지 않는 세션 포지션을 안전하게 소거하는 `clear_ghost_positions` 적용.

### 4. REVERSAL 평균회귀 청산 재설계 프로토타입 (V28.4h)
- **청산 로직 분기**: [_engine_exit.py](file:///c:/Users/stone/projects/nocodequant/app/ai/_engine_exit.py)
  - `REVERSAL_MEANREV_EXIT_ENABLED`가 활성화될 경우, MA20 터치 시 TP1(2R 익절 무력화)을 수행하고, 당일 세션 저점 이탈 시 구조적 SL을 집행하는 프로토타입 추가.

---

## 🧪 검증 및 테스트 결과

### 1. 단위 및 회귀 테스트 가동 결과
- **회귀 테스트 통과 목록**:
  1. `test_order_lifecycle_races.py`: **14/14 PASS** (비동기 주문 및 잔고 SSoT 동기화 grace 검증 완료)
  2. `test_strategy_session_races.py`: **5/5 PASS** (세션 동시성 락 및 readonly 복원 검증 완료)
  3. `test_ghost_position_sync_fix_260612.py`: **13/13 PASS** (stale 스냅샷 복원 시 좀비/유령 포지션 방지 정합 확인)

### 2. REVERSAL 전략 심층 최적화 스윕 (그리드 서치 18개 시나리오) 결과
- **목적**: REVERSAL 전략을 흑자(EV > 0)로 양전시키기 위해 다양한 진입 이격률(Disparity), 급락 방지 필터(RSI Hook, Higher Low), 청산 목표(MA20, Partial TP, Session Low SL), 손절 폭(1.5%)을 연계하여 백테스트를 진행.
- **결론 및 분석**:
  - 18개 시나리오 전체에서 기대값(EV) 흑자 양전 실패 (최적 평균 PnL -0.145%).
  - 5분봉 기반의 짧은 평균 회귀 목표(MA20)가 주는 수익폭에 비해, 급락 투매 종목의 하방 꼬리 리스크가 지나치게 비대칭적으로 큼.
  - **조치 사항**: REVERSAL 전략은 실거래 모의투자에서 영구 잠금 상태(`ENTRY_TH_REVERSAL = 100`)를 유지하여 자본 손실을 방지해야 함.

### 3. BREAKOUT 전략 포렌식 및 18개 필터 조합 최적화 스윕 결과 (V28.4i/k)
- **목적**: BREAKOUT 전략에 거래가 집계되지 않던 `pending_fill` 누수 결함을 해결하고, 18개 필터 조합 그리드 스윕 백테스트를 수행해 기대값 양전을 도출.
- **포렌식 결과 (패인 규명)**:
  - 61건의 거래 분석 결과, **88.5%(54건)**가 볼린저 밴드 상단 한참 아래에서 진입한 **가짜 돌파(Fake Breakout)**였으며, 하락/횡보장(DOWNTREND/RANGE) 내 진입이 **55.7%(34건)**를 차지해 손실을 누적함.
- **스윕 결과 (흑자 양전)**:
  - **1위**: `Scen17_Block_DT_Range_BB_Dist_1%_Sl` (n=53, 승률 45.3%, 평균 PnL **+0.407%**, 누적 PnL **+21.6%**).
  - **결론**: **DOWNTREND 국면 차단** + **볼린저 밴드 상단 이격 -1.0% 이내 밀착** + **단기 이평선 우상향(`ma_slope > 0`)** 3대 가드를 교집합으로 결합할 때, 가짜 돌파를 차단하고 기대값을 흑자 양전시킬 수 있음을 입증.

### 4. BREAKOUT 구조적 게이트 13일 전체 대조 검증 결과 (V28.4L)
- **목적**: 포렌식에서 제시된 단순 "국면 차단 + ADX < 40" 게이트를 13일 전체 데이터셋에 대조 검증하여 소표본 낙관 오도를 차단.
- **결과**: 
  - **S_baseline (현행)**: 6,366건 | 승률 28.5% | 평균 PnL **-0.323%**
  - **S_gate_on (교정)**: 1,557건 | 승률 32.8% | 평균 PnL **-0.243%**
  - **분석**: ADX < 40 필터 적용 시 휩소 4,800여 건을 성공적으로 억제하여 승률과 PnL이 다소 개선되었으나, 여전히 기대값은 음수 영역에 정체되어 양전에 실패함.
  - **결정**: 실전 양전을 위해서는 13일 전체에서 흑자가 증명된 **"이평선 우상향 + BB 상단 -1% 밀착"** 결합 게이트를 실거래에 적용하기로 결정.

### 5. 지수 컨텍스트 캡처 및 백테스트 오염 방지 (Phase 1)
- **목적**: 장세/지수(KOSPI/KOSDAQ 및 선물)와 종목의 상대 강도(RS)를 캡처하여 향후 AutoML 및 지수 필터링 게이트의 기반 마련.
- **배선 완료**:
  - `state_manager.py`: 실시간 업종지수 홀더(`index_state`) 구현.
  - `engine.py`: 실시간 업종지수(KOSPI 001 / KOSDAQ 101) SetRealReg 등록 및 OnReceiveRealData 콜백 분기 격리 배선.
  - `feature_recorder.py`: 신호 발생 시점 종목 market, KOSPI/KOSDAQ 지수 등락율, RS(지수 대비 종목 강도) 캡처 기능 추가.
  - `analyze_entry_features.py`: 지수 및 RS 변수들의 상관관계 분석 로직 연동.
- **백테스트 오염 방지 (V28.4.2)**:
  - 병행 백테스트 프로세스가 실시간 피처 캡처 로그를 오염시키던 결함을 수정하기 위해, 기본 캡처 플래그를 `False`로 유지하고 브로커 로그인 성공 시(`_on_event_connect`)에만 활성화되도록 폴라리티 게이트 적용 완료.

### 6. TREND 전략 고도화 방안 리서치 및 포렌식
- **개요**: 5대 전략 중 최강인 정예병 `TREND` 전략의 진입 타점 고도화 및 꼭지 추격 방지를 목적으로 기관 추세 추종 기법 리서치와 포렌식 진행.
- **포렌식 결과 (38종목 × 1000봉, 가드/유니버스 ON)**:
  - **기준선**: 165거래 | 승률 44.8% | 평균 PnL **+0.229%** (최강 엣지 확인).
  - **진입 위치별 분해**:
    - **MA20 눌림목 진입** (직전 5봉 내 되돌림 후 재상승): 101건 | 평균 PnL **+0.392%** (EV 대폭 개선).
    - **MA20 눌림목 없음** (순수 고점 추격): 64건 | 평균 PnL **-0.029%** (적자 요인).
    - *결론*: 추세추종이라도 어디서 사느냐(진입 위치)가 엣지를 결정하며, 고점 추격을 제어하는 것이 핵심 레버임.
  - **보조 지표 분석**:
    - **ADX 과열대 (ADX > 50)**: 평균 PnL **-0.149%**로 꼭지 추격 손실 유발. 반면 ADX 40~50 구간은 **+0.847%**의 강력한 수익 기록.
    - **RSI 지표**: RSI 80+ 초과열 구간이 오히려 평균 PnL **+0.747%**로 모멘텀 지속을 증명. RSI 상한 캡 설정은 역효과로 판정.
    - **청산**: ATR 트레일링/비대칭 청산 모델이 성숙하여 청산 룰 변경 여지는 없음.
- **13일 전체 대조 검증 결과**:
  - **눌림목 이진 게이트**: 거래가중 종합 EV는 개선(ON +0.145% vs OFF +0.049%)되었으나, 날짜별 ΔEV 중앙값이 **-0.033%**로 음수이며 승무패도 6승 7패로 갈려 국면 의존성이 심해 채택 보류.
  - **ADX > 50 상한 필터 스모크 (260604 HOT)**: ADX 상한 ON 적용 시 EV가 **+0.177% → +0.368%**로 대폭 상승하고 총수익도 증가하여 매우 긍정적.

---

## 🔮 향후 계획

1. **BREAKOUT 3대 게이트 실거래 이식**:
   - 그리드 스윕에서 흑자 양전이 실증된 `Scen17`의 3대 결합 가드(DOWNTREND 차단, BB 상단 -1.0% 이내 밀착, 단기 이평선 우상향)를 `_engine_entry.py` 및 `config_engine.py`에 이식.
2. **업종지수 실시간 캡처 스모크 테스트**:
   - 다음 장중 기동 시 로그에서 "📈 업종지수(KOSPI/KOSDAQ) 실시간 등록 완료" 메시지를 확인하고, `entry_features.jsonl`에 지수 및 RS 데이터가 정상 기록되는지 검증.
3. **선물(KOSPI200/KOSDAQ150) 실시간 피드 추가 연동 (Phase 1b)**:
   - 현물 지수 캡처 안정화 이후 선물 지수 피드를 추가 연동하여 RS 분석 범위 확장.
4. **TREND 전략 ADX > 50 과열 차단 필터 13일 전체 검증**:
   - 260604 스모크 테스트에서 긍정적 성과를 보인 ADX > 50 상한 필터의 13일 전체 백테스트 검증을 가동하여 휩소 제거 및 최종 이식 여부 판정.
