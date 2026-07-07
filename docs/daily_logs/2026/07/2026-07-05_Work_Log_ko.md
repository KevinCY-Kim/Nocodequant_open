# 📅 2026-07-05 업무 일지 (Daily Work Log)

> **[거버넌스 준수]**
> * **작업 등급**: L3 (프리미엄 React 웹 대시보드 구축 및 찐테마 점유율 랭킹 개선)
> * **참조 문서**:
>   * [ranking_manager.py](file:///c:/Users/stone/projects/nocodequant/app/services/managers/ranking_manager.py)
>   * [theme_report_window.py](file:///c:/Users/stone/projects/nocodequant/app/ui/theme_report_window.py)
>   * [rotation_web_server.py](file:///c:/Users/stone/projects/nocodequant/app/ui/web/rotation_web_server.py)
>   * [rotation_dashboard.html](file:///c:/Users/stone/projects/nocodequant/app/ui/web/rotation_dashboard.html)
> * **영향 받는 모듈 (Impacted Modules)**: `ranking_manager.py`, `theme_report_window.py`, `rotation_web_server.py` (신설), `rotation_dashboard.html` (신설)
> * **불변성 위반 여부 (Invariant Check)**: 위반 없음 (Violated: None)
> * **경계 준수 선언**: 기존 QSS 스타일 Qt UI의 디자인적 한계를 극복하기 위해 루프백 기반의 경량 React 웹서버를 신설하고 프리미엄 다크 모드 대시보드를 브라우저에 띄우도록 개선하였습니다. 찐테마 랭킹의 단순 사이클 기반 정렬 오류를 수정하고, 도넛 차트의 시각적 혼동을 유발하던 중앙 앵커 값을 정렬 누락 없이 보정하였습니다.

---

## 🎯 주요 목표 및 이슈 정의

1. **프리미엄 React 웹 대시보드 런칭**: Qt 기본 렌더러의 시각적 한계 및 차트 디자인 한계를 극복하고, 와이드 화면에 한눈에 들어오는 프리미엄 다크 모드 웹 대시보드를 구축.
2. **찐테마 정렬 왜곡 해소**: 단순 `hot_cycles` (주도 사이클 수) 기준 정렬 시, 주도 30사이클 vs 28사이클의 미세한 차이에서 전체 시장 점유율이 2배(29.4% vs 15.3%)나 차이 나는 테마가 하위로 밀리는 모순 수정.
3. **점유율 도넛 차트의 시각적 괴리 보정**: 점유율 차트에서 도넛 중앙에 찐테마명이 출력되고, 정렬되지 않은 슬라이스가 노출되어 사용자가 "점유율 1위 테마가 왜 중앙에 안 나오는가" 겪던 시각적 혼선 차단.

---

## ✅ 상세 작업 및 패치 내역

### 1. React UMD U1-Screen 웹 대시보드 및 경량 API 서버 구현
- **rotation_web_server.py (신규)**:
  - 127.0.0.1 루프백 주소 전용 경량 HTTP 서버 구축.
  - Windows의 `SO_REUSEADDR` 이중 바인딩 함정을 피하기 위해 `allow_reuse_address=False` 옵션 사용.
  - 포트 충돌 방지를 위해 8765 ~ 8775 포트를 순차 검색하며 데몬 스레드로 안전하게 구동.
  - `/` 경로로 React 대시보드 정적 HTML 서빙, `/api/rotation` 경로로 메모리 상에 캐싱된 리포트 JSON 스냅샷 서빙.
- **rotation_dashboard.html (신규)**:
  - React 18 UMD 및 Babel Standalone CDN을 활용하여 빌드 과정 없는 단일 HTML 프리미엄 UI 설계. (오프라인 폴백 가이드 문구 포함)
  - KPI 4종, SVG 기반 점유율 도넛 차트, 찐테마 랭킹 테이블, 타임라인(🌙 마커 포함), 오버나이트 레이더, LIVE 스냅샷 시각화 탑재. 10초 단위 자동 갱신.
  - 와이드 모니터에 최적화된 1화면 레이아웃(1438x909 뷰포트 내 전 섹션 수납, 내부 스크롤 적용).

### 2. 찐테마(주도 테마) 정렬 점수 수식 개편
- **ranking_manager.py (`get_rotation_report`)**:
  - 기존 `hot_cycles` 기준 정렬에서 **`hot_cycles × heat_sum`** ("주도한 시간 × 판의 크기") 수식으로 변경.
  - 이로써 주도 지속 시간과 전체 거래 강도를 종합 반영하여 왜곡 현상 해소. (주도 0사이클은 여전히 점수 0점으로 후보군에서 자동 탈락하는 정합성 유지)

### 3. 점유율 도넛 차트 정합성 수정
- **rotation_dashboard.html / theme_report_window.py**:
  - 웹 ShareBox에서 데이터 재정렬 누락을 보완하여 share 내림차순(점유율순) 정렬 후 상위 7개 슬라이스 표출.
  - 도넛 차트 중앙에 노출되는 테마를 기존 daily[0](주도 1위)에서 실제 당일 점유율 1위 테마(by_share[0])로 변경 (Qt/웹 공통).
  - 찐테마(주도 지속 1위)와 점유율 1위 테마가 불일치할 수 있는 설계적 의미를 살리고 시각적 정합만 확보.

---

## 🧪 검증 및 테스트 결과

### 1. 통합 동작 및 포트 바인딩 테스트
- 웹 서버 포트 바인딩, API 직렬화 실패 가드 및 404 폴백을 테스트하는 서버 전용 테스트 15건 올 PASS.
- 찐테마 랭킹 가중치 수식 개선에 따른 랭킹 순위 교차 검증을 포함한 기존 테스트 63건 올 PASS.
- 실브라우저 1438x909 환경 렌더링 검사 완료(콘솔 에러 0건, 스크롤 바운딩 박스 완벽 수납 확인).

---

## 🔮 향후 계획 및 최종 의도

1. **대시보드 라이브 렌더링 유지**: Qt 리포트 창 내부 새로고침 주기마다 웹 스냅샷을 PUSH하는 구조가 장시간 구동 시 스레드 락 없이 원활히 작동하는지 실전 모니터링.
2. **React UMD 로딩 안정성**: 네트워크 불량 등으로 CDN 접속 실패 시 오프라인 가이드가 친절하게 표시되는지 시나리오 점검.
