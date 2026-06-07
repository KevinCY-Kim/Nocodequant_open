# 📅 2026-05-15 업무 일지 (Daily Work Log)

## 🎯 주요 목표 및 이슈 정의
- [x] **런타임 콜드 스타트(Cold Start) 상태 복구 무결성 확보**: 재기동 시 `bars_held`, `peak_price`, `sl` 등이 초기화되는 현상 해결
- [x] **`TimeExit` 경과 봉 수 역산 로직 정밀화**: `entry_time` 영속성을 기반으로 프로그램 중단 시간 동안의 실제 거래 봉 수 복원
- [x] **`Ratchet SL` 하향 이동 버그 원천 차단**: 트리거 기준을 종가에서 장중 고점으로 변경하고, 재기동 시 `max_price` 원장 복구 연동
- [x] **Live MFE 대시보드 인터랙션 고도화**: 종목 선택 시 하단 Exit 전략 카드로 자동 스크롤되는 연동 로직 구현 (팩트체크 완료)

---

## ✅ 상세 작업 및 패치 내역

### 1. `strategy.py` & `main_window.py` 상태 복구 파이프라인 (L3)
* **원인 분석**: 서버 잔고 동기화 시 수신되는 데이터에 전략 메타데이터(`entry_time`, `strategy_type`, `max_price`)가 누락되어, 엔진이 매번 '신규 외부 포지션'으로 오인하고 초기값으로 덮어쓰는 구조적 결함.
* **해결 방안**:
    - `main_window.py`: `sync_position` 호출 직전, `state_manager`를 통해 SQLite에 저장된 원본 메타데이터를 추출하여 서버 데이터에 강제 주입.
    - `strategy.py`: 주입된 `entry_time`을 활용하여 2단계 복원 로직 구현 (DataFrame 인덱스 비교 + KST 장 시간 기준 시각 역산).
    - **효과**: 프로그램이 꺼져 있던 시간에도 실제 발생한 거래 봉 수를 정확히 카운트하여 `TimeExit` 및 `WarmupGuard`가 정합성을 유지함.

### 2. `_engine_exit.py` 하드닝 및 로직 정규화 (L3)
* **트리거 기준 변경**: `curr['close']` → `pos['peak_price']`. 
    - 봉 내부의 실시간 최고점(`high`)이 래쪳 라인을 터치하면 즉시 물리적 바닥 SL을 Lock-on 하도록 개선하여 휩쏘 대응력 강화.
* **[SSoT] ENUM 타입 정규화 (팩트체크 반영)**:
    - `Inactivity Guard` 판단 시 할당되던 `"시청산"` 문자열을 `DecisionLabel.SELL` ENUM으로 교체.
    - `ExitResult` 스키마와의 타입 일치성을 확보하여 주문 실행 및 포렌식 로그 레이어에서의 비교 연산 무결성 보장.

### 3. `live_mfe_tab.py` UI 인터랙션 및 시그널 하드닝 (L1)
* **시그널 연결 복구**: `QTableWidget`의 `cellClicked` 시그널이 핸들러(`_on_row_clicked`)와 연결되지 않았던 누락 사항을 확인하여 연결 완료.
* **스크롤 로직 정밀화**:
    - 종목명 파싱 시 마지막 괄호를 탐색하여 코드를 정확히 추출하도록 개선.
    - `ensureWidgetVisible` 및 `verticalScrollBar().setValue()` 듀얼 폴백을 통해 카드 이동의 확실성 보장.

---

## 🔒 거버넌스 및 구조 맵 체크리스트 (Governance Checklist)

1. **작업 등급**: L3 (핵심 런타임 상태 복구 및 SSoT 데이터 정규화)
2. **단일 진실 공급원(SSoT)**: `persistence_manager` 원장 데이터와 `schema.py` ENUM 규격을 유일한 권한으로 삼아 로직 정합성 유지.
3. **구조 맵 오토-싱크 검증**: 본 세션의 수정은 기존 파일 구조 내의 로직 강화에 국한되며, `Project_Structure_Map_ko.md` 상의 보호 구역(Protected Zone) 권한 규칙을 준수하였음을 선언함.
4. **불변성 준수**: 래쪳 SL의 단방향성 및 재시작 시 `peak_price` 복원을 통해 시스템 불변성(Invariant) 보호 완료.

---
**작성자**: Antigravity AI
**상태**: 2026-05-15 15:50 (KST) 팩트체크 반영 및 최종 보고 완료
