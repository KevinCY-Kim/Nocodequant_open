# 2026-03-26 작업일지 (Work Log)

## [V19.4] 거버넌스 룰스 동기화 및 매수 메카니즘 정합성 패치 (L3)
- **목표**: 거버넌스 문서와 코드의 완전한 일치(SSoT)를 달성하고, 매수 시 메타데이터가 `UNDEFINED`로 깨지는 현상을 추적하여 해결.
- **주요 변경**:
    - **거버넌스 문서 동기화 (V19.4)**:
        - `Decision_Flow_Architecture.md`, `Parameter_Invariants.md`, `Signal_Gating_Rules.md`를 최신 코드(Tiered EDA, MFE/MAE, PnL 공식 등)에 맞춰 업데이트.
        - `ADX_TREND_THRESHOLD` 및 `RANGE_THRESHOLD`를 `config_engine.py` 실측치인 **22**로 통일하여 문서-코드 간 수치 불일치 해소.
        - `AI_Work_Constitution_and_Reference_Map.md`를 v1.3으로 갱신하여 3-Layer Invariant Guard(Docs, Assert, Log) 체계 강화.
    - **메타데이터 유실(UNDEFINED) 원인 규명 및 수정 (`strategy.py`)**:
        - `Soft Entry Score` 도입 이후, `entry_score`는 통과했으나 `_classify_strategy_type`의 엄격한 판정(Hard Gate)에 걸려 `UNDEFINED`로 반환되던 데이터 크랙 버그 수정.
        - 매수 지시 시 깐깐한 기준에 미달하더라도 기여도가 높은 요소(수급, 추세, 가격 등)를 찾아 적절한 라벨(`VOLUME`, `TREND`, `BREAKOUT` 등)을 부여하는 Fallback 로직 도입.
    - **디버깅용 수동 매수 기능 도입 (`main_window.py`)**:
        - 매수 메타데이터 전파 과정을 실시간 추적하기 위해 'Hot Stocks (RankingWidget)' 우클릭 메뉴에 `🟢 임시 수동매수` 기능 추가.
        - 수동 주문 시에도 `strategy_type="MANUAL"`, `regime="TEST"`로 명확히 DB에 남도록 보완.
    - **계층형 감시(Tiered EDA) 운영 안정화 (`execution_manager.py`)**: 
        - Tier 1(실시간)과 Tier 2(배치) 간의 승격/강등 메커니즘을 문서화하고, 매수 신호 발생 시 메타데이터가 `TradeSignal` DTO를 거쳐 `OrderManager`까지 온전하게 전달되도록 파이프라인 정비.
- **결과**:
    - 거버넌스(Rules)와 실제 구현(Implementation) 사이의 괴리를 제거하여 시스템의 신뢰도를 높였으며, `trades.db`의 데이터 무결성을 확보하여 분석 대시보드의 통계 정확도를 근본적으로 개선함.

---
*NCQ Work Log System v1.2*
