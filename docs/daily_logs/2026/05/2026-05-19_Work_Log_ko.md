# 📅 2026-05-19 업무 일지 (Daily Work Log)

## 🎯 주요 목표 및 이슈 정의
- [x] **2026-05-05 백업 버전 골든-런 대비 성과 붕괴 포렌식 복구 및 튜닝**: 현재 운용 버전의 급격한 성과 하락(승률 0.0%, 0승 11패)을 극복하고 이전 강력한 성과(승률 46.2%, 프로핏 팩터 5.84)를 복구하기 위한 핵심 파라미터 및 로직 튜닝 적용.
- [x] **래칫(Ratchet) 청산의 고주파 틱 스파이크 조기 청산(Premature Stop-out) 차단**: 래칫 계산 기준 `_pnl_pct`을 실시간 틱 고점(`peak_price`)에서 5분봉 종가(`curr['close']`) 기준으로 복원하여 충분한 트레이딩 호흡 확보.
- [x] **진입 병목 해소 및 전략 임계값 Sweet Spot 환원**: TREND 임계값 68→60, BREAKOUT 임계값 54→50 하향 조정하여 랠리 초입 기민한 승차 기회 확보.
- [x] **L3 방어 가드의 섀도우 모드 복구**: `GUARD_SHADOW_MODE = True`로 전환하여 고수익 진입 시그널의 즉각적 실행 보장.
- [x] **orders_tracked DB UNIQUE 제약 조건 충돌 버그 해결**: 장중 재시작 또는 이전 세션의 미처리 `OPEN` 포지션이 잔존할 때, 동일 종목/전략의 새로운 `BUY` 진입 시 발생하는 SQLite `UNIQUE constraint failed` 에러 원천 차단.

---

## ✅ 상세 작업 및 패치 내역

### 1. `_engine_exit.py` 래칫 트리거 계산식 원상복구 (L3)
* **원인 분석**: 틱 레벨에서 1초마다 업데이트되는 실시간 `peak_price`를 기준으로 래칫 SL 단계를 조임에 따라, 장중 찰나의 틱 스파이크 후 지연이나 정상 눌림목 시 전진된 SL 이하로 가격이 하강하여 즉시 조기 시장가 매도가 터짐 (0승 11패의 주원인).
* **해결 방안**: 래칫 판단을 위한 `_pnl_pct` 계산을 5분봉 종가인 `curr['close']` 기준으로 단순화 및 원복하여, 캔들 마감 시점까지 가격 변동 노이즈를 견딜 수 있는 공간(Breathing Room) 제공.
* **패치 코드**:
  ```python
  if ACTIVE_CONFIG.get("RATCHET_ENABLED", True):
      # [V24.9 Revert] 래칫 트리거를 5분봉 종가 기준으로 원복.
      # peak_price는 틱(1초) 단위로 갱신되므로 장중 찰나의 스파이크에
      # 래칫이 과도하게 전진하여 정상 눌림목에서 즉시 청산되는 문제 해결.
      _pnl_pct = (curr['close'] - entry_p) / entry_p
  ```

### 2. `config_engine.py` 핵심 전략 임계값 및 가드 모드 최적 복구 (L3)
* **원인 분석**: 
  - `ENTRY_TH_TREND`가 68로 과도하게 상향되어 추세 극초기 랠리가 차단되고 상투 시점에서만 늦게 진입하는 병목이 생김.
  - `ENTRY_TH_BREAKOUT`이 54로 상향되어 고화질 필터링의 역설로 알파 돌파 거래가 상당수 유실됨.
  - `GUARD_SHADOW_MODE`가 `False`로 설정되어 L3 가드들(Risk, Regime, Reward, MTF)이 활성 매수 매달림을 강하게 차단함.
* **해결 방안**: 
  - `ENTRY_TH_TREND = 60` 및 `ENTRY_TH_BREAKOUT = 50`으로 골든-런 설정 환원.
  - `GUARD_SHADOW_MODE = True`로 설정하여 시그널을 Shadow 모드로 통과시키고 고수익 유도를 극대화.
  - 최근 반영된 틱 갭 모니터링, StateManager 재기동 복원 SL 정합성 매핑 등 모든 프로그램 강건성 코드는 그대로 보존하여 하이브리드 안정성 달성.

### 3. `persistence_manager.py` orders_tracked Stale OPEN 선제 차단 및 이력 보존형 자가 치유(Self-Healing) 적용 (L3)
* **원인 분석**: 
  - `orders_tracked` 테이블은 동일 종목이 특정 전략으로 동시에 두 번 `OPEN` 상태로 존재하지 못하도록 `idx_single_open_position (code, strategy_type) WHERE status='OPEN'` UNIQUE 인덱스를 통해 엄격히 제어함.
  - 세션 기동 오차, Kiwoom-로컬 간의 비동기 틱 지연으로 인해 이전 매도(`SELL`) 처리 시 `status='CLOSED'` 업데이트를 놓치거나 오더 추적 불일치가 발생할 경우, 디비에는 여전히 `OPEN`으로 남아있게 됨.
  - 이 상태에서 동일한 종목/전략으로 새로운 `BUY` 진입이 성립하여 `INSERT INTO orders_tracked`를 시도할 때 `ON CONFLICT(gtid)`는 작동하지만 `idx_single_open_position` 인덱스 충돌을 잡지 못해 `UNIQUE constraint failed` 에러가 발생하여 백그라운드 DB 큐가 동결/DLQ로 빠짐.
