# 워크로그: MainWindow 리팩토링 Step 2 (Builder 분리)

**날짜**: 2026-04-12
**상태**: ✅ 검증 완료
**작업 등급**: L4 (아키텍처 변경)

## 1. 작업 개요
모놀리식 구조였던 `MainWindow`의 UI 생성 로직을 독립적인 `Builder` 클래스들로 분리하여 코드 가독성과 유지보수성을 향상시켰습니다. 또한, 불필요한 중복 메서드와 커넥션을 제거하여 시스템의 안정성을 강화했습니다.

## 2. 주요 변경 사항

### 2.1 코드 규모 최적화
- **원본**: 3526라인
- **Step 1 + 사전정지**: 3493라인
- **Step 2 (현재)**: 2934라인
- **총 감소량**: -592라인 (-16.8%)

### 2.2 Builder 패턴 도입
UI 구성 로직을 다음과 같은 클래스들로 분리했습니다 (`app/ui/builders/`):
- `HeaderBuilder`: 계좌 정보, 상태바, 대시보드 버튼 등 상단 UI 구성
- `BodyBuilder`: 메인 제어판, 파라미터 설정, MME 시뮬레이터 등 본문 UI 구성
- `SidebarBuilder`: 우측 순위 위젯 및 포지션 테이블 구성 (DockWidget 연동)
- `ChartPanelBuilder`: 메인 차트 및 성능 요약 패널 구성

### 2.3 죽은 코드(Dead Code) 및 중복 제거
- `on_ranking_selected_REPLACED` 삭제
- `_on_trade_fill_pnl_sync` 메서드 및 관련 커넥션 삭제
- `save_pnl_history` 삭제 (Ledger 기반으로 통합)
- `load_pnl_history` 주석 처리된 죽은 코드 삭제
- `open_forensic_dashboard` 중복 정의 제거 (최신 버전인 L1680만 유지)
- `on_ranking_fav_add` 통합본(L1201)으로 단일화
- `_on_health_status` 스레드 가드, 툴팁, None 처리를 병합하여 하드닝 (L736)
- `sidebar btn_scope_settings` 중복 커넥션 제거
- `state_mgr.set_engine_state` 호출 위치를 `__init__` (L284)으로 이동하여 초기화 정합성 확보

## 3. 구조 최신화
- `app/ui/builders/` 디렉토리 신설 및 `__init__.py` 추가
- `Antigravity_rules/architecture/Project_Structure_Map_ko.md`에 신규 구조 반영 완료

## 4. 검증 결과
- **컴파일/부팅**: ✅ 정상 (V16.0 Stability Patch 로드 확인)
- **상태 관리**: ✅ StateManager와 UI 간 동기화 정합성 확인
- **이벤트 버스**: ✅ 리플레이 모드 및 헬스 텔레메트리 연동 정상
