# 📅 2026-06-13 업무 일지 (Daily Work Log)

> **[거버넌스 준수]**
> * **작업 등급**: L3 (가드 검증 및 설정 최적화, 정밀 캡처 인프라 구축, 형상 관리 완료)
> * **참조 문서**: 
>   * [guard_verification_260613_ko.md](file:///c:/Users/stone/projects/nocodequant/docs/backtests/guard_verification_260613_ko.md)
>   * [compare_0609_vs_current_report.md](file:///c:/Users/stone/projects/nocodequant/docs/backtests/compare_0609_vs_current_report.md)
>   * [config_engine.py](file:///c:/Users/stone/projects/nocodequant/app/core/config_engine.py)
>   * [analyze_shadow_guard_ev.py](file:///c:/Users/stone/projects/nocodequant/scripts/analyze_shadow_guard_ev.py)
> * **영향 받는 모듈 (Impacted Modules)**: `config_engine.py`, `schema.py`, `execution_manager.py`, `order_manager.py`, `state_manager.py`, `persistence_manager.py`
> * **불변성 위반 여부 (Invariant Check)**: 위반 없음 (Violated: None)
> * **경계 준수 선언**: 제미나이 S2 분석의 독립 검증을 수행하여, 실거래 대손실의 근본 원인이 L3 가드의 관찰 전용(Shadow ON) 모드 구동 때문이었음을 실증적으로 규명하였습니다. 또한 저점 가점의 순부정 효과를 규명하여 0으로 제거하고 라이브 배포하였으며, 섀도 모드의 한계(슬롯 잠식, 매칭률 20% 미만)를 극복하기 위해 포지션 비중 축소(10%)와 실시간 가드명 정밀 캡처 영속 인프라(V28.4g)를 구축하고 통합 검증을 완료했습니다.

---

## 🎯 주요 목표 및 이슈 정의

1. **L3 가드 Enforce(Shadow OFF) 성능 독립 검증**:
   - 가드를 실제로 집행했을 때와 관찰 모드(Shadow ON)로 돌렸을 때의 성과 갭을 계산일(260604)과 악장일(260611) 양쪽에서 실측하여, 가드 Enforce가 손익을 양전시키는 1순위 레버임을 증명.
2. **저점 위치 보너스(ENTRY_LOW_LOCATION_BONUS)의 인과 분리**:
   - V28.3 P4에서 추가되었던 저점 가점(+5)이 애매한 진입을 유도하여 성과를 희석하는 원인임을 밝히고 이를 제거.
3. **라이브 설정 긴급 반영**:
   - 저점 보너스 0을 라이브 설정(`config_engine.py`)에 즉시 반영. 단, 가드 Enforce(Shadow OFF)는 라이브 충격을 완화하기 위해 모니터링 후 단계 적용하기로 보류(Shadow ON 유지).
4. **V28.4 작업 일체 선별 커밋 및 형상 관리**:
   - 이번 세션 동안 진행된 VOLUME 전략 재정의, BREAKOUT 실패컷, 유니버스 게이트 복제, L0 청산 및 래칫 노브 등 V28.4 변경 사항을 선별하여 main 브랜치 직접 커밋 대신 가이드라인에 따라 `feat/v28.4-guard-bonus-and-gates` 브랜치로 커밋 처리.
5. **섀도 모드 데이터 확보 병목(슬롯 잠식) 해결 설계**:
   - 관찰 모드(Shadow ON) 상황에서 나쁜 거래가 먼저 입주하여 좋은 거래의 슬롯을 뺏어버리는 병목을 완화하고 풍부한 대조군 데이터를 모으기 위해 포지션 비중을 25% ➔ 10%로 낮추는 전략 수립.
6. **섀도 가드 정밀 캡처 인프라(V28.4g) 구축**:
   - 섀도 로그와 실거래 데이터의 낮은 대조 매칭률(~19.5%) 한계를 극복하고, 퍼지 조인 없이 100% 매칭률로 각 가드의 실제 정량 성과를 GROUP BY 집계할 수 있도록 주문-체결 영속화 전 구간에 섀도 가드 추적 필드(`entry_shadow_guards`)를 신설.

---

## ✅ 상세 작업 및 패치 내역

### 1. L3 가드 Enforce 및 저점 보너스 토글 검증
- **조치 내역**:
  - `backtests/run_v28_guard_verification_260613.py` 스크립트를 작성하여 8개의 테스트 시나리오를 구동.
  - 가드가 작동하지 않는 `Shadow ON` 상태가 누적 손실(-0.366% ~ -0.484%)의 주범이었으며, 가드 Enforce 시 거래당 EV가 0.4~0.5%p 개선되어 계산일 흑자(+0.237%), 악장일 본전(-0.016%)을 달성하는 극적 반전 실증.
  - 리포트 작성 완료: `docs/backtests/guard_verification_260613_ko.md`

### 2. 저점 보너스 제거 라이브 적용
- **조치 내역**:
  - [config_engine.py](file:///c:/Users/stone/projects/nocodequant/app/core/config_engine.py#L144): `ENTRY_LOW_LOCATION_BONUS = 0` 적용하여 애매한 꼭지점 매수를 유발하던 가점을 영구 제거 및 배포.

### 3. 섀도 가드 정밀 캡처 인프라 구축 (V28.4g)
- **조치 내역**:
  - **영속화 사슬 연쇄 패치**:
    * [schema.py](file:///c:/Users/stone/projects/nocodequant/app/core/schema.py): `TradeSignal` 데이터클래스에 `entry_shadow_guards: str = ""` 필드 신설.
    * [execution_manager.py](file:///c:/Users/stone/projects/nocodequant/app/services/managers/execution_manager.py): 진입 판단 결과인 `reasons`에서 `[SHADOW]` 패턴이 가드에 감지되면 해당 가드명을 추출하여 `TradeSignal`에 주입.
    * [order_manager.py](file:///c:/Users/stone/projects/nocodequant/app/services/managers/order_manager.py): 진입 메타 데이터를 체결 청산 시의 payload까지 정상 전달하도록 5개 지점 미러링 패치.
    * [state_manager.py](file:///c:/Users/stone/projects/nocodequant/app/services/managers/state_manager.py): `get_position_meta` 호출 시 섀도 가드 필드를 반환 딕셔너리에 매핑.
    * [persistence_manager.py](file:///c:/Users/stone/projects/nocodequant/app/services/managers/persistence_manager.py): DB 기동 시 멱등성을 보장하는 컬럼 핫 마이그레이션(`ALTER TABLE trades ADD COLUMN entry_shadow_guards TEXT DEFAULT NULL`)을 이식하고, INSERT문에서 31개의 컬럼과 바인딩 정합성을 일치시킴.
  - **하이브리드 분석기 업그레이드**:
    * [analyze_shadow_guard_ev.py](file:///c:/Users/stone/projects/nocodequant/scripts/analyze_shadow_guard_ev.py): DB 내에 정밀 캡처 필드(`entry_shadow_guards`)가 존재하면 최우선으로 이를 GROUP BY에 사용하고, 과거 이력이 없을 때는 퍼지 조인(Fuzzy Join)으로 자동 폴백되도록 어댑티브 하이브리드 연산 적용.

### 4. V28.4 작업 선별 커밋 완료
- **조치 내역**:
  - `feat/v28.4-guard-bonus-and-gates` 브랜치 생성 후 V28.4b~f 핵심 소스코드 커밋(`71fafa1`).
  - 이어서 V28.4g 정밀 캡처 인프라 및 분석기 수정 파일 일체를 컴파일 및 검증 완료 후 커밋(`5add6ad`) 완료.

### 5. 섀도 모드 개선 설계 (비중 10% 축소)
- **설계 내용**:
  - 포지션 비중을 `25%` ➔ `10%`로 축소 시, 동시 입주 슬롯이 `4개` ➔ `10개`로 확대.
  - 섀도 모드에서 통과된 불량 거래가 입주하더라도 여전히 6개 이상의 빈방이 남으므로 정예 주도주의 매수 타이밍이 튕겨 나가는 현상 방지.
  - 리스크 노출도가 2.5분의 1로 낮아져 섀도 모드 관찰 기간 동안의 계좌 훼손 리스크 완화.

---

## 🧪 검증 및 테스트 결과

### 1. 가드 Enforce + 저점 보너스 분리 스윕 결과 (TREND-only, 유니버스 ON)
- **검증 리포트**: [guard_verification_260613_ko.md](file:///c:/Users/stone/projects/nocodequant/docs/backtests/guard_verification_260613_ko.md)

| Dataset | Scenario (가드 / 보너스) | Trades | WinRate | TotalPnL | AvgPnL (EV) |
| --- | --- | :---: | :---: | :---: | :---: |
| **260604(계산일)** | 가드 ON / 보너스 5 | 466 | 33.91% | 59.94% | +0.129% |
| | **가드 ON / 보너스 0 (최적)** | **358** | **34.64%** | **84.98%** | **+0.237%** |
| | 가드 OFF / 보너스 5 (라이브) | 958 | 31.00% | -350.99% | -0.366% |
| | 가드 OFF / 보너스 0 (현재) | 797 | 33.00% | -230.09% | -0.289% |
| **260611(악장)** | 가드 ON / 보너스 5 | 728 | 36.13% | -44.80% | -0.062% |
| | **가드 ON / 보너스 0 (최적)** | **545** | **38.17%** | **-8.49%** | **-0.016%** |
| | 가드 OFF / 보너스 5 | 1556 | 34.13% | -753.24% | -0.484% |
| | 가드 OFF / 보너스 0 | 1269 | 35.15% | -595.98% | -0.470% |

- **주요 해석**:
  1. **가드 Enforce가 압도적 1순위 레버**: 가드 강제 집행 시 양일 모두 거래량이 절반 이하로 감소하고, 평균 PnL이 0.4~0.5%p 상승해 마이너스 영역에서 양수(+0.237%) 및 본전(-0.016%)으로 회생했습니다.
  2. **저점 보너스 제거의 효과**: 보너스 제거 시 거래 횟수가 약 25% 추가로 줄고 평균 손익이 한 단계 더 향상(계산일 +0.129% ➔ +0.237%, 악장일 -0.062% ➔ -0.016%)되었습니다. 저점 보너스는 휩소 구간 진입을 촉진하는 역효과가 있었음을 확인했습니다.

### 2. L3 정밀 캡처 인프라 통합 검증 (trade_exe32)
- **검증 스크립트**: `tests/test_shadow_guard_capture_260613.py`
- **결과**: **10개 검증 시나리오 전체 통과 (Pass 10/10)**
  - `TradeSignal` 속성 정상 바인딩 확인.
  - 다중 섀도 가드 감출 및 문자열 파싱 검증 완료.
  - `PersistenceManager` 임시 DB 핫 마이그레이션(ALTER TABLE) 실행 안정성 및 31컬럼/31플레이스홀더 INSERT 문 정합성 완료.
  - 격리 데이터베이스에 대한 라운드트립(Insert ➔ Select 복원) 100% 정상 작동 검증.

---

## 🔮 향후 계획

1. **1~2 영업일간 라이브 모니터링**:
   - 저점 보너스 0 적용으로 인해 아침 장초반이나 급상승 국면에서 오단 장대양봉 꼭지점 추격 빈도가 줄어드는지 관측.
2. **GUARD_SHADOW_MODE = False (가드 실제 차단) 적용 시점 조율**:
   - 모니터링 이후, 가드를 한꺼번에 enforce 시킬지 혹은 가드별로 순차 적용할지를 정하여 실거래 손익 흑자 전환 적용.
3. **포지션 비중 10% 조정**:
   - 라이브 UI/설정에서 수량 비중을 10%로 낮추어, 섀도 모드 하에서 슬롯 한도 초과로 좋은 종목이 기각당하는 현상을 방지하고 다양한 검증용 데이터를 안전하게 축적.
4. **안전 시간대 캡처 패치 배포**:
   - 주말 및 장마감 시간대를 활용하여 실거래 런타임 시스템에 정밀 캡처(ALTER TABLE 및 파이프라인 전달) 최종 배포 완료. 배포 시점 이후 생성된 신규 거래부터는 100% 매칭률의 가드 통계 리포트 생성 활성화.