* **해결 방안 및 비교 검토**:
  - *안건 A (INSERT OR REPLACE)*: 충돌 시 기존 행을 삭제하고 덮어씌움. 동작은 가능하나 기존 Stale 포지션의 이력(History) 정보가 물리적으로 완전히 소멸하여 포렌식 감사 무결성이 깨지는 심각한 부작용이 발견됨.
  - *안건 B (2단계 선제적 자가 치유 - 채택)*:
    1. **Stale OPEN 선제 격하**: 신규 `BUY` 진입 이벤트 처리 직전, 동일한 종목과 전략으로 이미 존재하는 모든 `OPEN` 포지션을 `CLOSED` 상태로 강제 전환(`UPDATE orders_tracked SET status='CLOSED' WHERE ... AND status='OPEN'`)하여 거래 이력을 안전하게 영구 보존함.
    2. **기존 안전 upsert 수행**: 이후 안전하게 충돌이 제거된 상태에서 원래의 `INSERT INTO ... ON CONFLICT(gtid) DO UPDATE` 쿼리를 완벽하게 집행함.
* **패치 코드**:
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


### 4. `engine_fsm.py` 슈미트 트리거(Schmitt Trigger) 기반 Whipsaw 방지 패치 (L3) — V15.0
* **원인 분석 (아이로보틱스 066430 포렌식)**:
  - 09:03:03 `score=82` → FSM `READY` (매수 진입, exposure 0.5) → **2초 후** 09:03:05 `score=67` → FSM `GHOST_GAP` (즉시 전량 매도, exposure 0.0) → 09:03:18 `score=90` → 재진입 → 09:03:24 `score=62` → 재청산. 단 20초 내 2회 왕복 매매(Whipsaw) 발생.
  - **근본 원인 1**: `stutter_count >= 1` 설정으로 히스테리시스(상태 전이 감쇠)가 사실상 무력화되어, 단일 노이즈 틱만으로도 FSM 상태가 즉시 전환됨.
  - **근본 원인 2**: 진입 임계값(score ≥ 75)과 청산 임계값이 동일하여, 75점 경계에서의 미세 진동이 포지션의 극단적 변동으로 직결됨.
* **해결 방안 (슈미트 트리거 패턴 - 제어공학 비대칭 히스테리시스)**:
  1. **비대칭 확인 틱**: 상향 전이(EARLY→READY 등 진입)는 **연속 2틱 확인**, 하향 전이(READY→EARLY 등 청산)는 **연속 3틱 확인** 필요. 방향이 바뀌면 카운터 즉시 리셋(노이즈 기각).
  2. **비대칭 스코어 밴드 (15점 Dead-Band)**: 진입은 `score ≥ 75`에서 유지하되, 이미 READY/ENTRY 상태인 경우 `score < 60`으로 지지가 완전히 깨져야만 하향 전이 허용. 60~74점 구간은 현재 상태를 유지하는 불감대(Dead-Band)로 작동.
  3. **타깃 방향 추적**: `last_target_state`를 통해 동일 방향의 연속성만 카운트하여, 노이즈로 인한 방향 진동 시 카운터가 절대 누적되지 않도록 설계.
* **기대 효과 (동일 로그 시뮬레이션)**:
  - 09:03:03 score=82 → 타깃 READY 1틱차 → 09:03:05 score=67 → 타깃 변경 → **진입 불발** (2틱 연속 미달)
  - 09:03:18~22 score=90→90→90→80 → **연속 4틱 ≥ 2 → 진입 성공**, 이후 score=62 (≥60 Dead-Band) → **청산 불발 (유지)**
  - **Whipsaw 완전 제거**, 불필요 왕복 수수료 차단.
* **패치 코드 (핵심부)**:
  ```python
  # Schmitt Trigger: Asymmetric hysteresis thresholds
  UPWARD_CONFIRM_TICKS = 2    # Ticks required to promote state
  DOWNWARD_CONFIRM_TICKS = 3  # Ticks required to demote state
  READY_EXIT_SCORE = 60       # Entry >= 75, Exit < 60 = 15-point dead-band

  # _determine_target() 내 비대칭 밴드
  elif score >= self.READY_EXIT_SCORE and currently_exposed:
       return self.current_state  # Dead-Band: HOLD, do NOT demote
  ```

---

## 🔮 장중 진행 상황 모니터링 예정 사항
1. **장중 진입 빈도 및 타이밍 검증**: 5분봉 종가 기준 래칫 복구 및 진입 임계값(60/50) 복원으로 장초반 및 장중 주도주(HOT) 편입이 조기에 정확하게 성립하는지 확인.
2. **청산 노이즈 감시**: 고주파 틱 스파이크로 인한 조기 손절(Premature shakeout)이 성공적으로 제어되고, 주도가 지속되는 흐름 동안 홀딩 호흡을 안정적으로 확보하는지 추적.
3. **독립 보고서 작성 완료**: `docs/strategy_restoration_v24_9.md`에 포렌식 대조 내역 및 검증 결과 영구 문서화.
4. **DB 라이터 에러 유실 모니터링**: Self-Healing 적용 후 백그라운드 스레드에서 `UNIQUE constraint failed` 에러가 더이상 보고되지 않고 정상적으로 트랜잭션이 완결되는지 검증.
5. **[신규] FSM Whipsaw 감시**: V15.0 슈미트 트리거 적용 후, 동일 종목에서 READY↔EARLY/GHOST_GAP 간의 20초 내 왕복 전이가 재발하지 않는지 ESL 로그를 통해 검증.

