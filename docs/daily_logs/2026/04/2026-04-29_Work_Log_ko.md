# 2026-04-29 Work Log

## 🎯 오늘의 목표
*   RankingManager의 필터링 로직 강화 및 저품질 종목 제거
*   Hybrid Run-Rate Gate (런레이트 + 가속도 기반) 유동성 필터 구현
*   기존 실시간 랭킹 엔진의 데이터 오염 버그 수정
*   [v] 매도 메타데이터 휘발 버그(UNDEFINED/MANUAL) 원인 규명 및 해결
*   [v] Surge(급등) 탭 저품질 종목(ETF, 우선주 등) 정밀 필터링
*   [v] KRX 테마 매핑 엔진 고도화 및 오분류(전력인프라/전선) 수정

## 🛠️ 작업 내용

### 1. V23.6 Hybrid Run-Rate Gate 구현
*   **배경**: 단순 300억 하드 컷오프는 장 초반 폭발하는 종목을 놓치는 맹점이 있음. 이를 보완하기 위해 "현재 페이스로 장 마감까지 300억을 달성할 확률"을 계산하는 런레이트 방식 도입.
*   **시간 기반 필터링**:
    *   `09:00~09:20`: 탐색 구간 (Warmup). 필터 미적용.
    *   `09:20 이후`: 런레이트 필터 활성화.
*   **가속도 보정 (Dynamic Discount)**:
    *   `_calc_realtime_surge` 점수를 활용하여 거래량이 급증하는 종목은 컷오프 허들을 낮춰줌 (최대 50% 할인).
    *   연속 함수형 설계를 통해 임계값 부근에서의 IN/OUT 플리커링 현상 방지.

### 2. RankingManager 구조적 버그 수정 (Critical)
*   **[C3] val_absolute 데이터 오염 수정**: 
    *   `apply_boost`에서 `GAIN`(거래량) 및 `VOLUME`(급증량) 데이터가 `VALUE`(거래대금) 필드를 덮어쓰던 버그 수정.
    *   이제 `val_absolute`는 VALUE 리스트 처리 시에만 갱신됨.
*   **[C2] heat_history 오염 방지**:
    *   유동성 필터(Run-Rate Gate)를 점수 이력 저장(`heat_history.append`) 이전 단계로 이동.
    *   필터링된 종목의 점수가 가속도(accel) 및 속도(velocity) EMA 계산에 영향을 미치지 않도록 격리.

### 3. Config 시스템 통합
*   모든 필터 파라미터를 `config_engine.py`로 이관하여 실시간 튜닝 가능하도록 개선:
    *   `RUNRATE_TARGET_BILLION`: 300억
    *   `RUNRATE_WARMUP_MINUTES`: 20분
    *   `RUNRATE_ABS_MIN_BILLION`: 20억 (절대 최소 대금)
    *   `RUNRATE_MAX_DISCOUNT`: 0.5 (최대 50% 할인)

### 4. V23.7 매도 메타데이터 무손실 전달 체계 구축 (Critical)
*   **증상**: 앱 재시작 후 매도 체결 시 `strategy_type`이 `UNDEFINED`, 청산방식이 `MANUAL`로 오기되는 현상 (SK네트웍스, 솔루스첨단소재 등).
*   **원인 분석**: 
    *   `pending_signal_meta`가 인메모리(dict)로만 관리됨.
    *   11:18분경 앱 재시작 시점에 Kiwoom에 걸려있던 미체결 LIMIT SELL 주문의 정보가 휘발됨.
    *   11:32분 체결 시 봇이 "내가 보낸 주문"임을 인지하지 못해 수동 매도로 오판.
*   **해결책 (Dual-Layer Guard)**:
    *   **영속화(Persistence)**: `data/pending_meta.json` 파일을 통해 주문 발주 시점의 메타데이터를 디스크에 실시간 백업/복원.
    *   **폴백(Fallback)**: 파일 유실 시에도 `StateManager`가 보유한 진입 시점의 포지션 정보(`position_meta_cache`)를 참조하여 `strategy_type` 및 `regime`을 강제 복구하는 로직 구현.
    *   **오표기 방지**: 강제 복구된 경우 청산 방식을 `MANUAL`이 아닌 `RESTORED`로 표기하여 분석 데이터 오염 차단.

### 5. Surge(급등) 탭 정밀 필터링 (V23.7)
*   **개선**: 급등 탭에서 전략과 무관한 지수 추종 상품 및 변동성만 높은 우선주 제거.
*   **필터 대상**:
    *   `증권사 상품`: 키움, KODEX, TIGER, ACE, RISE 등
    *   `파생/인덱스`: 레버리지, 인버스, 선물, ETF, ETN
    *   `우선주(정밀)`: 정규표현식(`([0-9])?우([BC123L]|\(전환\))?$`)을 통한 SK우, 현대차2우B 등 전수 차단.

### 6. KRX 테마 매핑 고도화 (Coverage 85.8% 달성)
*   **배경**: 테마 정보 부재 시 실행되는 2-hop 추론 엔진이 전선주를 "스마트 그리드"로 오분류하는 현상 발생 (산업 분류상 인접 동료들의 테마를 읽어오기 때문).
*   **해결**:
    *   **Subsector 매핑 확장**: 전선/케이블, 조선, 우주항공, 2차전지 등 28개 subsector 직접 매핑 추가 (커버리지 68% → 85.8%).
    *   **Product 키워드 보정**: KRX 업종상 '금속'으로 분류된 세명전기, 보성파워텍, 제룡산업 등을 `product` 내 "송배전", "송배전용" 키워드로 필터링하여 '전기장비/전력인프라' 테마로 강제 보정.
    *   **캐시 관리**: `krx_master.json` 갱신 후 `stock_graph.json` 삭제 및 재기동을 통해 테마 정보 즉시 갱신 가이드.

### 7. 진단 및 모니터링 강화
*   **[M2] 단위 검증 로그**: 장 개시 후 첫 5개 종목에 대해 `val_million` -> `val_absolute` 변환 과정을 로그로 출력하여 실측 검증 지원.
*   **[M3] 연산 최적화**: `rt_surge` 계산 결과를 루프 내에서 재사용하여 이중 호출 제거.

## 📈 향후 계획
*   장 개시 후 `M2 Verify` 로그를 통해 Kiwoom TR의 거래대금 단위(백만원 vs 천원) 최종 실측 및 보정.
*   런레이트 필터 도입 후 HOT 랭킹의 품질 변화 모니터링.
*   `trades.db`의 `reason` 컬럼이 `PersistenceManager`에서 누락되는 문제 추가 수정 예정.
*   섹터/테마 매핑 커버리지 90% 이상으로 추가 확장.
