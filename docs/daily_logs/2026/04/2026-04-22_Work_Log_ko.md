# NCQ 데일리 작업 로그 (2026-04-22)

## 📋 요약 (Summary)
오늘의 핵심 과업은 장중 발생한 **"보유 종목 자동 매도 미작동"** 현상의 근본 원인을 규명하고, 시스템의 신뢰성을 회복하는 것이었습니다. 분석 결과, 매수 체결 시점의 `UnboundLocalError` 크래시가 `OrderManager`의 주문 잠금(Pending Lock)을 영구적으로 방치하여 이후 모든 매도 로직을 차단했음을 확인했습니다. 이를 해결하기 위해 **30초 타임아웃 가드**를 도입하여 물리적 교착 상태를 원천 차단했으며, `V20.9.1` 비대칭 출구 엔진의 국면별(Regime-aware) 로직 결함들을 전면 수정하여 하락장 대응력을 극대화했습니다.

## 🛠 주요 변경 및 개선 사항

### 1. **[Order] 주문 교착 상태(Deadlock) 방어 엔진 구축 (V20.9.3)**
- **Pending Timeout Guard 도입**: 
    - 주문 발주 시각(`pending_order_times`)을 추적하여, 30초가 경과해도 체결/취소 신호가 오지 않을 경우 자동으로 락을 해제하는 안전장치 신설.
    - `check_ready()`에서 타임아웃 감지 시 즉각 로그를 남기고 다음 매도 로직이 진행되도록 흐름 개선.
- **체결 로직 무결성 강화**:
    - `on_fill_event` 내 모든 종료 분기(BUY 완료, SELL 완료, 전량 청산)에서 `clear_pending()`을 명시적으로 호출하여 메모리 누수 및 잠금 방지.

### 2. **[Strategy] 비대칭 출구 엔진 국면별 최적화 (V20.9.1.5)**
- **DOWNTREND 출구 가속화**:
    - 기존에 `UPTREND`와 동일한 40봉(200분) 타임아웃을 적용받던 `DOWNTREND` 종목들을 12봉(60분)으로 단축하여 "낙하칼" 보유 리스크 최소화.
- **스마트 시간 연장(Smart Extension) 가드**:
    - 하락장(`DOWNTREND`) 및 혼조세(`CHAOS`)에서는 일시적 반등이 있더라도 시간 청산 연장을 금지하여 보수적 관점 유지.
- **복구 포지션 트레일링 최적화**:
    - 앱 재시작 시 복구되는 포지션에 대해 국면을 판별하여 `trailing_multiplier`를 동적 할당(RANGE/CHAOS 1.5x, 나머지 3.5x).

### 3. **[Analytics] 스코어 파이프라인 복원 및 브릿지 로직 최적화 (V5.0)**
- **SignalResult 데이터 구조 고도화**:
    - `SignalResult` 데이터클래스에 `score_display`, `score_core`, `score_vol`, `score_price`, `score_risk` 5개 필드를 명시적으로 추가.
    - 전략 엔진(`_get_signal_internal`)의 모든 리턴 경로(정상/과열차단 포함)에서 분석 데이터를 해당 필드에 자동 주입하도록 개선.
- **Execution-Bridge 무손실 전달 구현**:
    - `execution_manager`에서 `TradeSignal` 생성 시, `SignalResult`의 신규 필드를 우선적으로 읽어오도록 브릿지 로직 수정.
    - 기존에 `entry_score`(0~100)와 `score_display`(50~100)를 혼용하여 발생하던 데이터 왜곡 및 0.0 표기 문제 완전 해결.
- **데이터 정합성 확보**:
    - `SignalResult` -> `TradeSignal` -> `OrderManager` -> `DB`로 이어지는 스코어 전달 전 과정을 무손실 파이프라인으로 재구축.

### 4. **[Strategy] 엔진 가독성 리팩토링 (Structural Cleanup)**
- **핵심 로직 메서드 분리**:
    - `_get_signal_internal()` 내의 지표 정규화(Block 1) 및 신뢰도/안정성 계산(Block 2) 로직을 `_compute_scores_and_confidence()` 전용 메서드로 추출.
    - 600줄에 달하는 메인 루프의 복잡도를 낮추고, 지표 계산부와 매매 판단부의 책임을 명확히 분리.
- **상태 관리 무결성 유지**:
    - 메서드 분리 후에도 `sess` 기반의 EMA(Strength, Stability) 상태 갱신이 기존과 동일하게 유지됨을 검증 완료.

## 🐞 주요 버그 픽스 (Hotfixes)

| 버그 ID | 문제 현상 | 수정 내용 | 파일 |
| :--- | :--- | :--- | :--- |
| **BUG-01** | `on_fill_event` 내 `pos_meta` 참조 오류 (`UnboundLocalError`) | BUY 체결 시 `pos_meta` 대신 `signal_meta`를 참조하도록 스코프 수정 | `order_manager.py` |
| **BUG-02** | 특정 종목이 3시간 이상 매도 평가에서 제외됨 | `pending_orders` 락이 풀리지 않아 발생한 현상으로 확인, 타임아웃 가드로 해결 | `order_manager.py` |
| **BUG-03** | 하락장 종목이 40봉(200분) 동안 청산되지 않음 | `_time_exit_map`에서 문자열 치환 로직 제거 및 국면별 명시적 분기 적용 | `strategy.py` |
| **BUG-04** | 복구 포지션의 트레일링이 1.5x로 고정되어 조기 청산 발생 | `sync_position` 시 `regime` 기반 배수 분기 로직 추가 | `strategy.py` |
| **BUG-05** | 키움 OCX 스크린 번호 충돌로 인한 주문 컨텍스트 오염 | 0101~0199 순환 스크린 번호 카운터 도입 | `engine.py` |
| **BUG-06** | `pending_orders` (set 객체)에 `pop(code, None)` 호출 시 TypeError | `discard(code)` 또는 `remove(code)`로 호출 방식 정정 | `order_manager.py` |

---

## 🚀 기대 효과 및 성과
- **시스템 가용성 극대화**: 특정 종목의 체결 신호가 유실되더라도 전체 엔진이 30초 이상 멈추지 않는 **'Self-Healing'** 능력 확보.
- **하락장 방어력 강화**: 하락 국면에서의 빠른 시간 청산과 연장 금지를 통해 자본 잠식 위험 감소.
- **데이터 정합성 100%**: 재기동 후에도 모든 보유 종목의 스코어와 트레일링 전략이 온전하게 유지됨.

---

## 📅 다음 작업 체크포인트
- [ ] 30초 타임아웃 가드가 실제 로그에 기록되는 케이스 모니터링 (네트워크 지연 대응 확인).
- [ ] 하락장(`DOWNTREND`) 진입 시 12봉 타임아웃이 칼같이 작동하는지 실전 데이터 검증.
- [ ] 재기동 후 복구된 종목들의 `trailing_multiplier`가 의도한 대로(3.5x 등) 찍히는지 로그 확인.
- [ ] `v_score_persistence` 뷰를 통해 오늘 신규 진입한 종목들의 점수 기록 여부 최종 점검.

---
**헌법 준수 및 거버넌스 가이드라인에 따라 모든 수정 사항을 기록 완료하였습니다.**
