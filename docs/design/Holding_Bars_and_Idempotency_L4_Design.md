# [L4 DESIGN NOTE] 보유봉(Holding Bars) 정밀화 및 체결 멱등성(Fill Idempotency) 아키텍처 개선

## 1. 개요 (Overview)
본 문서는 실현손익 왜곡의 근본 원인인 키움 API의 중복 체결 이벤트를 차단하고, 오버나이트/휴장 시간 노이즈가 제거된 정확한 보유봉수(Holding Bars) 산출을 위한 시스템 아키텍처 변경 사항을 기술합니다.

## 2. 변경 등급 (Change Classification)
- **등급**: **L4 (아키텍처 및 데이터 흐름 변경)**
- **사유**: `Strategy` -> `Execution` -> `Order` -> `Persistence`에 이르는 전 경로에 `bar_index` 데이터 필드를 추가하고, `OrderManager`의 실행 흐름에 멱등성(Idempotency) 가드를 삽입함.

## 3. 영향을 받는 모듈 (Impacted Modules)
- `app/ai/strategy.py`: `SignalResult` 구조 확장 및 인덱스 추출 로직 추가.
- `app/services/managers/execution_manager.py`: 신호 전파 시 메타데이터 매핑 레이어 수정.
- `app/services/managers/order_manager.py`: 전역 체결 캐시(`processed_fills`) 및 하이브리드 보유봉 계산 로직 구현.
- `app/services/managers/schema.py`: `TradeSignal` 데이터 클래스 확장.
- `app/services/managers/utils.py`: [NEW] 장 운영시간 필터링 유틸리티.
- `app/ui/strategy_dashboard.py`: 필터링 및 컬럼 폭 최적화.

## 4. 아키텍처 차분 (Architecture Diff)

### AS-IS
- **보유봉 계산**: 청산 시점의 단순 시간 차이(`Exit - Entry`)를 분단위로 나누어 계산. (장 마감/주말 노이즈 포함)
- **체결 처리**: API 이벤트를 수신하는 즉시 DB에 기록. 중복 수신 시 데이터 증폭 발생.
- **데이터 흐름**: `TradeSignal`은 가격과 수량 정보 위주로 구성됨.

### TO-BE
- **보유봉 계산**: `(Current Index - Entry Index)` 기반의 절대 봉수 계산. (데이터 원본의 정수 인덱스 활용)
- **폴백 메커니즘**: 인덱스 유실 시 KOSPI 장 운영시간(09:00~15:30)만 가산하는 `calculate_holding_bars` 적용.
- **체결 처리**: `processed_fills` (Set-based Cache)를 통한 입구 컷 필터링 적용.
- **데이터 흐름**: `bar_index` 및 `timeframe`이 전략 생성 시점부터 영속화 레이어까지 동기화되어 전파됨.

## 5. 불변성 가드 (Invariant Guards)
- **Fill Idempotency**: 모든 체결은 `fill_id`를 기반으로 단 1회만 처리되어야 함.
- **Time Integrity**: 어떠한 경우에도 `bars_held`는 음수가 될 수 없으며, 실제 개장 시간을 초과할 수 없음.

## 6. 마이그레이션 전략 (Migration Strategy)
- 기존 `trades.db` 레코드는 `bars_held`가 0으로 기록된 상태이나, 신규 로직 적용 이후 발생하는 모든 매매부터 정밀 수치가 기록됨.
- 소급 적용이 필요한 경우 `entry_time`과 `exit_time`을 기반으로 `calculate_holding_bars`를 재실행하는 복구 스크립트를 가동할 수 있음.

## 7. 롤백 플랜 (Rollback Plan)
- `OrderManager`의 `processed_fills` 캐시를 비우거나, `schema.py`의 필드 확장을 되돌려 이전의 단순 시간 기반 로직으로 즉시 복구 가능.
