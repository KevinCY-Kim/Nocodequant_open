# 📋 [Issue Report] 국면(Regime) 판정 및 제어 체계의 불일치 현황

> **작성일**: 2026-04-07  
> **대상 모듈**: `app/ai/strategy.py`, `app/core/config_engine.py`, `app/ui/strategy_dashboard.py`  
> **상태**: 현황 분석 완료 (해결 방안 논의 대기)

---

## 1. 아키텍처적 불일치 현황 (Architectural Inconsistencies)

현재 NoCodeQuant 엔진 내에서 시장 국면(Market Regime)을 판정하고 이를 로직에 적용하는 과정에서 다음과 같은 세 가지 불일치가 발견되었습니다.

### 1.1 국면 분류 체계(Taxonomy)의 다원화
시스템 내부적으로 국면을 정의하는 명칭과 범주가 위치에 따라 서로 다르게 운영되고 있습니다.

- **엔진 내부 판정 (`Level 1`)**: `TREND`, `RANGE` (2분법적 분류)
- **엔진 외부 출력 (`Level 2`)**: `UPTREND`, `DOWNTREND`, `RANGE` (방향성 포함 분류)
- **설계 가이드 문서 (Architecture)**: `UPTREND`, `DOWNTREND`, `RANGE`, `CHAOS` (4대 국면 분류)
- **설정 엔진 (`ACTIVE_CONFIG`)**: `TREND`, `RANGE`, `CHAOS` (시간 청산 및 임계값 맵핑용 키)

### 1.2 제어 로직과 판정 데이터의 단절 (Logic-Data Gap)
설계 의도와 달리 실제 제어권(Control Flow)이 논리적으로 도달할 수 없는 구간이 존재합니다.

- **CHAOS 국면의 사장(Dead Code)**: 설정값(Config)과 매핑 테이블에는 `CHAOS` 국면에 대한 임계값(72점)과 시간 청산(10봉)이 정의되어 있으나, 엔진이 내부 제어에 사용하는 변수(`current_regime`)는 `TREND`/`RANGE`만 반환하므로 `CHAOS` 관련 로직은 실제 시장 상황과 관계없이 실행되지 않는 상태입니다.
- **방향성 정보의 제어권 미부여**: `UPTREND`와 `DOWNTREND`에 대한 세부 판정은 이루어지고 있으나, 이는 UI 출력용으로만 사용될 뿐 진입 임계값(Threshold)을 국면별로 미세 조정하는 제어 루프에는 연결되어 있지 않습니다.

### 1.3 데이터 흐름의 선후 관계 역전 (Flow Inversion)
국면 정보가 정교화되는 시점이 매매 결정(Decision Making) 이후에 위치하여, 결정의 근거로 사용되지 못하고 있습니다.

- **후행적 정교화**: 국면을 `UPTREND`/`DOWNTREND`/`CHAOS`로 세분화하는 로직이 `_get_signal_internal` 메서드의 최하단(결과 반환 직전)에 위치합니다. 이로 인해 정작 중요한 매수/매도 결정 단계에서는 세분화된 국면 정보를 활용하지 못하고, 단순화된 `Level 1` 정보에만 의존하고 있습니다.

---

## 2. 시스템 영향 분석 (System Impact)

1.  **전략 성과보드(UI)와의 괴리**: 사용자는 화면에서 `UPTREND`를 보고 있으나, 실제 엔진은 이를 일반적인 `TREND`로 취급하여 동일한 임계값을 적용함에 따라 사용자의 기대 동작과 실제 로직 간의 간극 발생 가능성이 존재합니다.
2.  **리스크 관리 공백**: 변동성이 높고 방향성이 없는 `CHAOS` 국면에서 설계된 방어 로직(높은 문턱값, 짧은 보유 시간)이 작동하지 않아 과변동성 장세에서의 리스크 노출 가능성이 있습니다.
3.  **코드 유지보수 혼선**: 설정 파일(`config_engine.py`)에는 존재하는 파라미터가 실제 엔진 로직에서는 참조되지 않는 '유령 파라미터'화 되어 있어 향후 튜닝 시 혼선을 줄 수 있습니다.

---
*Reported By Antigravity (Powered by Advanced Agentic Coding)*
