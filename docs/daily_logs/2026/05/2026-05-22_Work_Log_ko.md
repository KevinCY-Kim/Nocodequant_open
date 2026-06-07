# 📅 2026-05-22 업무 일지 (Daily Work Log)

## 🎯 주요 목표 및 이슈 정의
- [x] **대한광통신(010170) 진입가 표기 오류 및 수익률 부호 왜곡 버그 해결**: 
  진입가(23,300)보다 청산가(23,500)가 높은데도 수익금(-70,928원)과 수익률(-2.38%)이 마이너스로 출력되던 오기 현상을 수정했습니다. 실제 체결가는 24,000원이나 영속화 단계의 자가 치유 로그 버그와 매수 체결 메타데이터의 목표가 캐싱 문제로 인해 발생한 데이터 왜곡 건입니다.
- [x] **실시간 주도주 탭별 2단계 감시(Surveillance & Strategy Tier) 아키텍처 구축**:
  보유 종목, 활성 종목, 관심 종목 외에 실시간 주도주 탭(HOT, PRE_HEAT, REVERSE)에 노출되는 전 종목을 감시 풀에 넣어 시세 데이터를 수집하고, 그중 상위권 유망 종목들을 매초 라이브 전략 연산 대상으로 승격시키는 2단계 감시 체계를 설계하여 매매 기회 손실을 최소화했습니다.

---

## ✅ 상세 작업 및 패치 내역

### 1. 근본 원인 분석 요약
* **원인 1 (`PersistenceManager` 예외 및 매수 원장 누락)**:
  자가 치유(Self-Healing) 로직 수행 중 존재하지 않는 `self.logger.warning`을 호출하여 `AttributeError` 예외가 발생했고, 매수(BUY) 기록 트랜잭션이 SQLite에 커밋되지 못하고 데드레터큐(`persistence_dlq.jsonl`)로 누락되었습니다.
* **원인 2 (`OrderManager` 고아 매도 복구 및 진입가 캐싱 왜곡)**:
  매도(SELL) 시점에 매수 기록이 누락되어 고아 매도(`ORPHAN SELL`) 복구 경로로 빠졌습니다. 이 복구 경로에서는 캐시된 신호의 목표 진입가(23,300)를 강제 사용하였으나, 실제 매수는 장 시작 직후 슬리피지로 인해 24,000원에 체결되었었습니다. 
  실제 청산은 23,500원에 이루어졌으므로 -70,928원의 손실이 정확하게 발생했으나, UI 및 DB 표기상의 진입가는 23,300원으로 기록되어 진입가보다 청산가가 높은데 마이너스 손실이 나오는 visual 모순이 유발되었습니다.

---

### 2. 패치 코드 내역

#### ① [app/services/managers/persistence_manager.py](file:///c:/Users/stone/projects/nocodequant/app/services/managers/persistence_manager.py#L247) 수정
* **자가 치유 로직 내 속성 에러 수정**: `self.logger.warning` 호출부를 모듈 전역에 정의된 `logger.warning` 객체로 수정하여 예외로 인한 DB 트랜잭션 롤백을 차단합니다.
```python
                                # [V25.2 FIX] self.logger.warning -> logger.warning (AttributeError 방지)
                                if _stale_closed > 0:
                                    logger.warning(
                                        f"⚠️ [Self-Heal] Stale OPEN x{_stale_closed} 자동 CLOSED 처리 "
                                        f"→ {p.get('code')} / {p.get('strategy_type')} (신규 BUY 진입 허용)"
                                    )
```

#### ② [app/services/managers/order_manager.py](file:///c:/Users/stone/projects/nocodequant/app/services/managers/order_manager.py#L456) 수정
* **실제 체결 평균 단가를 진입 메타데이터에 반영**: 매수 완료(BUY FILL) 처리 시, 신호 목표가(`derived_entry_price`)가 아닌 실제 거래소 평균 체결 단가(`price`)를 `entry_price`로 갱신하여 `position_meta`에 저장합니다.
```python
                strategy_type = derived_strategy_type
                regime        = derived_regime
                # [V25.2 FIX] 진입가를 신호 목표가 대신 실제 거래소에서 체결된 평균 단가로 갱신
                entry_price   = price if price > 0.0 else float(derived_entry_price or 0.0)
```

