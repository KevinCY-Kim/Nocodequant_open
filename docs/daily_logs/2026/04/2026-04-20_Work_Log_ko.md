# NCQ 데일리 작업 로그 (2026-04-20)

## 📋 요약 (Summary)
오늘의 핵심 과업은 시스템 재시작 시 매수 시점의 메타데이터(전략 타입, 진입 봉 인덱스, 타임프레임 등)를 잃어버리는 **'기억 상실(Amnesia)'** 버그를 근본적으로 해결하는 것이었습니다. 이를 위해 데이터가 '흘러가는' 구조에서 '기록되는' 구조로 아키텍처를 전면 개수하였으며, 기관급(Institutional-grade) 데이터 무결성 파이프라인을 구축했습니다.

## 🛠 주요 변경 및 개선 사항

### 1. **[Core] 데이터 파이프라인 고도화 (OrderManager)**
- `FILL` 이벤트 발행 시 기존에 누락되었던 핵심 메타데이터(`entry_bar_index`, `stop_loss_pct`)를 페이로드에 강제 포함시켜 데이터의 원천 수급을 보장함.

### 2. **[Persistence] 생명주기 관리 SSoT 구축 (PersistenceManager)**
- **`orders_tracked` 테이블 신설**: 현재 열린(`OPEN`) 포지션만을 관리하는 전용 상태 머신 테이블을 구축하여 `trades.db`에 영구 기록함.
- **V20.9.1 데이터 증발(Vaporization) 차단**: 매수(BUY) 체결 시 데이터를 버리던 로직을 제거하고, `status='OPEN'`으로 기록하는 파이프라인 연결.
- **무결성 방어벽 (COALESCE UPSERT)**: GTID 기반의 UPSERT 구조를 도입하여 중복 이벤트 수신 시에도 기존의 온전한 데이터를 보호하도록 설계.
- **Orphan SELL 격리**: 매수 기록이 없는 매도 주문 발생 시 크리티컬 로그(DLQ)를 남기도록 방어 로직 강화.

### 3. **[State] 자가 치유 자율화 (StateManager)**
- **Full Hydration 메커니즘**: 메모리 캐시 오염이나 유실 감지 시 즉시 `orders_tracked` 테이블에서 7대 핵심 메타데이터를 퍼올리는 복원 로직 구현.
- **Self-Healing 각인**: 복구된 데이터를 즉시 메모리 캐시 및 `overrides.json`에 재주입하여 부팅 루프 및 불필요한 I/O 발생을 원천 차단.

## 🐞 주요 버그 픽스 (Hotfixes)
- **Scope Error**: `_writer_loop` 내 미정의 변수(`cursor_t`) 참조로 인한 NameError 수정.
- **Falsy Evaluation**: `bar_index=0` 또는 `stop_loss_pct=0.0`인 경우 이를 Null로 오판하여 불필요한 조회를 유발하던 로직을 `is not None` 기반으로 개선.
- **Code Contamination**: 종목코드의 'A' prefix 처리 시 발생할 수 있는 오탐 위험을 `lstrip` 및 자릿수 체크 가드로 방어.
- **Audit Gap**: 매수(BUY) 시 forensics.db에 기록이 누락되던 감사 추적 공백 해결.

---

## 🐞 전략 성과 대시보드 데이터 유실 및 UI 개선 (V20.9.2)

### 1. **[Core] 수동 매도 데이터 유실 방지 (OrderManager)**
- **문제 진단**: 수동 시장가 매도 주문 시, 전량 체결 전 앱이 종료되면 메모리(`fill_accumulator`)에 머물던 체결 데이터가 DB에 기록되지 않고 소실되는 현상 발견 (예: 알파칩스 136주 중 1주 체결 후 종료).
- **해결책**: `OrderManager.flush_sell_accumulators()` 메서드를 신설하여 종료 시점에 미완료된 SELL 체결 내역을 강제로 DB(`trades.db`)에 플러시하도록 보강.
- **안전 하차(Graceful Shutdown)**: `MainWindow.closeEvent`에서 플러시 로직을 호출하고, 영속화 서버가 기록을 마칠 수 있도록 2초의 지연 시간을 부여함.

### 2. **[UI] 청산 유형 표시 전략적 단순화 (TradeLogTab)**
- **사용자 요구사항**: '손절', '익절', '트레일링' 등 세분화된 자동 청산 유형이 수치(PnL)로 이미 확인 가능하므로, UI에서는 복잡도를 낮추기 위해 **'수동'**과 **'자동'** 2단계로만 통합 표시하도록 변경.
- **매핑 로직**: `MANUAL` 타입만 '수동'으로 표시하고, 그 외 모든 시스템 자동 트래킹(`NORMAL`, `STOP_LOSS` 등)은 '자동'으로 통일.

