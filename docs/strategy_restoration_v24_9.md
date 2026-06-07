# 🛡️ 주도주 트레이딩 엔진 성과 복구 및 래칫/진입 임계값 포렌식 개선 보고서

> **문서 상태**: 승인 및 적용 완료 (ACTIVE)  
> **최종 수정일**: 2026-05-19  
> **해당 모듈**: `app/ai/_engine_exit.py`, `app/core/config_engine.py`, `app/services/managers/persistence_manager.py`  
> **목적**: 2026-05-05 백업 버전 골든-런 대비 급락한 운용 버전의 성능(승률 0%) 원인을 규명하고, 시스템 강건성을 유지하면서 핵심 트레이딩 엣지(Edge)를 완벽하게 원복하고, DB 영속화 레이어의 크래시 요인을 완전 제거함.

---

## 1. 배경 및 포렌식 분석 (Forensic Diagnostic)

### 1.1 성능 격차 지표 (Performance Delta)
* **백업 버전 (Good, 2026-05-05 시점)**: 승률 **46.2%**, 누적 수익 **+611,919 KRW**, 프로핏 팩터 **5.84**.
* **현재 운용 버전 (Bad, 변경 후)**: 승률 **0.0%**, 누적 수익 **-486,214 KRW**, 프로핏 팩터 **N/A (0승 11패)**.

### 1.2 4대 핵심 결함 판명

#### ① 고주파 틱(Tick) 레벨 래칫(Ratchet) 청산 오작동
* **상황**: 수익 구간(+0.5% 초과) 진입 시, 래칫 평가 기준 PnL(`_pnl_pct`)에 실시간 틱 단위 고점인 `peak_price`가 연동되었습니다.
* **영향**: 틱 데이터 수신 주기(1초)마다 최고가가 갱신되는 특성상, 순간적인 급등 틱 스파이크로 인해 래칫 손절선(SL)이 최고 단계로 긴박하게 끌어올려졌습니다. 이후 정상적인 단기 숨고르기(Pullback) 구간에서 가격이 살짝 쳐지자마자 전진된 래칫 SL을 하향 이탈하며 **즉각 조기 청산 시장가 매도**가 실행되었습니다. 이로 인해 트레이딩 호흡이 상실되어 모든 거래가 실시간 손실/약수익으로 조기 누수되었습니다.

#### ② 핵심 전략 진입 임계값(Threshold)의 극단적 상향
* **상황**: 추세 추종(`ENTRY_TH_TREND`) 임계값이 60 → 68로, 돌파 매매(`ENTRY_TH_BREAKOUT`) 임계값이 50 → 54로 인상되었습니다.
* **영향**: 스코어 68점은 모든 보조지표와 모멘텀이 시세 분출 후기에 다다른 과열 국면(상투)에서만 포착되는 수치입니다. 따라서 추세 초입에서의 스마트한 선탑승 기회가 구조적으로 원천 차단되고, 극소수의 지각 진입(상투 잡기)만 허용되어 손실 가능성이 급증했습니다.

#### ③ L3 방어 가드(Risk/Regime/Reward/MTF)의 과활성화
* **상황**: `GUARD_SHADOW_MODE` 정책 상수가 `True` (Shadow/진단 모드)에서 `False` (Active/실거래 차단 모드)로 명시 변경되었습니다.
* **영향**: 고화질의 돌파/수급 알파 신호들이 보수적인 리스크 이격 필터와 기대수익 조건에 가로막혀 **매수 보류(HOLD_BLOCK)**됨으로써 진입 빈도가 극히 위축되었습니다.

