# 2026-03-27 작업일지 (Work Log)

## [V19.4.1] 메타데이터 무결성 최종 수호 & 이벤트 레이스 컨디션 해결 (L4)
- **목표**: 전일(26일) `strategy.py` 패치 이후에도 실거래에서 `UNDEFINED` 메타데이터가 발생하던 현상의 근본 원인(Root Cause)을 추적하고 완전 해결.
- **주요 변경**:
    - **이벤트 레이스 컨디션(경쟁 상태) 발굴 및 제거 (`main_window.py`, `engine.py`)**:
        - 원인: Kiwoom 엔진에서 매수 체결 시, `sig_order_confirmed` 이벤트가 `sig_order_fill` 이벤트보다 단 *마이크로초* 먼저 발생.
        - 문제: `main_window.py`에서 `sig_order_confirmed`를 `order_mgr.clear_pending`에 연결해둔 탓에, 체결 정보(`sig_order_fill`)가 처리되기도 전에 메모리상의 메타데이터 캐시(`pending_signal_meta`)가 비워지는 치명적 논리 오류 발견.
        - 조치: 중복/조기 호출되던 `self.engine.sig_order_confirmed.connect(self.order_mgr.clear_pending)` 연결을 삭제. 체결(`on_fill_event`) 로직 내부에서 자연스럽게 `clear_pending`이 호출되도록 흐름 정상화.
- **결과**:
    - 전략 로직단(`strategy.py`: Categorical Classification) 패치와 시스템 이벤트단(`main_window.py`: Race Condition) 패치가 모두 완료되어, 자동 및 수동 거래 시 메타데이터(`strategy_type`, `regime`)가 `trades.db`에 누락 없이 정확하게 영속화됨.
    - SSoT(Single Source of Truth) 기반 대시보드의 데이터 정합성 100% 확보.
    - **[추가 패치] 수동 매도 식별 (Manual Exit Tracking)**: 사용자가 UI를 통해 직접(수동) 매도한 항목은 내부적으로 `OrderManager.send_order` 파이프라인을 우회하여 들어옴을 감지하고, 해당 경우 `청산방식(exit_type)`을 `NORMAL` 대신 **`MANUAL`**로, 사유를 `USER_MANUAL_SELL`로 명시하도록 `order_manager.py` 구조 개선.
    - **시스템 매도 식별 (System Exit Tracking)**: 자동화된 위험 관리(손절) 시 `STOP_LOSS`, 타이머/오버나잇 청산 시 `TIME_EXIT` 등 시스템의 의도가 그대로 `청산방식`에 기록되도록 `execution_manager.py` 보완.

## [V19.4.6] 시스템 안정성 강화 및 비동기 하이드레이션 (L3)
- **목표**: 포렌식 대시보드 로딩 시 GUI 프리징(응답 없음) 현상 해결 및 Windows 환경 저수준 호환성 확보.
- **주요 변경**:
    - **비동기 데이터 로딩 (`ForensicDataLoaderWorker`)**: 중량 쿼리가 포함된 `forensics.db` 조회를 메인 쓰레드에서 분리(`QThread` 적용). 대시보드 오픈 시 발생하던 '응답 없음' 현상 완전 제거.
    - **Unicode/Emoji 클리닝**: Windows `cp949` 콘솔 인코딩 환경에서 `UnicodeEncodeError`를 유발하던 로그 내 이모지(Emoji)를 전수 제거하여 엔진 안정성 확보.
    - **32-bit 아키텍처 가드**: 키움 OpenAPI(32비트 전용)와의 호환성을 위해 `main.py`에 32비트 실행 환경 강제 체크 로직 재구축.

## [V19.5] Flight Recorder: Deep-Search 메타데이터 자가 치유 (L5)
- **목표**: 프로그램 재시작 또는 캐시 오염 시 발생하는 `UNDEFINED` 트레이드 분류 문제의 근본적 방어.
- **주요 설계 (3-Tier Safety Net)**:
    - **1단계 (Memory)**: 실시간 `positions` 딕셔너리 참조.
    - **2단계 (Persistent Cache)**: `overrides.json` 파일 기반 복원.
    - **3단계 (Flight Recorder Recovery)**: 위 두 단계에서 데이터가 없을 경우, `forensics.db` 원장을 직접 쿼리하여 진입 당시의 `BUY` 시그널 메타데이터를 역추적하여 발굴.
- **결과**: 메모리 휘발이나 캐시 유실 상황에서도 과거 로그 원장을 기반으로 전략명(VOLUME, TREND 등)과 국면(Regime)을 100% 정합성 있게 복원하는 **'자가 치유(Self-Healing)'** 능력 확보.

---
*NCQ Work Log System v1.2*
