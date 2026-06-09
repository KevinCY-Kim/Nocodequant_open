# Reason Code 3계층 분류 체계 (Reason Code Architecture)

> 출처: ArmoraAI 차단 사유 체계 → NCQ 주식 시장 도메인 적응
> Tier: 2 (Policy)

---

## 1. 목적 (Purpose)

본 문서는 NoCodeQuant 시스템이 매매 신호를 **거부(Reject)**할 때 부여하는 차단 사유(Reason Code)의 **3계층 분류 체계**를 정의한다.

> "방어가 곧 알파다(Defense as Alpha)."
> 시스템이 매매를 *하지 않기로* 결정한 것은, 자본을 보존하기로 결정한 것과 기능적으로 동일하다.

---

## 2. 3계층 분류 (Three-Layer Classification)

### Layer 1: 인지 필터 (Cognitive Filters — 전략 품질)

> "AI가 현재 시장을 이해하고 있는가?"

| Reason Code | 설명 | NCQ 대응 Guard |
|:------------|:-----|:---------------|
| `NEUTRAL_ZONE` | 점수가 40~60점 중립 구간 | `Scoring Buffer` (Failure Patterns §3.2) |
| `LOW_CONFIDENCE` | 신뢰도가 `MIN_CONFIDENCE` 미만 | `rules/guards/LOW_CONFIDENCE.md` |

### Layer 2: 구조적 안전장치 (Structural Guards — 시스템 무결성)

> "시스템이 실행 가능한 안정 상태인가?"

| Reason Code | 설명 | NCQ 대응 Guard |
|:------------|:-----|:---------------|
| `STARTUP_GUARD` | 시스템 기동 후 데이터 안정화 대기 | `rules/guards/STARTUP_GUARD.md` |
| `STAMP_DUPLICATE` | 동일 봉/방향 중복 신호 차단 | Failure Patterns §2.2 (Double Entry) |
| `POSITION_CONFLICT` | 포지션 무결성 위반 (보유 중 재진입) | Order Lifecycle FSM |

### Layer 3: 시장 환경 필터 (Market Regime Filters — 환경 적합성)

> "현재 시장 환경이 전략에 적합한가?"

| Reason Code | 설명 | NCQ 대응 Guard |
|:------------|:-----|:---------------|
| `VSA_GATE` | 거래량 확증 실패 (수급 미달) | `rules/guards/VSA_GATE.md` |
| `GLOBAL_BEAR` | HTF 이동평균 하회 (하락장 판단) | Parameter Invariants §2.4 |
| `CRASH_GUARD` | 당일 등락률 -7% 초과 급락 | Market_Leader_Filter §2단계 |

---

## 3. 운영 활용 가이드

Reason Code 분포는 시스템과 시장 간 **적합도(Fit)**를 판단하는 핵심 건강 지표이다:

| 우세 계층 | 의미 | 권장 조치 |
|:---------|:-----|:---------|
| **Layer 1 우세** | 전략 예측력 저하 (높은 엔트로피) | 진입 임계값 상향 또는 전략 재검토 |
| **Layer 2 우세** | 시스템 과도 제약 또는 불안정 | 파라미터 튜닝 필요 |
| **Layer 3 우세** | 시장 비활성 또는 극단적 환경 | 변동성 회복까지 대기 |

---

## 4. 선물 도메인 제거 확인

- ✅ 레버리지/마진 용어 없음
- ✅ HMM/OVS 전략 참조 없음
- ✅ 틱 기반 파라미터 없음
- ✅ 모든 Guard가 NCQ 기존 문서에 매핑됨