#### ④ SQLite `orders_tracked` 테이블 `UNIQUE constraint failed` 에러 유발
* **상황**: `orders_tracked` 테이블은 `idx_single_open_position (code, strategy_type) WHERE status='OPEN'` 인덱스를 통해 동일 종목/전략의 단일 활성 포지션(OPEN)만 유지하도록 강제합니다.
* **영향**: 장중 비동기 네트워크 지연, 예외 복구 시점 오차 등으로 인해 이전 `SELL` 시 `status='CLOSED'` 업데이트에 도달하지 못해 DB 상에 stale `OPEN` 포지션이 잔존하는 현상이 간헐적으로 발생합니다. 이 상황에서 동일 종목/전략에 대한 새로운 `BUY` 시그널이 발생하여 `INSERT INTO orders_tracked`를 시도할 때, SQLite UNIQUE 인덱스가 충돌하여 `UNIQUE constraint failed` 에러를 뿜으며 백그라운드 DB 라이터 큐가 중단되고 이벤트가 DLQ(`persistence_dlq.jsonl`)로 버려졌습니다.

---

## 2. 정밀 복구 및 패치 내역 (Applied Patches)

최근 시스템에 반영된 **재기동(Restart) 상태 복원력 및 틱 갭 모니터링 강건성 코드는 100% 보존**하면서, 트레이딩 알파와 시스템 예외 회복력을 극대화하도록 패치했습니다.

### 2.1 래칫 청산 트리거 5분봉 종가 기준 복원
* **파일**: `C:\Users\stone\projects\nocodequant\app\ai\_engine_exit.py`
* **수정**: `_pnl_pct` 판단을 틱 최고가가 아닌 **5분봉 캔들의 마감 종가(`curr['close']`)** 기준으로 완전히 환원했습니다. 가격의 일시적인 흔들림 노이즈를 충분히 견디도록 숨쉬기 공간을 마련했습니다.
```python
if ACTIVE_CONFIG.get("RATCHET_ENABLED", True):
    # [V24.9 Revert] 래칫 트리거를 5분봉 종가 기준으로 원복.
    # peak_price는 틱(1초) 단위로 갱신되므로 장중 찰나의 스파이크에
    # 래칫이 과도하게 전진하여 정상 눌림목에서 즉시 청산되는 문제 해결.
    _pnl_pct = (curr['close'] - entry_p) / entry_p
```

### 2.2 진입 파라미터 및 섀도우 가드 복구
* **파일**: `C:\Users\stone\projects\nocodequant\app\core\config_engine.py`
* **수정**:
  - `ENTRY_TH_TREND = 60` (68에서 복원 - 추세 초입 편입)
  - `ENTRY_TH_BREAKOUT = 50` (54에서 복원 - 고알파 돌파 기회 확보)
  - `GUARD_SHADOW_MODE = True` (False에서 복원 - 섀도우 로깅 진입 통과)
```python
ENTRY_TH_BREAKOUT = 50  # 돌파 매매: 엄격분류는 score 자체가 높아 어차피 통과
ENTRY_TH_VOLUME   = 49  # 수급 매매 (-1)
ENTRY_TH_TREND    = 60  # [V23.3 FIX] 추세 추종: 50→60 복원 (노이즈 진입 차단)
...
GUARD_SHADOW_MODE      = True
```

### 2.3 `orders_tracked` DB Stale OPEN 선제 차단 및 이력 보존형 자가 치유(Self-Healing) 로직 도입
* **파일**: `C:\Users\stone\projects\nocodequant\app\services\managers\persistence_manager.py`
* **수정**: 
  - 단순 `INSERT OR REPLACE` 구조는 충돌 해결에는 용이하나 기존 Stale 포지션의 이력(History) 정보가 물리적으로 완전 삭제되는 포렌식 무결성 부작용이 있었습니다.
  - 이를 해결하기 위해 **2단계 자가 치유(Self-Healing) 모델**을 적용했습니다:
    1. **Stale OPEN 선제 격하**: 신규 `BUY` 진입 발생 직전, 동일한 종목과 전략으로 열려있던 기존 `OPEN` 레코드들의 상태를 `CLOSED`로 안전하게 업데이트하여 포렌식 추적 이력을 고스란히 보존합니다.
    2. **기존 upsert 수행**: 충돌이 사라진 상태에서 원래의 `INSERT INTO ... ON CONFLICT(gtid) DO UPDATE` 구문을 집행하여 신규 포지션을 `OPEN` 상태로 무결하게 등록합니다.
