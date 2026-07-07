# 📅 2026-07-06 업무 일지 (Daily Work Log)

> **[거버넌스 준수]**
> * **작업 등급**: L2 (대장 교체 타임라인 이벤트 로깅 및 오버나이트 UI 표기 개선)
> * **참조 문서**:
>   * [ranking_manager.py](file:///c:/Users/stone/projects/nocodequant/app/services/managers/ranking_manager.py)
>   * [theme_report_window.py](file:///c:/Users/stone/projects/nocodequant/app/ui/theme_report_window.py)
>   * [rotation_dashboard.html](file:///c:/Users/stone/projects/nocodequant/app/ui/web/rotation_dashboard.html)
>   * [test_theme_rotation_report.py](file:///c:/Users/stone/projects/nocodequant/tests/test_theme_rotation_report.py)
>   * [README.md](file:///c:/Users/stone/projects/nocodequant/README.md)
> * **영향 받는 모듈 (Impacted Modules)**: `ranking_manager.py`, `theme_report_window.py`, `rotation_dashboard.html`, `test_theme_rotation_report.py` (신설), `README.md`
> * **불변성 위반 여부 (Invariant Check)**: 위반 없음 (Violated: None)
> * **경계 준수 선언**:  테마의 대장주가 교체될 때 별도의 로그 없이 조용히 처리되어 인지하기 어렵던 부분을 명확한 타임라인 이벤트로 자동 기록하도록 개선하였습니다. 오버나이트 후보 표기 시 음수 마진에 대한 이중부호 버그를 교정하고, 작성된 마켓 로테이션 리포트에 대해 8가지 유스케이스 기반 영구 회귀 테스트를 보강 완료하였습니다.

---

## 🎯 주요 목표 및 이슈 정의

1. **대장주 실시간 변동 탐지 강화**: 테마 내 1위 대장주가 수시로 바뀌는데도 별도 알림이 없어 오독되던 구조를 개선하고, 대장주 교체 사건을 시각적으로 파악할 수 있도록 타임라인 이벤트 화.
2. **오버나이트 후보 표기 버그 교정**: 오버나이트 후보의 점유율 순증(late_gain)이 음수일 때 부호가 중첩되어 `+-2.0%p`와 같이 노출되던 이중부호 버그 수정.
3. **리포트 UI의 일관성 있는 폰트 밀도 적용**: 리포트 창 내 LIVE 섹션의 카드와 라벨들이 다른 테이블에 비해 과도하게 커서 화면 밀도를 떨어뜨리던 문제 조율.
4. **마켓 로테이션 검증용 회귀 테스트 영구화**: 세션 스크래치로만 존재하던 V29.7 정렬 테스트를 정식 테스트 스위트 경로에 편입하여 향후 랭킹 계산식 오작동 방지.

---

## ✅ 상세 작업 및 패치 내역

### 1. 대장 교체 타임라인 이벤트 로깅 구현
- **ranking_manager.py**:
  - 테마 정보 스냅샷 갱신 시, 이전 대장주와 현재 대장주가 달라지면 `"👑 대장 교체: 테마 — [A] → [B]"` 포착 이벤트 자동 로깅.
  - 너무 잦은 이벤트를 방어하기 위해 동일 테마에 대해 10분 단위의 쿨다운을 적용하고, 개장 시 최초 포착은 이벤트 등록 대상에서 배제.
  - Qt 리포트 뷰어(`_fmt_event` 연동, 보라색 표시) 및 웹 대시보드 타임라인(보라색 테두리 CSS 렌더링)에 각각 맞춤 스타일 적용.
  - 기존 "대장주" 컬럼명을 "대장주(현재)"로 명명하여 실시간 1위 종목임을 사용자에게 보다 명확히 명시.

### 2. 오버나이트 음수 순증 이중부호 제거 및 UI 보완
- **theme_report_window.py / rotation_dashboard.html**:
  - late_gain이 음수인 상태로 넘어올 때 `+-` 부호가 동시에 찍히던 버그 수정.
  - 웹 버전: 음수일 경우 텍스트에 하락색(적색) 및 `p` 단위를 매끄럽게 처리.
  - Qt 버전: `:+ .1f` 포맷 자동 부호를 사용하여 중첩을 제거하고 색상 분기 로직 적용.
  - Qt KPI 주도 지속시간 하드코딩(30s)을 실제 스냅샷 주기(`snap_sec`)에 연동되도록 유연화.
  - 리포트 LIVE 섹션의 엣지 및 예열 카드를 26px 높이로 고정하고 폰트를 11px로 축소하여 전체 화면 밀도와 정합.

### 3. 마켓 로테이션 검증 테스트 세트 영구화
- **test_theme_rotation_report.py (신규)**:
  - 8가지 경계조건에 대응하는 단위 테스트 작성:
    - 28사이클x2배판 vs 30사이클x1배판 가중치 정렬 검증.
    - 주도 0사이클 테마 제외 검증.
    - 오버나이트 경계조건 및 전일 후보 적중 여부 판정 로직.
    - 웹/Qt가 소비하는 데이터 DTO 스키마 필드 계약 준수성 검증.
- **README.md**:
  - 마켓 로테이션 분석 리포트 및 성능 추이 스크린샷 이미지를 문서에 신규 추가하여 유저 가이드성 보강.

---

## 🧪 검증 및 테스트 결과

### 1. 정적 회귀 테스트 실행
- 새로 편입한 `test_theme_rotation_report.py`를 포함하여 마켓 로테이션 연산 관련 신규 테스트 8건 및 기존 전체 테스트를 실행한 결과, 100% 정상 통과함을 확인.
- 브라우저 레이아웃 실측: 대장 교체 시 `ev.captain` CSS 속성이 정상적으로 매핑되어 보라색 테두리로 부드럽게 렌더링되고, 음수 마진도 하락색으로 정확히 반영됨을 확인.

---

## 🔮 향후 계획 및 최종 의도

1. **테마 대장 교체 빈도 분석**: 10분의 쿨다운이 적절한지 모니터링하여 장중 테마 쏠림이나 노이즈성 교체 이벤트가 너무 많이 유입되지 않도록 임계 조정 검토.
