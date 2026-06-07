# 2026-02-27 작업 기록 (V18.0 Minimal Soft Entry Architecture)

## 헌장 준수 확인 (AI Work Constitution Compliance)

### 작업 등급 식별 (Category Identification)
- **등급**: **L4** (아키텍처 및 데이터 흐름 변경)
- **근거**: 엔진의 진입 판단 로직을 결정론적 AND 게이트 구조에서 확률적 가중합(Soft Scoring) 구조로 전면 전환. 이는 의사결정 방식(Decision Flow)과 신호 제어(Signal Gating)의 아키텍처적 근예를 변경하는 고부하 작업임.

### 참조 문서 (Doc Reference)
| 문서 | 역할 |
|------|------|
| `Antigravity_rules/governance/AI_Work_Constitution_and_Reference_Map.md` | 최상위 거버넌스 (v1.2) |
| `Antigravity_rules/rules/Signal_Gating_Rules.md` | 신호 제어 규칙 (v18.0 업데이트) |
| `Antigravity_rules/rules/core/Parameter_Invariants.md` | 파라미터 불변성 (v18.0 업데이트) |

### 영향 모듈 (Impacted Modules)
| 모듈 | 변경 유형 | 영향도 |
|------|-----------|--------|
| `app/ai/strategy.py` | L4 Core | `_calc_entry_score` 도입, AND 게이트 제거, DTO 필드 확장 |
| `app/core/config_engine.py` | L2 Config | `ENTRY_THRESHOLD`, `ENTRY_W_*` 파라미터 5건 등록 |
| `app/ui/main_window.py` | L1 Visual | S/E 복합 스코어 표시, 4블록 기반 스테이지, 콤팩트 툴팁 적용 |

### 불변성 검증 (Invariant Check)
- **Violated**: None
- **Hardcoding 금지**: ✅ 모든 점수 가중치 및 임계값을 `ACTIVE_CONFIG`로 관리 (매직넘버 제거)
- **경계 존중**: ✅ UI에서 매매 판단 로직 없음, `main_window.py`는 `entry_detail` 블록 점수를 단순 시각화하는 역할에 충실
- **금지 패턴**: ✅ 하드코딩된 RSI/Volume 조건 제거 및 가중치 파라미터화

---

## 오늘 완료된 태스크

### 1. [L4] Minimal Soft Entry Architecture 전환

**목적**: 기존 6중 AND 게이트의 과도한 필터링으로 인한 기회손실(매매 빈도 저하)을 해결하고, 확률 우위에 기반한 반복 매매가 가능한 유연한 진입 구조로 진화.

**변경 내용 (V18.0)**:
- **Layer 1 (Hard Safety)**: 자본 보호를 위한 3대 게이트(Warmup, Extreme Volume, Data Integrity)만 강제 유지.
- **Layer 2 (Soft Score)**: 4대 핵심 블록 가중합으로 `entry_score` 산출.
    - **Core Technical (40%)**: MA+RSI+BB 종합 (기존 `raw_score` 재활용, 신뢰도 이중 패널티 제거).
    - **Volume Quality (20%)**: 거래량 강도 구간별 점수화.
    - **Price Trigger (20%)**: 전봉 돌파 수준 계층화 + 과매도 완화 로직(RSI<35).
    - **Risk Block (20%)**: 신뢰도(Confidence) + 국면(Regime) + 장기추세(MA240) 통합 관리.

---

### 2. [L2] 전략 파라미터 SSoT 확립

**내용**: 엔진 오버라이드 및 하드코딩 요소를 제거하고 `config_engine.py`를 통한 단일 제어 체계 구축.
- `ENTRY_THRESHOLD = 62` (GPT/Grok 권장 균형값 적용)
- `WARMUP_MIN_BARS = 50` 복원 (헌장 준수)
- 각 블록별 가중치(`ENTRY_W_CORE/VOL/PRICE/RISK`) 명시적 등록.

---

### 3. [L1] UI/UX 가독성 및 반응성 고도화 (V18.1~V18.2)

**문제**: S/E 수치 혼선, 신뢰도 하락 시 신호등 소등 지연, 툴팁 크기 과다.

**해결**:
- **Score Composite**: `S:41 | E:52` 포맷으로 지표 점수(S)와 매수 확률(E)을 명확히 분리.
- **Tooltip**: S|E 상세 설명 및 블록별 충족 현황을 보여주는 **Compact Style** 툴팁 표준화.
- **Latency**: 신뢰도 Blackout 임계값을 30% → 15%로 완화하여 시장 변화에 대한 시각적 반응 속도 개선.
- **Staging**: 4개 블록 충족 개수(0/4 ~ 4/4)에 따른 날씨 아이콘 및 단계 정보 제공.

---

## 아키텍처 비교 (V17.0 vs V18.0)

| 항목 | 기존 (V17.0) | 신규 (V18.0) |
|------|--------------|--------------|
| 진입 논리 | 6중 AND Gate (Strict) | 4-Block Soft Score (Probabilistic) |
| 신뢰도 처리 | 점수에 직접 곱셈 (Double Penalty) | 리스크 블록 내 가중치 편입 (Single Penalty) |
| 판단 지점 | UI Display Score 위주 | Backend Entry Score 기준 |
| 기대 빈도 | 일 0~1회 (패턴 완성형만) | 일 2~5회 (확률적 우위형) |

---

## 향후 주요 목표 (Roadmap)
- **Live Trading 샘플링**: V18.0 기반의 실제 매매 성공률 및 슬리피지 분석.
- **블록 확장**: 거래원(Orderbook) 또는 수급(Hot Index) 블록을 추가하여 정교함 고도화.
- **UI 애니메이션**: Entry Score 임계값(62) 근접 시 게이지 깜빡임 효과 등 시각적 큐 강화.

---
**Status: ✅ L4 Architecture Upgrade Complete | ✅ 4-Block Soft Implementation | ✅ UI Clarity & Tooltip Refined | 🛡️ Constitution v1.2 Compliant**
