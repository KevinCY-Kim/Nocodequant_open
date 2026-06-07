# NCQ 데일리 작업 로그 (2026-04-21)

## 📋 요약 (Summary)
오늘의 핵심 과업은 전략 실행 엔진(`strategy.py`)의 의사결정 프로세스 저변에 깔린 구조적 버그들을 제거하여 **'신호 차단(Signal Blocking)'** 현상을 해결하는 것과, 문서상으로만 존재하던 **'비대칭 출구 엔진(Asymmetric Exit Engine)'**을 실제 코드로 정밀 이식하는 것이었습니다. 특히 R-Multiple(리스크 배수) 기반의 동적 트레일링 스탑과 전략별 TP1 분리 정책을 통해 "수익은 길게, 손실은 짧게"라는 퀀트의 비대칭성을 시스템 심장부에 구축 완료했습니다. 또한, V5.0 분석 파이프라인 연계를 위해 진입 시점의 4블록 스코어 정보를 DB에 각인하는 작업을 완료했습니다.

## 🛠 주요 변경 및 개선 사항

### 1. **[Strategy] 비대칭 출구 엔진(Asymmetric Exit Engine) 실구현**
- **R-Multiple 수학 모델 도입**:
    - 모든 포지션에 `initial_sl`을 각인하고, 1R(리스크 단위) 기반의 수익 비율(`current_r`)을 실시간 계산하는 엔진 구축.
    - `initial_sl` 영속화 파이프라인 신설 (신규 매수 및 세션 복구 시 1:1 보존).
- **Asymmetric 매도 4계 이식**:
    - **제1계(BE Switch)**: 2.2R 도달 시 손절선을 진입가 위(+0.2R)로 전진 배치하여 '절대 지지 않는 싸움' 구현.
    - **제2계(Dynamic Tightening)**: 수익권별 트레일링 배수를 4.5x → 3.0x → 2.0x → 1.5x로 자동 변속하여 수익 잠금 강화.
    - **제3계(TP1 분리)**: 주도주(TREND, BREAKOUT, VOLUME)는 1.5% 익절(TP1)을 전면 폐지하고 트레일링으로 무부하 수익 극대화. 역추세용만 2.0R TP1 적용.
    - **제4계(Smart Time Exit 2.0)**: 수익권 + MA20 상단 + MA20_Slope 우상향 시 시간 제한을 2배 연장하는 생존 패스 부여.
- **Panic Shield 우선순위 최적화**:
    - 시장 급락 시(MA240 하회) 비대칭 엔진의 배수를 무력화하고 `1.0x` 타이트닝을 최우선 적용하는 자본 보호 최우선 정책 코딩.

### 2. **[Analytics] 전략 성과 분석 엔진 V5.0 구현**
- **스코어 영속화 파이프라인 완성**:
    - 진입 시점의 4대 핵심 요인(Technical, Volume, Price, Risk)을 `trades` 테이블에 영속적으로 저장(Score-Persistence).
    - `TradeSignal` -> `OrderManager` -> `PersistenceManager` (DB INSERT)로 이어지는 데이터 흐름 구축.
- **분석 무결성 가드레일 도입 (`schema_views.py`)**:
    - **PnL 왜곡 방지**: 성과 계산 시 비중 합산 방식을 적용하여 부분 청산 시의 데이터 노이즈 차단.
- **UI 시각적 고도화**:
    - `trade_log_tab`: 스코어 히스토리 및 세부 점수 툴팁 구현으로 복기 기능 강화.

## 🐞 주요 버그 픽스 (Hotfixes)

