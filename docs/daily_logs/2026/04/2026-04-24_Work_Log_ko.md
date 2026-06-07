# NCQ 데일리 작업 로그 (2026-04-24)

## 📋 요약 (Summary)
오늘의 주요 과업은 '전략 의사결정 허브' 내 **AI Insights(인사이트) 대시보드의 레이아웃 최적화 및 시각적 안정성 확보**였습니다. 초기 기동 시와 종목 선택 시의 박스 크기 불일치로 인한 UI 요동 현상을 해결하기 위해 **"Baseline Height Stabilization (기준 높이 안정화)"** 패턴을 도입했습니다. 또한, Qt 렌더링 엔진의 특성상 발생하는 미세한 하단 공백과 항목 간 뭉침 현상을 `QTimer` 비동기 처리와 `documentLayout` 정밀 측정을 통해 완벽히 제어하였습니다. 이를 통해 제도권 수준의 정갈하고 견고한 대시보드 인터페이스를 완성했습니다.

## 🛠 주요 변경 및 개선 사항

### 1. **[UI] AI Insights 레이아웃 안정화 및 최적화 (V19.9.6g)**
- **Baseline Height 고정 패턴 적용**:
    - `_base_height(130px)`를 설정하여 종목 선택 시 박스가 급격히 줄어드는 현상 차단.
    - 초기 기동 상태와 데이터 업데이트 상태의 높이 계산 로직을 단일화하여 레이아웃 점프(Jumping) 현상 근본 해결.
- **하단 공백 제로(Zero-Padding) 달성**:
    - `QFrame.NoFrame`과 `ScrollBarAlwaysOff` 설정을 통해 물리적 점유 공간 최소화.
    - `documentMargin(0)` 및 전역 스타일시트 강제를 통해 HTML 기본 여백 원천 봉쇄.
- **정밀 렌더링 타이밍 제어**:
    - `QTimer.singleShot(0, ...)`을 활용하여 Qt 레이아웃 엔진이 렌더링을 마친 직후의 실제 높이를 계산하여 위젯 크기 동기화.

### 2. **[UI] 콘텐츠 가독성 및 디자인 정밀 조정**
- **항목 뭉침 현상 해결**:
    - `span` 인라인 방식에서 발생하던 뭉침 현상을 확실한 줄바꿈을 보장하는 `div` 블록 구조로 환원하여 해결.
    - 각 요소별 `margin: 0`, `padding: 0` 강제 주입으로 다닥다닥 붙는 현상 방지.
- **시각적 일체감 확보**:
    - "다음 타겟" 박스의 배경색을 전체 위젯 배경색(`#1a1a1a`)과 동기화하여 시각적 단절감 제거.
    - `line-height: 1.3` 및 `margin-top: 10px` 조정을 통해 정보 간의 논리적 구분 명확화.
- **상단 Regime Panel 시각적 안정화 (V19.9.6h)**:
    - 대세·국면·수급 패널 높이를 `48px`로 절대 고정하여 아이콘 변화에 따른 높이 요동 차단.
    - 초기 상태(`-`)와 활성 상태(`▲`)의 폰트 크기를 `18px`로 통일하여 물리적 크기 불일치 해결.
- **전체 의사결정 허브 레이아웃 하드닝 (V19.9.6i)**:
    - DecisionBox(상단 흰색 박스) 높이를 `55px`로 최적화하고 점수(130px)/결정(110px) 레이블 폭을 재조정하여 중앙 정렬의 미적 균형 복구.
    - 결정 상태 메시지 단일화: **전략 엔진(`strategy.py`) 레벨**에서 "전략 준비중"을 "매수 준비"로 직접 수정하여 데이터 일관성(SSoT) 확보 및 UI 연산 최소화.
    - **버그 픽스 (V19.9.6o)**: "매수 준비" 문구에 포함된 "매수" 키워드 때문에 발생하던 `UNDEFINED` 배지 노출 현상 해결 (조건문을 실제 진입 신호로 한정).
    - **툴팁 시스템 통합 (V22.1)**: `_build_tip()` 모듈 헬퍼 도입으로 대시보드 내 모든 툴팁(10개소 이상)의 디자인(폰트, 색상, 레이아웃) 표준화 및 시각적 고도화 완료.
    - 전략 배지(Strategy Badge)의 폭을 `110px`로 유지하여 영문 포함 텍스트(`VOLUME`, `BREAKOUT`) 가독성 사수.
    - 신호등(Traffic Lights) 구역의 상하 레이블 폭을 `70px`로 통일하여 수직 정렬축 완벽 일치.

### 3. **[Strategy] 전략 로직 하드닝 및 핵심 버그 픽스 (V22.1)**
- **포지션 관리 로직 정교화**:
    - `_in_profit` 조건을 도입하여 실제 수익 구간에서만 BE SL(본전 손절선) 상향이 작동하도록 개선 (손실 구간 조기 청산 원천 차단).
- **데이터 무결성 및 가드 강화**:
    - `_get_bb_raw`에서 세션 참조 오류를 수정하고 `regime`을 직접 전달받도록 파라미터화하여 계산 정확도 확보.
    - `upper_bb`의 NaN/Zero 데이터 유입 시 익절(TP2) 오작동 방지를 위한 하한선 가드(`> 0`) 추가.
- **코드 품질 및 유지보수성 최적화**:
    - 미사용 데드 파라미터(`vol_ratio`) 제거 및 함수 내부 `import` 구문을 파일 상단으로 통합하여 런타임 효율 및 가독성 향상.

