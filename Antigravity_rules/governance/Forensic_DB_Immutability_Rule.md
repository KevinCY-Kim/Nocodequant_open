# Forensic & DB Immutability Rule [V17.0]

## 0. Purpose

본 문서는 **NoCodeQuant 플랫폼의 신뢰성, 재현성, 감사 대응 능력**을 장기적으로 보장하기 위한 최상위 불변 규칙을 정의한다. 본 규칙은 기술 구현보다 상위 개념이며, 모든 개발·운영·AI 작업의 판단 기준으로 작동한다.

---

## 1. Core Principle (절대 원칙)

### 1.1 Historical Data Immutability (과거 데이터 불가침)

* **저장된 과거 거래(Trade) 및 신호(Signal) 데이터는 절대 수정·보정·덮어쓰기 하지 않는다.**
* 버그, 누락, 로직 오류가 발견되더라도 **기존 데이터는 그대로 보존**한다.
* 모든 수정은 **Forward-Only 방식**으로만 허용된다.

> 과거를 고치는 시스템은 신뢰를 잃고,
> 신뢰를 잃은 시스템은 반드시 붕괴된다.

---

## 2. Allowed Correction Mechanisms (허용되는 보정 방식)

과거 데이터에 문제가 발견되었을 경우, 아래 방식만 허용된다:

### 2.1 Versioned Metrics (버전화된 지표)
* 새로운 계산 로직은 **v2, v3 지표**로 추가한다.
* 기존 지표(v1)는 유지하며 UI에서 선택적으로 비교 가능하게 한다.

### 2.2 Annotation / Incident Records
* 과거 데이터 오류는 **건너뛰거나 일지(Work Log)에 기록**하여 명시한다.

---

## 3. Forbidden Actions (금지 사항)

다음 행위는 **어떠한 상황에서도 금지**된다:
* ❌ 과거 로그 UPDATE / DELETE
* ❌ 통계 수치 맞추기 위한 데이터 재작성
* ❌ 테스트 결과를 실데이터에 반영

> “수치가 보기 싫다”는 이유는
> **데이터 수정의 정당한 사유가 아니다.**

---

## 4. Forensic Layer Policy [Updated V17.0]

### 4.1 AI 판단 책임 추적 (Decision Trace)
* **Signal Result 스냅샷**: 의사결정 시점의 MA, RSI, BB 지표 값과 산출된 점수, 신뢰도를 모두 기록한다.
* **Metadata**: 당시 적용된 `runtime_config`의 스냅샷을 보존하여 재현 가능성을 확보한다.

### 4.2 Engine-Driven EventBus Pipeline (V17.0 SSoT)
* **발행 권한**: `ExecutionManager`만이 포렌식 이벤트의 유일한 발신자. UI에서의 `event_bus.publish()` 호출은 금지.
* **자동 구독**: `PersistenceManager.subscribe_to_bus(event_bus)` → `PERSISTENCE_WHITELIST` 내 이벤트 자동 구독.
* **영속화**: EventBus → `PersistenceManager._handle_bus_event()` → SQLite INSERT → `on_updated_callback`.
* **시각화**: `forensic_bridge` → `ForensicDashboard` (개별 로그 행 표시, 집계 배제).

---

## 5. UI / Dashboard Separation Rule [Updated V17.0]

* **Dashboard**: 실시간 시장 데이터와 AI의 현재 인식을 시각화한다.
* **Log Panel**: 시스템의 물리적 동작(API 호출, 체결 등)을 기록한다.
* **ForensicDashboard**: EventBus → PersistenceManager 경로로 수신된 감사 로그를 실시간 표시한다 (`SIGNAL_GENERATED`, `ORDER_FILLED`, `ERROR` 등).
* 모든 시각적 데이터는 백엔드(`StrategyManager`)의 계산 결과에 충실해야 한다.

---

## 6. AI Tool Usage Rule

AI 도구를 통한 작업 시 반드시 다음을 준수한다:
1. 본 문서를 작업 시작 전에 참조한다.
2. 데이터 수정이 필요한 작업은 즉시 중단하고 보정 로직을 설계한다.
3. 임시 땜질 로직은 허용되지 않는다.

---

## 7. Final Doctrine

> **데이터는 증거다.**
> 증거를 고치는 시스템은
> 결국 자기 자신을 부정하게 된다.

본 규칙은 모든 기술적 판단보다 우선한다.

---

## 8. Data Repair 예외 절차 (Exceptional Forensic Repair) [v1.2 Appended]

> 출처: ArmoraAI Historical_Data_Immutability_Policy → NCQ 도메인 적응 병합

"데이터 보정(Data Repair)"은 **예외적 포렌식 작업**으로 분류되며, 기본 수정 방식이 아니다.

아래 **4가지 조건을 모두 충족**한 경우에만 허용된다:

1. **명시적 승인 기록**: 보정 사유와 승인자가 기록으로 남아야 한다.
2. **원본 데이터 보존**: 보정 전 원시 데이터를 별도로 보존해야 한다.
3. **별도 테이블/뷰 사용**: 보정 데이터는 원본 테이블을 덮어쓰지 않고 **새 테이블 또는 뷰**에 기록한다.
4. **대시보드 표식**: UI에 "보정된 데이터(Reconstructed Data)" 표식을 명확히 표시한다.

위 조건 미충족 시, 데이터 보정은 **금지**된다.

---

## 9. AI 필수 사전 선언 의무 (Mandatory AI Declaration) [v1.2 Appended]

AI 또는 개발자가 과거 데이터 수정을 제안하기 전, 반드시 다음을 선언해야 한다:

> "본 작업은 영속화된 과거 데이터를 수정하므로, Forensic & DB Immutability Rule을 위반합니다."

위 선언이 참(True)인 경우, 해당 제안은 **즉시 거부**되어야 한다.
단, Section 8의 예외 절차를 모두 충족하는 경우에 한해 승인 검토가 가능하다.

---

## 10. DB 운영 불변 표준 (DB Operational Invariants) [v1.2 Absorbed]

> 출처: KevinCY-Kodex rules.md §11 Log Integrity & Audit Trail → NCQ 흡수

### 10.1 Atomic Schema Evolution (원자적 스키마 변경)
DB 스키마 변경 시 반드시 다음 패턴을 사용한다:
1. `BEGIN IMMEDIATE`
2. `CREATE TABLE new_table ...`
3. `INSERT INTO new_table SELECT ... FROM old_table`
4. `DROP TABLE old_table`
5. `ALTER TABLE new_table RENAME TO old_table`

트랜잭션 무결성을 보장하며, 중간 실패 시 원본 테이블이 보존되어야 한다.

### 10.2 Silent Failure Defense (침묵 실패 방어)
모든 `INSERT`/`UPDATE` 작업 후에는 반드시 `rowcount` 또는 `changes()`를 체크하여 **침묵하는 쓰기 실패(Silent Write Failure)**를 감지해야 한다.

### 10.3 Space ISO Format Sync (타임스탬프 통일)
모든 타임스탬프는 `'T'` 구분자가 없는 **`YYYY-MM-DD HH:MM:SS.mmm`** (Space ISO) 포맷으로 통일하여 문자열 정렬 무결성을 유지한다.
