# 2026-04-14 작업일지 (Work Log)

## 1. 개요 (Overview)
- **작업명**: MainWindow 리팩토링 Step 4B 완료 및 Institutional Trade Guards 최적화 (V21.9)
- **작업 등급**: **L4(Refactoring) / L1(Strategy Architecture)**
- **작업자**: Antigravity (AI Assistant)
- **참조 문서**:
    - `Antigravity_rules/governance/AI_Work_Constitution_and_Reference_Map.md` (v1.8)
    - `Antigravity_rules/rules/Failure_Patterns_and_Guards_ko.md`
    - `docs/planning/bug_fix_overheat_checklist.md`

## 2. 작업 상세 (Task Details)

### 2.1 [L4] MainWindow 리팩토링 Step 4B 최종 검증
- **목표**: `update_mme_preview` 로직 분리 및 `ConfigManager`로의 이관
- **라인 수 감축**: 2,308 → 2,180. 총 **128라인** 감축 완료. 누적 1,346라인 감축(-38.2%).
- **위젯/빌더 안정화**: `BodyBuilder` (좌측 패널 폭 조정) 및 `PremiumHeaderWidget` (부모 참조 제거 및 콜백 적용) 패치 적용.

### 2.2 [L3] Decision Hub (SignalStatusWidget) 과열 차단 버그 수정
- **증상**: 엔진이 과열(OVERHEAT)되어 진입을 차단하는 경우, 위젯 하단의 진행률 게이지와 4가지 체크리스트(✓/✕)가 완전히 사라지고 "추세 분석 중"만 출력됨.
- **원인 (확정)**: `strategy.py`의 `_get_signal_internal`에서 `vol_ratio` 초과(Layer 1 Hard Guard) 시 조기 리턴(Early Return)하는 로직이, `entry_detail` 및 `logic_gap` 파싱 로직보다 **앞쪽**에 위치하여 데이터가 소실됨.
- **수정 사항 (`app/ai/strategy.py`)**:
    - `entry_score`, `entry_detail`, 그리고 `logic_gap`(체크리스트 내용) 계산 로직을 `Hard Safety Gate` 진입 **이전으로 이동**.
    - 차단 발동 시 계산된 데이터를 `SignalResult`에 모두 탑재(`logic_gap=logic_gap, entry_detail=entry_detail, entry_score=entry_score`)하여 리턴.
    - `action` 메시지를 더욱 명확하게 `과열 차단 (거래량 {vol_ratio_val:.1f}x, 상한 {vol_cap}x)`로 업데이트.

### 2.3 [L1] Institutional Trade Guards 최적화 (V21.8 ~ V21.9.2)
- **V21.8 가드 엔진 Config화**: `execution_manager.py`의 하드코딩된 `score < 60` 차단 로직을 제거하고, `config_engine.py`의 `ENTRY_SCORE_FLOOR` 및 국면별 임계치(`ENTRY_THRESHOLD_UPTREND` 등)와 연동하여 제어권 완전 확보.
- **V21.9.1~2 Tier 1 자동 승격 (Auto-Promotion) 정밀 수리**: 
    - **Surveillance Gap 수정**: 승격된 종목이 Top 20 밖으로 밀려날 경우 OCX 구독이 해제되어 데이터가 단절되는 치명적 결함 수정 (`SurveillanceController.reconcile` 4-arg 확장 및 `promoted_pool` 보호 로직 삽입).
    - **로그 폭탄 제거**: 30초마다 15줄씩 발생하던 `🚀 [Promotion]` 로그를 `is_new` 체크를 통해 신규 진입 시 1회로 제한.
    - **Signal 파이프라인 개선**: `MainWindow` → `SurveillanceController` 간의 랭킹 스냅샷 시그널을 4-arg(`top_codes, held, fav, promoted`)로 확장하여 상태 동기화 정합성 확보.
- **엄격한 TTL(Time-To-Live) 정책 적용**: `ExecutionManager.on_timer`에서 틱마다 TTL을 연장하던 Keep-alive 로직을 삭제하여, 외부 탐색기에 의한 명시적 연장이 없는 한 정확히 5분 후 자동 강등(Demotion)되도록 아키텍처 원칙 준수.
- **전 구간 매매 개입 분석 (L0~L5)**: 스캐너(L0)부터 주문(L5)까지의 차단 요인을 6단계 레이어로 정립하여 `trade_intervention_landscape_report_ko.md` 작성.

### 2.4 [L0] Shadow Log 기반 주도주 차단 원인 정밀 진단
- **현황**: `UPTREND` 종목이 감지되고 있으나 실제 체결로 이어지지 않는 원인을 `shadow_guard_log.jsonl` 분석을 통해 규명.
- **주요 차단 요인**:
    - **Reward Check (1위)**: UPTREND 국면에서도 MIXED 전략의 1.5% 리워드 기준이 동일하게 적용되어 5건 차단 확인.
    - **MTF Guard**: 60분봉 MA240 하방 정렬 가드에 의해 1건 차단 확인.
- **결론**: 엔진의 탐색 능력(Tier 1 승격)은 복구되었으나, 기계적 가드(Reward/MTF)가 강세장 초입의 신호를 억제하고 있음.

## 3. 불변성 및 안정성 검사 (Invariant & Stability Check)
- **불변성 위반**: 없음 (Violated: None)
- **수정 정합성**: `surveillance.py` 슬롯 수정 → `main_window.py` 시그널 확장 순서로 적용하여 런타임 연결 오류 원천 차단. AST 검증 통과.

## 4. 향후 계획 (Next Steps)
- **전략 임계치 튜닝 (P1)**: Shadow Log 분석 결과를 바탕으로 `UPTREND` 국면 내 `Reward Check` 임계값을 1.5% → 1.2% 등으로 하향 조정 검토.
- **MTF 가드 예외 적용 (P2)**: 강한 `UPTREND` 돌파 시 60분봉 이평선 가드를 일시적 유예하거나 완화하는 로직 검토.
- **RankingManager 가중치 미세 조정 (P3)**: `HEAT` 점수 산출 시 거래대금(40%)과 등락률(30%) 외에 체결강도 변화량을 더 민감하게 반영할지 검토.

