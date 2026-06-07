# Design Note: 랭킹 수집 엔진 정책 중심 리팩토링 (V99)

본 문서는 **AI 작업 헌장 v1.1**에 따라 **L4 (아키텍처 및 데이터 흐름 변경)** 등급 작업의 설계 및 무결성을 증명하기 위해 작성되었습니다.

## 1. 개요 (Overview)
- **작업 등급**: L4
- **목적**: 하드코딩된 TR 수집 로직을 제거하고, 메타 테이블 기반의 정책 중심(Config-driven) 수집 엔진으로 전환하여 운영 안정성 및 확장성을 확보함.
- **주요 변경**: `KiwoomEngine` 수집 계층 아키텍처 개편 및 `StateManager` 가드 도입.

## 2. 필수 참조 문서 (Mandatory References)
- [x] `AI_Work_Constitution_and_Reference_Map.md` (거버넌스 준수)
- [x] `Failure_Patterns_and_Guards_ko.md` (이상 데이터 수신 방어 패턴 적용)
- [x] `Parameter_Invariants.md` (랭킹 파라미터 불변성 유지)
- [x] `Smart_Heat_Index_Spec.md` (Heat Index 연산 정합성 보장)

## 3. 아키텍처 변경 내역 (Architecture Diff)

| 구분 | AS-IS (Procedural) | TO-BE (Meta-driven) |
| :--- | :--- | :--- |
| **수집 관리** | `if-else` 분기를 통한 하위 메서드 호출 | `RANKING_TR_META` 테이블 기반 동적 매핑 |
| **데이터 파싱** | 필드명 하드코딩 및 직접 인덱싱 | 메타 정의(`fields`)를 통한 자동 순회 파싱 |
| **방어 로직** | 예외 발생 시 개별 `try-except` 처리 | `safe_get` 패턴을 통한 전역적 방어 수집 |
| **확장성** | TR 추가 시 코드 3~4곳 수정 필요 | 메타 테이블 1줄 추가로 신규 TR 대응 가능 |

## 4. 임팩트 분석 (Impacted Modules)
- `app/ai/engine.py`: 수집 엔진 코어 리팩토링 (중심부)
- `app/services/managers/state_manager.py`: Phase 업데이트 불변성 가드 추가
- `app/services/managers/ranking_manager.py`: 데이터 구조 변경에 따른 연산 로직 정합성 조정
- `app/ui/main_window.py`: 고도화된 랭킹 수신 로그 반영

## 5. 불변성 및 경계 점검 (Invariant & Integrity Check)
- **SSoT 준수**: 모든 시장 상태 및 랭킹 데이터 권한은 백엔드(`engine`, `state_mgr`)에 유지됨.
- **경계 준수**: UI 위젯에 대한 직접 참조가 없으며, 데이터는 시그널을 통해 전달됨.
- **무결성 선언**: 비정상적인 외부 데이터(`None`, 빈 문자열)가 유입되어도 기존 시스템 상태를 절대 훼손하지 않음.

## 6. 하위 호환성 및 마이그레이션 (Backward Compatibility)
- **Data Schema**: 기존 `RankingManager`과 UI에서 사용하는 리스트 구조(Positional list)를 유지하기 위해, 딕셔너리로 파싱된 데이터를 최종 단계에서 패치 처리하여 전달함.
- **Migration**: 별도의 DB 마이그레이션은 필요하지 않으며, 엔진 재시작 시 즉시 신규 아키텍처가 적용됨.

---
**AI 작업 헌장 v1.1 준수 확인 완료**
*작성일: 2026-02-26*
*버전: 1.0*
