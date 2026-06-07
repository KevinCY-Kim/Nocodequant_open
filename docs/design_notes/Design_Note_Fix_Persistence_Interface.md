# Design Note: PersistenceManager 인터페이스 불일치 수정 (L3)

본 문서는 **AI 작업 헌장 v1.1**에 따라 **L3 (핵심 로직 및 실행부 수정)** 등급 작업의 설계 및 무결성을 증명하기 위해 작성되었습니다.

## 1. 개요 (Overview)
- **작업 등급**: L3
- **이슈**: `OrderManager.on_fill_event`에서 `PersistenceManager.emit_event` 호출 시 인수 개수가 맞지 않아 `TypeError` 발생.
- **원인**: `emit_event`는 `(event_type, data)` 두 인수를 기대하나, `OrderManager`는 `(event)` 하나만 전달함.
- **해결**: 호출 시 `event.event_type`을 첫 번째 인수로 전달하도록 수정.

## 2. 필수 참조 문서 (Mandatory References)
- [x] `AI_Work_Constitution_and_Reference_Map.md` (L3 등급 산출물 준수)
- [x] `Failure_Patterns_and_Guards_ko.md` (데이터 저장 실패 시 로깅 및 방어 패턴 확인)

## 3. 변경 상세 (Implementation Detail)

### [MODIFY] [order_manager.py](file:///c:/Users/stone/projects/nocodequant/app/services/managers/order_manager.py)
- `on_fill_event` 메서드 내 `self.persistence_manager.emit_event(event)` 호출부를 리팩토링함.
- **수정안**: `self.persistence_manager.emit_event(event.event_type, event)`

## 4. 임팩트 분석 (Impacted Modules)
- `app/services/managers/order_manager.py`: 체결 이벤트 기록부 (직접 수정)
- `app/services/managers/persistence_manager.py`: 데이터 저장부 (영향 확인 완료, 수정 불필요)

## 5. 불변성 및 경계 점검 (Invariant & Integrity Check)
- **데이터 일관성**: 저장되는 이벤트의 GTID 및 타입은 `TradeEvent` 객체 내의 값과 일관되게 유지됨.
- **무결성 선언**: 본 수정은 인터페이스 정합성만을 맞추는 것이며, 기존의 리스크 관리 로직이나 주문 집행 로직을 위반하지 않음.

## 6. 롤백 계획 (Rollback Plan)
- 수정 전 코드로 원복 시 다시 `TypeError`가 발생하므로, 문법적 오류가 없는 본 수정안을 유지하는 것이 기본 정책임.

---
**AI 작업 헌장 v1.1 준수 확인 완료**
*작성일: 2026-02-26*