### 4. **[Strategy] 거대 모듈 리팩토링 및 관심사 분리 (V22.1)**
- **Strategic Orchestration 패턴 도입**:
    - 1,000라인이 넘어가는 `strategy.py`의 복잡도를 제어하기 위해 핵심 판단 로직을 3대 서브 엔진으로 분리하고 `StrategyManager`를 오케스트레이터로 재정의.
- **신규 서브 엔진 그룹 구축**:
    - **`_engine_entry.py` (EntryEngine)**: 신규 진입 판단 및 L3 가드(Risk/Regime/Reward/MTF) 로직 전담.
    - **`_engine_exit.py` (ExitEngine)**: 포지션 보유 중 비대칭 출구 4계(SL/TP1/TP2/TimeExit) 및 트레일링 스탑 업데이트 전담.
    - **`_engine_penalty.py` (PenaltyEngine)**: 과열 차단(VOL_RATIO_CAP), 패널티 계산 및 UI용 Logic Gap 분석 전담.
- **기대 효과**: 모듈 간 결합도(Coupling)를 낮추고 개별 엔진 단위의 단위 테스트 및 로직 확장이 용이한 구조적 기반 마련.

## 🐞 주요 버그 픽스 (Hotfixes)

| 버그 ID | 문제 현상 | 수정 내용 | 파일 |
| :--- | :--- | :--- | :--- |
| **BUG-11** | AI Insights 박스가 빈 공간을 차지하며 비정상 확장됨 | 레이아웃 하단에 `addStretch(1)` 추가하여 모든 위젯 상단 밀착 | `signal_status_widget.py` |
| **BUG-12** | 상태 설명 항목들이 한 줄로 뭉쳐서 표시됨 | `span(inline)` 태그를 `div(block)`로 환원하여 줄바꿈 강제 | `signal_status_widget.py` |
| **BUG-13** | 종목 변경 시 박스 크기가 요동치며 하단 버튼 위치가 변함 | `Baseline Height(130px)` 기준값 및 `max()` 함수로 높이 하한선 고정 | `signal_status_widget.py` |
| **BUG-14** | HTML 마지막 줄 아래에 1~3px의 미세 공백 잔존 | `documentLayout().documentSize()` 기반 정밀 측정 로직 도입 | `signal_status_widget.py` |
| **BUG-15** | 상단 Regime Panel 아이콘 교체 시 미세한 높이 변화 발생 | 패널 높이 `48px` 고정 및 초기/이후 폰트 크기 동기화 | `signal_status_widget.py` |
| **BUG-16** | 포지션 없는 상태에서 SELL action + FULL_TP 발생 (Critical) | decision 표시용 레이블만 변경하고 action/exit_type HOLD 유지 | `strategy.py` |
| **BUG-17** | WEAK_VSA 패널티 미등록 및 config 누락 (Critical) | `volume_ratio < VSA_WARN_RATIO` 기준 주입 및 `config_engine.py`에 0.8 기본값 추가 | `strategy.py`, `config_engine.py` |
| **BUG-18** | `_get_bb_raw` 잘못된 세션 참조로 엉뚱한 국면 조회 (Medium) | regime 파라미터 명시화 및 세션 참조 완전 제거 | `strategy.py` |
| **BUG-19** | 손실 포지션 즉시 청산 유발 (Medium) | `_in_profit` 조건 추가로 수익 구간에서만 BE SL 상향 | `strategy.py` |
| **BUG-20** | TP2 upper_bb 0.0일 때 즉시 발동 가드 누락 (Medium) | `upper_bb` zero/NaN 방어를 위해 `> 0` 가드 추가 | `strategy.py` |
| **BUG-21** | 함수 내부 import 3곳 잔존 (Minor) | `math`, `json`, `os`, `pandas` import를 파일 상단으로 통합 | `strategy.py` |
| **BUG-22** | `_calc_entry_score`의 `vol_ratio` 데드 파라미터 존재 (Minor) | 미사용 파라미터 제거 및 호출부 인자 정리 | `strategy.py` |

---

## 🚀 기대 효과 및 성과
- **UI 일관성 극대화**: 어떤 상황에서도 흔들리지 않는 레이아웃 기준점을 확보하여 사용자 경험(UX) 안정성 향상.
- **전략 엔진 신뢰도 확보**: 7종의 핵심 버그 픽스를 통해 의도치 않은 청산 및 진입 오류를 원천 차단하고 논리적 완결성 달성.
- **정보 전달력 강화**: 불필요한 여백은 제거하고 논리적 간격은 확보하여 AI 분석 결과의 가독성 증대.
- **고도화된 Qt 위젯 제어**: 렌더링 루프와 레이아웃 엔진 간의 충돌을 해결하는 고급 기술 패턴(Baseline & Async Resize) 정립.

---

## 📅 다음 작업 체크포인트
- [ ] AI 분석 결과가 220px(최대 높이 제한)을 초과할 때 스크롤 없이 가독성이 유지되는지 확인.
- [ ] 저해상도 모니터에서 130px의 베이스라인 높이가 적절한지 피드백 수렴.
- [ ] 다른 위젯(Log, Performance 등)에도 동일한 Baseline Stabilization 패턴 적용 검토.

---
**헌법 준수 및 거버넌스 가이드라인에 따라 모든 수정 사항을 기록 완료하였습니다.**
