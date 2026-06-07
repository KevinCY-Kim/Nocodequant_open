# NCQ Ranking System Algorithm Specification (V22.0)

본 문서는 NoCodeQuant 시스템의 실시간 시장 분석 엔진인 `RankingManager`의 순위 산출 알고리즘과 데이터 처리 메커니즘을 정의합니다.

## 0. 설계 원칙 (Design Principles)
- **교차 검증 (Cross-Verification)**: 단일 지표가 아닌 거래대금, 등락률, 체결강도 등 다수 지표에 동시 노출된 종목에 가중치를 부여합니다.
- **데이터 복구 (Data Restoration)**: 수천 퍼센트 단위로 튀는 데이터 오염(Inflation)을 정규 범위로 복구하여 신호 무결성을 유지합니다.
- **비선형 압축 (Non-linear Compression)**: 로그 스케일링을 통해 지표의 포화(Saturation)를 방지하고 미세한 변화를 포착합니다.

---

## 1. HOT (Smart Heat Index)
시장의 종합적인 '열기'를 측정하는 핵심 지표입니다.

### 1.1 산출 공식 (Formula)
$$RawScore = (Val_{norm} \times 0.4 + Mom_{norm} \times 0.3 + Acc_{norm} \times 0.2 + Pow_{norm} \times 0.1) \times CrossBoost$$

- **$Val_{norm}$ (40%)**: 거래대금 순위 점수 (1위=1.0 ~ 최하위=0.0)
- **$Mom_{norm}$ (30%)**: 가격 모멘텀 점수 (3~12% 구간에서 최대점, 과열/급락 시 자동 감쇄)
- **$Acc_{norm}$ (20%)**: 거래량 가속도 순위 점수
- **$Pow_{norm}$ (10%)**: 체결강도/호가잔량 점수 (200% 기준 정규화)
- **$CrossBoost$**: 다수 랭킹 동시 진입 시 보너스 (`1.0 + (hits-1) \times 0.25`). [Fix-1] 필드별 중복 카운팅 방지 로직 적용.

### 1.2 보정 로직
- **Penny Damping**: 1,000원 미만 동전주에 대해 0.6x~0.8x 점수 삭감.
- **Liquidity Quality (3억 기준) [Fix-8]**: 거래대금 3억(3.0) 미만 종목에 대해 비례적 페널티 부여 (`val_total / 3.0`). 

---

## 2. HOT2 (Burst Energy Index)
종목의 '속도'가 아닌 '가속도(에너지)'에 집중하여 급발진하는 종목을 포착합니다.

### 2.1 산출 공식
$$AccelScore = \ln(1 + |a_{smooth}| \times 1000) \times 18$$

- **$a_{smooth}$**: Heat Index의 2차 미분값(가속도)을 EMA(0.5)로 평활화한 값.
- **Logarithmic Scaling**: `log1p` 함수를 사용하여 지표가 100점에 조기 고착되는 것을 방지하고 민감도를 유지합니다.
- **Signal Purity [Fix-3]**: 가속도가 없는 경우(빈 HOT2) 억지로 채우지 않고 비워둠으로써 신호의 순수성을 유지합니다.

---

## 3. 급등 (Rocket/GAIN)
가격 상승폭과 체결 에너지의 하이브리드 결합 지표입니다.

### 3.1 산출 공식
$$HybridScore = Chg \times \sqrt{NormPower}$$

- **$Chg$**: 당일 등락률.
- **$NormPower$**: 0~2000% 범위로 정규화된 체결강도.
    - **Anomaly Recovery [Fix-2]**: 수만~수십만%로 오염된 데이터를 자릿수(10000, 1000, 100)별로 역산하여 0~2000% 범위 내로 실시간 복구합니다. 복구 후에도 범위를 벗어나면 원천적으로 차단합니다.

---

## 4. 거래대금 (Liquidity/VALUE)
시장의 0차 진실인 유동성 데이터를 제공합니다.

- **필터링**: ETF, ETN, 선물, 우선주, 스팩(SPAC) 등 파생/특수 종목을 키워드 기반으로 원천 차단합니다.
- **Damping**: -7% 이하 급락 종목은 순위 산정에서 제외하거나 강력한 감쇄(Damping)를 적용합니다.

---

## 5. 특별 전시 테이블 (Special UI Tables)

| 이름 | 표시 개수 | 정렬 기준 | 특징 |
| :--- | :---: | :--- | :--- |
| **HEAT** | 50 | Smart Heat Index 순 | 시장 주도주 및 수급 집중주 포착 |
| **HOT2** | 50 | Acceleration Score 순 | 단기 급발진 및 수급 유입 에너지 포착 |
| **GAIN** | 50 | Hybrid Score 순 | 가격 탄력성과 체결 강도 동시 충족주 |
| **VALUE** | 50 | 거래대금 순 | 대형주 및 시장 유동성 근간 확인 |

---
*Document Version: V22.0*
*Last Updated: 2026-04-10 (Ranking Manager V22.0 Migration)*
