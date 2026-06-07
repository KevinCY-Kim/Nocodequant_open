# 🛡️ [Final Master Plan] 국면(Regime) 판정 체계 통합 및 CHAOS 실전 가동

> **작성일**: 2026-04-07  
> **상태**: 최종 승인 대기 (Final Review)  
> **주제**: ADX 히스테리시스(25/22) 안정화를 기반으로, 엔진 내부의 2분법적 국면 판정을 폐기하고 설계 도안과 일치하는 4대 국면(`UPTREND / DOWNTREND / RANGE / CHAOS`) 체계로 파이프라인을 전면 개편함.

---

## 1. 아키텍처 개편의 핵심 원칙

1.  **CHAOS 국면의 부활**: ADX 22 미만(`RANGE`) 구간 내에서 `VOL_RATIO_WARN(1.8)`을 초과하는 이상 변동성을 `CHAOS`로 격리하여 72점 임계값과 10봉 시간 청산 로직을 가동함.
2.  **데이터 흐름의 전진 배치**: 함수 최하단에서 UI용으로만 수행되던 국면 변환 로직을 `_update_regime_state`로 이동하여 파이프라인 최상단에서 국면을 확정함.
3.  **단일 진실 공급원(SSoT)**: 모든 후속 로직(임계값, 청산, 지표 계산)이 전방에서 확정된 `current_regime` 하나만을 바라보게 함.

---

## 2. 상세 단계별 구현 명세 (Execution Steps)

### [Step 1] 하방 리스크 차단 설정 (`config_engine.py`)
하락 추세(`DOWNTREND`)에서의 '낙하하는 칼날 잡기(Falling Knife)'를 방지하기 위해 보수적 임계값을 추가합니다.

```python
# app/core/config_engine.py
ENTRY_THRESHOLD_DOWNTREND = 70  # 하락장: 극도로 보수적 진입 필터
```

---

### [Step 2] 엔진 핵심 로직 개편 (`strategy.py`)

#### 2.1 `_update_regime_state` 시그니처 및 분기 확장
- **입력 확장**: `vol_ratio`를 국면 판정의 변수로 추가.
- **로직**: `RANGE` 구간 내에서 `VOL_RATIO_WARN(1.8)` 시 CHAOS 확정.

```python
def _update_regime_state(self, code: str, adx: float, adx_slope: float = 0.0, vol_ratio: float = 1.0) -> str:
    # ... 기존 ADX 25/22 히스테리시스 로직 ...
    
    # [신규] CHAOS 격리: RANGE 구간 내 변동성 스파이크
    if new_regime == 'RANGE':
        vol_warn = ACTIVE_CONFIG.get("VOL_RATIO_WARN", 1.8) # CAP(2.2)보다 낮은 기준 사용
        if vol_ratio > vol_warn:
            new_regime = 'CHAOS'
            
    return new_regime
```

#### 2.2 호출부(`_get_signal_internal`) 데이터 주입 수정
```python
# L546 부근
vol_ratio_val = self._safe_float(curr.get('vol_ratio', 1.0))
current_regime = self._update_regime_state(code, adx, adx_slope, vol_ratio_val)
```

---

### [Step 3] 의존성 범위 확장 및 포지션 방어 (`strategy.py`)

#### 3.1 `_get_bb_raw` 역행성 의존성 해결 (L419)
4대 국면 문자열을 처리할 수 있도록 비교 범위를 확장합니다.

```python
if regime in ("TREND", "UPTREND", "DOWNTREND"):
    if is_downtrend or regime == "DOWNTREND":
        # BB_DOWNTREND_REVERSAL_SCORE(85.0) 적용 구간
        ...
```

#### 3.2 포지션 방어 로직 강화 (`_adapt_existing_positions`)
`DOWNTREND` 전환 시 즉극적으로 수익 보존 및 손절선을 방어합니다.

```python
if new_regime in ("TREND", "UPTREND"):
    # (기존) 추세 확인 시 목표가 상향 및 SL 이동
elif new_regime == "DOWNTREND":
    # (신규) 하락 추세 전환: 즉시 방어 모드
    pos['tp_trend_target'] = None
    pos['trailing_multiplier'] = 1.5
    pos['sl'] = max(current_sl, entry_price)
    pos['adaptation_log'] = "⚠️ 하락추세 전환 -> 즉시 방어 모드"
else: # RANGE, CHAOS
    # 방어적 출구 전략
```

---

### [Step 4] 파라미터 맵핑 통합 및 후행 로직 폐기

#### 4.1 임계값 맵핑 통합 (L731)
```python
_regime_th_map = {
    "UPTREND":   ACTIVE_CONFIG.get("ENTRY_THRESHOLD_TREND", 58),
    "DOWNTREND": ACTIVE_CONFIG.get("ENTRY_THRESHOLD_DOWNTREND", 70), # 신규
    "TREND":     ACTIVE_CONFIG.get("ENTRY_THRESHOLD_TREND", 58), 
    "RANGE":     ACTIVE_CONFIG.get("ENTRY_THRESHOLD_RANGE", 65),
    "CHAOS":     ACTIVE_CONFIG.get("ENTRY_THRESHOLD_CHAOS", 72), # 활성화
}
```

#### 4.2 `exported_regime` 폐기 (L784~786)
더 이상 함수 최하단에서 변환할 필요가 없으므로 `current_regime`을 `SignalResult`에 직접 주입합니다.

---

## 3. 리스크 및 정책 결정 사항

1.  **DOWNTREND 이중 패널티**: `ENTRY_THRESHOLD_DOWNTREND(70)`와 `_calc_entry_score` 내부의 `-20점 패널티`가 **이중으로 적용**됩니다. 이는 폭락장에서의 매수를 원천 차단하겠다는 설계 의도를 가장 안전하게 구현하는 방식입니다 (옵션 A 채택).
2.  **순환 의존성**: `_get_bb_raw`가 세션에 저장된 국면명을 읽으므로, 3단계를 완료하기 전까지는 2단계의 CHAOS 판정이 지표 계산에 에러를 유발하지 않도록 주의해야 합니다.

---
*Created By Antigravity (Powered by Advanced Agentic Coding Framework)*
