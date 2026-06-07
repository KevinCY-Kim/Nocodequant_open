# 워크로그: MainWindow 리팩토링 및 모듈화 단계별 진행 기록

**날짜**: 2026-04-12
**상태**: ✅ Step 3B 완료 (안정화 단계)
**작업 등급**: L4 (아키텍처 변경 및 데이터 흐름 재설계)

---

## 1. 누적 성능 지표 (Cumulative Metrics)

리팩토링 단계별 `MainWindow` 코드 변화량입니다.

| 단계 | 주요 작업 | 라인 수 | 변화량 | 비율 |
| :--- | :--- | :---: | :---: | :---: |
| **Origin** | 리팩토링 시작 전 모놀리식 상태 | 3,526 | - | 100% |
| **Step 1** | 사전 정지 및 간단한 로직 정리 | 3,493 | -33 | 99.1% |
| **Step 2** | UI Builder 분리 (Header/Body/Sidebar/Chart) | 2,934 | -559 | 83.2% |
| **Step 3A** | ConfigManager 위임 및 도메인 재배치 | 2,624 | -310 | 74.4% |
| **Step 3B** | POEHealthPanel 분리 및 콜백 개선 | **2,423** | **-201** | **68.7%** |
| **누적 결과**| **총 1,103라인 최적화** | - | **-1,103** | **-31.3%** |

---

## 2. 단계별 상세 내역 (Step-by-Step Details)

### [Step 1] 사전 정지 및 안정화
- **내용**: 32비트 환경에서의 키움 OCX 로드 안정화 및 인코딩 오류(CP949 이모지) 해결.
- **결과**: 엔진 초기화 시 안정성 확보.

### [Step 2] UI Builder 패턴 도입 (Decoupling)
- **내용**: `MainWindow`의 거대한 `setup_ui` 로직을 4개의 독립 빌더로 분리.
    - `HeaderBuilder`, `BodyBuilder`, `SidebarBuilder`, `ChartPanelBuilder`
- **제거**: `on_ranking_selected_REPLACED`, `save_pnl_history` 등 레거시/중복 코드 대거 삭제.
- **개선**: `state_mgr.set_engine_state` 호출 위치 조정을 통한 초기화 정합성 확보.

### [Step 3A] ConfigManager 위임 및 재배치 (Delegation)
- **내용**: 설정 관리 로직(Group A 메서드)을 `ConfigManager`로 위임.
- **재배치**: `app/services/managers/`에 있던 `config_manager.py`를 UI 전용 매니저 폴더인 `app/ui/managers/`로 이동.
- **위임 메서드**:
    - `get_ui_config`, `save_config`, `load_config`, `reset_config`
    - `init_runtime_config_from_ui`, `apply_initial_defaults`, `apply_preset`

### [Step 3B] POEHealthPanel 분리 및 콜백 개선 (Delegation & Bugfix)
- **내용**: `MainWindow`에 남아있던 POE(수익 최적화 엔진) 상태 및 튜닝 제어 로직을 `POEHealthPanel`로 위임.
- **신설**: `app/ui/managers/poe_health_panel.py`
- **버그 수정 (MME Preview Callback)**:
    - *문제점*: `premium_header_widget`의 `parent()` 체인이 실제 `MainWindow`에 도달하지 않아 `update_mme_preview`가 호출되지 않는 버그 발생.
    - *해결*: 레이아웃 중첩도에 의존하는 안티패턴을 배제하고, `set_mme_callback`을 통해 콜백 함수(`update_mme_preview`)를 직접 주입받아 실행하도록 `PremiumHeaderWidget` 및 `BodyBuilder`의 코드를 결합도(Decoupling) 관점에서 수정(콜백 패턴 적용).

---

## 3. 구조 최신화 현황 (Structure Map Sync)

`Antigravity_rules/architecture/Project_Structure_Map_ko.md`에 다음 변경 사항이 반영되었습니다.
- [NEW] `app/ui/managers/`: UI 전용 제어 매니저 계층 신설
    - `config_manager.py`: 실행 계층에서 표현 계층 매니저로 도메인 이동
    - `poe_health_panel.py`: POE 상태/건강 지표 및 튜닝 패널 위임 목적 신규 파일 생성
- [SYNC] `app/ui/builders/`: Step 2에서 신설된 빌더 계층 명시

---

## 4. 최종 검증 결과
- **컴파일/부팅**: ✅ 정상 (V16.0 Stability Patch 및 ConfigManager/POEHealthPanel 로드 확인)
- **기능 정합성**: ✅ POE(수익 최적화 엔진) 튜닝 결과 수신 및 UI 패널 콜백 동작 이상 없음
- **버그 해결**: ✅ `parent()` 체인 버그가 수정되어 `update_mme_preview` 콜백 실시간 연동 완료
- **환경 적응**: ✅ 32비트 Python 환경에서 키움 OCX 시그널 연결 유지
- **거버넌스**: ✅ AI 작업 헌장 v1.8에 따른 L4 변경 절차 준수 완료
