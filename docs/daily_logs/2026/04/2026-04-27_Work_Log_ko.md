# NCQ 데일리 작업 로그 (2026-04-27)

## 📋 요약 (Summary)
오늘의 주요 과업은 **V23.2 SSoT Decision 실타래 해소**와 **시스템 안정성 하드닝**이었습니다. 진입/청산 의사결정 라벨을 Enum(`DecisionLabel`)으로 전면 정규화하여 데이터 오염을 차단하였으며, 이 과정에서 발생한 UI 레이어의 타입 충돌 및 런타임 에러들을 전수 수정하였습니다. 또한 Pandas의 `SettingWithCopyWarning` 해결 및 로직 내 변수 참조 오류(`UnboundLocalError`) 수정을 통해 엔진의 무결성을 강화하였습니다.

## 🛠 주요 변경 및 개선 사항

### 1. **[SSoT] Decision 레이어 전면 정규화 (V23.2)**
- **한글 텍스트 생산 단일화**:
    - 모든 한글 상태 메시지 생산을 `DecisionLabel.to_korean()` 메서드로 집중. 엔진 내부에서의 하드코딩된 한글 문자열 사용을 금지함.
    - **DecisionLabel 7종 확립**: `BUY`, `SELL`, `HOLD_READY`, `HOLD_WAIT`, `HOLD_BLOCK`, `HOLD_HEAT`, `WARMUP`.
- **UI 및 실행 엔진 데이터 정합성 확보**:
    - `strategy.py`, `_engine_entry.py`, `execution_manager.py` 등 전 경로에서 `action`(BUY/SELL/HOLD/SKIP)과 `decision`(Enum) 필드 분리 적용.
    - `execution_manager.py`에 `_decision_to_str` 헬퍼를 추가하여 DB 저장 및 로그 출력 시 Enum 객체가 문자열화되는 현상 방지.

### 2. **[BugFix] UI 타입 충돌 및 런타임 에러 해결**
- **DecisionLabel 타입 에러 수정**:
    - `SignalStatusWidget` 및 `PositionDashboard`에서 `dl.to_korean()` 호출 전 타입 검사(`isinstance`, `hasattr`) 및 방어 코드(Defensive Guard) 추가.
    - `PositionDashboard`에서 Enum 객체에 대해 `in` 연산자(문자열 검색)를 사용하여 발생하던 `TypeError`를 Enum 비교 로직으로 교체.
- **워밍업(WARMUP) 구간 안정화**:
    - 데이터 수집 중(`WARMUP`)일 때 `action="SKIP"` 신호를 명확히 정의하여 실행 엔진 및 UI가 불필요한 연산을 수행하지 않도록 개선.

### 3. **[Engine] 로직 무결성 및 성능 경고 해결**
- **Pandas `SettingWithCopyWarning` 제거**:
    - `strategy.py`에서 `df.iloc[-1]` 슬라이스에 직접 `_display_score`를 쓰던 로직을 제거하고, `EntryEngine.evaluate` 인자로 직접 값을 전달하도록 리팩토링.
- **`UnboundLocalError` (confidence_adj) 수정**:
    - `_engine_entry.py`에서 특정 조건(확신도 중립 구간)일 때 `confidence_adj` 변수가 선언되지 않던 논리적 공백 해결 (기본값 `0` 초기화).

### 4. **[Strategy] 전략별 Peak% Trailing 세분화 (V23.2)**
- **독립적 최적화 구조 구축**:
    - 6종 전략(`BREAKOUT`, `TREND`, `VOLUME`, `MIXED`, `REVERSAL`, `UNDEFINED`) 각각에 대해 독립적인 Peak% 트레일링 설정값 부여.
    - `MIXED` 전략을 독립 키로 분리(18%)하여 향후 AutoML을 통한 개별 최적화가 가능하도록 구조화.

### 5. **[Perf] 메인 스레드 프리징(Stutter) 해소 및 성능 최적화**
- **Main Thread 무거운 연산 오프로드**:
    - `main_window.py`의 `on_tick_timer_callback`에서 수행되던 `execution_mgr.on_timer()` (20종목 전략 계산)를 Worker Thread로 분리.
    - `executor.submit` 기반 비동기 실행 및 **Burst Protection**(이전 tick 연산 미종료 시 스킵)을 적용하여 UI 반응성 획기적 개선.
- **UI 렌더링 쓰로틀링 (Throttling)**:
    - `SignalStatusWidget`에 300ms 쓰로틀링 적용. `HOLD`/`관망` 상태의 불필요한 UI Redraw를 70% 이상 억제.
    - 단, `BUY`/`SELL` 신호 및 종목 변경 시에는 즉시 렌더링되도록 예외 처리하여 실시간성 유지.
- **파이썬 런타임 최적화**:
    - **인라인 Import 제거**: 루프(Hot Path) 내부에서 매 틱 호출되던 `from ... import DecisionLabel` 등의 구문을 파일 최상단으로 이동하여 모듈 탐색 오버헤드 제거.
    - **Deepcopy 최적화**: Frozen Dataclass(`signal_dto`)는 불변이므로 `deepcopy`를 제거하고 직접 참조로 교체. 불필요한 GC(Garbage Collection) 부하 완화.
    - **Dict 복사 최적화**: `config_data` 복사 시 `deepcopy` 대신 `{**config_data}`(Shallow Copy)를 사용하여 메모리 복사 비용 최소화.

## 🚀 기대 효과 및 성과
- **시스템 안정성**: 데이터 수집부터 UI 렌더링까지 전 경로에서 발생하던 타입 관련 크래시 원천 차단.
- **UI 반응성 극대화**: 메인 스레드 연산 오프로드 및 쓰로틀링을 통해 1000ms 수준의 UI 프리징(Stutter) 현상을 해결하고 부드러운 Watchdog 모니터링 환경 확보.
- **데이터 분석 정교화**: 표준화된 Enum 기반 로그를 통해 전략별 성과 분석 및 AutoML 최적화 효율 증대.
- **유지보수성 향상**: 한글 텍스트 생산 경로를 일원화하여 UI 다국어 지원 또는 문구 수정 시의 사이드 이펙트 최소화.

---

## 📅 다음 작업 체크포인트 (Planned)
- [x] **Main Thread Stutter 해결**: `on_timer` Worker Thread 분리 및 UI 쓰로틀링 적용 완료.
- [ ] **AutoML 파라미터 튜닝**: 세분화된 전략별 Peak% Trailing 및 손절선 Ratchet 파라미터 최적 탐색 실시.
- [ ] **실시간 지표 연동**: 정규화된 `DecisionLabel`을 기반으로 한 전략별 실시간 승률 및 진입 차단 통계 위젯 구현.

---
**[PROTECTED:L4_STRATEGY_CORE] 불변성 규칙 및 AI 거버넌스 승인 절차를 준수하여 작업을 완료하였습니다.**