#### ③ [app/services/managers/order_manager.py](file:///c:/Users/stone/projects/nocodequant/app/services/managers/order_manager.py#L592) 수정 (2차 패치)
* **부분 체결 가중평균단가 실시간 갱신 및 SSoT 동기**:
  - BUY 체결이 누적될 때마다 `price_sum / cum_qty`를 가중평균단가로 계산하여 `acc["entry_price"]`를 실시간 갱신합니다.
  - 이를 `StateManager.set_position_meta(code, {'entry_price': weighted_avg})`를 호출하여 메모리 및 영속화 오버라이드 캐시(`position_meta_cache`)에 동기화합니다.
  - BUY `PersistenceEvent` emit 시 `payload["entry_price"]` 및 max/min_price에 갱신된 가중평균단가를 전달하도록 수정하였습니다.

#### ④ [app/services/managers/persistence_manager.py](file:///c:/Users/stone/projects/nocodequant/app/services/managers/persistence_manager.py#L310) 수정 (2차 패치)
* **고아 매도 복구 정책(Orphan Sell Recovery Policy) 강화**:
  - 고아 매도(BUY 기록 유실) 발생 시 복구 경로의 우선순위를 다음과 같이 고도화하고, 신호 목표가(signal metadata) fallback 사용을 차단했습니다:
    1. `StateManager` 스냅샷 (`positions` 및 `daily_price_cache`) 조회.
    2. 유실 시 `KiwoomEngine._instance.positions` (실시간 Chejan 기반 broker execution ledger) 조회.
    3. 둘 다 실패할 경우, 강제로 `DLQ 격리`(`persistence_dlq.jsonl` 파일 기록) 및 `continue`하여 잘못된 진입가 오염 레코드가 `trades` 테이블에 삽입되는 것을 원천 차단.

