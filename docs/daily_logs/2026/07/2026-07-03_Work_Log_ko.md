# 📅 2026-07-03 업무 일지 (Daily Work Log)

> **[거버넌스 준수]**
> * **작업 등급**: L3 (HTS Reconciliation 시스템 복원, 세법 동기화, UI 스냅샷 복구 및 휩소 분석)
> * **참조 문서**:
>   * [trade_analysis_report.md](file:///C:/Users/stone/.gemini/antigravity-ide/brain/182a2a6f-c8f0-4ac5-a48c-3116ebb871de/trade_analysis_report.md)
>   * [_engine_entry.py](file:///c:/Users/stone/projects/nocodequant/app/ai/_engine_entry.py)
>   * [config_engine.py](file:///c:/Users/stone/projects/nocodequant/app/core/config_engine.py)
>   * [composition_root.py](file:///c:/Users/stone/projects/nocodequant/app/core/composition_root.py)
>   * [execution_manager.py](file:///c:/Users/stone/projects/nocodequant/app/services/managers/execution_manager.py)
>   * [persistence_manager.py](file:///c:/Users/stone/projects/nocodequant/app/services/managers/persistence_manager.py)
>   * [state_manager.py](file:///c:/Users/stone/projects/nocodequant/app/services/managers/state_manager.py)
>   * [main_window.py](file:///c:/Users/stone/projects/nocodequant/app/ui/main_window.py)
> * **참고자료 (공개 저장소)**:
>   * [자동매매 집행 정책 및 가이드](https://github.com/KevinCY-Kim/nocodequant_open/blob/main/docs/Trading_Logic_Guide_ko.md)
>   * [AI 전략 최적화 및 튜닝 가이드 (공통)](https://github.com/KevinCY-Kim/nocodequant_open/blob/main/docs/POE_User_Guide_ko.md) / [AI 전략 최적화 및 튜닝 가이드 (튜닝)](https://github.com/KevinCY-Kim/nocodequant_open/blob/main/docs/AI_Tuning_User_Guide_ko.md)
>   * [전략 개선 성과 분석 보고서](https://github.com/KevinCY-Kim/nocodequant_open/blob/main/docs/strategy_performance_v24_2.md)
> * **영향 받는 모듈 (Impacted Modules)**: `_engine_entry.py`, `composition_root.py`, `execution_manager.py`, `persistence_manager.py`, `state_manager.py`, `main_window.py`
> * **불변성 위반 여부 (Invariant Check)**: 위반 없음 (Violated: None)
> * **경계 준수 선언**: 7/3 오늘 발생한 실거래 휩소(Churning) 원인 포렌식을 수행하였으며, 6개 소스코드 모듈에 걸친 `[V28.0]` HTS Server Reconciliation 동기화 및 UI Execution Snapshot 로깅 수정 내역을 검토하고 데일리로그에 완벽히 반영하였습니다. 백테스팅을 포함한 별도의 코드 제어/실행 행위는 유저 요청에 따라 전면 배제하였습니다.

---

## 🎯 주요 목표 및 이슈 정의

1. **약보합 장세 내 빈번한 매매 원인 규명**: 오늘 오전 9시 00분~9시 12분 사이 발생한 8건의 매매(누적 손익 약 -65만 원)가 발생하여, 보합/하락장에서 매매가 타이트하게 관리되지 않고 Churning(무한 재진입)이 발생한 이유 분석 및 쿨다운 설정 확인.
2. **`[V28.0]` HTS Server Reconciliation 패치 확인**: HTS의 실제 체결 내역과 로컬 엔진의 MDD, 잔고, 실현손익을 완전 동기화하고 누수를 방지하기 위한 패치 내역 교차 검증.
3. **UI Execution Snapshot 복구**: 차단된 신호(`SIGNAL_BLOCKED`) 및 발생된 신호(`SIGNAL_GENERATED`)의 상세 데이터 및 종목명을 UI/forensics 데이터베이스에 유실 없이 로깅하기 위한 직렬화 구조 개선 검증.
4. **세법 및 수수료 정합성 현실화**: 2025년 기준 KRX 세법 개정 세율(거래세+농특세 0.23%)과 실질 거래비용 추정 공식(0.245%) 동기화.

---

## ✅ 상세 작업 및 패치 내역

### 1. 매매 원인 정밀 분석 (forensics.db + trades.db)
- **현황 분석**:
  - `data/trades.db` 확인 결과, 금호건설(4회), 남화토건(2회), 계룡건설(1회), 삼성전자(1회) 매매 기록 확인.
  - 이 중 금호건설과 남화토건은 진입 후 1~2분 이내에 빠른 손절 청산이 발생했음에도 불구하고, 청산 직후 수십 초~1분 만에 동일 종목에 무차별적으로 재진입(Churning)하여 휩소 손실이 누적됨.
- **근본 원인 규명**:
  - **재진입 쿨다운 비활성화**: [config_engine.py](file:///c:/Users/stone/projects/nocodequant/app/core/config_engine.py#L246-L248) 내 `REENTRY_COOLDOWN_ENABLED = False` 상태로 6봉(30분)의 재진입 금지 장치가 작동하지 않음.
  - **국면 판정 오차 및 지수 필터 부재**: 개별 종목의 단기 수급 지표 튐 현상으로 인해 국면이 UPTREND로 오판(Threshold가 49로 하향)되었고, 종합 주가지수 연동 필터(KOSPI/KOSDAQ)는 "읽기 전용"으로 실 매매 결정에 반영되지 않아 방어 장치가 작동하지 못함.
- **조치 제안**:
  - `REENTRY_COOLDOWN_ENABLED = True` 즉각 변경 및 `ENTRY_LOCATION_HARDGATE_ENABLED = True` 설정을 대안으로 제시하는 [포렌식 분석 보고서](file:///C:/Users/stone/.gemini/antigravity-ide/brain/182a2a6f-c8f0-4ac5-a48c-3116ebb871de/trade_analysis_report.md)를 작성함.

### 2. `[V28.0]` HTS Server Reconciliation 및 세법 동기화
- **composition_root.py**:
  - `engine.sig_realized_pnl`과 `state_mgr.force_sync_pnl`의 시그널-슬롯 연결을 복원하여 실시간 HTS 실현손익을 StateManager에 즉각 전파.
- **state_manager.py**:
  - 실거래 계좌 동기화 시 증권사 평균 매입단가(`avg_price`)를 로컬 포지션 진입가(`entry_price`)로 최우선 적용(SSoT)하여 호가 갭 괴리 해결.
  - 2025~ KRX 세법 기준에 맞추어 실거래 Gross PnL 계산 시 증권거래세+농특세율을 기존 0.20%에서 **0.23%**로 조정.
- **persistence_manager.py**:
  - 모의/백테스트용 예상 수수료 계산(0.22% -> **0.245%**) 공식을 상향 현실화하여 실체결과의 차이 보정.
- **main_window.py**:
  - HTS P&L 실시간 동기화를 위해 정기 타이머 루프에 `request_realized_pnl()` 호출을 추가하여 주기적으로 증권사 데이터를 연동.
  - 일일 P&L 요약 표시 로직(`update_realized_pnl` 등)이 캐싱 DB 대신 `state_mgr`의 실시간 HTS 동기화 통계 필드를 일괄 참조하게 변경.

### 3. UI Execution Snapshot 로깅 구조 개선
- **_engine_entry.py**:
  - `SIGNAL_BLOCKED` 비동기 DB 로깅 시 `StateManager`를 사용해서 실시간 종목명(`stock_name`)을 주입하고, 구조화된 `blocked_snapshot` 정보를 `forensics.db`에 저장하도록 함.
- **execution_manager.py**:
  - `SIGNAL_GENERATED` 이벤트 발생 시 분석 지표 인스턴스를 안전하게 재귀적으로 직렬화(`_serialize_res`)하여 `snapshot` 및 `signal` 페이로드로 온전히 저장.
- **persistence_manager.py**:
  - `FORENSICS_ONLY_TYPES`에 `SIGNAL_BLOCKED`를 추가하여 `trades.db` 오염 없이 `forensics.db` 로그로만 깔끔히 격리 저장.
  - 이전에 payload 딕셔너리가 이중으로 중첩되는 역직렬화 헬퍼 버그를 완화하여 nested parsing이 가능하도록 복구.

---

## 🧪 검증 및 테스트 결과

- 유저의 백테스팅 진행(클로드 단독 진행 중) 요청에 따라, 로컬 환경에서의 추가 백테스팅 및 코드 수정 일체를 진행하지 않고 정적 변경 사항 확인으로 검증을 갈음함.

---

## 🔮 향후 계획 및 최종 의도

1. **설정값 활성화**: 유저 컨펌 후 `REENTRY_COOLDOWN_ENABLED = True` 반영 및 다음 영업일 휩소 진입 차단 현황 모니터링.
2. **지수 가드(Market Index Guard) 연동 고려**: 중장기적으로 `EntryEngine`에 실제 종합지수 가드를 결합하여 보합/하락 장세에서의 매수 허들 자동 조절 설계.