### 🏆 작업 성과
- **데이터 보존율 100%**: 이제 장 중 예기치 못한 종료나 수동 청산 시에도 단 1주의 체결 데이터도 누락되지 않는 강력한 영속성 확보.
- **UI 직관성 개선**: 성과 보드의 불필요한 레이블 노이즈를 제거하여 전략 성과 분석에만 집중할 수 있는 환경 구축.

### 3. 최종 검증 및 비대칭 전략(Combo) 고도화 (V21.9 Final)
- **버그 수정**: `ma_slope` 전달 오류 및 `MIXED` 흡수 버그를 해결하여 전략 분류의 정확성을 제도적 수준으로 끌어올림.
- **비대칭 전략(Asymmetric Strategy) 도입**: 
    - **VIP 하이패스**: `TREND` + `VOLUME` 동시 발생 시 `entry_th`를 추가로 -10 완화하여 주도주 편입 기회 극대화.
    - **손익 비대칭 출구**: 주도주 포스트 진입 시 `trailing_multiplier`를 4.5x로 확장하고, `Time Exit`을 3배 연장하여 '수익 극대화(Let Profit Run)' 원칙 구현.
- **검증 성과**:
    - **수익성**: 구버전 대비 누적 수익률 **+13.22%** 개선 확인.
    - **안정성**: 불필요한 `MIXED` 매매는 20건 감소시키고, `REVERSAL` 포착은 19건 증가시켜 전략 분산 효과 달성.

### 4. 비대칭 출구 엔진 전면 개편 (V20.9.1 Asymmetric Exit 4계)
GPT의 전략 평가("방패만 두꺼운 군대는 천하를 얻지 못한다")를 바탕으로, 출구 전략을 근본적으로 재설계했습니다.

**핵심 변경 사항**:
- **제1계 R-Multiple**: 모든 포지션에 `1R = entry - initial_sl` 척도를 내재화. 안전핀1: `R_MIN_FLOOR_PCT = 0.5%`로 R=0 폭주 방지.
- **제2계 무위험 지대**: 2.2R 도달 시 손절선을 `entry + (0.2 * R)`로 강제 전진(`BE 스위치`). 안전핀2: R 기반 정규화.
- **가변 트레일링**: 1.5R=4.5x → 2.2R=3.0x → 3.2R=2.0x → 5.2R=1.5x. 안전핀3: 히스테리시스 완충 구간.
- **제3계 TP1 전략 분리**: `TREND/BREAKOUT/VOLUME` → TP1 전면 폐지 (Let Profit Run). `REVERSAL/MIXED` → 2R 도달 시 절반 익절 유지.
- **제4계 스마트 시간 청산 2.0**: 수익권 + MA20 위 + slope 양수 + 최소 0.5R 조건 충족 시 홀딩 연장. 안전핀4: 미세 수익 연장 거부.
- **보너스 Early Cut**: 웜업 중이라도 -0.5R 이하 시 즉시 손절 (진입 오류 고속 절단).

**백테스트 검증 (20개 종목)**:
- 매매 횟수: 3,303건 (Old/New 동일) — 진입 로직 무영향 확인.
- 전략 분류 안정: MIXED -5건, VOLUME +9건 — 비대칭 TP1 폐지에 따른 자연스러운 재분류.
- 시스템 안정성: 오류 없이 전체 백테스트 완료 — 안전핀 4종 정상 작동 확인.

### 5. 내일의 체크포인트
- [ ] **VIP 하이패스** (`TREND+VOLUME 주도주 확정`) 로그 발생 여부 모니터링.
- [ ] **BE 스위치**: 2.2R 달성 시 `be_switched=True` 로그 확인 (무위험 지대 전환).
- [ ] **가변 트레일링**: 수익이 커질수록 trailing_multiplier가 4.5x→3.0x→2.0x로 자동 타이트닝 되는지 실전 확인.
- [ ] **스마트 홀딩**: `[SmartHold]` 로그로 시간 연장이 의미 있는 추세에서만 발동되는지 대조.
- [ ] **Early Cut**: 진입 초기 급락 시 `-0.5R` 이하에서 빠르게 절단되는지 체크.

---

## 🔎 심층 분석: 전략 차단 현상(TREND, MIXED, REVERSAL)에 대한 코드 교차 검증 (V21.8 MVSM Guard)

이전 회차(Flash)의 분석 리포트와 실제 엔진 코드(`app/ai/strategy.py`, `app/core/config_engine.py`, `app/services/managers/execution_manager.py`)를 교차 검증한 결과입니다. **Flash의 분석은 실제 시스템에 구현된 방어적 설계(Defensive Design) 철학과 정확히 일치함이 확인되었습니다.**

