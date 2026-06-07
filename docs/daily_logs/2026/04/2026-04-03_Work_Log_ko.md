# 2026-04-03 작업일지 (Work Log)

## [V19.10] 전략성과보드 데이터 파편화 해결 및 논리적 집계(Aggregation) 구현 (L4)
- **목표**: 부분 체결(Partial Fills)로 인해 발생하는 대시보드 내 중복 로그 및 승률 왜곡 문제를 해결하고, 단일 주문(Order Identity) 기반의 일관된 성과 지표 산출.
- **주요 변경**:
    - **이벤트 라우팅 격리 (`persistence_manager.py`)**:
        - `ORDER_FILLED` 이벤트를 `forensics.db` 전용 감사 로그로 격리하여 성과 분석용 `trades.db` 오염 방지.
        - `FILL` (OrderManager 집계 완료) 이벤트만 성과 테이블에 기록되도록 필터링 규칙 강화.
    - **SQL 쿼리 레벨의 논리적 병합 (`persistence_manager.py`)**:
        - `get_recent_trades` 및 `get_strategy_performance_summary`에 `GROUP BY COALESCE(order_id, gtid)` 기술 적용.
        - 파편화되어 이미 DB에 기록된 기존 데이터에 대해서도 UI 상에서 하나의 논리적 거래로 자동 합산되도록 사후 보정 로직 구현.
    - **수학적 정밀도 확보**:
        - 수익률(`pnl_ratio`) 계산 시 단순 평균 방식에서 탈피하여, `SUM(pnl_amount) / (MIN(entry_price) * SUM(qty))` 공식을 통한 가중 합산 방식으로 전면 개편.
    - **DB View 무결성 동기화**:
        - `v_strategy_performance` 뷰 정의를 CTE(Common Table Expression) 기반의 2단계 집계 쿼리로 업데이트.
        - 앱 재시작, 외부 BI 도구 연동, 직접 SQL 조회 시에도 대시보드와 동일한 '단일 진실 공급원(SSoT)' 보장.