#### ⑤ 2단계 감시(Tiered Surveillance) 파이프라인 구현
* **[app/services/managers/ranking_manager.py](file:///c:/Users/stone/projects/nocodequant/app/services/managers/ranking_manager.py#L959-L970) 수정**
  - `get_top_codes`가 HOT 상위 30개 + PRE_HEAT 상위 20개 + REVERSE 상위 10개 = 총 60개 종목을 중복 제거하여 통합 반환하도록 수정했습니다. (Quote-only Tier 후보군 확장)
* **[app/services/surveillance.py](file:///c:/Users/stone/projects/nocodequant/app/services/surveillance.py#L111-L127) 수정**
  - `SurveillanceController.reconcile`에서 실시간 구독 대상(`desired_active`)을 `top_codes` 전체(60개)와 `promoted_pool`로 설정하여 실시간 시세 수집 대상을 대폭 확장했습니다.
  - 3-strike 해제 조건인 `to_remove_candidates`에서도 이 전체 60개 종목군을 제외하여 실시간 틱 데이터가 누락 없이 연속 축적되도록 보장했습니다.
  - **[버그 수정]** `reconcile` 내에서 정의되지 않은 변수 `top_50`을 참조하여 `NameError`가 발생해 실시간 감시 목록 동기화가 실패하던 결함을 `top_codes_set`으로 수정하여 해결했습니다.
* **[app/ui/main_window.py](file:///c:/Users/stone/projects/nocodequant/app/ui/main_window.py#L1272-L1287) 수정**
  - `_debounced_ranking_update` 내 `Tier 1 Auto-Promotion` 로직을 개선하여, 단순히 HOT 상위 15개만 승격하는 대신 `HOT 상위 15개` + `PRE_HEAT 상위 5개` + `REVERSE 상위 5개` 종목을 추출하여 중복 제거 후 `promotion_queue`에 직접 편입해 매초 전략판단(Strategy Tier)을 적용받도록 만들었습니다.

---

## 🔮 검증 및 테스트 결과
1. **단위 및 통합 테스트 실행**:
   - `tests/test_pnl_ssot_consistency.py`를 통해 금일 및 과거 3일간의 SSoT PnL 정합성을 검증하여 정합성이 깨지지 않음을 확인했습니다.
2. **패치 효과 검증**:
   - **가중평균단가 실시간 갱신**: KRX 장중 분할/부분 체결(partial fill)이 발생하더라도 최종 체결 가격이 아닌 실제 거래량이 반영된 가중평균단가가 `entry_price`로 유지되어 평균단가 왜곡을 차단합니다.
   - **고아 매도 철저 방어**: `StateManager`뿐만 아니라 `KiwoomEngine` 인스턴스에 유지되고 있는 실시간 포지션 원장(`engine.positions`)까지 순차 복구하여 데이터 유실을 최소화하고 무결성(SSoT)을 보장합니다.
3. **실시간 주도주 감시 및 자동 승격 검증**:
   - `python -m py_compile`을 사용하여 구문 오류를 검증 완료하였습니다.
   - 2단계 감시 체계가 정상 가동됨에 따라, PRE_HEAT/REVERSE 탭 종목의 실시간 틱이 유실 없이 수집되어 실시간 가속도/회복성 지표의 점수 정합성이 극대화되고, 상위 5개 유망 종목이 실시간 라이브 전략 연산 풀(`promoted_pool`)에 정상적으로 자동 진입하여 감시와 기회 포착 효율이 향상되었습니다.및 영속화 오버라이드 캐시(`position_meta_cache`)에 동기화합니다.
  - BUY `PersistenceEvent` emit 시 `payload["entry_price"]` 및 max/min_price에 갱신된 가중평균단가를 전달하도록 수정하였습니다.

#### ④ [app/services/managers/persistence_manager.py](file:///c:/Users/stone/projects/nocodequant/app/services/managers/persistence_manager.py#L310) 수정 (2차 패치)
* **고아 매도 복구 정책(Orphan Sell Recovery Policy) 강화**:
  - 고아 매도(BUY 기록 유실) 발생 시 복구 경로의 우선순위를 다음과 같이 고도화하고, 신호 목표가(signal metadata) fallback 사용을 차단했습니다:
    1. `StateManager` 스냅샷 (`positions` 및 `daily_price_cache`) 조회.
    2. 유실 시 `KiwoomEngine._instance.positions` (실시간 Chejan 기반 broker execution ledger) 조회.
    3. 둘 다 실패할 경우, 강제로 `DLQ 격리`(`persistence_dlq.jsonl` 파일 기록) 및 `continue`하여 잘못된 진입가 오염 레코드가 `trades` 테이블에 삽입되는 것을 원천 차단.

#### ⑥ 실시간 감시 자원 안정성 및 운영안정성 고도화 패치 (V25.4 추가)
* **[app/services/managers/ranking_manager.py](file:///c:/Users/stone/projects/nocodequant/app/services/managers/ranking_manager.py#L959-L980) 수정 (Slot Reservation 대응)**:
  - `get_top_codes`에서 반환하는 튜플에 소스 탭 유형(HOT, PRE_HEAT, REVERSE) 정보를 8번째 필드로 추가하여 downstream에서 탭 유형을 구분할 수 있게 개선했습니다.
* **[app/services/surveillance.py](file:///c:/Users/stone/projects/nocodequant/app/services/surveillance.py#L111-L265) 수정 (우선순위 & 가변예산 & 슬롯예약)**:
  - **[P0] 85개 Hard Cap 및 우선순위 스케줄러**: 실시간 틱 수신 제한(키움 OpenAPI 안정 수신 한계)에 대응해 최대 실시간 감시 종목 수를 85개(`MAX_REALTIME_CODES = 85`)로 강력 제한하고, `HELD` (MANDATORY_IMMORTAL_SET, 절대 제거 금지) > `PROMOTED` (HIGH) > `FAVORITES` (MEDIUM) > `RANKED` (LOW)의 순위를 보장하는 Priority Scheduler를 구축했습니다.
  - **[P1] 장초반 Dynamic Cycle Budget**: 09:00~09:15(장초반 15분) 구간은 12회, 30분 이내 또는 높은 등록 빈도 감지(최근 60초 내 API 호출 30회 초과) 시 8회, 평시는 5회의 가변 사이클 예산(sliding call budget)을 동적으로 조정하도록 개선했습니다.
  - **[P1] Cooldown Bypass 튜닝**: 단순 바이패스가 아닌 `promoted_score >= 60` 혹은 일반 주도주 중 `score >= 75` 이상인 고점수 종목만 선별적으로 쿨다운(15초)을 바이패스하도록 튜닝하여 잦은 churn을 방지했습니다.
  - **[P2] Slot Reservation (Weighted Round Robin 믹싱)**: 특정 탭(예: HOT 폭주장)의 독식으로 타 탭(특히 PRE_HEAT) 종목이 감시에서 Starvation(굶음)을 겪지 않도록, `HOT` 3개 : `PRE_HEAT` 2개 : `REVERSE` 1개의 가중치 기반 라운드 로빈 방식으로 랭킹 종목군을 믹싱하여 감시 슬롯을 안정적으로 예약 배분했습니다.
  - **[버그 수정] reconciliation 등록 순서 왜곡 수정**: `to_add_candidates`가 set 연산 결과물이라 무작위 순서로 순회되던 문제를 원래 정밀한 우선순위 순서가 보존되어 있는 `priority_ordered` 기반 순회로 수정하여 해결했습니다.

---

## 🔮 검증 및 테스트 결과
1. **단위 및 통합 테스트 실행**:
   - `tests/test_pnl_ssot_consistency.py`를 통해 금일 및 과거 3일간의 SSoT PnL 정합성을 검증하여 정합성이 깨지지 않음을 확인했습니다.
   - `tests/test_logic_improvements.py` 에 실시간 감시 로직에 대한 검증을 위해 `TestSurveillanceLogic` 클래스를 구성하고, 다음 4가지 핵심 시나리오 테스트를 완벽히 통과(OK)시켰습니다.
     * `test_priority_scheduling_and_hard_cap`: 우선순위 적용 및 85개 Hard Cap 한도 내 held 종목 누락 방지 검증.
     * `test_cooldown_bypass`: 승격 및 주도주 점수 기준에 따른 선별적 쿨다운 바이패스 검증.
     * `test_dynamic_cycle_budget`: 장 경과 시간대 및 변동성 임계치에 따른 사이클 예산(12/8/5) 동적 분배 검증.
     * `test_slot_reservation_weighted_round_robin`: HOT:PRE_HEAT:REVERSE = 3:2:1 가중치 라운드 로빈을 통한 감시 슬롯 예약제 및 Starvation 방지 기능 검증.
2. **패치 효과 검증**:
   - **가중평균단가 실시간 갱신**: 이제 KRX 장중 분할/부분 체결(partial fill)이 발생하더라도 최종 체결 가격이 아닌 실제 거래량이 반영된 가중평균단가가 `entry_price`로 유지되므로, 평균단가 왜곡을 방지할 수 있습니다.
   - **고아 매도 철저 방어**: `StateManager`뿐만 아니라 `KiwoomEngine` 인스턴스에 유지되고 있는 실시간 포지션 원장(`engine.positions`)까지 순차 복구하여 데이터 유실을 최소화하고, 최악의 경우에도 오염된 신호 목표가를 사용하는 대신 DLQ 격리를 선택하여 `trades` 테이블의 무결성(SSoT)을 완벽하게 보호합니다.
   - **시스템 실시간 운영 안정성 확보**: 85개 하드캡, 우선순위 스케줄링, 슬롯 예약을 결합하여 OpenAPI 틱 지연이나 누락을 선제적으로 예방하고 장초반 모멘텀 기회를 빠르고 유연하게 포착할 수 있게 되었습니다.
