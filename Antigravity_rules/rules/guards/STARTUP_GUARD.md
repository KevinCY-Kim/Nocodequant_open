# Reason Code Specification: STARTUP_GUARD (데이터 수집 대기) [V27.0]

**Category:** Structural Guard
**Logic Type:** Initialization Lock

---

## 1. 개요 (Overview)
`STARTUP_GUARD`는 NoCodeQuant 엔진의 "워밍업 단계(Warm-up Phase)"입니다. 시스템이 기동되거나 타임프레임이 변경된 직후, 지표 계산을 위한 충분한 시계열 데이터가 쌓일 때까지 매매 신호를 차단합니다.

## 2. 기술적 정의 [V27.0]
*   **트리거 조건:**
    $$ N_{candles} < \text{WARMUP\_MIN\_BARS} $$
    *   `N_candles`: 현재 메모리에 적재된 확정 캔들의 개수.
    *   `WARMUP_MIN_BARS`: `ACTIVE_CONFIG.get("WARMUP_MIN_BARS", 50)`을 통해 시스템 설정값(기본 50개)을 런타임에 동적으로 주입합니다.
*   **동작**: `SignalResult.decision` 필드에 `DecisionLabel.WARMUP` ("데이터 수집 중") 메세지를 담아 반환하며, 실제 매매 로직 실행을 유보합니다.

## 3. 방어 목적 (Defense Goal)
*   **이동평균선(MA) 및 RSI 안정화**: 이들 지표는 최소 20~40개 이상의 과거 데이터가 있어야 수학적으로 의미 있는 값을 도출합니다. 데이터가 부족한 상태(`Cold Start`)에서의 오판을 방지합니다.
*   **정규화 파이프라인(Normalization) 보호**: 최근 표준편차를 계산하기 위해서는 일정량의 표본이 필수적입니다.

## 4. 운영 가이드 (Operational Guide)
*   **대기 시간**: 선택한 타임프레임 및 `WARMUP_MIN_BARS` 설정에 따라 대기 시간이 달라집니다. (기본 50개 기준)
    - 1분봉: 약 50분 대기
    - 5분봉: 약 4시간 대기
*   **참고**: 장 시작 전 미리 프로그램을 켜두거나, 과거 데이터를 충분히 조회(`GetPriceHistory`)하여 채우는 것이 좋습니다.

> **💡 핵심 요약 (Key Insight):**
> *"지표의 정확도는 데이터의 양에 비례합니다. 초기 50개의 캔들은 시스템이 시장의 리듬을 배우는 시간입니다."*

---
*Document Version: v27.0 (2026-05-25 V27.0 Sync)*
