# 시각화 엔진 아키텍처 사양서 (Visualization Engine Spec)

본 문서는 NoCodeQuant 시스템의 시각화 엔진(차트 및 데이터 맵)에 대한 아키텍처 표준과 불변성 규칙을 정의합니다.

## 1. Intent 기반 Reducer 패턴 (Intent-based Reducer)
UI 상태의 안정성과 추적 가능성을 보장하기 위해 모든 네비게이션은 순수 Reducer 패턴을 따릅니다.

- **Intents**: 모든 UI 이벤트(예: `SHIFT_TO`, `ZOOM`)는 불변의 Intent 객체 또는 문자열로 발행되어야 합니다.
- **Pure Reducer**: `ViewportReducer`는 `(State, Intent, Value) -> NewState` 형태의 순수 함수여야 합니다. 부수 효과(Side-effect)나 외부 의존성이 없어야 합니다.
- **불변성 규칙(Invariants)**:
    - `start_idx`는 항상 `[0, total_len - view_count]` 범위 내로 제한(Clamp)되어야 합니다.
    - `view_count`는 시스템이 정의한 한계치(예: 10-500) 내에 있어야 합니다.
    - **우측 앵커 줌 (Anchor-Right Zoom)**: 줌 동작 시 데이터 경계에 도달하지 않는 한, 시각적인 우측 끝 인덱스는 고정되어야 합니다.

## 2. 좌표계 불변성 (Coordinate System Invariants)
사용자의 인지 부하를 줄이고 시각적 정밀도를 유지하기 위해 차트 영역은 엄격한 경계 규칙을 따릅니다.

### 2.1 가격/거래량 분리 (80/20 규칙)
- **주 영역 (80%)**: 가격 액션(캔들) 및 기술적 지표(MA, BB)를 위해 예약됩니다.
- **보조 영역 (20%)**: 거래량 바 및 오실레이터형 지표(RSI)를 위해 예약됩니다.
- **절대 경계 (Absolute Boundary)**: 좌표 변환 시 `price_bottom` 한계점를 설정하여 지표 간의 기하학적 겹침(Overlap)을 0%로 유지합니다.

## 3. 고성능 렌더링 (High-Performance Rendering)
- **Backing Store (QPixmap)**: 정적 데이터(캔들, 지표)는 Backing Store에 사전 렌더링되어야 합니다.
- **오버레이 레이어 (Overlay Layer)**: 동적 요소(크로스헤어, 툴팁)는 별도의 패스에서 렌더링하여 커서 이동 시 지연 없는(Zero-Lag) 반응성을 보장합니다.
- **업데이트 가드**: 실시간 데이터 주입 시 `patch_data()` 경로를 활용하여 전체 기하 구조의 재계산을 방지합니다.

## 4. 네비게이션 UX 표준
- **데이터 맵 스케일링**: 스크롤바 핸들 크기는 전체 데이터 대비 가시 영역 비율(`view_count / total_len`)에 비례해야 합니다.
- **정밀 트레이싱**: 드래그 스크롤 시 정수 반올림 오차를 방지하기 위해 픽셀 누적기(Pixel Accumulator)를 사용하여 부드러운 이동을 구현합니다.

---
*버전: v1.0 (2026-02-19 개정)*
