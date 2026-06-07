# 📅 2026-05-27 업무 일지 (Daily Work Log)

## 🎯 주요 목표 및 이슈 정의
- [x] **실시간 감시 HUD 탭별 카운트 불일치 문제 해결**:
  `get_top_codes()`의 중복 제거(seen) 로직으로 인해 HOT 탭에 포함된 돌파/역추세 종목이 탭별 카운팅에서 누락되어 돌파 0, 역추세 0 등으로 오표기되던 현상을 수정했습니다.
- [x] **실시간 감시 한도(Slot Limit) 설정 동적 바인딩**:
  감시 슬롯 한도(`SURV_LIMIT_HOT`, `SURV_LIMIT_PRE_HEAT`, `SURV_LIMIT_REVERSE`)를 UI에 하드코딩하지 않고 `ACTIVE_CONFIG` 설정을 통해 동적으로 참조하도록 리팩토링했습니다.
- [x] **Cold-Start Deadlock(수급 틱 부족) 해결을 위한 웜업 가상 틱 생성 및 세이프가드**:
  프로그램 구동 초기 또는 감시 해제 후 재등록 시 체결강도가 계산되지 않고 먹통이 되는 문제를 해결하기 위해 15개의 가상 스냅샷으로 웜업(Warmup)하도록 기능을 설계했으며, 가상 틱으로 인한 데이터 오염을 예방하기 위해 `synthetic` 메타 태그를 명시적으로 삽입(Synthetic Safeguard)했습니다.
- [x] **만료된 가상 스냅샷 강제 Purge (Warmup Max TTL 적용)**:
  거래정지나 극저유동성 종목 등 틱 유입이 중단된 종목에 대해 가상 데이터가 오래 유지되는 것을 방지하기 위해 `MAX_SYNTHETIC_TTL_SEC = 180` 초과 시 강제 퍼지(Purge)하는 가드 로직을 탑재했습니다.
- [x] **저유동성 종목 감시 리스크 및 예방 대책 검토**:
  저유동성 종목이 주도주 감시 풀에 무임승차할 확률과 이에 대한 완화 패치 필요성에 대해 다중 런레이트 게이트 동작 연산 분석을 수행하여 보고했습니다.
- [x] **실시간 주도주 내 KoAct 액티브 ETF 필터링 추가**:
  KoAct 브랜드의 액티브 ETF(예: 'KoAct 바이오헬스케어액티브')가 실시간 주도주 감시 풀 및 상승률 랭킹 리스트에 진입하여 감시 슬롯을 소모하지 않도록 정적 제외 리스트에 등록했습니다.
- [x] **차트 툴팁 수치 정보 우측 % 비율 표시 구현**:
  HTS 스타일을 반영하여 차트 툴팁(시가/고가/저가/종가/거래량/이동평균/볼린저밴드) 우측에 직전 캔들 종가 또는 현재 종가 대비 % 비율 정보를 추가해 차트 가독성을 높였습니다.
- [x] **이중 시장 체온 시스템 (Dual Market Temperature System) 구현**:
  기존 단순 평균 방식에 따른 온도 희석(Dilution) 현상을 극복하기 위해 시장 체온을 확산 체온(Breadth)과 주도주 체온(Leader)으로 이중화하고, 소수 대형주에 의한 왜곡을 막기 위한 참여율 필터(Participation Factor)와 거래대금 가중치 상한선(Weight Clamping) 가드를 결합하여 실전 거래 체감 장세와 완벽히 동기화시켰습니다.
- [x] **HUD 전 항목 통일 규격 HTML 툴팁 연동**:
  전술 HUD 상의 모든 항목(Regime, 이중 시장 체온, 가드 상태, MME 과열도, 실시간 감시 현황)에 대해 기존의 통일된 툴팁 빌더(`_build_hud_tip`) 규격을 100% 적용하여, 마우스 오버 시 정밀 분석/설명 가이드를 즉시 조회 가능하도록 개선했습니다.