```python
# [V25.1 Self-Healing] 충돌 전 선제적 Stale OPEN 포지션 처리
# UNIQUE 인덱스 idx_single_open_position(code, strategy_type) WHERE status='OPEN'
# 충돌 방어: 동일 종목/전략의 이전 Stale OPEN 레코드를 CLOSED로 격하하여
# 이벤트 이력을 보존하면서 새로운 BUY의 삽입을 안전하게 허용.
_stale_closed = cursor.execute("""
    UPDATE orders_tracked SET status = 'CLOSED', last_update_time = ?
    WHERE code = ? AND strategy_type = ? AND status = 'OPEN'
""", (now_utc, p.get("code"), p.get("strategy_type"))).rowcount
if _stale_closed > 0:
    self.logger.warning(
        f"⚠️ [Self-Heal] Stale OPEN x{_stale_closed} 자동 CLOSED 처리 "
        f"→ {p.get('code')} / {p.get('strategy_type')} (신규 BUY 진입 허용)"
    )

# [V20.9.1 Bugfix 1] cursor_t -> cursor (NameError 방지, 올바른 스코프 사용)
cursor.execute("""
    INSERT INTO orders_tracked (
        gtid, code, status, strategy_type, regime, entry_price, 
        entry_time, entry_bar_index, timeframe, stop_loss_pct, last_update_time
    ) VALUES (?, ?, 'OPEN', ?, ?, ?, ?, ?, ?, ?, ?)
    ON CONFLICT(gtid) DO UPDATE SET 
        strategy_type = COALESCE(excluded.strategy_type, orders_tracked.strategy_type),
        regime = COALESCE(excluded.regime, orders_tracked.regime),
        entry_price = COALESCE(excluded.entry_price, orders_tracked.entry_price),
        entry_time = COALESCE(excluded.entry_time, orders_tracked.entry_time),
        entry_bar_index = COALESCE(excluded.entry_bar_index, orders_tracked.entry_bar_index),
        timeframe = COALESCE(excluded.timeframe, orders_tracked.timeframe),
        stop_loss_pct = COALESCE(excluded.stop_loss_pct, orders_tracked.stop_loss_pct),
        last_update_time = excluded.last_update_time
""", (
    p.get("gtid") or gtid, p.get("code"), p.get("strategy_type"), p.get("regime"),
    p.get("entry_price"), p.get("entry_time"), p.get("entry_bar_index"),
    p.get("timeframe"), p.get("stop_loss_pct"), now_utc
))
```

---

## 3. 기대 및 검증 시나리오

1. **승률의 원래 수준 회복**: 틱 휩소로 인해 잘려나가던 11패 분량이 홀딩 후 정상 익절(TP1, TP2) 또는 스마트 트레일링 구간으로 연동되어, 원래의 **46% 승률 및 높은 프로핏 팩터**로 복귀가 예상됩니다.
2. **조기 진입 안정성**: 주도주의 상승 초기 구간(임계점 60/50 수준)에서 시그널 포착 즉시 주문이 기민하게 집행되어 손익비가 극대화됩니다.
3. **복구 회복력 상호 작용**: 장중 재기동 시에도 `strategy.py`의 강화된 `sync_position`이 기존 SL 상태와 최고점 정보를 정확히 바인딩하므로 안심하고 운용이 가능합니다.
4. **DB 영속화 무결성 보장**: `INSERT OR REPLACE` 구조의 투입으로 인해 장중 재기동, 틱 오차에 관계없이 백그라운드 DB 큐 동결 및 UNIQUE 제약 조건 위반 오류가 0%로 수렴하여 견고한 포지션 트래킹 환경을 제공합니다.

