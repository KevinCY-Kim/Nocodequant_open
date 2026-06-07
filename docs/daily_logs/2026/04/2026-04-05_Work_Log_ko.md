# 작업일지: V20.9 Time-Explorer Hardening & Analytics UX (2026-04-05)

## 0. 작업 등격 및 필수 체크리스트 (Compliance)

*   **작업 등급**: **L3 (Institutional-Grade Logic & UI Hardening)**
*   **필수 참조 문서**:
    - [공식 헌장](file:///C:/Users/stone/projects/nocodequant/Antigravity_rules/governance/AI_Work_Constitution_and_Reference_Map.md)
    - [구조 맵](file:///C:/Users/stone/projects/nocodequant/Antigravity_rules/architecture/Project_Structure_Map_ko.md)
*   **영향 받는 모듈**: `range_calendar.py`, `strategy_dashboard.py`
*   **불변성 확인**: 위반 없음. 단일 진실 공급원(SSoT) 원칙에 따라 `selectedDate()` 기반 좌표 매핑 수행.
*   **경계 잠금**: 수정 범위는 명시된 UI 모듈 내로 엄격히 제한됨.

---

## 1. 📅 RangeCalendar v5 — 5대 버그 수정 (Hardening)

단순한 UI 버그가 아닌, Qt 엔진의 타이밍 및 이벤트 가로채기 설계 결함을 해결하는 **뿌리 단계의 수정(Root-Level Fix)**을 완료했습니다.

| ID | 분류 | 내용 | 조치 사항 |
| :--- | :--- | :--- | :--- |
| **BUG-1** | 크리티컬 | `self.viewport` 이름 충돌 | `self._vp`로 리네임하여 내장 메서드 오염 방지 |
| **BUG-2** | 중간 | 드래그 중 이탈(Leave) 클린업 | `leaveEvent` 처리로 마우스 이탈 시 드래그 상태 자동 해지 |
| **BUG-3** | 중간 | `update()` 충돌 | BUG-1 해결을 통해 `AttributeError` 원천 차단 |
| **BUG-4** | 중간 | 흐릿한 날짜(Faded Cell) 간섭 | `_get_date_from_pos()`에 월 범위 검증 로직 추가 |
| **BUG-5** | 낮음 | 기본 클릭 동작 간섭 | `return True` 전략을 정교화하여 드래그 시 날짜 강제 이동 차단 |

---

## 2. 📊 StrategyDashboard — 9대 UX 개선 (Analytics UX)

분석의 깊이와 컨텍스트 인지력을 높이기 위한 **Prop-Grade** 기능들을 대거 투입했습니다.

### 2.1 비교 분석 도구 고도화
- **[NEW] CompareRangeDialog**: 수동 고정 모드에서 B기간을 드래그 달력으로 정교하게 선택 가능.
- **[NEW] B-Range Snapshot**: 마지막으로 사용한 수동 B기간을 기억하고 버튼 하나로 즉각 복원(Snapshot).
- **[NEW] lbl_b_range**: 현재 고정된 B기간을 컨트롤 패널에 상시 표시하여 상황 인지력 강화.

### 2.2 메타데이터 시각화 (Context Awareness)
- **[NEW] 기간 편차 산출**: 배너에 A와 B의 기간 길이 차이(동일/차이 일수)를 자동 표기하여 통계적 왜곡 방지.
- **[NEW] 타이틀 뱃지**: 비교 모드 활성 시 윈도우 제목에 `●` 표시를 추가하여 모드 상태 직관화.
- **[FIX] 동기화 버그**: `combo_b_sync` 변경 시 무반응이던 이슈를 해결하여 즉각적인 분석 대전환 가능.

### 2.3 시스템 하드닝
- **Style Consistency**: `_combo_style()` / `_preset_btn_style()` 헬퍼로 스타일 정의 일원화.
- **Clean Delta**: 비교 모드 OFF 시 잔류하던 델타(🔼/🔽) 수치를 명시적으로 소거하도록 로직 보강.

---

## 3. 검증 결과 (Verification Results)

- [x] **드래그 무결성**: 달력 월 경계 및 위젯 외부 이탈 시에도 크래시 없이 상태가 유지됨.
- [x] **비교 모드 스위칭**: 자동(슬라이딩) ↔ 수동(앵커링) 전환 시 데이터와 UI가 0.3초 내 동기화됨.
- [x] **표기 정확도**: A, B 기간의 일수가 다를 경우 배너에 "A기간 3일 더 김" 등의 정보가 정확히 산출됨.

---
> [!NOTE]
> 이제 **NCQ Time-Explorer**는 단순히 날짜를 정하는 도구가 아니라, 과거의 특정 국면(B)과 현재(A)를 한눈에 대조하며 전략의 우위를 판별하는 **지능형 분석기**가 되었습니다.

---

## 4. 🧩 Institutional Refactoring v1 (V20.10.1)

`main_window.py`의 비대화(4,629줄)를 해결하고 유지보수성을 극대화하기 위해, 독립적인 UI 컴포넌트들을 물리적으로 분리하는 **Module Segregation**을 단행했습니다.

### 4.1 컴포넌트 모듈 분리 (6개)
- **[NEW] signal_status_widget.py**: Decision Hub 핵심 로직 및 히스토그램 분리 (616줄)
- **[NEW] ranking_widget.py**: 시장 주도주 랭킹 및 탭 제어 로직 분리 (313줄)
- **[NEW] premium_header_widget.py**: 상단 고밀도 헤더 및 시세 연동 로직 분리 (173줄)
- **[NEW] recommendation_widget.py**: 실시간 스캐너 추천 종목 위젯 분리 (123줄)
- **[NEW] trading_scope_dialog.py**: 자동매매 범위 설정 다이얼로그 분리 (70줄)
- **[NEW] signal_status_widget.py**: `ScoreHistogramWidget` 포함 통합 관리

### 4.2 구동 안정성 패치 (Stability Patch)
- **Context Injection**: 분리된 `SignalStatusWidget`이 AI 분석 캐시(`ai_cache`)에 접근할 수 있도록 `main_window`에서 `context`를 명시적으로 주입.
- **Signal Connectivity**: `RankingWidget` 내부에 누락되었던 [⚙️ 설정] 버튼과 `open_scope_settings` 메서드를 `main_window`에서 수동 연결하여 기능 무응답 이슈 해결.
- **Strict Exports**: `widgets/__init__.py` 및 `dialogs/__init__.py`에 클린 export 구문을 추가하여 패키지 접근성 강화.

---

## 5. 🚀 향후 과제 (Next Steps - V20.10.2+)

### Step 1: MainWindow Internal Mixins (Segregation)
- 3,471줄 남은 `MainWindow` 내부 로직을 기능별 Mixin 클래스로 2차 분리
- `ConfigMixin`: 설정 로드/저장 로직 (save_config, load_config)
- `RankingMixin`: 실시간 데이터 수신 및 랭킹 처리 로직
- `ReplayMixin`: 시뮬레이션 모드 전환 및 인터페이스 락 제어 로직

### Step 2: RangeCalendar One-Week Bug (Re-Analysis)
- `indexAt` 기반 V6의 부작용을 분석하고, 좌표 매핑 무결성을 유지하면서 1주일 밀림 현상을 박멸하는 **Surgical Math Fix** 적용.
