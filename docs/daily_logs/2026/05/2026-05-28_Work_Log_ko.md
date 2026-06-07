# 📅 2026-05-28 업무 일지 (Daily Work Log)

## 🎯 주요 목표 및 이슈 정의
- [x] **폴라리스오피스(041020) 유실 거래 기록 데이터 복구**:
  앱 재시작 시점의 매매 체결 이벤트 수신/매칭 문제로 인해 누락되었던 폴라리스오피스 거래 내역을 정밀 복원하여 데이터베이스에 직집 입력 완료했습니다.
- [x] **Orphan Sell(미확인 매도) 매칭 및 복원 가드 로직 개선**:
  앱이 예기치 않게 재시작되어 GTID 매핑이 유실되더라도 동일 종목/전략의 `OPEN` 매수 포지션과 자동 매칭하고 진입 정보와 실현 손익을 자동 복원하도록 보완했습니다.
- [x] **멀티차트 timeframe_minutes AttributeError 크래시 해결**:
  멀티차트 활성화 시 `CandleManager` 인스턴스에 존재하지 않는 `timeframe_minutes` 속성을 참조해 프로그램이 종료되던 UI 버그를 실제 정의된 `tf` 속성 참조로 즉시 수정했습니다.
- [x] **오늘자 매매 성과 분석 보고서 작성 및 이관**:
  Profit Factor 2.70 및 당일 실현수익 +830,536원을 달성한 매매 데이터를 감시탭/전략유형/국면별로 분석하여 별도 분석문서로 저장했습니다.

---

## ✅ 상세 작업 및 패치 내역

### 1. 폴라리스오피스(041020) 유실 거래 기록 복구 및 DB 강제 정합성 동기화
* **대상 파일**:
  - [data/trades.db (sqlite)](file:///c:/Users/stone/projects/nocodequant/data/trades.db)
* **현상**:
  - 5월 28일 오전 10:10:56에 매수(590주 @ 4,583.35원)된 폴라리스오피스가 10:21:49에 청산 완료되었으나 성과보드(UI)에서 해당 거래 기록이 완전히 유실되어 보이지 않았습니다.
  - 원인은 청산 시점(10:21:49, 111주 부분체결 직후)에 앱이 재시작되면서 실시간 체결 수신 이벤트의 parent_gtid(`GTID_UNKNOWN`)가 매칭되지 않았고, 재시작 극초기 상태의 메모리 캐시 누락으로 판정되어 거래 이력이 DB 기록에서 완전히 누락(Orphan Sell)되었기 때문입니다.
* **조치**:
  - `reconstruct_trade.py` 복구 스크립트를 작성하여 당시 매수 체결가 및 매도 체결 수량 정보를 토대로 정밀한 수수료/세금 비용(약 0.22% 가산)을 연산했습니다.
  - 성과보드 주 데이터베이스인 `trades.db` 내 `trades` 테이블에 복원된 체결 건(590주, 진입 4,583.36원, 청산 4,495원, 실현손익 -57,834.73원, 수익률 -2.1388%)을 정합성에 맞춰 입력 완료했습니다.

---

### 2. Orphan Sell 방지 및 자가 치유(Self-Healing Fallback) 매칭 레이어 구축
* **소스 파일**:
  - [persistence_manager.py](file:///c:/Users/stone/projects/nocodequant/app/services/managers/persistence_manager.py#L301-L322)
* **현상**:
  - 매도 체결 정보 수신 시, 앱 재시작 등의 변수로 인메모리 GTID 맵이 유실되어 매칭 대상 매수 레코드(parent_gtid)를 찾지 못할 경우 복원 단계에서 복원에 실패하면 거래 자체가 유실(DLQ 격리)되는 치명적인 문제가 있었습니다.
* **조치**:
  - **동일 종목/전략 OPEN 레코드 백업 매칭 (3차 Fallback)**: GTID 기반 매칭 실패 시, `orders_tracked` 테이블 내에서 **동일 종목 코드(code) 및 동일 전략(strategy_type)을 가지며 `status = 'OPEN'` 상태인 레코드**가 존재하는 경우 해당 레코드를 매칭 대상으로 자동 할당하게 하였습니다.
  - **진입 정보 및 PnL 수동 보정**: 백업 매칭 성공 시, 매칭된 레코드에서 `entry_price`와 `entry_time`을 추출하여 체결 데이터 페이로드를 보정합니다. 또한, 진입/청산 단가가 존재함에도 PnL 계산이 유실된 경우를 위해 세금(0.2%) 및 수수료(0.015%*2) 거래 비용을 감안한 실전형 수익률/수익금을 자체적으로 재산출하여 보정 반영하도록 보완했습니다.

---

### 3. 멀티차트 timeframe_minutes 속성 불일치 크래시(AttributeError) 패치
* **소스 파일**:
  - [multi_chart_window.py](file:///c:/Users/stone/projects/nocodequant/app/ui/components/multi_chart_window.py#L269)
  - [candle_manager.py](file:///c:/Users/stone/projects/nocodequant/app/utils/candle_manager.py#L5)
* **현상**:
  - 주도주 감시 화면에서 종목 변경 또는 타임프레임 전환 시, 멀티차트 핸들러인 `_on_marshalled_history_received`에서 `AttributeError: 'CandleManager' object has no attribute 'timeframe_minutes'` 에러를 내며 GUI 프로그램이 즉시 강제 종료되었습니다.
* **조치**:
  - `CandleManager` 클래스는 내부적으로 타임프레임 분 단위를 `self.tf` 속성으로 정의하여 관리하고 있으나, UI 컴포넌트(`multi_chart_window.py`)에서 `self.candle_managers[code].timeframe_minutes`로 오참조하던 코드를 찾아냈습니다.
  - 해당 비교 구문을 `self.candle_managers[code].tf != tf_val`로 알맞게 정정하여 에러 유발 및 크래시 현상을 차단했습니다.

---

### 4. 2026-05-28 전략 성과 정밀 분석 수행
* **대상 파일**:
  - [docs/analysis/2026-05-28_Strategy_Analysis_ko.md](file:///c:/Users/stone/projects/nocodequant/docs/analysis/2026-05-28_Strategy_Analysis_ko.md)
* **내용**:
  - **오늘자 매매 요약**: 17건 매매, 승률 52.9%(9승 8패), 누적 수익 +830,536원, Profit Factor 2.70, 평균 익절 +2.70%, 평균 손절 -1.28% 달성.
  - **세부 강점**: 돌파 탭과 TREND 전략의 강력한 기여(+61만원 및 +84만원)와 UPTREND 국면에서의 집중도 극대화. 횡보장(RANGE) 진입 억제 기능의 성공적 동작.
  - **보완 과제**: 급등 탭 종목의 초단기 슬리피지 방지 및 윗꼬리 저항 제어, MIXED 전략 진입 시 체결강도(Volume Power) 가드 결합 검토 기술 분석 제언.

---

## 🔮 향후 대응 및 실거래 운영 안정화 방안
1. **자가 치유(Self-Healing) 동작 사후 검증**:
   - 향후 장중 앱 재시작 등의 돌발 상황 발생 시, Orphan Sell 상태의 매도 주문들이 DB 내 `orders_tracked` 상의 OPEN 레코드와 정상 백업 매칭되어 `trades`로 안정적으로 유입되는지 모니터링합니다.
2. **슬리피지 최소화 런타임 제어**:
   - 급등주 추격이나 MIXED 전략 오작동을 제어하기 위해 차주 중 실시간 거래량 집중 지표를 추가 바인딩할 계획입니다.
