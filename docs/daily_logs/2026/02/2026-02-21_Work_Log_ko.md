# 2026-02-21 AI System Upgrade Work Log

## 1. 개요 (Overview)
V14.0 업그레이드와 V14.7 수익 최적화 엔진(POE) 구축을 통해 시스템의 '자기 가변성(Self-Adaptability)', '투명성(Transparency)', 그리고 '수익 효율성(Profit Efficiency)'을 완성했습니다. 이제 시스템은 단순 관측을 넘어, 스스로를 진단하고 수익을 위해 파라미터를 자동 튜닝하는 'Institutional-grade' 최적화 루프를 가동합니다.

## 2. 주요 변경 사항 (Key Changes)

### [1] ADX 기반 지능형 국면 전환 (Regime Switching with Hysteresis)
- **Hysteresis 구현**: ADX가 25 상향 돌파 시 `TREND`로 진입하고, 22 하향 돌파 전까지는 `TREND`를 유지하는 **Dead-zone(22~25)** 기법을 도입하여 잦은 국면 전환(Chattering)을 방지했습니다.
- **3-bar Cooldown**: 국면 전환 직후 3개 캔들 동안 추가 전환을 금지하여 시스템 안정성을 극대화했습니다.

### [2] 동적 포지션 출구 전략 (Dynamic Position Adaptation)
- **보유 포지션 실시간 가변**: 국면이 전환되면 기존에 보유한 포지션의 TP/SL/Trailing 파라미터가 실시간으로 재설정됩니다.
    - **TREND 시**: 수익 극대화를 위해 익절가 상향 및 트레일링 완화.
    - **RANGE 시**: 리스크 관리를 위해 빠른 익절 및 타이트한 추적.

### [3] AI 도슨트(AI Docent) UI 현대화
- **Penalty Icons**: 엔진이 페널티를 부여한 이유를 아이콘(🐻 글로벌 베어, ⚠️ 수급 미흡)과 툴팁으로 즉각 확인할 수 있게 개선했습니다.
- **Regime Monitor**: 하단 패널에 현재 국면(Regime), 수급(VSA), 글로벌 추세(HTF) 상태를 실시간 시각화하는 모니터를 구현했습니다.
- **Surgical UX Compression**: 기존 280px 높이 내에서 모든 신규 정보를 수용하기 위해 여백 및 폰트 크기를 최적화한 초고밀도 레이아웃을 적용했습니다.

### [4] 중앙 파라미터 거버넌스 (ACTIVE_CONFIG Alignment)
- `ADX_TREND_THRESHOLD`, `VOL_SPIKE_RATIO` 등 모든 핵심 하드코딩 리터럴을 `ACTIVE_CONFIG`로 격상하여 관리 일관성을 확보했습니다.

### [5] 수익 최적화 엔진 (v14.7 Profit Optimization Engine - POE)
- **Monetized FSM**: Ready(75%), Entry(100%) 등 엔진 상태에 따라 **자본 투입 비중(Capital Exposure)**을 차등 분배하는 수익 지향적 상태 머신을 구현했습니다.
- **Engine Monitor (Dual-Path)**: 실시간 감시용 JSONL과 고속 분석용 **SQLite(ESL_cold.db)**를 동시에 운용합니다. 32비트 환경(trade_exe32)에서도 100% 안정성을 보장하기 위해 표준 라이브러리 기반으로 설계되었습니다.
- **Parameter Auto-Tuner**: 과거 헛발질(False Break) 비율을 분석하여 `MIN_CONFIDENCE` 임계값을 스스로 높이거나 낮추는 자가 학습 루프를 완성했습니다.
- **1-min Heartbeat Stress Test**: 1분봉 고밀도 상황에서 엔진의 누수 없는 상태 전이와 가속 로직을 실시간 검증할 수 있는 테스트 모드를 UI에 통합했습니다.

## 3. 기술적 상세 (Technical Details)

- **Impacted Modules**:
    - `app/ai/strategy.py`: `_update_regime_state`, `_adapt_existing_positions` 및 Gap 가속 로직.
    - `Main_NCQ.py`: `SignalStatusWidget` 고밀도 레이아웃 및 **POE 우측 제어 패널** 추가.
    - `app/core/engine_fsm.py`: [NEW] 자본 가중치 기반 상태 머신.
    - `app/core/engine_monitor.py`: [NEW] SQLite 기반 고속 텔레메트리 엔진.
    - `app/core/parameter_autotuner.py`: [NEW] 자가 진단 및 파라미터 최적화 루프.
    - `app/services/managers/execution_manager.py`: 실시간 관측 및 상태 전이 훅(Hook) 통합.

## 4. 인과관계 및 연결성 (System Logic Connectivity)
1. **주도주 검색 (Smart Heat Index)**: 거래량/등락률/체결강도를 조합하여 '잠재적 후보'군을 발굴.
2. **의사결정 허브 (Decision Hub)**: 발굴된 종목을 대상으로 3중 지표 정규화 및 수급 확증(VSA) 필터를 적용하여 최종 진입/관망 결정.
3. **국면 분석 (Regime Analysis)**: 진입 후 시장 환경 변화를 감지하여 '사후 관리(Adaptive Exit)' 모드로 자동 전환.
4. **AI GPT 분석 (Deep Insight)**: 이 모든 과정을 인간이 이해할 수 있는 자연어로 풀어서 설명.

---
*Constitution Category: L4 Architecture Change*
