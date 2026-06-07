# 2026-03-17 작업일지 (Work Log)

## [Phase 14.1] 실현손익 SSoT 동기화 및 정합성 강화 (L4)

### 1. 개요 (Overview)
- **목표**: 시스템 상의 실현손익과 HTS(Home Trading System) 데이터 간의 괴리를 해결하기 위해, 로컬 계산 방식을 정교화하고 서버(opt10077) TR 데이터를 진실 공급원(SSoT)으로 채택하여 1원 단위까지 완벽한 동기화 구현.
- **수정 등급**: **L4 (아키텍처 및 데이터 흐름 변경)** - 데이터의 소유권을 로컬 매니저에서 서버 TR 응답으로 전환하고, 엔진-매니저 간 동기화 채널(Signal)을 재설계함.

### 2. 주요 변경 사항 (Proposed Changes)

#### [1] 실현손익 데이터 흐름 재설계 (Architecture & Data Flow)
- **서버 동기화 가드 (SSoT Sync)**: `SELL` 체결 확인 시 2초 후 `opt10077`(당일실현손익상세) TR을 자동으로 요청하도록 `KiwoomEngine` 수정.
- **강제 동기화 슬롯 (Force Sync Slot)**: `StateManager` 내에 서버 데이터로 로컬 상태를 덮어쓰는 `force_sync_pnl()` 메서드 구현. 
- **시그널 채널 연결**: `engine.sig_realized_pnl`과 `state_mgr.force_sync_pnl`을 `composition_root.py`에서 연결하여 데이터 흐름 자동화.

#### [2] 로컬 계산 공식 고도화 (Local Calculation Refinement)
- **세금/수수료 분리 계산**: 기존 `0.23%`의 단순 버퍼 방식을 폐기하고, HTS와 동일한 **증권거래세 0.20% + 양방향 수수료 0.015%** 공식을 적용하여 서버 동기화 전 오차 범위를 최소화함.

#### [3] 로그 폭주 방지턱 복구 (Log Spam Buffer - L3)
- **점수 및 쿨다운 가드**: `BUY` 점수 60점 미만 필터링 및 동일 종목 60초 쿨다운 적용.
- **긴급 예외 처리**: `SELL` 및 `EMERGENCY` 신호는 60초 가드를 우회하여 즉시 반응하도록 설계하여 생존성 확보.

#### [4] TR 파싱 버그 수정 (Engine Robustness)
- **GetCommData 파라미터 교정**: `opt10077` 파싱 시 `rq_name`을 `record_name`으로 잘못 사용하던 버그를 수정하여 정상적인 데이터 추출 보장.
- **다중 행 합산 로직**: 당일 여러 종목을 매도했을 경우를 대비하여 모든 레코드 행을 순회하며 실현손익 총합을 집계하도록 로직 강화.

### 3. 영향을 받은 모듈 (Impacted Modules)
- `app/ai/engine.py`: TR 파싱 버그 수정 및 실시간 체결 후 동기화 트리거 추가.
- `app/services/managers/state_manager.py`: 실현손익 공식 수정 및 동기화 메서드 추가.
- `app/core/composition_root.py`: 엔진-매니저 간 신규 데이터 채널 연결.

### 4. 불변성 및 경계 점수 (Invariants & Governance)
- **Invariant Check: Pass** - `AI 작업 헌장 v1.2`의 SSoT 원칙(§1.2)을 완벽히 준수하며, 로컬 추정치보다 서버 확정치를 우선하도록 설계됨.
- **Boundary Check: Pass** - 모든 데이터 연산 및 상태 관리는 백엔드(`engine`, `state_mgr`)에서 수행하며, UI는 이를 반영만 하도록 설계됨.
- **Failure Pattern Registration**: '실현손익 정합성 괴리(PnL Drift)' 실패 패턴을 `Failure_Patterns_and_Guards_ko.md`에 공식 등록 완료.

---

## 5. 최종 검증 (Final Verification)
- [x] **구문 검사**: 수정된 3개 모듈(`engine`, `state_manager`, `composition_root`)에 대한 `py_compile` 통과 확인.
- [x] **로직 검증**: 매도 체결 시 2초 후 TR 요청 및 시그널을 통한 상태 전파 프로세스 설계 확인.

---
*NCQ Work Log System v1.2*
