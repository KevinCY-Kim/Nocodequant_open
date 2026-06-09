# Failure Patterns and Guards (실패 패턴 및 방어) [V27.0]

## 1. Purpose (목적)
본 문서는 NoCodeQuant 시스템이 실제로 방어하고 있는 **알려진 실패 모드(Known Failure Modes)**와 그에 대한 **방어 로직(Defense Logic)**을 매핑합니다. 본 문서에 기술된 리스크 제어 로직은 [AI 작업 헌장 v1.8](file:///C:/Users/stone/projects/nocodequant/Antigravity_rules/governance/AI_Work_Constitution_and_Reference_Map.md)의 **L3(로직/수행)** 등급으로 관리됩니다.

---

## 2. Technical Failures (기술적 실패)

### 2.1 The "API Timeout" (통신 지연)
-   **Scenario**: 키움 증권 서버로부터 주문 응답이 오지 않거나, 데이터 수신이 중단됨.
-   **Guard**: `KiwoomEngine` 로그 및 `main_window.py` 상태 표시줄.
-   **Action**: `lbl_status`에 통신 상태 표시, 일정 시간 응답 없을 시 재접속 권유 메세지 노출.

### 2.2 The "Double Entry" (중복/스팸 주문) - [Updated V18.11]
-   **Scenario**: 동일 봉(Candle) 내에서 신호가 반복 발생하거나 지연된 데이터가 들어와 중복 주문이 나가는 현상.
-   **Guard**: **Strong Dedup Hash (30s TTL) & Position Integrity**.
-   **Logic**: 
    - `(code, qty, side)` 기반의 고유 해시를 생성하여 30초간 동일 주문 재발송 원천 차단.
    - 주문 전송 직전 포지션 엔진 잔고 조회(`BUY` 시 qty=0, `SELL` 시 qty>0)를 통한 물리적 가드.
    - 체결 또는 취소 확인 시 해당 해시 즉시 해제하여 다음 신호 대기.

### 2.3 The "Painter Crash" (렌더링 충돌)
-   **Scenario**: 멀티스레드 환경에서 UI 갱신 중 `QPainter` 개체가 중첩되어 "Active Painter" 오류 및 크래시 발생.
-   **Guard**: **Painter Lifecycle Mandatory End & Snapshot Iteration**.
-   **Logic**:
    - `paintEvent` 내 `try...finally` 블록을 통한 `painter.end()` 보장.
    - 공유 리스트 순회 시 원본 대신 복사본(`list(self.data)`)을 활용하여 `RuntimeError` 방지.

### 2.4 The "Silent Write Failure" (침묵하는 저장 실패) - [New v15.9.0]
-   **Scenario**: SQLite 명령은 실행되었으나, 트랜잭션 충돌이나 저널링 오류로 인해 실제로 로우(Row)가 삽입되지 않은 상태.
-   **Guard**: **Precision Write Confirmation (Rowcount Check)**.
-   **Logic**:
    - `INSERT` 쿼리 실행 직후 `cursor.rowcount`가 1인지 엄격히 검증.
    - `conn.total_changes` 비교를 통해 물리적 DB 상태 변화를 이중 확인.
    - 실패 시 로그 생성 후 50ms 대기 후 즉시 재시도(Retry Pattern).

### 2.5 The "Database Lockout" (데이터베이스 락 지연) - [Updated v16.0]
-   **Scenario**: 대규모 로그 마이그레이션이나 통계 집계 중 `OperationalError: database is locked`가 발생하여 실시간 매매 로그 저장이 차단됨.
-   **Guard**: **Batch Record Copy & Independent Read/Write Connections**.
-   **Logic**:
    - **No-ATTACH Policy**: 500MB 이상의 대용량 WAL 파일 존재 시 `ATTACH DATABASE`는 체크포인트 지연으로 인해 반드시 실패함. 이를 폐기하고 독립된 두 커넥션을 통한 **배치 복사(Batch Copy)**로 전환.
    - **Vacuum-in-Isolation**: 마이그레이션 완료 후, 어떤 트랜잭션도 열려있지 않은 상태에서만 별도 프로세스/커넥션으로 `VACUUM`을 실행하여 파일 용량 축소.
    - `busy_timeout=20000` (20초) 및 어플리케이션 레벨의 **Exponential Backoff Retry** 유지.

### 2.6 The "Ghost DB Instance" (유령 DB 인스턴스) - [New v16.0]
-   **Scenario**: `TradeLogger`, `ForensicDashboard` 등 여러 모듈에서 `PersistenceManager()`를 각각 호출하여 서로 다른 커넥션이 동일 DB 파일을 점유, 교착 상태 유발.
-   **Guard**: **Strict Singleton & Process-level Lock**.
-   **Logic**: `__new__` 메서드 내에서 `threading.Lock`을 사용하여 단일 인스턴스 생성을 강제하고, 기존 인스턴스 존재 시 즉시 반환하여 커넥션 중복을 원천 차단.

### 2.7 The "Fill Inflation" (체결 데이터 증폭) - [Updated V19.8.9c]
-   **Scenario**: 키움 API가 동일 주문에 대해 다수의 체결 코드를 송신하거나, 모의투자 환경에서 부분 체결 시 동일한 체결번호(FID 912)를 반환하여 후속 체결이 차단(Silent Block)되는 현상.
-   **Guard**: **Composite fill_id Normalization**.
-   **Logic**: 
    - `fill_id = f"{base_id}_{unfilled_qty}"` (FID 912 + FID 902) 조합을 강제함.
    - 미체결 수량은 매 부분 체결 시마다 반드시 변동되므로, 브로커 식별자가 중복되더라도 엔진 레벨에서 절대적 고유성을 확보하여 누락 없는 DB 저장을 보장함.

### 2.8 The "Holding Time Inflation" (보유 시간 왜곡) - [New V19.6]
-   **Scenario**: 단순 시간 차이(Exit - Entry)로 보유봉수를 계산할 경우, 장 마감 이후나 주말이 포함되어 수익률 대비 보유 시간이 뻥튀기되는 현상.
-   **Guard**: **Candle-Based Time Calculation**.
-   **Logic**: Timestamp 대신 차트의 실제 봉(Bar) 개수를 기반으로 시간을 추산하거나, KOSPI 장 운영시간(09:00-15:30)만 가산하는 전용 유틸리티 적용.

### 2.9 The "Zombie Trade Resurrection" (좀비 트레이드 부활) - [New V19.8.9]
-   **Scenario**: 수동 매수 시 메타데이터 유실로 인해 `StateManager`가 DB에서 과거의 `BUY` 기록을 무작위로 찾아와 현재 포지션에 덮어씌움으로써 보유 시간 및 지표가 심각하게 왜고되는 현상.
-   **Guard**: **Last-Event Aware Recovery (Stateful Recovery)**.
-   **Logic**: 
    - `forensics.db`에서 특정 종목의 가장 최근 기록을 조회할 때 무조건 `BUY`만 찾는 것이 아니라, **절대적 최근 이벤트**를 조회함.
    - 만약 그 최신 이벤트가 `SELL`이라면, 과거 포지션이 이미 종결되었음을 뜻하므로 복구 로직을 즉시 취소(`return {}`)함.

### 2.10 The "Null-Time Comparison Crash" (Null 시점 비교 크래시) - [New V19.8.9c]
-   **Scenario**: 매도 체결 시 메타데이터 부재로 `entry_time`이 `None`일 경우, `calculate_holding_bars` 함수 내에서 `datetime` 객체와 비교 연산 시 `TypeError`가 발생하여 엔진이 중단됨.
-   **Guard**: **Defensive Type Guard (Pre-Comparison Check)**.
-   **Logic**: 
    - `utils.py:calculate_holding_bars` 함수 진입 시점에 `entry_time`의 유효성(`None`, `'-'`, `""` 등)을 우선 검사하여 부적절한 값이 감지될 경우 비교 연산 수행 전 즉시 `0`을 반환함.

### 2.11 The "Layout Collapse" (레이아웃 붕괴 및 밀림 현상) - [Updated V19.9.2]
-   **Scenario**: 실시간 데이터 팽창, 긴 종목명, 혹은 OS/테마에 따른 숨은 여백 폭발로 인해 3단 컬럼 구조가 깨지는 현상.
-   **Guard**: **Constraint-Based Layout System**.
-   **Logic**: 하위 컴포넌트의 Stretch 요소를 강제 격리하고 중앙 확장만 허용.

### 2.12 The "Zero-R Explosion" (R-Multiple 수치 폭주) - [New V20.9.1]
-   **Scenario**: 초저변동성 종목에서 진입가와 손절가가 거의 같을 경우, 분모인 R값이 0에 수렴하여 `current_r`이 무한대로 발산하고 로직이 붕괴되는 현상.
-   **Guard**: **R-Value Floor Guard**.
-   **Logic**: `R = max(entry - initial_sl, entry * 0.005)`. 리스크 단위를 최소 진입가의 0.5%로 강제 보정하여 정규화 안정성 확보.

### 2.13 The "Trailing Chattering" (트레일링 배수 급변동) - [New V20.9.1]
-   **Scenario**: 특정 R-Multiple 경계선(예: 3.2R) 근처에서 가격이 미세하게 진동할 때, 트레일링 배수가 매 봉마다 바뀌며 의도치 않은 청산을 유발하는 현상.
-   **Guard**: **Exit-Hysteresis Buffer**.
-   **Logic**: 전환 임계값에 `+0.2R` 수준의 여유 마진을 부여하여 상태 천이의 관성을 확보.

---

## 3. Logic & Scoring Failures (로직 및 점수 실패)

### 3.1 The "Numerical Explosion" (수치적 발산)
-   **Scenario**: 급변동장에서 지표 값이 상상을 초월하는 수준으로 튀어 정규화 로직이 붕괴됨.
-   **Guard**: **Soft Dynamic Clipping ($\pm 10\sigma$)** (`strategy.py`).
-   **Mechanism**: 입력된 Raw 지표 값을 최근 100개 캔들의 표준편차로 나누되, 그 결과가 10을 넘지 않도록 클램핑(Clamping). 
-   **Rationale**: 10-sigma는 '블랙 스완'을 수용하기에 충분하면서도, 수학적 계산이 파괴되는 것을 막아줌.

### 3.2 The "Neutral Noise" (중립 소음)
-   **Scenario**: 박스권 횡보장에서 지표들이 미세하게 상하방을 오가며 매매 횟수만 늘리는 현상.
-   **Guard**: **Scoring Buffer & Confidence Gate**.
-   **Rule**: 40~60점 구간은 무조건 회색(중립)으로 처리하며, 신뢰도가 25% 미만이면 신호 생성 자체를 차단 (`MIN_CONFIDENCE`).

### 3.3 The "Regime Shock" (국면 전환 충격)
-   **Scenario 1 (Transition)**: 타임프레임 변경 직후, 이전 데이터의 잔상이 남아 잘못된 신호를 생성함.
-   **Scenario 2 (Chattering)**: ADX 임계값 부근에서 국면이 잦게 변동하며 전략적 혼선을 야기함. [V14.0]
-   **Guard**: **Reset-on-Change & ADX Hysteresis**.
-   **Action**: 
    - 타임프레임 스위칭 시 `ignore_next_signal` 플래그 활성화.
    - ADX 22-25 히스테리시스 구간 설정을 통해 국면의 잦은 스위칭 방지.

### 3.4 The "Falling Knife" (폭락주 뇌동매매) - [Updated v18.45]
-   **Scenario**: ADX의 비방향성 맹점으로 인해 급락 중인 종목이 '추세적 수급'으로 오인되어 매수 신호가 발생하는 현상.
-   **Guard**: **SSoT Directional Guard & Downtrend Penalty**.
-   **Logic**: 
    - **SSoT Direction**: `_get_signal_internal`에서 MA20/MA60 이격 및 기울기를 통해 `is_downtrend`를 확정하고 하위 연산에 강제 주입.
    - **Hard Cutoff**: -7% 미만 종목 즉각 퇴출 (Lightning Cutoff).
    - **Penalty**: 하락장 국면에서 리스크 블록 -20점 즉각 부여 및 BB 반등 임계값(`85.0`) 상향.

### 3.5 The "Institutional Noise" (파생 상품 소음) - [New v13.8.10]
-   **Scenario**: KODEX, TIGER 등 ETF/ETN 종목들이 거래대금 상위를 점유하여 개별 주식의 수급 분석을 방해함.
-   **Guard**: **Keyword-based Institutional Filter**.
-   **Action**: 종목명 키워드 필터링을 통해 파생 상품군을 유니버스에서 원천 배제.

### 3.6 The "Futures Market Fake Open" (선물장 개장 페이크) - [New V18.9]
-   **Scenario**: 08:45분 선물/옵션 시장 개장 시 키움 API가 장운영구분 코드 3(장중)을 송신하여, 시스템이 이를 정규장 개장으로 오인해 장중 필터를 조기 적용하는 현상.
-   **Guard**: **Time-based Market Phase Guard**.
-   **Logic**:
    - `StateManager.update_phase` 내에서 현재 시간이 09:00 이전일 경우, API의 '장중' 신호를 무시하고 `PRE_OPEN` 상태를 강제 유지.
    - 본장 개장 정시(09:00)가 되어야만 `DURING_MARKET`으로의 전환을 허용하여 수급 필터의 조기 오작동 방지.

### 3.7 The "Aggregated State Desync" (집계 상태 동기화 오류) - [New v16.0]
-   **Scenario**: `ForensicDashboard` 등에서 데이터 모델 리셋 시 이전의 집계 인덱스(`last_event_row`)가 남아있어 `IndexError`를 유발하거나 엉뚱한 데이터를 참조하는 현상.
-   **Guard**: **Mandatory State Reset on Model Update**.
-   **Logic**: 데이터 수신(`set_data`) 또는 초기화 시 해당 위젯의 커스텀 집계 상태 변수들을 즉각 `None` 또는 `0`으로 리셋하는 의존성 주입 코드를 강제함.

### 3.8 The "Ranking Damping Leak" (랭킹 감쇄 로직 누수) - [New V23.3]
-   **Scenario**: 하락 종목의 점수를 깎기 위해 `damping` 계수를 계산했음에도 불구하고, 실제 최종 점수 합산 과정에서 이를 곱해주지 않아 하락 투매주가 랭킹 상위에 노출되는 현상.
-   **Guard**: **Explicit Multiplicative Damping**.
-   **Logic**: `val_norm`, `mom_norm`, `acc_norm` 등 각 점수 컴포넌트 생성 시 `float(damping)`을 명시적으로 곱하여 하락률에 따른 페널티가 수학적으로 누락 없이 반영되도록 강제함.

### 3.9 The "Negative Acceleration Paradox" (음의 가속도 역설) - [New V23.3]
-   **Scenario**: 급락하는 종목의 가속도 절대값(`abs(accel)`)을 사용하여 가속지수를 산출할 경우, 오히려 '더 빨리 떨어지는 종목'이 높은 가속 점수를 받아 HOT2 리스트에 표시되는 현상.
-   **Guard**: **Sign-Aware Acceleration Filter**.
-   **Logic**: 가속지수 산출 시 반드시 `accel > 0` (양의 가속) 및 `chg >= 0` (상승 중) 조건을 필수 게이트로 설정하여 하락 가속 종목을 원천 차단함.

---

## 4. Execution Risk Guards [Updated V17.0]

### 4.1 The "Stop Loss" (강제 손절) - [Updated V18.11]
-   **Scenario**: 진입 후 가격이 예상과 반대로 급하게 움직임.
-   **Guard**: `ExecutionManager.on_timer` 내 실시간 수익률 체크 (Backend SSoT).
-   **Rule**: `if pnl_pct < -config_data['stop_loss']: engine.send_order(...)`.
-   **Action**: 
    - **Emergency Interrupt**: 손절 발생 시 기존의 모든 미체결 주문(`pending_orders`)을 즉시 취소하여 주문 충돌 방지.
    - **State Sync**: 앱 재시작 시 `sync_position`을 통해 평단가 및 고점(`peak_price`)을 복원하여 연속성 있는 트레일링 스탑 보장.
-   **SSoT**: ⚠️ UI에서의 손절 판단은 금지하며, 반드시 `ExecutionManager` 백엔드에서만 실행.

### 4.2 The "Kill Switch" (운용 정지)
-   **Scenario**: 당일 누적 손실이 예수금 대비 일정 비율을 초과함.
-   **Guard**: `ExecutionManager` 내 Kill Switch 로직 (Backend SSoT).
-   **Rule**: 당일 손실이 설정된 `%`를 초과하면 모든 신규 진입 주문을 엔진 레벨에서 차단.
-   **SSoT**: ⚠️ UI에서의 Kill Switch 판단은 금지 (헌장 §1.3 경계 존중).

### 4.3 The "Forensic Pipeline" (감사 추적 파이프라인) - [Updated V18.45]
-   **Scenario**: 매매 이벤트 정보 누락 또는 무분별한 중복 생성으로 인한 감사 가독성 저해.
-   **Guard**: **Engine-Driven EventBus → PersistenceManager → ForensicDashboard**.
-   **Logic**:
    - **Single-Source Ownership**: `ForensicBridge` 시그널 중복 연결 제거로 데이터 이중 생성 원천 봉쇄.
    - **Smart Deduplication**: `_seen_log_ids` (GTID Set) 필터를 통한 중복 로그 입구 컷.
    - **Payload Filter**: `PersistenceEvent.payload` 구조를 정확히 파싱하여 `EXECUTION_FILL` 등 고빈도 노이즈 차단.
-   **SSoT**: UI는 이벤트 발행 권한을 갖지 않으며, Engine만이 유일한 포렌식 이벤트 발신자.

### 4.4 The "Unauthorized Database Mutation" (비승인 DB 변형) - [New V17.2.8.5]
-   **Scenario**: AI가 시스템 성능 개선이나 정교화를 목적으로 운영 중인 프로덕션 DB(`trades.db`)에 대규모 `UPDATE` 또는 `DELETE` 스크립트를 직접 실행하여 데이터 무결성을 파괴함.
-   **Guard**: **Design Note Validation & Mandatory Backup-First Policy**.
-   **Rule**: 
    - 1,000건 이상의 레코드 변형이 동반되는 작업은 반드시 **L4 등급**으로 분류.
    - 실행 전 반드시 **물리적 DB 백업**(`copy-item`)이 선행되어야 함.
    - `UPDATE` 쿼리 대신 시스템 환경 설정(`to_local_naive`)을 통한 **Ingress Hardening**을 우선 권장 (비파괴적 해결).
-   **Action**: 위반 시 즉시 백업본으로 롤백 및 거버넌스 복구 절차 수행.

### 4.6 The "Overnight Gap-down Risk" (오버나이트 갭하락 리스크) - [New V18.15]
-   **Scenario**: 전일 보유한 포지션이 당일 약세로 마감하여 익일 시가에 큰 폭의 갭하락 손실이 확정되는 현상.
-   **Guard**: **HedgeGate (15:10 Automated Risk Exit)**.
-   **Logic**: 15:10 기준 `pnl_pct < -2.0%`인 약세 종목을 익일로 넘기지 않고 당일 즉시 전량 청산하여 기회비용 및 갭하락 위험 차단.

### 4.7 The "Late Session Entry Noise" (장 막판 진입 노이즈) - [New V18.15]
-   **Scenario**: 15:00 이후 세력의 물량 정리나 심리적 요동으로 발생하는 가짜 매수 신호에 속아 진입하여 익일 물리게 되는 현상.
-   **Guard**: **TimeGate (15:00 Smart Cutoff)**.
-   **Logic**: 15:00 이후 발생하는 모든 신규 `BUY` 신호를 엔진 레벨에서 원천 차단하여 오버나이트 진입 리스크 관리.

### 4.8 The "Morning Panic Sell-off" (시가 갭하락 패닉 셀) - [New V18.15]
-   **Scenario**: 시가 갭하락 시 기계적인 손절 집행으로 인해 V자 반등의 기회를 놓치는 현상.
-   **Guard**: **GapGuard (Conditional 30s Grace Period)**.
-   **Logic**: 갭이 -5% 이내일 경우 30초 유예.

### 4.9 The "Early Entry Fatality" (진입 초기 치명상) - [New V20.9.1]
-   **Scenario**: 진입 직후 예상과 완전히 반대로 움직여 웜업 기간(2봉) 가드 내에서 심각한 손실이 발생하는 현상.
-   **Guard**: **Early Cut (High-Speed Hypothesis Termination)**.
-   **Logic**: 보유 시간이 2봉 이하라도 손익비가 **-0.5R** 에 도달하면 가드를 무시하고 즉시 매도하여 자본을 보호함.

### 4.10 The "Fee-Eating Time Extension" (기회비용 낭비성 시간 연장) - [New V20.9.1]
-   **Scenario**: 수익은 미미한데 가격이 MA20 위에 있다는 이유로 시간 청산이 계속 연장되어 다른 주도주에 진입할 자본이 묶이는 현상.
-   **Guard**: **Smart Time Extension (Min-R Qualifier)**.
-   **Logic**: 수익권(PnL > 0)일지라도 현재 수익이 **0.5R** 미만인 경우 시간 연장 혜택을 부여하지 않고 냉정히 청산함.

### 4.11 The "Winning Trade PnL Vaporization" (수익 거래의 본전 이하 회귀) - [New V20.9.1]
-   **Scenario**: 2R 이상의 큰 수익이 났음에도 불구하고 익절선을 올리지 않아 급락 시 수익을 모두 반납하고 손실로 전환되는 허탈한 현상.
-   **Guard**: **BE Switch (Free-Ride Zone Transition)**.
-   **Logic**: 수익이 **2.2R** 에 도달하면 손절선을 즉시 `진입가 + 0.2R`로 상향 배정하여 해당 포지션의 '무손실'을 시스템적으로 보장함.

### 4.12 The "Metadata Race Condition" (이벤트간 메타데이터 유실) - [New V19.6]
-   **Scenario**: 키움 OpenAPI의 비동기적 특성으로 인해 주문 확인(`sig_order_confirmed`)과 체결(`sig_order_fill`) 이벤트가 순차적으로 발생할 때, 체결 시점에 원본 `TradeSignal`의 컨텍스트(전략명, 국면 등) 정보가 소실되어 DB에 `UNDEFINED`로 기록되는 현상.
-   **Guard**: **Metadata Context Bridge (`pending_signal_meta`)**.
-   **Logic**:
    - 주문 전송(`send_order`) 시점에 해당 종목의 메타데이터를 `pending_signal_meta` 딕셔너리에 임시 보관.
    - 체결 이벤트 수신 시 해당 딕셔너리를 조회하여 메타데이터를 복원한 뒤 `PersistenceManager`에 전달.
    - 사용 완료된 캐시는 즉시 삭제하여 메모리 누수 방지.

### 4.13 The "WAL Sync Invisibility" (WAL 동기화 지연) - [New V19.8.2]
-   **Scenario**: SQLite의 WAL 모드로 인해 최신 거래 데이터가 `-wal` 보조 파일에만 머물러 있어, 웹 대시보드 서버가 메인 `.db` 파일만 서빙할 경우 당일 데이터가 0으로 표시되는 현상.
-   **Guard**: **On-demand WAL Checkpointing**.
-   **Logic**:
    - 대시보드 서버(`strategy_dashboard.py`)가 DB 파일을 전송하기 직전에 `PRAGMA wal_checkpoint(TRUNCATE)` 명령을 실행하여 WAL 데이터를 메인 파일로 강제 병합.
    - HTTP 응답 헤더에 `Cache-Control: no-cache`를 설정하여 브라우저의 정적 파일 캐싱 차단.

### 4.14 The "Non-Reentrant Lock Self-Deadlock" (비재진입성 락 자체 데드락) - [New V26.1]
-   **Scenario**: `StateManager`에서 `get_active_surveillance_set()` 호출 시 `self.lock`을 획득(Acquire)한 컨텍스트 내에서, 내부적으로 락을 또다시 획득하려는 `self.surveillance_set` 프로퍼티 게터에 접근하여 자가 교착 상태(Self-Deadlock)에 빠져 프로그램이 정지되는 현상.
-   **Guard**: **Direct Private Variable Access under Locked Context**.
-   **Logic**: 락을 획득한 상태의 내부 메서드에서는 외부 게터 프로퍼티 대신 프라이빗 멤버 변수인 `self._surveillance_set`에 직접 접근하여 이중 락 획득 시도를 차단.

### 4.15 The "GUI Main Thread Stutter on High Logging" (로그 폭주시 GUI 프리징) - [New V26.2]
-   **Scenario**: 실거래 중 실시간 로그 수천 건이 대시보드로 직접 유입 및 UI 렌더링되면서 메인 스레드가 블로킹되어 "응답 없음"이 발생하는 현상.
-   **Guard**: **Circular Memory Buffer (Flight Recorder) & Offline Analysis**.
-   **Logic**: 실시간 GUI 로그 렌더링을 억제하고 메모리 내 순환 버퍼(`deque(maxlen=2000)`)에 저장하며, 비정상 종료나 Coordinated Shutdown 시점에 `flight_recorder_dump.json` 파일에 일괄 디스크 덤프하여 오프라인으로 사후 분석하도록 우회.

### 4.16 The "Forensic Dashboard Summary Metrics Scale Distortion" (요약 카드 승인/차단율 왜곡) - [New V26.6]
-   **Scenario**: `TOTAL SIGNALS` 카드 분모는 전체 DB 로그 수(수십만 건)인 반면, 승인/차단 분자는 최근 로드된 슬라이딩 윈도우(2000건)를 기준으로 세어 비율이 0에 수렴하는 등 왜곡되는 현상.
-   **Guard**: **Sliding-Window Uniform Scaled Recalculation**.
-   **Logic**: `recalculate_summary_metrics()` 메서드를 신설하여, 분모와 분자를 모두 현재 화면에 로드된 슬라이딩 윈도우 데이터셋(`self.model._data`) 기준으로 일치시켜 동적 집계.

### 4.17 The "Tooltip Stylesheet Cascade Pollution" (스타일시트 캐스케이드 오염) - [New V26.7]
-   **Scenario**: `QLabel` 등 개별 위젯에 스타일을 타입 셀렉터 없이 날것(`setStyleSheet("background-color: ...")`)으로 지정하면 스타일시트가 하위 `QToolTip` 위젯으로 전파되어 툴팁 글씨 판독이 불가능해지는 현상.
-   **Guard**: **Explicit Type Selector Isolation**.
-   **Logic**: `setStyleSheet` 호출 시 `QLabel { background-color: ... }`와 같이 타입 셀렉터를 명시하여 스타일시트 상속 오염을 차단.

### 4.18 The "Wall-Clock Leakage in Replay/Backtest" (시간 흐름 비결정성 누출) - [New V26.6]
-   **Scenario**: 백테스팅 및 리플레이 모드 구동 중 `time.time()`, `datetime.now()` 등 원시 시스템 시계를 호출하여 시뮬레이션 일관성과 결정성(Determinism)이 파괴되는 현상.
-   **Guard**: **Deterministic AST Lint & Virtual TradingClock**.
-   **Logic**: AST 분석기(`tests/test_determinism_lint.py`)를 통해 핵심 엔진 도메인 내 시간 관련 호출을 전수 검출하고, 가상 `TradingClock` 사용을 의무화하거나 허용된 로직에 한해 `# noqa: determinism` 주석 명시.

### 4.19 The "Dirty Shutdown State Loss" (비정상 종료 시 데이터 유실) - [New V26.2.2]
-   **Scenario**: 닫기 버튼(`closeEvent`) 클릭 시 스레드 종료 대기 엉킴, DB 커밋 누락, 락 미해제로 인해 상태 오염 및 메모리 누수가 발생하는 현상.
-   **Guard**: **EngineLifecycleManager Coordinated Shutdown**.
-   **Logic**: `EngineLifecycleManager` 오케스트레이터를 통합 연동하여 수동 매도 누적기 플러시, 감시 타이머 정지, DB 영속화 커밋, 락 해제, 소켓 닫기를 선형적 순서로 안전하게 통제 및 실행.

### 4.20 The "NaTType strftime crash" (NaTType 문자열 포맷 크래시) - [New V27.2]
-   **Scenario**: 키움 OpenAPI에서 주봉/월봉 조회 시 무효 날짜 데이터(예: `00000000`)가 유입되어 Pandas가 `NaT`로 변환하고, 이후 X축 라벨 포맷팅 중 `NaTType does not support strftime` ValueError 발생하여 크래시.
-   **Guard**: **Pre-filtering & Null Check Guard**.
-   **Logic**:
    - `engine.py`에서 무효한 날짜 문자열("0", "0000", 비어있음 등)을 감지하여 사전에 스킵합니다.
    - `chart_widget.py` 내 `_convert_to_dto`에서 `pd.isnull()` 검사를 적용해 `NaT`인 경우 안전하게 공백(`""`) 처리 및 타임프레임별 동적 포맷팅(일/주봉 `"%m/%d"`, 월봉 `"%Y/%m"`) 적용.

### 4.21 The "Engine-GUI Initialization Race" (_tr_queue AttributeError) - [New V27.2]
-   **Scenario**: 프로그램 재기동 시 마지막 차트 구분을 복원(`apply_initial_defaults`)하는 흐름이 QComboBox 이벤트를 조기 트리거하여, 메인 윈도우 생성자 내에서 `self._tr_queue` 버퍼가 인스턴스화되기도 전에 `request_minute_data_throttled`가 호출되어 AttributeError 유발.
-   **Guard**: **Eager Initialization & Lazy Initialization Guard**.
-   **Logic**:
    - 생성자(`__init__`) 내부에서 UI 조립 및 설정 로드 이전 시점인 최상단부로 `self._tr_queue = []` 초기화 순서를 앞당겨 배치.
    - `request_minute_data_throttled` 내에 `hasattr(self, '_tr_queue')`를 검증하여 미생성 시 즉시 동적 리스트 인스턴스화를 수행하는 Lazy 가드 탑재.

### 4.22 The "GetCommDataEx Column Shift" (컬럼 인덱스 매핑 밀림) - [New V27.3]
-   **Scenario**: `opt10081`(일봉)과 달리 `opt10082`(주봉)/`opt10083`(월봉)은 첫 컬럼에 종목코드가 없고 바로 종가가 시작되어, 인덱스를 공용으로 사용 시 모든 OHLCV 수치가 밀려 종가에 거래량, 거래량에 거래대금이 매핑되어 캔들 스케일이 극도로 왜곡되는 현상.
-   **Guard**: **Regime-Aware Column Layout Fork**.
-   **Logic**:
    - `_parse_chart_data` 내 `is_macro` 조건문 하단에서 일봉(`rq_name == "req_chart_day"`)과 주/월봉 케이스를 분리하여, 주/월봉의 경우 컬럼 인덱스를 1개 열씩 좌측으로 시프트한 보정 인덱스 맵을 사용하도록 강제.

### 4.23 The "Multi-Chart Control Coupling" (멀티차트 제어 결합 간섭) - [New V27.4]
-   **Scenario**: 메인 차트와 별개로 거시적인 차트를 참고하기 위해 서브 윈도우를 띄울 때, 서브 윈도우의 타임프레임 변경이나 조회 요청이 메인 루프(의사결정허브, strategy)의 지표 정보나 타임프레임을 오염시켜 주문 오발동을 유발하는 현상.
-   **Guard**: **Pipeline Isolation & One-Way Event Binding**.
-   **Logic**:
    - `multi_req_chart` 접두사 기반의 격리된 TR 요청 코드와 `sig_multi_chart_data` 시그널을 별도로 정의하여 의사결정 파이프라인에서 완전히 제외.
    - 메인 윈도우의 종목 변경 이벤트(`on_code_changed`)가 발생할 때만 서브 윈도우로 종목 코드를 복사 주입하는 단방향 동기화 처리.

### 4.24 The "P&L Layout Bloat Null-Crash" (P&L 위젯 제거 시 Null 참조 오류) - [New V27.5]
-   **Scenario**: 하단의 "실시간 운용 현황(P&L)" 영역을 사용하지 않으려 레이아웃에서 제거했을 때, 메인의 토글 이벤트 등에서 해당 컴포넌트들을 직접 참조하여 Null/AttributeError가 발생하는 현상.
-   **Guard**: **L1 Layout Isolation & hasattr Defensive Guards**.
-   **Logic**:
    - `chart_panel_builder.py`에서 패널 및 토글 바 컴포넌트 생성을 완전히 제거하고, `main_window.py`의 `toggle_performance_panel` 등 관련 제어 메서드 시작 시점에 `hasattr` 검사 가드를 일괄 적용해 미조립 상태 시 동작 없이 즉각 복귀(Early Exit)하도록 구현.

### 4.25 The "AI Insights Overwrite by Realtime Tick" (실시간 틱 갱신에 의한 AI 인사이트 강제 덮어쓰기) - [New V27.7]
-   **Scenario**: "AI종합 분석 (심층 인사이트)" 버튼 클릭 후 GPT 분석 텍스트가 노출되어 있는 동안, 실시간 수급 틱이 유입되거나 300ms 주기 렌더링 타이머가 작동하여 `update_status`가 호출되면서 GPT 분석 결과를 강제로 지우고 원래의 체크리스트(`insights_html`)로 덮어쓰는 현상.
-   **Guard**: **View Lock Flag & Dual-Mode Toggle Button**.
-   **Logic**:
    - `SignalStatusWidget`에 `is_ai_showing` 상태 플래그를 두어, 이 값이 `True`일 때는 `update_status` 하단부의 `txt_ai.setHtml(insights_html)` 호출(화면 갱신)을 명시적으로 스킵(Guard Bypass).
    - 분석 성공 시 버튼 텍스트를 `"↩️ 돌아가기 (AI 분석 종료)"`로 변경하여, 사용자가 다 읽고 돌아가기 버튼을 클릭했을 때만 플래그를 해제하고 원래의 틱 수급 체크리스트로 되돌아가는 양방향 뷰 스위칭 구조 구현.

### 4.26 The "Document-Width-Induced Text Clipping" (문서 너비 설정 누락 및 상태 전이 시 갱신 누락으로 인한 텍스트 조기 줄바꿈) - [Updated V27.10]
-   **Scenario**: `txt_ai` QTextBrowser 내의 텍스트가 위젯 실제 가로 크기보다 훨씬 좁은 너비에서 조기 개행(10자 내외)되어 가로 빈 공간을 활용하지 못하고 가독성이 극도로 악화되는 현상. 특히 AI 결과 수신, 에러 발생, 분석 로딩 등의 상태 전이 시점에 래핑 폭 계산이 연동되지 않아 조기 개행 상태가 유지되는 문제 발생. 또한, 텍스트가 너무 길 때 높이가 220px까지 동적으로 팽창하여 고정된 660px 컨테이너 한도를 초과해 전체 대시보드 레이아웃을 찌그러트리고 붕괴시키는 부작용 발생.
-   **Guard**: **Dynamic Text-Width Scaling, Resize-Bound Layout Re-calculation, State-Bound Recalculation & Vertical Height Clamping**.
-   **Logic**:
    - `_adjust_txt_ai_height` 내에서 텍스트 도큐먼트 `adjustSize()` 호출 전, `doc.setTextWidth(self.txt_ai.width() - 20)`를 실행하여 위젯 가로 폭 전체를 온전히 활용하도록 래핑(Wrap) 기준선을 확장.
    - `SignalStatusWidget.resizeEvent` 핸들러를 정의하여 창 가로 크기 조절 시 비동기로 레이아웃 계산을 즉시 수행해 반응형 너비 동기화를 완성.
    - `on_ai_result`, `on_ai_error`, `request_ai_analysis` 등 AI 분석 텍스트 내용이 변경되는 모든 생명주기 상태 전이 지점에서 `QTimer.singleShot(0, self._adjust_txt_ai_height)`을 호출하여 즉각적으로 래핑 폭이 재조정되도록 강제.
    - `txt_ai`가 세로로 130px을 초과하여 팽창하지 않도록 최대 높이(`self._base_height`)로 상한을 고정하며, 초과 텍스트는 얇고 세련된 다크 테마 커스텀 스크롤바(`width: 6px`)를 통해 내비게이션 가능하도록 처리하여 레이아웃 안정성을 확보.

---

## 5. Final Doctrine
"**모든 코드는 실패할 수 있음을 전제하며, 각 단계마다 물리적/논리적 이중 방어막을 유지한다.**"  
보호 로직은 결코 타협의 대상이 아니며, 하드웨어나 네트워크의 한계조차 논리적 가드(`assert`)로 극복하는 것을 지향합니다.

### 5.1 Defense Integrity
본 문서의 모든 가드는 헌장 1.2조에 의거하여 **문서-코드-로그**의 삼중 체계로 강제됩니다. 특히 스탑로스(Stop Loss), 킬스위치(Kill Switch), 실현손익 동기화(PnL Sync) 및 **DB 무결성 가드**는 시스템의 생존과 직결된 L3/L4 핵심 로직으로, 정규 승인 없이는 어떠한 변경도 불가능합니다.

---
*Failure Patterns Version: v27.10 (AI Text Content Transition, Responsive Wrapping & Height Guard Patch)*

