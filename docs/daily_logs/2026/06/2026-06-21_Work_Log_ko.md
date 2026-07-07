# 📅 2026-06-21 업무 일지 (Daily Work Log)

> **[거버넌스 준수]**
> * **작업 등급**: L3 (양 시장 디커플링 연동 구현, 지수 시초가 실시간 수신 안정화, 주도주 노출 가중치 엔진 구현 및 검증)
> * **참조 문서**: 
>   * [quant_baseline_260619_ko.md](file:///c:/Users/stone/projects/nocodequant/docs/backtests/quant_baseline_260619_ko.md)
>   * [_engine_entry.py](file:///c:/Users/stone/projects/nocodequant/app/ai/_engine_entry.py)
>   * [ranking_manager.py](file:///c:/Users/stone/projects/nocodequant/app/services/managers/ranking_manager.py)
> * **영향 받는 모듈 (Impacted Modules)**: `_engine_entry.py`, `engine.py`, `ranking_manager.py`, `config_engine.py`, `index_history.json`
> * **불변성 위반 여부 (Invariant Check)**: 위반 없음 (Violated: None)
> * **경계 준수 선언**: 실거래 지수 시초가 누락을 근본적으로 차단하기 위해 실시간 업종지수 구독 시 FID 16(시가)을 완벽히 매핑하였습니다. 또한 코스피가 상승할 때 코스닥이 급락하는 등의 양 시장 디커플링 현상을 정합하게 백테스팅 및 라이브 매매에 반영하기 위해 개별 종목의 소속 시장(KOSPI/KOSDAQ)을 자동 분리하고 각각 독립적으로 `adaptive_gap` 스위칭을 수행하도록 엔진을 패치하였습니다. 나아가 실시간 주도주 노출 랭킹 엔진에 양 시장 상대 강도(스프레드) 격차에 기반한 동적 노출 가중치를 부스팅/댐핑하여 기회 비용을 확보하도록 개선하고 단위 테스트 검증을 완료하였습니다.

---

## 🎯 주요 목표 및 이슈 정의

1. **실시간 지수 시초가 수신 누락 방어**:
   - 실거래 장 초반 키움 API로부터 지수 시초가가 유실되거나 0으로 받아와져 갭캡 필터 계산 시 예외가 발생할 위험을 완벽히 방어.
   
2. **양 시장(KOSPI/KOSDAQ) 디커플링 현상 시뮬레이션 및 실거래 대응**:
   - 코스피 상승세에 코스닥 종목이 휩쓸려 무분별하게 상투 매수되거나 반대로 코스닥 강세장에서 코스피 종목의 진입이 원천 차단되는 지수 왜곡 현상 방지.
   - 종목별 소속 시장을 분리하여 각각의 시장 지수 등락율에 맞춘 독립적인 탄력적 갭캡을 적용.

3. **고정 갭 vs 디커플링 적용 탄력 갭의 백테스트 교차 검증**:
   - 단순 3% 고정 갭을 사용한 대조군(`fixed_gap_3`)과 디커플링 분리를 적용한 실험군(`adaptive_gap`)을 비교하여 실질적인 성과 개선율 입증.

4. **실시간 주도주 노출 디커플링 동적 가중치 제어**:
   - 코스피-코스닥 양 시장 간의 실시간 강세 격차를 계량화하여 주도주 랭킹 스코어(Heat Score)를 동적 부스팅/댐핑.
   - 강세 시장의 주도주들이 랭킹 상단에 노출되도록 유도하여 제한된 계좌 슬롯 내 거래 기회비용 극대화.

---

## ✅ 상세 작업 및 패치 내역

### 1. 실시간 업종지수 수신 정밀화 및 방어 코드 구축
- **조치 내역**:
  - [engine.py](file:///c:/Users/stone/projects/nocodequant/app/ai/engine.py#L1423-L1425): 실시간 업종지수 등록(`SetRealReg`) 호출 시 기존 수신 필드에 `FID 16`(시가)을 추가(`"10;11;16"`)하여 분모가 되는 시초가 누락을 차단.
  - [state_manager.py](file:///c:/Users/stone/projects/nocodequant/app/services/managers/state_manager.py#L763-L776): 실시간 시가 정보가 수집되면 `get_index_chg_from_open` 연산의 정합성을 담보하고, 미수신 시에는 `index_history.json` 캐시를 거쳐 기존 시장 국면(`current_regime`)에 기반한 갭캡으로 안전하게 백업 및 폴백하도록 방어.

### 2. 코스피-코스닥 소속 시장별 독립 갭 스위칭 구현
- **조치 내역**:
  - [_engine_entry.py](file:///c:/Users/stone/projects/nocodequant/app/ai/_engine_entry.py#L433-L497): `_market_of(code)` 함수를 도입하여 종목 소속 시장을 분리하고, 해당 시장의 실시간 등락율(`change_from_open`)에 맞춰 독립 적용하도록 로직 개편.
  - 백테스팅 환경 지원을 위해 [index_history.json](file:///c:/Users/stone/projects/nocodequant/data/index_history.json) 역사적 지수 데이터베이스를 구축하여 과거 시뮬레이션 시에도 동일한 디커플링 규칙이 반영되도록 배선 완료.

### 3. 비교군(fixed_gap_3) 추가 및 검증 백테스트 수행
- **조치 내역**:
   - [run_quant.py](file:///c:/Users/stone/projects/nocodequant/backtests/run_quant.py#L39-L43): 대조군 실험을 위해 3% 고정 갭 변이(`fixed_gap_3`)를 추가하고 4개 윈도우에 대해 전체 시뮬레이션 재수행 및 결과 자동 집계 완료.

### 4. 양 시장 디커플링 기반 주도주 노출 가중치 엔진 구현
- **조치 내역**:
  - [config_engine.py](file:///c:/Users/stone/projects/nocodequant/app/core/config_engine.py#L324-L327): 동적 노출 민감도(`MARKET_DECOUPLING_SENSITIVITY = 0.20`) 및 부스팅/댐핑 캡 범위(`0.70 ~ 1.30`)를 설정에 추가하여 동적 제어 가능하도록 구성.
  - [ranking_manager.py](file:///c:/Users/stone/projects/nocodequant/app/services/managers/ranking_manager.py#L785-L810): 실시간 지수 시가 대비 등락율 격차(Spread)에 비례하여 코스피와 코스닥 소속 종목들의 주도주 스코어(Power)를 부스팅/댐핑하는 가중치 연산 삽입. 랭킹 스코어 요동 방지를 위해 `0.2%` 데드존 필터 적용.

---

## 🧪 검증 및 테스트 결과

### 1. 고정 갭 3% vs 디커플링 반영 탄력 갭 최종 비교 (TREND 전략)
* **검증 조건**: 4개 윈도우 교차 시뮬레이션
* **장세별 기대값(EV) 및 거래 수 비교**:

| 장세 국면 | 고정 갭 3.0% (`fixed_gap_3`) | 탄력적 갭 1.0%~3.0% (`adaptive_gap`) | 성과 비교 우위 (개선율) |
| :--- | :---: | :---: | :--- |
| **보합장 (EARLY)** | `-0.151%` (n384) | **`+0.029%` (n243)** | **흑자 양전 성공** (불필요한 진입 36% 차단) |
| **하락장 (CRASH)** | `+0.035%` (n1022) | **`+0.095%` (n662)** | **EV 2.7배 상승** (하락장 추격 35% 차단) |
| **상승장 (RISE)** | `+0.089%` (n760) | **`+0.353%` (n408)** | **EV 3.96배 상승** (디커플링 휩소 46% 차단) |

### 2. 성과 해석
* **디커플링 분리 필터의 유효성 검증**:
  - 기존의 통합 3% 고정 방식은 코스피가 상승해도 코스닥이 무너지는 디커플링 국면에서 코스닥 종목들에 3% 갭을 완만하게 허용하여 손실 휩소에 노출되었습니다.
  - 시장 독립 적용 방식은 코스닥 종목에 대해 1.0% 갭제한을 엄격히 고수하여 휩소 거래를 **46% 차단**하였고, 그 결과 상승장 EV가 `+0.089%`에서 **`+0.353%`로 극대화**되었습니다.

### 3. 랭킹 디커플링 가중치 엔진 단위 테스트 검증
- **검증 시나리오**: KOSPI +1.0% 상승, KOSDAQ -1.0% 하락 (스프레드 2.0%) 설정
- **결과**:
  - KOSPI 종목(삼성전자) 점수: `39` (부스팅 가중치 `1.30` 상한 캡 적용)
  - KOSDAQ 종목(에코프로) 점수: `8` (댐핑 가중치 `0.70` 하한 캡 적용)
  - 양 시장의 상대 강도에 따라 주도주 노출 우선순위가 정합하게 격리 및 유도되는 것을 성공적으로 입증.

---

## 🔮 향후 계획 및 최종 의도

1. **실거래 구동 확인**:
   - 실거래 장중 로그를 모니터링하여 코스피/코스닥 실시간 등락율이 종목별 소속 시장에 매칭되어 갭캡 제한이 유연하고 정합하게 스위칭되는지 관찰.
2. **형상 관리 완료**:
   - 백테스트 결과 및 구현 코드를 깃허브 원격 저장소(`feat/v28.4-guard-bonus-and-gates` 브랜치)에 push 완료하여 정합성 확보.