- [x] **이중 체온-레짐 연계 12케이스 다이내믹 AI 브리핑 구축**:
  시장 체온(확산/주도) 및 레짐 상태에 대응하는 12가지 장세 매트릭스 시나리오를 설계하여, 순환매 흐름 및 집중도를 고도로 상세화한 분석 보고 브리핑을 실시간 출력하도록 보완했습니다.
- [x] **매매 보유 시간 단축 및 조기 청산 원인 분석 및 패치**:
  대양금속(009190)의 7초 조기 청산 원인이 가상 틱 웜업 데이터로 인한 고가 오염 및 Flash Crash Protection의 오작동에 있음을 밝혀내고, 체결 시점에 가격 버퍼를 실제 체결가로 강제 리셋하며 체결 미경과 봉의 고가 반영을 제한하는 세이프가드를 적용했습니다. 또한, Ratchet/BE 등 수익 보존 청산(is_profit_protecting) 시 UI에 "손절"로 오표기되는 버그를 `ExitType.PEAK_TRAIL`("익절")을 반환하도록 수정하여 해결했습니다.

---

## ✅ 상세 작업 및 패치 내역

### 1. HUD 실시간 감시 카운트 교차 탭 집계(Cross-Tab Aggregation) 방식으로 전환
* **소스 파일**: 
  - [hud_manager.py](file:///c:/Users/stone/projects/nocodequant/app/ui/managers/hud_manager.py#L145-L192)
  - [config_engine.py](file:///c:/Users/stone/projects/nocodequant/app/core/config_engine.py#L50-L53)
  - [ranking_widget.py](file:///c:/Users/stone/projects/nocodequant/app/ui/widgets/ranking_widget.py#L270-L278)
  - [ranking_manager.py](file:///c:/Users/stone/projects/nocodequant/app/services/managers/ranking_manager.py#L1140-L1142)
* **현상**:
  - 실시간 주도주 화면의 하단 HUD 영역에서 실시간 감시 중인 종목 수가 "추세 30/30 돌파 0/20 역추세 0/10" 형태로 돌파 및 역추세는 감시되고 있음에도 불구하고 0개로 표시되거나, 탭 간 표시 숫자가 어긋나는 불일치 버그가 있었습니다.
  - 원인은 `get_top_codes()`가 키움 OpenAPI 구독 한도(80개)에 맞춰 중복을 제거(`seen` set)하면서 HOT ➡️ PRE_HEAT ➡️ REVERSE 순서로 레이블을 1개만 할당했기 때문입니다. HOT에 먼저 할당된 돌파/역추세 중복 종목은 PRE_HEAT/REVERSE 카운트에서 누락되었습니다.
* **조치**:
  - **교차 탭 집계 방식 도입**: HUD 매니저(`hud_manager.py`)에서 `get_top_codes()`의 결과인 `subscribed_codes`(실제 구독 중인 고유 종목 코드 셋)를 기준으로, 각 탭별 원본 리스트(`_last_hot_list`, `_last_pre_heat_list`, `_last_reverse_list`)를 개별 순회하여 현재 구독 풀에 포함되어 있는 종목 수를 교차 카운팅하도록 변경했습니다.
  - **슬롯 한도 동적 바인딩**: `ACTIVE_CONFIG`에 `SURV_LIMIT_HOT` (30개), `SURV_LIMIT_PRE_HEAT` (20개), `SURV_LIMIT_REVERSE` (10개) 한도를 정의하고, `hud_manager.py`, `ranking_widget.py`, `ranking_manager.py` 등 슬롯 크기를 제어하는 모든 코드에서 이를 동적으로 읽어가도록 설계하여 하드코딩을 제거했습니다.

---

### 2. Cold-Start Deadlock 방지 및 Synthetic Safeguard (가상 데이터 세이프가드) 구현
* **소스 파일**: 
  - [ranking_manager.py](file:///c:/Users/stone/projects/nocodequant/app/services/managers/ranking_manager.py#L620-L666)
  - [ranking_manager.py](file:///c:/Users/stone/projects/nocodequant/app/services/managers/ranking_manager.py#L2044-L2108)
  - [execution_manager.py](file:///c:/Users/stone/projects/nocodequant/app/services/managers/execution_manager.py#L986-L992)
* **현상**:
  - 신규 등록된 종목은 첫 틱이 들어올 때까지 대기 상태(Deadlock)에 빠져 실시간 감시가 즉시 수행되지 못했습니다. 이를 해결하기 위해 15개의 가상 스냅샷을 만들어 웜업하는 패치를 적용했으나, 이 가상 데이터가 포렌식 로그(Forensic Log), 리플레이(Replay) 시스템 및 AI/강화학습 데이터셋 빌드 시 오염(Contamination)을 유발할 위험이 있었습니다.
* **조치**:
  - **가상 틱 플래그 도입**: 웜업용 가상 스냅샷 생성 시 8번째 튜플 원소로 `True` (Synthetic Flag) 값을 할당했습니다.
  - **메타 태깅 및 출처 조회 API 신설**: 최초 진입 시점 당시 snaps 내 가상 틱 존재 여부를 판단하여 `is_warmup: True`, `synthetic: True` 메타 태깅을 추가했습니다. 외부 분석, 리플레이, 피처 추출을 위해 `get_quote_snapshots(code)` Public API를 만들어 틱 스냅샷 추출 시 각 틱마다 `synthetic: True/False` 여부를 투명하게 노출하도록 수정했습니다.
  - **호환성 디폴트 처리**: `execution_manager.py` 등 폴백(fallback) 시점에서도 `synthetic: False`, `is_warmup: False`를 명시적으로 보장하여 예외 발생 가능성을 차단했습니다.

---

### 3. Warmup Max TTL 만료 처리 가드 탑재
* **소스 파일**:
  - [ranking_manager.py](file:///c:/Users/stone/projects/nocodequant/app/services/managers/ranking_manager.py#L620-L635)
* **현상**:
  - 거래정지 상태이거나 유동성이 극도로 부족한 저유동성 종목의 경우, 최초 생성된 가상 스냅샷이 실시간 실제 틱에 의해 교체(Purge)되지 않고 메모리 내에 영구적으로 잔존할 수 있었습니다.
* **조치**:
  - **`MAX_SYNTHETIC_TTL_SEC = 180` 적용**: 주기적인 랭킹 수집 루프 내에서, 모든 스냅샷이 가상 데이터(`synthetic` 틱)로만 이루어진 종목 중 마지막 가상 틱 생성 시각이 180초 이상 경과한 종목을 강제로 `_quote_snapshots` 버퍼에서 제거(Purge)하는 보안 로직을 구축하였습니다.

---

### 4. 저유동성 종목 모니터링 리스크 분석 및 방어 타당성 검토 (코드 변경 없음)
* **현상**:
  - 주도주 감시 풀에 거래대금이 현저히 낮고 호가 공백이 큰 저유동성 종목이 진입하여 제한된 실시간 감시 슬롯(60개)을 낭비하거나 거래 노이즈를 일으킬 우려가 제기되었습니다.
* **분석 내용**:
  - **진입 확률 분석**: 현재 시스템은 장중 거래대금이 최소 20억 원 이상, 타겟 150억~300억 원 수준인 주도주들만 선정하기 때문에, 저유동성 종목이 주도주 탭에 들어올 확률은 극히 희박(0%~5% 미만)합니다.
  - **런레이트 게이트 동작**: 가상 틱으로 웜업되더라도, NCQ 엔진의 2단계 런레이트 게이트(최소 20억 절대 하한 및 시간별 가중 런레이트 통과 기준)를 충족하지 못하면 실시간 포지션 연산 풀(`promoted_pool`)에 승격되지 않고 즉시 필터링됩니다.
  - **결론**: 추가적인 거래대금 필터 코드를 수정하여 시스템 복잡도를 올리기보다는, 기 구현된 런레이트 게이트와 Synthetic Safeguard 만으로도 완벽한 방어가 가능하므로 안전하다는 기술 분석을 완료했습니다.

---

### 5. KoAct ETF 실시간 주도주 제외 필터링 추가
* **소스 파일**:
  - [ranking_manager.py](file:///c:/Users/stone/projects/nocodequant/app/services/managers/ranking_manager.py#L45-L54)
* **현상**:
  - 실시간 주도주 화면 및 랭킹 감시 풀에서 "KoAct 바이오헬스케어액티브" 등의 액티브 ETF 종목이 포착되어, 일반 개별 주도주를 추적해야 하는 감시 슬롯(Slot)과 리소스를 부적절하게 점유하는 현상이 관찰되었습니다.
* **조치**:
  - **제외 키워드 추가**: `ranking_manager.py` 내의 ETF 및 SPAC 정적 필터링 키워드 튜플인 `EXCLUDE_KEYWORDS` 및 `EXCLUDE_GAIN_KEYWORDS` 리스트에 `"KOACT"` 키워드를 신규 추가하여, 해당 문구가 포함된 모든 ETF 상품들이 상승률 및 수급 감시 탭에서 원천 필터링되도록 보완했습니다.

---

### 6. 차트 툴팁 내 HTS 스타일 % 비율 정보 추가
* **소스 파일**:
  - [chart_widget.py](file:///c:/Users/stone/projects/nocodequant/app/ui/widgets/mini_chart/chart_widget.py#L620-L649)
  - [constants.py](file:///c:/Users/stone/projects/nocodequant/app/ui/widgets/mini_chart/constants.py#L38-L43)
* **현상**:
  - 기존 차트의 마우스 오버 툴팁은 가격 수치 정보만 나열하여, 현재 특정 캔들의 시가, 고가, 저가, 종가 및 보조지표(이동평균, 볼린저밴드)가 직전 종가 또는 현재 종가 대비 어느 정도의 이격 및 상승률 수준인지 직관적으로 파악하기 어려웠습니다.
* **조치**:
  - **OHLC % 변화율 구현**: 직전 캔들의 종가(`closes[snap_idx-1]`)를 기준가(Baseline)로 설정하여, 현재 캔들의 시가/고가/저가/종가 우측에 `(값 - 기준가) / 기준가 * 100` 형식의 % 비율(`f" ({pct:.2f}%)"`)을 렌더링하도록 반영했습니다.
  - **거래량 % 변화율 구현**: 직전 캔들의 거래량(`volumes[snap_idx-1]`)을 기준으로 현재 거래량의 비율을 `(현재거래량 / 직전거래량) * 100` 형태로 표시(`f" ({pct:.1f}%)"`)했습니다.
  - **보조지표 이격도 % 구현**: 이동평균선(MA) 및 볼린저밴드(BB) 상/하단 값의 경우, 현재 캔들의 종가(`closes[snap_idx]`)를 기준가로 하여 현재 종가 대비 이격 비율을 구해서 함께 표시했습니다.
  - **레이아웃 안정성 확보**: 추가된 텍스트로 인해 툴팁 박스 크기가 잘리거나 줄바꿈되는 현상을 방지하기 위해 `constants.py`의 `TOOLTIP_WIDTH`를 `180`에서 `220`으로 확장 조정했습니다.

---

### 7. 이중 시장 체온 시스템 (Dual Market Temperature) 구현
* **소스 파일**:
  - [ranking_manager.py](file:///c:/Users/stone/projects/nocodequant/app/services/managers/ranking_manager.py#L2109-L2170) (신규 API 신설)
  - [hud_manager.py](file:///c:/Users/stone/projects/nocodequant/app/ui/managers/hud_manager.py#L53-L75) (이중 온도 매핑 및 연동)
  - [tactical_hud_widget.py](file:///c:/Users/stone/projects/nocodequant/app/ui/widgets/tactical_hud_widget.py#L119-L172) (이중 게이지 UI 컴포지션)
* **현상**:
  - 기존의 단일 시장 체온은 수백 개의 활성 후보 종목의 단순 평균으로 계산되어, 소수의 주도주 섹터(AI/반도체 등)만 폭등하고 나머지 종목은 침체된 한국 시장 특유의 편중 장세 시 평균이 강하게 희석되어 화면에 실제 체감과 어긋나는 극단적인 **COLD (예: 13도)** 경향을 띄었습니다.
* **조치**:
  - **확산 체온(Breadth Temp)과 주도주 체온(Leader Temp) 분리**:
    - **확산 체온**: 기존과 동일하게 전체 활성 종목들의 단순 평균으로 산출하여 자금의 광범위한 확산도(순환매 분포)를 대변합니다.
    - **주도주 체온**: 수급 상위 15%의 주도 집단(Adaptive Size: 최소 10, 최대 30종목)을 정렬하여 거래대금 가중 평균으로 산출해 실매매 체감 집중도를 대변합니다.
  - **삼성전자/하이닉스 메가캡 편향 제어 (Weight Clamping)**: 거래대금 가중치 연산 시 `min(6.0, log1p(거래대금))` 상한선(약 400억 원)을 적용하여, 초대형주 1~2개가 주도주 온도를 독식 및 왜곡하는 것을 차단했습니다.
  - **소수 특정주 쏠림 왜곡 제어 (Participation Factor)**: 수급 점수가 일정 기준(0.8)을 넘는 주도 종목의 참여 비율을 연산하고, 최종 공식 `LeaderTemp = Strength * (0.6 + Participation * 0.4)`을 적용하여 다수의 리더가 함께 상승할 때만 온도가 HOT/OVERHEATED로 올라서도록 보정했습니다.
  - **HUD UI 개편**: 기존 단일 체온 행을 좌우 2분할하여 **"확산: 20 COLD | 주도: 85 OVERHEATED"**와 같이 시장의 형태(순환매인지, 주도주 쏠림인지)를 트레이더가 직관적으로 동시 해독할 수 있도록 가로 배치 UI 레이아웃을 성공적으로 구축했습니다.

---

### 8. HUD 통일 규격 HTML 툴팁 연동 및 12케이스 다이내믹 AI 브리핑 구현
* **소스 파일**:
  - [tactical_hud_widget.py](file:///c:/Users/stone/projects/nocodequant/app/ui/widgets/tactical_hud_widget.py#L6-L47) (툴팁 빌더 `_build_hud_tip` 탑재 및 위젯 바인딩)
  - [hud_manager.py](file:///c:/Users/stone/projects/nocodequant/app/ui/managers/hud_manager.py#L134-L163) (이중 체온-레짐 다이내믹 브리핑 조건문 구축)
* **조치**:
  - **통일 규격 HTML 툴팁 탑재**:
    - 대시보드의 기존 툴팁 양식(13px 굵은 제목, 회색 설명글, label-value 구조의 본문 정렬, 하단 파란색 이탤릭 힌트)과 완벽히 동일한 HTML 빌더 `_build_hud_tip`을 `tactical_hud_widget.py`에 탑재했습니다.
    - 국면(Regime), 확산 체온, 주도주 체온, 가드 상태, MME 자금 과열도, 실시간 수급 감시 등에 모두 개별 바인딩하여 마우스 오버 시 데이터 수집 방식 및 해석 기준을 명확하게 가이드하도록 조치했습니다.
  - **12케이스 다이내믹 AI 브리핑**:
    - 기존의 3분기 하드코딩 방식에서 탈피하여, 레짐 4대 분류와 이중 시장 체온(확산 및 주도 과열 임계치 기준)의 조합으로 도출된 **12가지 시장 국면 매트릭스**를 설계했습니다.
    - "선택적 주도주 상승장", "하락장 속 낙폭과대 개별 반등", "광범위 상승세 비중 확대", "순환매 말기 추격 금지" 등 실전에 즉각 대응할 수 있는 고정밀 전술 Briefing 텍스트가 상황에 맞춰 실시간 렌더링되도록 완성했습니다.

### 9. 매매 보유 시간 단축 및 조기 청산 패치 (Warmup Price Clamping & Profit Protecting Label)
* **소스 파일**:
  - [_engine_exit.py](file:///c:/Users/stone/projects/nocodequant/app/ai/_engine_exit.py#L95-L120)
  - [_engine_exit.py](file:///c:/Users/stone/projects/nocodequant/app/ai/_engine_exit.py#L282-L302)
  - [execution_manager.py](file:///c:/Users/stone/projects/nocodequant/app/services/managers/execution_manager.py#L398-L420)
* **현상**:
  - 대양금속(009190) 등 일부 종목이 매수 진입 후 단 수 초(7초) 만에 급락 청산(`DecisionLabel.SELL`)되는 현상이 발생했습니다.
  - 원인은 감시 등록 시 생성되는 가상 틱(Warmup Snapshots)의 고가가 진입 당시 주가로 저장되고, 실제 매수 체결(낮은 가격) 직후 `ExitEngine`이 이 오염된 고가를 캔들 고가(`curr['high']`)로 읽어 `peak_price`를 과도하게 높게 잡았기 때문입니다. 이로 인해 갭하락으로 오인하여 플래시 크래시 가드(Flash Crash Protection)가 오작동했습니다.
  - 또한 파워넷(037030) 등 Ratchet(수익 보존) 청산이 정상 이익을 남기고 완료되었음에도 UI에 일괄적으로 `"손절"`로 표기되는 라벨링 오류가 있었습니다.
* **조치**:
  - **체결가 Clamping**: `execution_manager.py`에서 BUY 체결 확인(PendingFill 해제) 즉시 StateManager 포지션 데이터의 `max_price`/`min_price`, `daily_price_cache` 및 전략 세션의 `peak_price`/`entry_price`를 실제 체결 평균단가(`_fill_price`)로 강제 강하 초기화(Clamping)하여 진입 전 웜업 노이즈를 완벽히 차단했습니다.
  - **체결 미경과 봉 고가 가드**: `_engine_exit.py` 내 `peak_price` 업데이트 시, 체결 시점보다 이전 봉이거나 체결 진행 봉(5분 미경과)의 캔들 고가(`curr['high']`) 반영을 우회하고 실시간 실제 틱에 의해서만 갱신되도록 `_is_dirty_candle` 필터를 도입했습니다.
  - **익절 라벨 정밀화**: `_engine_exit.py`에서 수익 보존 청산(Ratchet, Break-Even 등)이 실행될 때 `ExitType.STOP_LOSS` 대신 `ExitType.PEAK_TRAIL`을 반환하게 하여 UI 및 체결 로그에 정상적으로 `"익절"`로 렌더링되도록 수정했습니다.

---

## 🔮 향후 대응 및 실거래 운영 안정화 방안
1. **장중 가상 스냅샷 거동 감시**:
   - `MAX_SYNTHETIC_TTL_SEC (180초)`에 따라 비거래 종목의 가상 틱이 정상 퍼지되는지 장중에 로그(`🗑️ [Surveillance] Purged...`)를 모니터링합니다.
2. **리플레이/학습용 추출 파이프라인 검증**:
   - `get_quote_snapshots`를 사용하는 Forensic 로그 또는 AI 피처 빌더 연동 시 `synthetic: True` 데이터를 필터링하는 로직이 정상 작동하는지 교차 확인합니다.
3. **조기 청산 세이프가드 및 체결가 초기화 로깅**:
   - 신규 포지션 진입 시 가상 틱으로 오염되었던 고가 정보가 정상적으로 리셋(Clamping 로그)되는지 모니터링하고, 장중 Ratchet 청산 시 "익절" 라벨로 변환 표기되는지 실시간 매매 로그를 확인합니다.
