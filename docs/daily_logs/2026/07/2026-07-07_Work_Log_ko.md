# 📅 2026-07-07 업무 일지 (Daily Work Log)

> **[거버넌스 준수]**
> * **작업 등급**: L3 (UI 설정 조립 안정화, DB 성능 최적화, 락 경합 및 쿨다운 안전 조치)
> * **참조 문서**:
>   * [main.py](file:///c:/Users/stone/projects/nocodequant/app/main.py)
>   * [main_window.py](file:///c:/Users/stone/projects/nocodequant/app/ui/main_window.py)
>   * [composition_root.py](file:///c:/Users/stone/projects/nocodequant/app/core/composition_root.py)
>   * [state_manager.py](file:///c:/Users/stone/projects/nocodequant/app/services/managers/state_manager.py)
>   * [persistence_manager.py](file:///c:/Users/stone/projects/nocodequant/app/services/managers/persistence_manager.py)
>   * [strategy_settings_dialog.py](file:///c:/Users/stone/projects/nocodequant/app/ui/dialogs/strategy_settings_dialog.py)
>   * [config_manager.py](file:///c:/Users/stone/projects/nocodequant/app/ui/managers/config_manager.py)
>   * [forensics_retention.py](file:///c:/Users/stone/projects/nocodequant/scripts/tools/forensics_retention.py)
> * **영향 받는 모듈 (Impacted Modules)**: `main.py`, `main_window.py`, `composition_root.py`, `state_manager.py`, `persistence_manager.py`, `strategy_settings_dialog.py`, `config_manager.py`, `forensics_retention.py` (신설), `test_order_lifecycle_races.py`
> * **불변성 위반 여부 (Invariant Check)**: 위반 없음 (Violated: None)
> * **경계 준수 선언**: UI 전략 설정 조립 과정 및 백엔드 지표 통신 상에서 발생할 수 있는 스레드 경합, 파일 입출력 손상, 파싱 예외 지점을 전면 방어하였습니다. 또한 StateManager RLock의 DB 쿼리 분리를 통해 스레드 락 프리 구조로 전환하고 SQLite NaN 직렬화로 인한 복구 오염 문제를 수정하는 등 HTS Reconciliation 안정성을 획기적으로 개선하였습니다.

---

## 🎯 주요 목표 및 이슈 정의

1. **설정 조립 경로(UI ↔ 엔진) 입력 방어 및 안전성 제고**: UI 스핀박스, 수량 입력란의 한글/한국 로케일 그룹 구분자(,) 삽입 시 float 파싱 에러로 장중 UI 프로세스가 크래시되어 기동이 멈추는 리스크 완화.
2. **ACTIVE_CONFIG 스레드 경합 차단**: 지표 계산 워커 스레드가 전역 설정 딕셔너리를 호출하는 시점에 클리어(clear) 및 재구성 경합이 발생하여 일시적으로 빈 데이터를 읽는 현상 해결.
3. **설정 파일 입출력 정합성 보완**: UTF-8 저장 및 CP949 로드 불일치 및 깨진 설정 파일에 의한 기동 불가 문제 해결.
4. **StateManager RLock 경합 해소**: HTS 체결 및 UI 동기화 루프가 RLock을 잡은 채 forensics.db를 조회(수백 ms 소요)하여 전체 스레드가 프리징되는 병목 해결.
5. **SQLite NaN 직렬화로 인한 복구 불가 오류 차단**: ADX 지표 등의 NaN이 SQLite에 RFC 표준 위반 형식으로 기록되어 `json_valid()=0` 판정으로 영구적으로 스캔이 실패되던 현상 해결.
6. **대용량 DB 성능 개선 및 디바이스 보존 스크립트 작성**: 4.7GB/265만 행 규모의 forensics.db 풀스캔 N+1 문제 해결 및 상시 retention 유지보수용 자동화 도구 도입.
7. **재진입 쿨다운 우회 경로 봉쇄**: TREND 청산 후 BREAKOUT으로 분류된 재진입이 쿨다운 감시망을 벗어나던 로직 홀 보완.

---

## ✅ 상세 작업 및 패치 내역

### 1. UI 설정 조립기 및 입력 파싱 방어 (V29.12)
- **strategy_settings_dialog.py / config_manager.py**:
  - AND/OR 진입 로직 및 RSI 매수/매도존이 엔진에 실질적으로 연동되지 않는 미연동 죽은 노브였음을 사용자 다이얼로그에 라벨 및 툴팁으로 명시.
  - rsi_buy/rsi_sell 설정 로드 시의 저장-복원 대칭 누락 수정 및 값 범위 제한(1~99) 명시.
  - `input_qty`에 QDoubleValidator C 로케일을 강제 바인딩하여 "1,000" 등 콤마 입력 시 float() 변환에서 생기는 ValueError 원천 차단.
  - UI 틱 콜백 경로 내 파싱은 예외 없이 안전하게 마지막 유효값 또는 1000% 상한으로 폴백하도록 `_parse_order_qty` 적용.
  - 전역 excepthook을 적용하여 예외 발생이 UI 프로세스를 즉사시키지 않도록 방어.
- **runtime_config.py**:
  - `initialize_runtime_config` 시 딕셔너리를 비우지 않고, 완성본 병합 후 원자적 교체하여 지표 계산 워커와의 경합창 폐쇄.
  - encoding='utf-8' 명시 입출력 및 손상 config 발생 시 `.corrupt-*.bak`으로 복구 백업 인프라 마련.

### 2. StateManager DB RLock 분리 및 성능 개선 (V29.11)
- **state_manager.py / test_order_lifecycle_races.py**:
  - `get_position_meta` 함수 내 RLock 내부에서는 메모리 상태 판독만 수행하고, 무거운 DB 조사기 기능은 락 외부로 이탈시켜 체결 스레드/UI 스레드와의 경합 제거.
  - `_deep_search_inflight` 플래그를 도입하여 동일 종목에 대한 중복 DB 조회를 억제.

### 3. NaN 직렬화 차단 및 SQLite 복구 기능 정상화
- **persistence_manager.py**:
  - `json.dumps(allow_nan=True)` 사용 시 ADX NaN 등의 값이 그대로 SQLite로 넘어가 `json_valid()` 에러를 유발하던 문제 해결.
  - `_dump_rfc_json` 헬퍼를 추가하여 NaN 값을 `null`로 사전에 변환 치환하여 직렬화하도록 구조 개선.

### 4. DB 레이어 성능 극대화 및 Retention 툴 배치 (V29.10)
- **persistence_manager.py / logger.py / forensics_retention.py (신규)**:
  - `get_last_signal_for_code` 조회 범위를 최근 5만 행으로 제한하여 PK 범위 스캔으로 0.6초 이내 완료 보증.
  - SQLite WAL+NORMAL 유지 모드로 커넥션을 상시 오픈하여 대량 트랜잭션 오버헤드 최소화.
  - `forensics_retention.py` 스크립트를 작성하여 BLOCKED 14일, GENERATED/SIGNAL 60일, 기타 영구 유지 정책을 백그라운드로 안전하게 스케줄링할 수 있게 함.

### 5. 차트 1봉 플래시 및 tick 렌더 지연 수정 (V29.9)
- **candle_manager.py / main_window.py**:
  - 종목 변경 후 비동기 데이터 로딩 중 tick이 df를 patch하여 1봉만 풀스케일 렌더링되던 문제 해결을 위해 `_chart_history_pending` 가드 적용.
  - 컬럼 벡터화 렌더러 `patch_last_bar`을 구현하여 tick 수신 시 rolling 지표를 메인 스레드 연산 없이 효율적으로 갱신.

---

## 🧪 검증 및 테스트 결과

### 1. 시나리오 및 회귀 검사
- `py_compile`을 사용하여 UI 조립기 및 DB 레이어 패치 대상 7개 파일 빌드 통과.
- `test_order_lifecycle_races.py`에 새로 추가된 RLock 해제 검증 및 자가치유 회귀 테스트 케이스 정상 통과 확인.
- `forensics_retention.py` 시뮬레이션을 통해 아카이브 행 개수 유효성 검증 실패 시 본체 미삭제 안전 기동 프로세스 검증 완료.

---

## 🔮 향후 계획 및 최종 의도

1. **상시 DB 용량 감시**: 장기 운행 시 `forensics_retention.py` 자동 배치를 통하여 일일 ~70MB 씩 늘어나는 DB 리소스를 1.5GB 미만으로 영구 억제 조치.
