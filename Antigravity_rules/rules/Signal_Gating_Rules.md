# Signal Gating Rules (신호 제어 규칙) [V27.0]

## 1. 목적 (Purpose)
본 문서는 잘못된 신호가 실행 단계로 넘어가지 못하도록 막는 **방어벽(Gating)** 규칙을 정의합니다. 본 문서는 [AI 작업 헌장 v1.8](file:///C:/Users/stone/projects/nocodequant/Antigravity_rules/governance/AI_Work_Constitution_and_Reference_Map.md)의 **L3(로직/수행)** 등급에 해당하며, HOTFIX 수정을 엄격히 금지합니다.

---

## 2. 결정 레벨 게이트 (Decision Level Gates) [V18.5]

### 2.0 아키텍처: 2계층 소프트 진입 (2-Layer Soft Entry)
-   **계층 1 (강력한 안전장치)**: 자본 보호를 위한 최소 게이트 (3건).
-   **계층 2 (소프트 진입 점수)**: 4블록 가중합. `entry_score >= ENTRY_THRESHOLD` 시 매수.
-   **기존 AND 6중 구조 폐지** (V18.0): 신뢰도(confidence) 이중 패널티 및 기회비용 과다 해소.

### 2.1 웜업 게이트 (Warmup Gate, 수집 대기) — 계층 1
-   **조건 (Condition)**: 현재 종목의 누적 캔들 수 < `ACTIVE_CONFIG["WARMUP_MIN_BARS"]` (기본 50).
-   **조치 (Action)**: `관망 강제 (Force HOLD)` (사유: "데이터 수집 중").
-   **사유 (Reason)**: 지표(MA, RSI 등)의 안정적인 계산을 위한 최소한의 데이터 확보 기간.

### 2.2 극한 거래량 게이트 (Extreme Volume Gate) — 계층 1
-   **조건 (Condition)**: `vol_ratio > ACTIVE_CONFIG["VOL_RATIO_CAP"]` (기본 2.2).
-   **조치 (Action)**: `관망 강제 (Force HOLD)` (사유: "과열 차단").
-   **사유 (Reason)**: 극단적 거래량 과열은 피크아웃 위험이 높으며, 지표 왜곡을 유발함.

### 2.3 소프트 진입 점수 (Soft Entry Score) — 계층 2
-   **구성**: 4블록 가중합 (총 100점)
-   **핵심 기술 지표 (Core Technical)** (40%): MA + RSI + BB의 가중 결합 점수 (`raw_score`).
-   **거래량 품질 (Volume Quality)** (20%): `vol_ratio` 구간별 품질 점수 (0.7x ~ 1.8x 최적).
-   **가격 트리거 (Price Trigger)** (20%): 전봉 고점 돌파 및 캔들 형태별 점수.
-   **리스크 블록 (Risk Block)** (20%): 신호 안정성 + 시장 국면 + 장기 이평선(240선) 위치 반영.
-   **매수 (BUY)**: `entry_score >= ENTRY_THRESHOLD` (UPTREND/TREND=50, RANGE=58, DOWNTREND=60, CHAOS=65). **[V21.9.3 탐색 모드 적용]**
-   **🚀 VIP 하이패스 (Combo Booster) [V21.9]**: `TREND` + `VOLUME` 콤보 포착 시, 진입 임계값을 추가로 **-10점 완화**하여 주도주 편입 기회를 극대화함.
-   **하락장 감점 (Downtrend Penalty) [V18.45]**: `is_downtrend` (MA20 < MA60, 0.3% Hysteresis 및 MA20_slope < 0) 감지 시 리스크 블록에서 **-20점 즉각 부여**.
-   **사유 (Reason)**: 단일 지표의 우연한 일치를 배제하고, 여러 차원의 근거가 확보된 시점에 진입하여 승률을 극대화함.

### 2.4 영업 종료 임박 게이트 (TimeGate) [V18.15] — 계층 1
-   **조건 (Condition)**: 현재 시간 >= `ACTIVE_CONFIG["LATE_ENTRY_CUTOFF_TIME"]` (기본 15:00).
-   **조치 (Action)**: 모든 신규 `BUY` 신호 차단 (Force BLOCK).
-   **사유 (Reason)**: 장 마감 직전은 정보 비대칭 및 변동성이 극심하여 방향성 예측이 불가능하며, 오버나이트 리스크가 가중되는 구간임.

### 2.5 진입 게이트 역선택 보정 (Anti-Adverse-Selection Calibration) [V28.3]
> 근거: 진입게이트 비교분석 (2026-06-11) — 시간·거래량 조건부 음성 필터의 교집합이 "저점 신호는 차단하고 클라이맥스 꼭지 봉만 통과"시키는 역선택을 유발함이 확인됨.

-   **[P1] Trend-Only Mode 단계적 해제**: `ENTRY_TH_BREAKOUT 100→55`, `ENTRY_TH_MIXED 100→65`. VOLUME/REVERSAL은 잠금(100) 유지, 섀도 측정 후 결정.
-   **[P2] 아침 동적 수급 가드 분모 수정**: 당일 봉 1~2개 구간(09:05~09:15)은 분모가 시초봉(당일 최대 거래량)이 되어 클라이맥스 봉만 통과시키던 결함 → 절사평균이 성립하는 당일 봉 3개 이상 구간에서만 평가.
-   **[P3] Bar-Maturity 눌림 구조 면제**: 진행봉이 음봉/보합이거나 세션 고점 대비 `ENTRY_PULLBACK_RETRACE_PCT`(1%) 이상 되돌림이면 240s 대기 면제 — "추격은 막고 눌림은 잡는" 비대칭 설계.
-   **[P4] 저점 위치 양성 가점 (Low-Location Bonus)**: 세션 VWAP ±1% 이내 또는 세션 고점 -2% 이상 되돌림 시 `entry_th -5pt` — 음성 필터 일변도 구조에 유일한 '좋은 위치' 직접 가점.
-   **[P5] 하드 가드 섀도 통일**: `ENTRY_MIN_VALUE_5M`·`MORNING_VOLUME_GUARD`가 `GUARD_SHADOW_MODE`를 준수하도록 통일 — 측정(가상 손익) 후 차단 여부를 데이터로 결정.

---

## 3. 청산 거버넌스 (Exit Governance) [V20.9.1]

매수보다 중요한 매도를 관리하기 위해 시스템적으로 강제되는 청산 레이어입니다. **R-Multiple(리스크 배수)** 기반의 비대칭 구조를 채택합니다.

### 3.1 비대칭 익절 및 방어 (Asymmetric Rules)
-   **무위험 지대 (BE Switch)**: 수익이 **2.2R** 도달 시 손절선을 `Entry + 0.2R`로 전진 배치하여 원금 보호.
-   **가변 트레일링 스탑**: 수익 구간(1.5R~5.2R+)에 따라 ATR 배수를 **4.5x에서 1.5x까지 단계적 타이트닝**하여 수익 잠금.
-   **전략별 TP1 차등**:
    -   `TREND/BREAKOUT/VOLUME` (추세계): TP1 완전 폐지 (수익 극대화).
    -   `REVERSAL/RANGE` (역추세계): **2.0R** 도달 시 50% 분할 익절.

### 3.2 강력한 자본 보호 (Stop Loss & Early Cut)
-   **Early Cut [V20.9.1]**: 진입 후 2봉 이내라도 손실이 **-0.5R** 에 달하면 즉시 절단 (진입 가설 즉시 폐기).
-   **Panic Shield**: 장기 이평선(MA240) 하회 시 트레일링 배수를 **1.0x**로 조여 리스크 최소화.
-   **Doomsday SL**: 시스템이 설정한 절대 손절선(기본 4%) 이탈 시 가드 무시 즉시 청산.

### 3.3 시가 갭 보호 규칙 (GapGuard) [V18.15]
-   **조건 (Condition)**: 장 초반(09:00:00 ~ 09:00:30) 시가 갭하락 발생 시.
-   **패닉 셀 차단**: 갭이 `-5%` 이내일 경우, 30초간 손절을 유예(Grace Period)한다.
-   **즉각 대응**: 갭이 `-5%`를 초과하여 급락할 경우, 유예 없이 즉시 `Hard Stop`을 집행한다.

### 3.4 오버나이트 리스크 관리 (HedgeGate) [V18.15]
-   **조건 (Condition)**: 현재 시간 >= `15:10` 및 종목별 수익률 < `-2.0%`.
-   **조치 (Action)**: 해당 종목 강제 전량 청산 (OVERNIGHT_RISK_EXIT).
-   **Fake Open Guard [V14.0]**: 08:45~09:00 사이의 선물 시장 가격 급변으로 인한 가짜 장동 신호를 차단하고 `PRE_OPEN` 상태를 강제 고수.

---

## 4. 데이터 무결성 게이트 (Data Integrity Gates)

### 4.1 이상치 차단 게이트 (Clipping Gate)
-   **조건 (Condition)**: 지표 원본(Raw) 값이 최근 표준편차의 `±10σ`를 초과.
-   **조치 (Action)**: `소프트 클립 (Soft Clip)` (Raw 값을 ±10σ 수준으로 정규화).
-   **사유 (Reason)**: 극단적 스파이크(Spike) 데이터로 인한 지표 왜곡 및 전략 엔진의 수치적 불안정성 방지.

### 4.2 글로벌 리셋 게이트 (Global Reset Gate, 타임프레임 스위칭)
-   **조건 (Condition)**: 타임프레임이 변경된 직후의 첫 번째 캔들.
-   **조치 (Action)**: `신호 무시 (Ignore Signal)` (플래그: `ignore_next_signal`).
-   **사유 (Reason)**: 이전 타임프레임의 잔상 데이터가 지표 계산에 영향을 미치는 것을 방지하고, 깨끗한 환경에서 첫 데이터를 수용.

### 4.3 실행 계층 게이트 (Execution Layer Gate, 스탬프 로직)
-   **조건 (Condition)**: 현재 신호의 타임스탬프(`signal_ts`)가 해당 종목/방향으로 이미 처리된 타임스탬프(`handled_ts`)와 같거나 이전인 경우.
-   **조치 (Action)**: `주문 실행 건너뛰기 (Skip Order Execution)`.
-   **로직 (Logic)**: `last_handled_signals[(symbol, action)] = signal_ts`.
-   **사유 (Reason)**: 동일 캔들 내에서 발생하는 미세한 티크(Tick) 가격 변화로 인한 중복 주문(Spam)을 물리적으로 차단.

---

## 5. 시장 열기 게이트 (Market Heat Gating)

개별 종목의 모멘텀뿐만 아니라, 시장 전체의 수급 질을 검정하여 진입 유니버스를 제한합니다.

### 5.1 기관급 소음 제거 필터 (Institutional Filter)
- **조건 (Condition)**: 종목명에 파생/상품군 키워드(`KODEX`, `TIGER`, `선물`, `인버스`, `레버리지`, `ETN` 등) 포함.
- **조치 (Action)**: `영구 제외 (Hard Exclusion)` (지수 산출 및 스캐닝 대상에서 원천 제외).
- **사유 (Reason)**: 개별 주식의 실질적 수급 탄력성을 측정하기 위해 지수화된 상품군 노이즈를 제거.

### 5.2 하락 투매 차단 필터 (Crash-dump Volume Spike Guard) [V23.3]
- **조건 1 (Hard)**: 당일 등락률 < `-3.0%`.
- **조치 (Action)**: `영구 제외 (Hard Exclusion)` (실시간 랭킹 및 감시 대상에서 즉각 퇴출).
- **조건 2 (Penalty)**: `-3.0% <= 등락률 < 0%`.
- **조치 (Action)**: `지수적 가중치 감쇄 (Exponential Damping)`. `exp(chg / 2.0)` 공식을 적용하여 미세 하락 시에도 모멘텀 점수를 엄격히 삭감.
- **사유 (Reason)**: 하락 투매로 인해 일시적으로 발생하는 거래량/거래대금 폭발이 '추세적 수급'으로 오인되어 랭킹 상위에 노출되는 위험을 원천 차단.

### 5.3 실시간 랭킹 엔진 V23.3 (Phase 1/2) [V23.3]
- **생존 검증 (Liveness Check)**: 실시간 체결 데이터(`RealtimeQuote`)의 타임스탬프를 감시하여, 최근 10분 이상 체결이 없는 '죽은 종목'(상한가 포함)의 점수를 단계적으로 0.05배까지 감쇄.
- **자체 모멘텀 엔진**: 키움 TR에 의존하지 않고 자체 스냅샷(deque)을 활용하여 최근 15분 가격 변화율 및 5분 거래량 급등세를 직접 산출.
- **하이브리드 블렌딩**: TR 기반 데이터와 자체 계산 지수를 **50:50**으로 혼합하여 데이터의 최신성과 신뢰도를 동시 확보.

### 5.4 동전주 필터 정책 (Penny Stock Policy) [V25.1]
- **조건 (Condition)**: `PENNY_FILTER_MODE` 옵션이 설정된 상태에서 종목 가격이 `PENNY_MIN_PRICE`(1000원) 미만인 경우.
- **조치 (Action)**:
    - `hard_exclude` 모드: 1000원 미만 종목을 랭킹 및 실시간 감시 유니버스에서 원천 배제. (실거래/운영 모드 기본)
    - `damped` 모드: 1000원 미만(`PENNY_DAMPING_RATIO=0.8`) 및 500원 미만(`PENNY_SEVERE_RATIO=0.6`)에 대해 점수를 감쇄.
- **사유 (Reason)**: 동전주 및 초소형 잡주 진입으로 인한 자본 잠식 위험과 슬리피지 리스크를 차단.

### 5.5 다중 포착 노이즈 제어 (Cross Boost Control) [V25.1]
- **조건 (Condition)**: 종목이 여러 조건식(VALUE, GAIN, VOLUME)에 동시 포착되어 다중 가산점(Cross Boost)이 발생하는 경우.
- **조치 (Action)**: 선형 증폭(최대 1.5x) 방식을 `log1p` 기반의 비선형 완화 곡선으로 교체하고 부스트 비율 절대 상한을 `1.25x` (`CROSS_BOOST_MAX`)로 제한.
- **사유 (Reason)**: 장 초반 동시다발적 조건 포착으로 인한 테마 급등 잡주의 점수 과열 및 지표 왜곡 현상을 방지.

### 5.6 실시간 주도주 출처 추적 (Theme Flow Attribution) [V25.1]
- **조건 (Condition)**: 실시간 랭킹 탭에 노출된 종목의 이동 흐름을 추적 및 로깅할 시.
- **조치 (Action)**: 최초 발굴 탭(HOT, PRE_HEAT, REVERSE)과 현재 소속 탭을 분리 기록하는 `Attribution` 맵을 주문 메타데이터에 이식하고, 1분 주기로 흐름을 `logs/ranking_snapshots_YYYYMMDD.jsonl`에 스냅샷 백업.
- **사유 (Reason)**: 사후 포렌식 분석 및 백테스트 검증 시 종목 수급 유입 경로를 역추적 가능케 하여 정보의 투명성 확보.

---

## 6. UI 가시성 및 레이아웃 규칙 (UI Visibility & Layout Rules)

### 6.1 신호등 동기화 (Traffic Light Synchronization)
- **규칙 (Rule)**: `strategy.py`의 지표 점수(`IndicatorScore`) 상태는 UI(`SignalStatusWidget`)의 램프 색상과 100% 일치해야 한다.
- **색상 매핑 (Color Mapping)**: 
    - `saturated > 10`: **초록 (GREEN)** (상승/과매도 해소)
    - `saturated < -10`: **빨강 (RED)** (하락/과매수 해소)
    - `기타 (Else)`: **회색 (GRAY)** (보합/중립)

### 6.2 주도테마 및 보유잔고 독립 QDockWidget 분리 (Dock Decoupling) [V27.0]
- **규칙 (Rule)**: "서로 다른 라이프사이클과 제약을 지닌 위젯은 독립된 독 프레임으로 분리되어야 한다."
- **구현**:
    - 기존 single dock 내 vertical splitter 구조를 폐지하고, `dock_theme_flow`("🔥 실시간 주도테마 흐름")와 `dock_pos`("💼 보유 잔고 & 백그라운드 감시")를 각각 독립된 `QDockWidget`으로 격리.
    - `saveState`/`restoreState` 영속성을 보장하도록 각 독 위젯에 유일한 `objectName`("dock_ranking", "dock_theme_flow", "dock_pos") 부여.
    - 닫기 버튼 오작동 방지를 위해 `DockWidgetClosable` 속성을 제거하고 `Movable | Floatable` 속성만을 부여.
    - 초기 세로 비율은 `[230, 310, 260]`으로 분할 보정하여 주도주 가시성과 보유잔고 영역의 시인성을 동시 확보.

---

## 7. 모니터링 게이트 (Monitoring Gates) [ARCH-EDA-01]

리소스 효율화 및 고빈도 신호 포착을 위해 감시 강도를 동적으로 조절합니다.

### 7.1 실시간 격상 게이트 (Promotion Gate)
- **조건 (Condition)**: 백그라운드 스캔(Tier 2) 결과 `entry_score > 50`.
- **조치 (Action)**: 실시간 감시 리스트(Tier 1)로 300초간 격상 및 우선 순위 부여.
- **사유 (Reason)**: 잠재적 매수 기회가 포착된 종목에 컴퓨팅 자원을 집중하여 신호 발생 지연(Latency) 최소화.

### 7.2 리소스 해제 게이트 (Demotion Gate)
- **조건 (Condition)**: 격상 후 300초 경과 및 추가적인 고점수 갱신 부재.
- **조치 (Action)**: Tier 2로 강등 및 `DEMOTION` 이벤트 발행을 통한 리소스 해제.
- **사유 (Reason)**: 장기 관망 종목의 실시간 리소스를 회수하여 전체 시스템 부하 방지.

---

## 8. Forensic Log Governance (포렌식 로그 거버넌스) [V19.8.9c]

매매의 모든 과정을 투명하게 기록하고 감찰하기 위한 규정입니다.

### 8.1 중복 배출 금지 (Deduplication Gate)
- **규칙**: 동일한 `GTID`(또는 `event_id`)를 가진 로그는 단일 세션 내에서 포렌식 대시보드에 **1회만 노출**되어야 한다.
- **구현**: `_seen_log_ids` (SSoT Set)를 통한 입구 컷 필터링.

### 8.2 Performance Integrity [V19.8.9c]
-   **1-Order-1-Row**: 성과 분석 데이터(`trades.db`)는 매도 완료(`is_position_closed`) 시점에만 1회 기록하여 데이터 중복 및 Ghost Row를 원천 차단한다.
-   **Fill Idempotency [Updated V19.8.9c]**: 
    - `OrderManager.processed_fills` 캐시와 **복합 식별자(FID 912 + FID 902)** 조합을 통해 중복 체결 이벤트를 물리적으로 차단한다. 
    - 모의투자 환경의 식별자 중복 버그(FID 912 고정)에 대응하여 미체결 수량을 결합함으로써 실현손익 왜곡을 원천 방지한다.
-   **MME Heat Update (Real-time Sync)**: 
    - 리스크(Heat) 등록 시점을 주문 신호 발생 시가 아닌, **실제 체결 완료(`is_buy_complete`)** 시점으로 정규화한다. 
    - 체결 평단가와 최신 총자산을 기반으로 산출된 '유효 리스크'를 등록하여 자금 관리의 정밀도를 확보한다.
-   **Metadata SSoT & Deep-Search**: 
    - 매수 시 메타데이터를 보관 후 복원하되, 소실 시 `forensics.db`를 역추적(Deep-Search)한다. 
    - **Zombie Guard**: 최신 이벤트가 `SELL`인 경우 복구를 즉시 취소하여 과거 기록 오염을 방지한다.
-   **Holding Bars Integrity [V19.8.9c]**: 
    - (Current Index - Entry Index) 기반 계산을 원칙으로 하며, `calculate_holding_bars` 유틸리티를 사용한다. 
    - **Null Guard**: `entry_time` 부재 시 비교 연산을 차단하고 `0`을 반환하여 시스템 크래시를 방지한다.

### 8.3 Micro Market Indicators & Forensic Tooltip [V26.2]
-   **규칙**: 가짜 진입(False Positive)의 미시적 원인을 포착하기 위해 호가 스냅샷에서 수집 가능한 핵심 지표 7종(`spread`, `spread_tick`, `spread_pct`, `bid_total_qty`, `ask_total_qty`, `imbalance_ratio`, `proximity_score`)을 `SignalSnapshot`에 주입 및 SQLite 영속화한다.
-   **시각화**: 포렌식 대시보드 UI에 `METRIC_GLOSSARY` 설명 사전을 탑재하고 툴팁(Tooltip)을 연동하여, 분석 시 사용자가 마우스 오버만으로 각 지표의 의미를 파악하도록 직관성을 확보한다.

---

## 10. Score-Performance Adaptive Analytics (V5.0)

진입 시점의 지표 점수(Score)와 실제 매매 결과(PnL) 간의 상관관계를 통계적으로 분석하여, 시스템의 진입 임계값을 동적으로 튜닝하기 위한 거버넌스입니다.

### 10.1 분석 무결성 가드레일 (Analysis Integrity Guards)
- **SSoT Isolation**: 모든 분석용 SQL 쿼리는 `app/services/managers/schema_views.py`에 격리 정의하여 관리한다.
- **Ledger-based PnL**: 수익률 계산 시 오염된 `MAX(pnl)` 대신 `SUM(pnl_amount) / SUM(invested_amount)` 공식을 사용하여 부분 청산 시의 왜곡을 원천 차단한다.
- **Statistical Significance**: 성과 리포트 출력 시 유효 표본(`trade_count`)이 최소 **30건** 이상(또는 전체 모수의 2%)인 경우에만 신뢰할 수 있는 데이터로 간주한다.

### 10.2 분위수 기반 적응형 제어 (Quantile-based Adaptive Tuning)
- **Top-Tier Corridors**: `v_score_quantile` 뷰를 통해 상위 20%(1분위)의 승률이 하위 분위수보다 낮을 경우, 해당 전략의 해당 스코어 가중치를 즉시 재검토한다.
- **Adaptive Threshold**: 특정 전략의 평균 진입 점수가 수익률과 역상관관계를 보일 경우, POE(Parameter Optimization Engine)는 해당 전략의 `ENTRY_THRESHOLD`를 상향하여 보수적 진입을 유도한다.

---

## 11. 최종 원칙 (Final Doctrine)
"**모호한 백 번의 진입보다, 확실한 한 번의 기회를 잡는다.**"  
모든 제어(Gating) 규칙은 사용자의 자산 보호를 위해 보수적으로(Conservative) 작동하는 것을 원칙으로 합니다.

---
*Signal Gating Rules Version: v27.0 (Attribution & UI Decoupling Sync)*
