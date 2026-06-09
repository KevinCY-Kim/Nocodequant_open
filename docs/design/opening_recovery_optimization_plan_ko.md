# 📐 NCQ 시초 감시 시스템(Opening Recovery) 고도화 설계 플랜 (실전 노이즈 대응형)

> [!IMPORTANT]
> 본 설계 플랜은 실시간 수십 개 종목의 이벤트 스트림 속에서 **"틱 노이즈에 의한 조기 탈락"**, **"재등록 폭주에 따른 리소스 낭비"**, **"최초 등록가 휩소 왜곡"** 등 장초반 극단적 변동성 노이즈를 제어하기 위한 실전 대응형 스펙 업데이트 버전입니다.

---

## 1. 정밀 조율 5대 핵심 방향

단순 이탈/탈락 메커니즘을 넘어, 실전 거래 환경에서의 틱 노이즈 및 왜곡을 제어하기 위해 아래 5가지 사양을 도입합니다.

```
                           [시초 감시 등록 대상]
                                     │ 
                        (Eviction Cooldown 180초 검증) -> 쿨다운 중이면 Skip
                                     ▼
                          [감시 등록 및 모니터링]
                                     │
                     ┌───────────────┴───────────────┐
                     ▼                               ▼
            [초경량 생존선 이탈]              [이격 한계선 초과]
                     │                               │
             (1틱 즉시 사형 방지)            (과열 고점 등록가 왜곡 방지)
             연속 5틱 이상 하회 시           시가(Day_Open) 기준 절대 앵커 적용
                     │                               │
                     ▼                               ▼
                 즉시 탈락                       즉시 탈락
```

### ① 틱 노이즈 완충 (Weak Ticks Buffer)
- **문제**: 장초반 09:05~09:15은 극단적 호가 충돌로 1틱 만에 VWAP을 순간 하회했다 복귀하는 패턴이 매우 흔합니다. 즉시 탈락시킬 시 진짜 대장주를 필터 노이즈로 잃게 됩니다.
- **해결**: 종목별 `weak_ticks_count`를 도입합니다. 가격이 지지선(시가 -2%, VWAP -0.5%)을 하회하더라도, **연속 5틱 이상** 하회 상태가 유지될 때만 최종 탈락(`Eviction`)시킵니다. 1~2틱 하회 후 복귀 시 카운터를 0으로 초기화합니다.

### ② 재등록 쿨다운 (Eviction Cooldown)
- **문제**: 급변하는 장초반에 동일 종목이 "탈락 $\rightarrow$ 신규 조건 충족 $\rightarrow$ 재등록"을 초당 반복하며 CPU 연산 낭비 및 DB 로그 폭증을 일으킬 위험이 큼니다.
- **해결**: `eviction_cooldown` 딕셔너리를 도입합니다. 한번 감시 풀에서 탈락한 종목은 탈락 시점부터 **180초(3분) 동안** 감시 풀 재등록을 완전히 차단합니다.

### ③ 등록가 왜곡 보정 (시가 기준 절대 이격 앵커링)
- **문제**: 타임 가드 차단 시점의 가격(`Registered_Price`) 자체가 당일 과열 고점(휩소 피크)일 수 있어, 이를 기준으로 이격도 한계를 설정하면 반등의 기회를 비합리적으로 차단하게 됩니다.
- **해결**: 하루 중 가장 명확한 불변의 앵커(Anchor)인 **당일 시가(`Day_Open`)**를 기준으로 이격도를 제어합니다.
  - **돌파/추세 전략**: 현재가 `Price > Day_Open * 1.03` (시가 대비 +3.0% 초과 상승 시 손익비 붕괴로 즉시 탈락)
  - **눌림목/역추세 전략**: 현재가 `Price > Day_Open * 1.015` (시가 대비 +1.5% 초과 상승 시 눌림 구간 이탈로 즉시 탈락)

### ④ 시간 가중 생존력 (Survival Priority)
- **개념**: 장초반의 거친 풍파 속에서 지지선을 깨지 않고 오랫동안 생존한 종목일수록 강력한 지지 매물대를 형성한 우량 주도주일 확률이 높습니다.
- **해결**: 감시 풀 등록 이후 경과 시간(`survival_seconds`)을 기록하여 시간 가중치를 부여합니다.
  - 생존 시간이 **60초를 초과한 종목**은 휩소 노이즈를 충분히 견딘 것으로 판정하여, 일시적 지지선 이탈 허용 틱 카운트를 기존 5틱에서 **10틱**으로 상향하여 유연성을 부여합니다.

---

## 2. 코드베이스 수정 스펙 및 구현 사양

### A. `OpeningRecoveryManager` 생성자 및 필드 확장 (`opening_recovery_manager.py`)
```python
def __init__(self, shadow_mode: bool = True, initial_pool: Optional[Dict[str, dict]] = None):
    self._pool: Dict[str, dict] = {}
    self._cooldowns: Dict[str, float] = {}  # code: expired_timestamp (eviction cooldown 180s)
    # ... 기존 속성 로드 ...
```

### B. `register()` 단계의 쿨다운 검증
```python
def register(self, code: str, day_open: float, entry_score: float, strategy_type: str, regime: str, curr_close: float, curr_high: float, curr_low: float, signal_time: str):
    # 쿨다운 검증
    curr_ts = time.time()
    if code in self._cooldowns and curr_ts < self._cooldowns[code]:
        return  # 180초 쿨다운 동안은 재등록 차단

    # ... 기존 등록 로직 ...
    # 신규 카운터 필드 추가
    candidate["weak_ticks_count"] = 0
    candidate["registered_ts"] = time.time()
```

### C. `evaluate_recovery()` 단계의 틱 노이즈 완충 및 시가 앵커 필터링
```python
def evaluate_recovery(self, code: str, price: float, current_time_str: str) -> Optional[str]:
    cand = self._pool.get(code)
    if not cand: return None
    
    # 1. 생존 시간 측정 (survival_seconds)
    survival_seconds = time.time() - cand.get("registered_ts", time.time())
    weak_ticks_limit = 10 if survival_seconds > 60 else 5
    
    # 2. 실시간 VWAP 산출
    vwap = curr_value / curr_volume if curr_volume > 0 else day_open
    
    # 3. 생존 지지선 판단 (시가 -2%, VWAP -0.5%)
    under_open = price < cand["day_open"] * 0.98
    under_vwap = price < vwap * 0.995
    
    if under_open or under_vwap:
        cand["weak_ticks_count"] += 1
        if cand["weak_ticks_count"] >= weak_ticks_limit:
            # 최종 탈락 처리 및 180초 쿨다운 적용
            self._cooldowns[code] = time.time() + 180
            self._pool.pop(code)
            self._sync_to_state_manager()
            return None
    else:
        cand["weak_ticks_count"] = 0 # 생존 영역 복귀 시 즉시 카운터 초기화
        
    # 4. 시가 기준 이격 한계 가드 (Registered Price의 과열 왜곡 극복)
    chase_limit_mult = 1.03 if cand["strategy_type"] in ("BREAKOUT", "TREND") else 1.015
    if price > cand["day_open"] * chase_limit_mult:
        self._cooldowns[code] = time.time() + 180
        self._pool.pop(code)
        self._sync_to_state_manager()
        return None
        
    # ... 기존 READY 판정 및 시초 고가 재돌파 연산 수행 ...
```
