# 📅 2026-07-04 업무 일지 (Daily Work Log)

> **[거버넌스 준수]**
> * **작업 등급**: L3 (마켓 로테이션 리포트 적응형 임계 및 오버나이트 레이더 도입)
> * **참조 문서**:
>   * [ranking_manager.py](file:///c:/Users/stone/projects/nocodequant/app/services/managers/ranking_manager.py)
>   * [theme_report_window.py](file:///c:/Users/stone/projects/nocodequant/app/ui/theme_report_window.py)
>   * [theme_flow_widget.py](file:///c:/Users/stone/projects/nocodequant/app/ui/widgets/theme_flow_widget.py)
>   * [main_window.py](file:///c:/Users/stone/projects/nocodequant/app/ui/main_window.py)
>   * [sync_krx_listings.py](file:///c:/Users/stone/projects/nocodequant/data/sync_krx_listings.py)
> * **영향 받는 모듈 (Impacted Modules)**: `ranking_manager.py`, `theme_report_window.py`, `theme_flow_widget.py`, `main_window.py`, `sync_krx_listings.py` (신설), `krx_master.json`, `etf_codes.json` (신설)
> * **불변성 위반 여부 (Invariant Check)**: 위반 없음 (Violated: None)
> * **경계 준수 선언**: 오후장의 매매 수급 둔화로 인해 발생하던 테마 감지 공백 문제를 적응형 분위수 임계 계산식을 통해 성공적으로 개선하였습니다. 이 과정에서 익일 대장 테마 후보를 실측하는 오버나이트 레이더를 도입하고, ETF를 코드 기반으로 완벽히 필터링하는 로직 및 장중 재시작 시 상태 복원용 아카이브 기획을 안전하게 적용 완료하였습니다.

---

## 🎯 주요 목표 및 이슈 정의

1. **오후장 테마 감지 공백 해소**: 고정 임계값(±5pt) 기준을 아침 유동성에 맞춰 설계하여, 유동성이 급감하는 오후장(14시 이후)에 테마 강도(velocity)가 묻혀 거래 기회 및 관측이 소실되는 구조적 한계 극복.
2. **익일 주도주(오버나이트) 예측 모델 탑재**: 오후 14시 이후의 점유율 순증 추이를 토대로 익일 대장 테마 후보를 선정하는 '오버나이트 레이더' 구축 및 당일 적중 검증 가이드 추가.
3. **자금이동 및 예열 테마 간 연계 회복**: 기존에 상호 독립적으로 계산되던 자금이동 경로와 예열 테마(🌱)를 연결하여, 낮은 강도로 인해 누락되던 유입 경로를 구제.
4. **ETF 필터링 취약점 극복**: 주도주(HOT) 탭에 '1Q K반도체' 등 특이 브랜드의 ETF가 노출되던 명칭 기반 필터링 취약성을 코드 기반 차단 방식으로 전환.
5. **장중 재시작 시 아카이브 복원**: 프로그램 재시작 시 당일 오전의 마켓 로테이션 지표 및 이벤트 누계가 소실되어 이중 카운팅되거나 30초 틱 게이트가 꼬이는 문제 해결.

---

## ✅ 상세 작업 및 패치 내역

### 1. 적응형 감지 임계(Adaptive Velocity Threshold) 및 Share 지표 신설
- **ranking_manager.py**:
  - 임계값 = `clamp(1.2 × 시장 전체 |velocity| 중앙값, 하한 1.5pt, 상한 5.0pt)` 수식 도입.
  - 임계값 변화에 연동하여 뱃지(⚡/🔥/🌱/🌫️) 판정도 2-패스(선연산 후배정)로 자동 조절되도록 재구조화.
  - 시장 전체 heat 대비 테마별 백분율을 나타내는 `share` 및 `share_vel` 지표를 새로 추가하여, 시장 유동성에 따른 왜곡 없는 정적 판단 지원.

### 2. 오버나이트 레이더 (Overnight Radar) 설계
- **ranking_manager.py / theme_report_window.py**:
  - 14:00+ 시간 윈도에서 각 테마별 점유율 순증(시초 대비 종가) 및 유입 강도 추적.
  - 오늘 생성된 후보가 익일 실제 찐테마 TOP3 내에 안착했는지를 체크하여 리포트에 적중 표시(✓) 구현.
  - 주말을 고려하여 최대 4일 전까지의 과거 아카이브 파일을 자동으로 역탐색하여 하루 1회 적중 통계 캐시.

### 3. 자금이동 ↔ 예열 연계 및 UI 리포트 전면 개편
- **theme_flow_widget.py / ranking_manager.py**:
  - 예열(🌱) 테마를 유입 경로(flow_edges) 목적지 후보군에 명시적으로 추가하여, 임계치에 걸쳐 누락되던 연결선을 최소 1개 보장하도록 수정.
  - 5초 데이터 캐싱과 30초 스냅샷 게이트를 구성하여 GUI 갱신 시 `velocity = 0.0`으로 수렴해 무력화되던 버그 완화.
  - `ThemeReportWindow` 내 찐테마 KPI 4종, 점유율 도넛 차트(QPainter 기반), 백데이터 랭킹, 타임라인 이벤트(🌙 장후반 마커 표시 포함)를 추가하여 리포트 디자인 전면 개편.

### 4. 장중 재시작 아카이브 (Intraday State Restoration)
- **ranking_manager.py**:
  - 30초 주기로 `logs/theme_rotation_state_YYYYMMDD.json`에 원자적 교체(`tmp` 생성 후 `os.replace`) 방식으로 런타임 상태 저장.
  - 날짜 롤오버 전 재기동 시 해당 날짜의 상태 데이터를 자동 파싱·복원하여 중복 로깅 및 틱 게이트 꼬임 차단.

### 5. 코드 기반 ETF 완벽 차단 및 신규상장 유니버스 동기화
- **sync_krx_listings.py (신규) / etf_codes.json (신규)**:
  - KRX `finder_secuprodisu`를 호출하여 전체 ETF 코드를 긁어와 `data/etf_codes.json`(1,143개)을 생성하는 idempotent 동기화 스크립트 작성.
  - `ranking_manager` 기동 시 이를 로드하여 코드 단위로 차단하도록 수정.
  - KRX `finder_stkisu`를 통하여 신규상장 주식 170종을 발굴, `krx_master.json`에 유가/코스닥/코넥스 스키마로 추가하여 시장구분 `None` 우회 차단.

---

## 🧪 검증 및 테스트 결과

### 1. 단위 테스트 보강
- 오후장 저활동 임계 clamp 기능, late 시간 윈도 및 전일 적중 판정 로직을 검증하는 61건의 유닛 테스트 케이스 실행 및 전원 통과 확인.
- `etf_codes.json` 로드 시 0182R0, 0103T0 등 신규 브랜드의 ETF가 완벽히 필터링 처리(True)되고 일반 주식은 통과(False)함을 입증.

---

## 🔮 향후 계획 및 최종 의도

1. **오후장 감지 성능 모니터링**: 14:00 이후 적응형 분위수 임계가 하한인 1.5pt 부근에서 뭉개지지 않고 유의미하게 예열 테마 및 자금이동 엣지를 포착해내는지 장중 확인.
2. **익일 적중 검증**: 오버나이트 레이더의 익일 매치율 통계를 누적하여 장 후반 수급과 다음 날 주도 테마 간의 통계적 상관관계를 정밀화.
