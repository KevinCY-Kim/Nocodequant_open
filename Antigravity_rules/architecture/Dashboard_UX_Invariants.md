# Dashboard UX 불변 조건 (Dashboard UX Invariants)

> 출처: ArmoraAI Dashboard_Invariants + Dashboard_UX_Principles → NCQ 도메인 적응
> Tier: 3 (Architecture)

---

## 1. 데이터 무결성 (Data Integrity)

- **Read-Only 원칙**: 대시보드(UI)는 백엔드 데이터를 **읽기만** 수행한다. UI에서 매매 판단 데이터를 변조하는 것은 금지된다 (헌장 §1.3).
- **SSoT 충실성**: 모든 시각적 데이터는 `StrategyManager` 또는 `ExecutionManager`의 계산 결과에 충실해야 한다.

---

## 2. 색상 체계 (Color Convention)

| 상태 | 색상 | 코드 | 한국 시장 관습 |
|:-----|:-----|:-----|:-------------|
| 수익 / 상승 | 빨간색 | `#f44336` | 한국은 상승=빨강 |
| 손실 / 하락 | 파란색 | `#2196f3` | 한국은 하락=파랑 |
| 중립 / 대기 | 회색 | `#eeeeee` | — |
| 위험 / 경고 | 주황색 | `#fd7e14` | — |

> 색상의 의미는 **전역적으로 일관**되어야 하며, 컴포넌트마다 임의 변경 금지.

---

## 3. 시간축 무결성 (Time Axis Integrity)

- **고유 타임스탬프**: 차트 렌더링에 사용되는 타임스탬프는 **고유(Unique)** 해야 한다. 중복 시 최신 값 1개만 유효.
- **엄격 오름차순**: 시각화 데이터는 **엄격한 오름차순(Strictly Ascending)** 정렬 필수. 역전 데이터는 무시 또는 보정.
- **메모리 관리**: 차트 데이터 포인트는 최대 **200개** 유지. 초과 시 가장 오래된 데이터부터 삭제 (Performance_Analytics_Design §3.2 참조).

---

## 4. 인지 부하 최소화 (Cognitive Load Reduction)

- **지표 투명성**: 핵심 지표(승률, MDD, 신뢰도 등)는 마우스 오버 시 **Tooltip**으로 정의와 해석 가이드를 제공해야 한다.
- **정보 밀도 제한**: 단일 화면에 표시하는 핵심 수치는 **7±2개** 이내를 목표로 한다 (밀러의 법칙).
- **상태 아이콘**: 복잡한 수치 대신 의미 아이콘(`🐻`, `⚠️`, `🔥`)으로 1차 인지를 지원한다.

---

## 5. 선물 도메인 제거 확인

- ✅ HMM/OVS Source Filtering 제거
- ✅ Observation/Review Pressure 개념 제거 (ArmoraAI 전용)
- ✅ WAL Polling HTMX 방식 제거 (NCQ는 PyQt5 단일 프로세스)
- ✅ 모든 참조가 NCQ 기존 문서에 매핑됨