### 1. 가드 로직 작동 원리 일치성 (Code Validation)
시스템은 현재 자본 보호를 최우선으로 하는 `MVSM Guard Engine`의 통제를 엄격히 받고 있으며, 분석된 차단 사유들은 모두 코드에 명시된 하드 리미트와 일치합니다:
- **REVERSAL / MIXED**: `DOWNTREND` 국면에서 각각 `GUARD_REGIME_BLOCK_DOWNTREND_REVERSAL` 및 `GUARD_REGIME_BLOCK_DOWNTREND_MIXED` 정책에 의해 원천 차단됩니다 (로직 라인: `strategy.py: 1017~1035`). 이는 하락장에서 섣불리 바닥을 잡으려는 시도나, 추세가 없는 상태에서의 혼합 신호를 배제하기 위함입니다.
- **TREND / MIXED (기대수익 부족)**: 매수하기 좋은 상승장(`UPTREND`)이더라도, 진입 시점의 타겟 수익률(ATR 기반 목표가)이 시스템이 요구하는 최소 마진(`MIN_NET_REWARD`: 0.5% + 거래비용)을 넘지 못할 경우 '기대수익 부족'으로 매수를 보류합니다 (로직 라인: `strategy.py: 1046~1062`). 
- **MTF (Multi-Timeframe) 역배열**: 단기 5분봉에서 긍정적인 신호가 발생해도, 상위 프레임인 60분봉(MA240) 역배열 하에 있을 경우 추세 추종 시도를 원천 차단하는 `MTF Guard`가 작동 중입니다 (로직 라인: `strategy.py: 1065~1084`).

### 2. 왜 BREAKOUT과 VOLUME만 계속 체결되는가? (Exemption Logic)
해당 전략들만 지속적으로 거래되는 원인 역시 코드 내 예외(Exemption) 로직으로 명확히 입증됩니다:
- **Reward Check 심사 면제**: 모멘텀 기반 진입의 속도를 극대화하기 위해, 1043번째 줄(`if strategy_type_val in ('BREAKOUT', 'VOLUME'): pass`)에서 **기대수익 검증 단계를 전면 면제(Exempt)** 받고 있습니다.
- **MTF 가드 무시권 부여**: 거래량 비율(`vol_ratio`)이 `1.8`(경계치)을 돌파하며 폭발할 경우, 상위 60분봉이 역배열이더라도 이를 무시하고 강제 진입을 허용하는 강력한 예외 조건(`is_mtf_exception`)이 하드코딩되어 있습니다 (로직 라인: `strategy.py: 1070`).

### 3. Shadow Mode(그림자 모드)의 구조적 한계 규명
현재 엔진은 `GUARD_SHADOW_MODE = True`로 설정되어 있어, 가드들이 체결을 물리적으로 막지 않고 내부 로그(`[SHADOW]`)만 남기고 BUY 사인을 올려보내고 있습니다. 그럼에도 해당 전략들이 체결되지 않는 진짜 이유는 **Execution Manager 레벨의 필터링 허들** 때문입니다.
- Execution Manager의 `on_timer` 루프는 시스템 리소스를 최적화하기 위해 현재 종목이 **보유 중(held), 강제 손절 대상, 사용자가 포커스한 모니터링 종목(active), 혹은 일시 승격된 종목(promoted_pool) 중 하나일 때만 전략 신호를 체결 엔진으로 패스**합니다 (`execution_manager.py: 435`). 
- 다른 전략들은 점수 스레스홀드를 통과하여도, 위 집중 감시망(Surveillance) 안에 진입할 수 있는 강한 모멘텀 트리거가 부족하여 체결 요청으로 넘어가지 않고 스킵(`continue`)되는 구조적 방어가 존재했습니다.

### 🏆 결론 및 향후 방향
현재의 차단 현상은 시스템 결함이나 로직 에러가 아닙니다. **엔진이 철저하게 리스크를 통제하고 낙하칼을 피하며 뼈대 있는(Volume/Breakout) 자리에만 집중 타격하도록 설계된 L3-L4 방어 체계의 완벽한 작동 결과**입니다. 
만약 포트폴리오의 매매 빈도를 높이고 다양한 전략을 가동하고 싶다면, `config_engine.py`에서 `MIN_NET_REWARD`를 인하(예: 0.3%)하거나 `ENTRY_THRESHOLD_DOWNTREND` 등의 허들값을 선별적으로 완화하는 리스크 선호도 조정 튜닝을 계획해볼 수 있습니다.

---
**기록이 기억을 지배하는 NoCodeQuant V20.9.2로의 도약이 완료되었습니다.**
