# 📅 2026-06-22 업무 일지 (Daily Work Log)

> **[거버넌스 준수]**
> * **작업 등급**: L3 (실거래 매매 0건 사후분석 및 거래대금 가드 적응형 전환 구현)
> * **참조 문서**:
>   * [_engine_entry.py](file:///c:/Users/stone/projects/nocodequant/app/ai/_engine_entry.py)
>   * [config_engine.py](file:///c:/Users/stone/projects/nocodequant/app/core/config_engine.py)
> * **영향 받는 모듈 (Impacted Modules)**: `_engine_entry.py`, `config_engine.py`
> * **불변성 위반 여부 (Invariant Check)**: 위반 없음 (Violated: None)
> * **경계 준수 선언**: 실거래 매매 0건이라는 이상 신호를 forensics.db(signals)·trades.db
>   원장으로 직접 근본원인까지 추적하였습니다. 고정 절대 거래대금 가드의 마진 부재와, 눌림목
>   성숙도 면제 도입(6/11) 이후 재발한 봉초반 오차단 회귀를 함께 확인하였고, 두 문제를
>   해소하는 적응형 가드를 도입하되 기존 안전장치(상한 클램프·표본 부족 시 고정값 폴백)를
>   유지해 과도한 완화를 방지하였습니다.

---

## 🎯 주요 목표 및 이슈 정의

1. **실거래 매매 0건 원인 규명**: 지난 금요일(6/19) 25건 체결 대비 오늘(6/22) 0건. forensics.db/
   trades.db 교차 분석으로 정확한 차단 지점을 특정.
2. **거래대금 가드 구조적 취약점 해소**: `ENTRY_MIN_VALUE_5M` 고정 5천만원 가드가 시장 전체
   유동성이 평소보다 살짝만 낮아도 전량 봉쇄로 이어지는 설계임을 확인하고 적응형으로 전환.
3. **눌림목 성숙도 면제 회귀 수정**: `ENTRY_MATURITY_PULLBACK_EXEMPT`(6/11 도입)가 진행 중인
   미성숙봉을 즉시 평가하게 하면서, V28.0이 의도했던 "성숙도 대기로 봉초반 대금 오차단을
   해소한다"는 보호 효과를 무력화시킨 회귀를 수정.

---

## ✅ 상세 작업 및 패치 내역

### 1. 원인 분석 (forensics.db + trades.db)
- **조치 내역**:
  - `data/trades.db` `trades` 테이블 조회 결과 2026-06-22 체결 0건 확인(6/15~6/19는 24~30건).
  - `data/forensics.db` `signals` 테이블 집계: 오늘 매수 신호 8,906건 중 8,907건(누적치 포함)이
    `SIGNAL_BLOCKED`. 이 중 96%(8,579건)가 `매수 보류: 대금 부족 (value=N < limit=50,000,000)`.
  - 봉이 거의 다 익은(121~240초 경과) 후보 3,129건만 따로 보면 오늘 최고 후보값은
    49,939,500원으로 5천만원 기준선을 근소하게 못 넘김. 반면 6/18·6/19도 차단된 후보 최고치가
    각각 49,928,400원·49,993,750원으로 거의 동일한 선에서 막혀 있었지만, 그날은 턱걸이로
    기준을 넘는 후보가 있어 26건·25건이 체결됨 — **가드 자체가 원래도 여유 없이 설계**됐음을
    확인.
  - 차단 신호 중 11%(938건)는 `🎯 눌림 구조 — 봉 성숙도 면제 (Ns/240s) [V28.3 P3]` 태그와 함께
    봉이 열린 지 30초 이내에 평가되어, 거래량이 1~5주 수준만 쌓인 상태로 무조건 대금 가드에
    걸림 — `ENTRY_MATURITY_PULLBACK_EXEMPT`(6/11, `af71edb`)가 `ENTRY_BAR_MATURITY_SEC`(6/10,
    `3f22d0a`) 도입 시 명시된 "성숙도 대기가 대금가드 봉초반 오차단도 동반 해소" 부수효과를
    면제 경로에서 무력화시키는 회귀로 판명.

### 2. 거래대금 가드 적응형 전환
- **조치 내역**:
  - [config_engine.py](file:///c:/Users/stone/projects/nocodequant/app/core/config_engine.py#L122-L133):
    `ENTRY_VALUE_FLOOR_ADAPTIVE`(마스터 스위치) / `ENTRY_VALUE_FLOOR_PERCENTILE`(0.70) /
    `ENTRY_VALUE_FLOOR_MIN`(1,500만원) / `ENTRY_VALUE_FLOOR_MAX`(기존 고정값 5천만원) /
    `ENTRY_VALUE_FLOOR_MIN_SAMPLES`(30) 신규 노브 추가.
  - [_engine_entry.py](file:///c:/Users/stone/projects/nocodequant/app/ai/_engine_entry.py):
    `EntryEngine`에 당일 성숙봉 거래대금 표본을 적립하는 day-scoped 롤링 버퍼
    (`_value_samples`, 날짜 변경 시 자동 리셋)와 `_record_value_sample` /
    `_get_adaptive_value_floor` 헬퍼 추가. 표본이 `MIN_SAMPLES` 미달이면(장 초반) 기존 고정값
    그대로 폴백, 충분하면 70th percentile을 `[MIN, MAX]`로 클램프해 그날의 "상위 후보"는 통과
    가능하도록 함. `MAX=기존 고정값`이므로 평소보다 더 엄격해지는 경우는 없음.

### 3. 눌림목 성숙도 면제 시 거래대금 계산 버그 수정
- **조치 내역**:
  - [_engine_entry.py](file:///c:/Users/stone/projects/nocodequant/app/ai/_engine_entry.py):
    Bar-Maturity Gate에서 `_pullback_exempt`가 발동하면(`curr`가 거래량 미적립 진행봉)
    `_value_ref_row`를 `df.iloc[-2]`(직전 완성봉)로 교체. 거래대금 가드는 이후 `curr` 대신
    `_value_ref_row`로 `curr_val`을 계산해 실측 완성봉 데이터로 평가. 직전봉이 없는 극단적
    케이스(세션 첫 봉)는 `curr`로 안전 폴백. 정상 성숙 경로(`_pullback_exempt=False`)는 변경 없음.

---

## 🧪 검증 및 테스트 결과

### 1. 정적 검증
- `py -3 -m py_compile app/ai/_engine_entry.py app/core/config_engine.py` 통과.

### 2. 오프라인 리플레이 (오늘자 차단 신호 8,663건 재시뮬레이션)
- 70th percentile 원시값 11,147,500원 → `[15M, 50M]` 클램프 적용 후 적응형 바닥값
  **15,000,000원**(하한 클램프 발동 — 오늘처럼 극단적으로 한산한 날엔 percentile보다 하한이
  우선 작동해 과도한 완화를 방지).
- 이 바닥값으로 재평가 시 오늘 차단됐던 8,663건 중 **1,975건(23%)**이 통과로 전환, 고유
  (종목·분) 조합 기준 **86건**의 실질적 매매 기회가 발생했을 것으로 추산.
  (단, 면제로 조기평가된 938건은 원본 로그에 직전봉 값이 없어 동일 curr_val로만 재계산했으므로
  실제로는 이보다 더 많은 기회가 살아났을 가능성 — 과소추산.)
- 대조군: 6/19(25건 체결일)에 동일 방식으로 계산하면 적응형 바닥값이 22,407,190원으로,
  기존 고정값보다는 낮지만 상한(5천만원)을 넘지 않음 — 정상 거래일에 가드가 기존보다
  더 엄격해지지 않음을 확인.

### 3. 기존 테스트
- `tests/` 디렉터리에 `_engine_entry`/거래대금 가드 관련 기존 테스트 없음 — 회귀 검사 대상 없음.

---

## 🔮 향후 계획 및 최종 의도

1. **실거래 구동 확인**: 다음 거래일 forensics.db `SIGNAL_BLOCKED` 사유 분포를 재집계하여
   "대금 부족" 비중 감소와, 면제 케이스의 `value=`가 더 이상 1~30초 구간에서 비정상적으로
   작은 값(수천~수만원)으로 찍히지 않는지 확인.
2. **상시 모니터링**: 적응형 바닥값이 `ENTRY_VALUE_FLOOR_MIN`(1,500만원) 클램프에 자주 걸리는지
   추적 — 너무 자주 하한에 머물면 해당 값 자체의 재조정 검토.