| 버그 ID | 문제 현상 | 수정 내용 | 파일 |
| :--- | :--- | :--- | :--- |
| **BUG-01** | UPTREND 상단 돌파 시 UI에 "하단이탈"로 반대 표기 | 정규화 점수 매핑에서 가격 직접 비교 로직으로 교체 | `strategy.py` |
| **BUG-02** | 상승 우위임에도 격차 미달 시 DOWNTREND 강제 편입 | `plus_di >= minus_di` 조건 시 UPTREND 판정 분기 추가 | `strategy.py` |
| **BUG-03** | 5분봉 ATR 기반 Reward Check 구조적 통과 불능 | `TREND` 전략에 대해 Reward Check 예외(Exempt) 적용 | `strategy.py` |
| **BUG-04** | 앱 재시작 후 포지션 복구 시 R-Multiple 계산 오염 | `sync_position` 딕셔너리에 `initial_sl` 명시적으로 주입 | `strategy.py` |
| **BUG-05** | 전략성과보드 오픈 시 `NoneType` 포맷팅 오류 | 구버전 DB 행의 NULL 스코어 값에 대해 `or 0` 방어 로직 추가 | `trade_log_tab.py` |
| **BUG-06** | 1봉 미만 초단기 거래 시 보유봉이 1봉으로 강제 표기되는 왜곡 | `max(1, bars)` 강제 로직 제거 및 0봉 추적 지원 | `utils.py`, `order_manager.py` |
| **BUG-07** | 웜업 중 UI 손절 설정값에 의한 광속 청산(Flash Exit) 발생 | `Doomsday SL`을 UI 설정값과 분리하여 독립 방어선(-4%)으로 격리 | `strategy.py` |
| **BUG-08** | `trades` 테이블 `reason` 컬럼 누락으로 분석 쿼리 실패 | SQLite Hot-Migration 수행하여 `reason` 컬럼 추가 | `trades.db` |
| **BUG-09** | 틱 레벨(`update_tick`)에서 웜업 중 Doomsday(-4%) 무조건 유예 현상 | 웜업 구간이어도 진입가 대비 -4% 하락 시 즉각 청산하도록 틱 가드 보완 | `strategy.py` |
| **BUG-10** | `initial_sl` 계산부 변수명 충돌 (실제 UI SL인데 Doomsday로 명명됨) | `doomsday_sl_price` → `ui_cap_sl_price`로 교체하여 네이밍 버그 해소 | `strategy.py` |
| **BUG-11** | 코드(0봉 렌더링)와 주석(최소 1봉 보장) 간의 문서 불일치 | `calculate_holding_bars` docstring 수정 및 미사용 import 제거 | `utils.py` |
| **BUG-12** | Panic Shield가 비대칭 트레일링 덮어쓰기에 의해 무력화됨 | flag 기반 최우선 적용 및 `min(shield, dynamic)` 로직 도입 | `strategy.py` |
| **BUG-13** | `time_limit` float 연산 시 정밀도 저하 | 정수 곱셈 후 명시적 `int()` 캐스팅 적용 | `strategy.py` |
| **BUG-14** | R < 1.5 구간 trailing_mult 하드코딩된 기본값(3.5) 불일치 | `pos`에 내장된 실제 초기 배수를 읽어오도록 fallback 수정 | `strategy.py` |
| **BUG-15** | RSI Inversion 로직의 텍스트 표기 오류 (RSI < 50 시 가점 표기) | UPTREND 시 RSI 강세(+) 방향성을 갖도록 스코어링 반전 적용 완료 | `strategy.py` |

---

## 🚀 기대 효과 및 성과
- **데이터 기반 자율 진화**: 누적된 스코어 데이터를 바탕으로 "어떤 점수대에서 수익이 집중되는가"를 머신러닝/통계적으로 분석하여 진입 임계값을 동적으로 튜닝할 수 있는 기반 마련.
- **신호 정합성 100%**: 의사결정 단계의 스코어와 최종 성과가 GTID로 연결되어, 블랙박스 없는 전략 검증 가능.
- **UI 신뢰도 향상**: 시각적 매핑 오류 교정 및 상세 툴팁 제공으로 사용자 직관성 극대화.

---

## 📅 내일의 체크포인트
- [x] 실전 매매에서 진입 시 발생한 스코어가 `trades.db`에 정확히 기록되는지 샘플링 검증.
- [x] `trades` 테이블에 `reason` 컬럼 추가 및 청산 사유 기록 여부 확인.
- [ ] 스코어 분위수 분석 뷰(`v_score_quantile`)가 실제 전략 튜닝의 임계값 설정에 유효한 데이터 세트를 도출하는지 확인.
- [ ] `UPTREND` 국면 판정이 지연 없이 발생하는지 모니터링하여 `TREND` 전략 가동성 재확인.
- [x] 장중 앱 재기동 테스트를 통해 포지션 복구 후 `initial_sl` 및 `score` 메타데이터가 유지되는지 최종 확인.
- [x] 초단기 거래(Flash Exit 현상) 발생 시 `reason` 필드에 정확한 사유(SL_TICK or STOP_LOSS)가 남는지 실전 매매 모니터링 (Doomsday 반영 완).
- [ ] Tick 레벨 웜업 중 Doomsday 하락(-4% 터치) 시 최대 5분 지연 없이 즉각 SL 쉴드가 발동하는지 검증.

---
**헌법 준수 및 거버넌스 가이드라인에 따라 모든 수정 사항을 기록 완료하였습니다.**