## [V20.0] AI 전략 튜닝 엔진 (Strategy Diagnostic Engine) 고도화 (L5)
- **목표**: 단순 리포트 수준의 튜닝 가이드를 점수화(Scoring)와 즉각적 액션(Actionable Advice)이 결합된 상위 1% '전략 의사결정 엔진'으로 격상.
- **주요 변경**:
    - **진단 엔진 구현 (`strategy_dashboard.py`)**:
        - `StrategyDiagnosticEngine` 클래스 내부화: 승률(20%), Profit Factor(40%), Edge Ratio(40%) 가중치 기반 정량적 전략 점수화 로직 구축.
        - MFE/MAE/Edge 분석을 통한 5단계 전략 상태 판정(EXCELLENT, GOOD, MIXED, WEAK, DISABLE).
        - `MIN_TRADES` 임계치를 5건으로 현실화하여 데이터 부족 시에도 신뢰도 경고와 함께 부분 진단 제공.
    - **데이터 집계 레이어 확장 (`persistence_manager.py`)**:
        - `get_strategy_performance_summary` SQL 쿼리를 CTE(Common Table Expression) 기반 2단계 집계로 전면 개편.
        - 거래 단위(`order_id`/`gtid`)별로 `avg_mfe`, `avg_mae`, `avg_edge_ratio`를 선행 계산하여 집계 오차(Cartesian Product) 원천 차단.
    - **UI/UX 디자인 혁신 (V20.0.4)**:
        - **초밀착 배지 스타일(Badge-style)**: 각 지표를 다크 골드(#1c1503) 톤의 라운드 박스로 시각화하여 정보 밀도 및 프리미엄 터미널 감성 확보.
        - **시선 집중(Focal Alignment)**: 'Eye Tracking Scatter' 해결을 위해 100% 폭 테이블을 배제하고 좌측 정렬 고정 폭(150px) 레이아웃 적용.
        - **QTextBrowser 최적화**: Flexbox 기능을 배제하고 HTML Table 기반의 안정적 레이아웃 체계 구축.
- **최종 결과**:
    - 전략별 실질적 '엣지(Edge)'와 '손절 효율(MAE)'을 기반으로 한 정밀한 튜닝 가이드 제공.
    - 시각적 피로도를 낮추고 데이터 가독성을 극대화한 기관급 거래 터미널 UI 완성.

## [V20.1] AI 튜닝 브릿지 & 코디네이터 (TuningCoordinator) 구현 (L5)
- **목표**: V20.0 진단 엔진의 제안(Action)을 실제 `ACTIVE_CONFIG` 파라미터 변경에 자동 연결하는 '자율 의사결정 브릿지' 구축.
- **주요 변경**:
    - **TuningCoordinator 신설 (`app/services/tuning/coordinator.py`)**:
        - `ACTION_PARAM_MAP`: 액션 타입(`RSI_FILTER`, `TRAILING_STOP`, `SCALE_UP`, `EDGE_LOW`)을 국면(TREND/RANGE/CHAOS)별 `ACTIVE_CONFIG` 키와 변동값으로 매핑.
        - **포지션 안전 정책**: 포지션 보유 중 → 큐잉(최신 1개 Overwrite), 포지션 없음 → 즉시 적용.
        - **멱등성(Idempotency)**: `tune_id(UUID)` 기반으로 동일 튜닝 중복 적용 원천 차단.
        - **24h TTL Auto-Expiry**: 큐에 대기 중이던 낡은 튜닝 자동 폐기.
        - **Governance Logging**: 모든 적용/큐잉/폐기 이력을 `forensics.db`에 Diff 포함 기록.
    - **UI 브릿지 (`strategy_dashboard.py`)**:
        - 진단 카드 내 액션별 `[적용하기 ▶]` 버튼 렌더링 (TuningCoordinator 미연결 시 자동 숨김).
        - `anchorClicked` → `_handle_action_clicked` → `process_proposal()` 완전 연결.
    - **시스템 연동 (`main_window.py`)**:
        - `MainWindow.__init__`에서 `TuningCoordinator` 생성.
        - `StrategyDashboardWindow` 생성 시 `tuning_coordinator=` 인자로 주입.
        - `OrderManager`에 post-init 방식으로 코디네이터 주입 (composition root 비침습).
    - **청산 후 자동 적용 훅 (`order_manager.py`)**:
        - `on_fill_event`의 `is_position_closed` 판정 직후 `tuning_coordinator.on_position_closed(strategy_type)` 호출.
        - 대기 중인 큐가 있으면 자동으로 파라미터에 반영.
- **최종 결과**:
    - **완전한 자율 루프 완성**: 진단 → 제안 → 클릭 → 큐잉/즉시 적용 → 청산 후 자동 반영 → Governance 이력 기록.
    - **무중단 안전 설계**: 코디네이터가 미주입 상태면 기존 동작에 전혀 영향 없음. 모든 예외는 로그로만 처리.

## [V20.4] 전역 프로젝트 구조 맵 (Project Structure Map) 동기화 (L4)
- **목표**: V20.1 자율 튜닝 아키텍처 도입에 따른 물리적/논리적 구조 변경 사항을 공식 맵에 반영하여 시스템 무결성(Integrity) 보장.
- **주요 변경**:
    - **비주얼 트리 갱신**: `app/services/tuning/coordinator.py` 신규 경로 추가 및 계층 구조 동기화.
    - **보호 구역(Protected Zone) 지정**: `TuningCoordinator`를 **[PROTECTED:L4]** 등급으로 승격하여 임의 수정 방전 처리.
    - **도메인 가이드 최신화**: `app/services/tuning/` 하위를 '자율 파라미터 튜닝 및 거버넌스 이력 관리' 도메인으로 공식 정의.
- **최종 결과**:
    - AI 작업 헌장 v1.7에 따른 '구조 맵 오토-싱크' 완료.
    - 권한 부여 체계(Permissions)와 실제 파일 시스템 간의 100% 정합성 확보.

---
*NCQ Work Log System v1.5*

