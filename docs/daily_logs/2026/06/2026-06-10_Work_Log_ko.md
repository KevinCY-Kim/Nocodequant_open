# 📅 2026-06-10 업무 일지 (Daily Work Log)

> **[거버넌스 준수]**
> * **작업 등급**: L3 (핵심 실행부 및 UI 위젯 결함 수정)
> * **참조 문서**: 
>   * [AI_Work_Constitution_and_Reference_Map.md](file:///c:/Users/stone/projects/nocodequant/Antigravity_rules/governance/AI_Work_Constitution_and_Reference_Map.md)
>   * [config_engine.py](file:///c:/Users/stone/projects/nocodequant/app/core/config_engine.py)
>   * [_engine_entry.py](file:///c:/Users/stone/projects/nocodequant/app/ai/_engine_entry.py)
> * **영향 받는 모듈 (Impacted Modules)**: `_engine_entry.py`, `persistence_manager.py`, `state_manager.py`, `main_window.py`, `signal_status_widget.py`
> * **불변성 위반 여부 (Invariant Check)**: 위반 없음 (Violated: None)
> * **경계 준수 선언**: 엔진의 핵심 결정 로직 외에 타 모듈의 리팩토링이나 코드 정리를 수행하지 않았으며, 변경은 선언된 모듈 영역 내로 격리되었습니다.

---

## 🎯 주요 목표 및 이슈 정의

1. **SIGNAL_BLOCKED 발생 시 UI AttributeError 해결**:
   * 차단 신호(`SIGNAL_BLOCKED`) 발생 시 UI 위젯(`signal_status_widget.py`)의 `update_status()`에서 `'dict' object has no attribute 'snapshot'` 오류와 함께 크래시가 일어나는 현상 해결.
2. **Regime-Adaptive 이격 가드(TREND_MA20_GAP_CAP) 오작동 교정**:
   * `config_engine.py`에 기본값(`0.01`)이 하드코딩 정의되어 있어 `ACTIVE_CONFIG`에 항상 세팅되는 현상 때문에, `UPTREND` 국면 시 이격을 3.0%로 완화해 주는 분기가 실행되지 않던 데드코드 문제 해결.
3. **오늘(2026-06-10) 실거래 무매매 포렌식 분석**:
   * 수정 패치 및 앱 재시작 이후에도 거래가 단 1회도 발생하지 않은 구체적인 사유 규명 및 주도주 휩소 회피 데이터 검증.

---

## ✅ 상세 작업 및 패치 내역

### 1. 차단 신호 UI 크래시 디펜스 및 구조화된 스냅샷 주입
* **원인 분석**:
  * 전략 엔진이 조건 미달 신호를 차단할 때 `SIGNAL_BLOCKED` 이벤트를 딕셔너리(`dict`) 포맷으로 방출했으나, UI 수신 슬롯에서는 `SignalResult` 클래스 객체를 전제로 `.snapshot` 및 속성들에 직접 접근을 시도하여 `AttributeError` 발생.
* **패치 내역**:
  * [_engine_entry.py](file:///c:/Users/stone/projects/nocodequant/app/ai/_engine_entry.py#L484-L525): `SIGNAL_BLOCKED` 페이로드에 구조화된 `blocked_snapshot` 정보를 세트로 구성하여 딕셔너리 내에 `"snapshot"` 필드를 명시적으로 바인딩 주입.
  * [main_window.py](file:///c:/Users/stone/projects/nocodequant/app/ui/main_window.py#L2217-L2229): 메인 스레드 안전 UI 슬롯(`_on_marshalled_signal_received`)에서 수신 결과가 `dict` 유형일 경우 `update_status` 호출을 사전에 차단하고 방어 코드 구축.
  * [signal_status_widget.py](file:///c:/Users/stone/projects/nocodequant/app/ui/widgets/signal_status_widget.py#L560-L566): `update_status` 진입 시 `result`가 `dict`일 경우 오류 없이 즉시 리턴(`Early Return`)하도록 조치.

### 2. Regime-Adaptive 이격 가드 정상화 (우선순위 교정)
* **원인 분석**:
  * 기존 조건문 `if "TREND_MA20_GAP_CAP" in ACTIVE_CONFIG:`는 `config_engine.py`에서 항상 기본값을 로드하므로 무조건 `True` 분기를 탔고, 이로 인해 `UPTREND` 국면 전환 시에도 3% 완화가 적용되지 못하고 1% 가드에 가로막히는 병목 현상이 발생했음.
* **패치 내역**:
  * [_engine_entry.py](file:///c:/Users/stone/projects/nocodequant/app/ai/_engine_entry.py#L293-L303): `current_regime == 'UPTREND'` 조건을 최우선으로 분기하여 **UPTREND일 때는 3.0% 가드**를 즉시 할당하고, 그 외의 추세(`TREND`) 국면에서만 `ACTIVE_CONFIG` 설정 값(기본 1.0%)을 할당하도록 우선순위 교정 완료.

### 3. 세금 및 거래비용 정밀 정규화
* **패치 내역**:
  * [state_manager.py](file:///c:/Users/stone/projects/nocodequant/app/services/managers/state_manager.py#L508): 증권거래세 세율을 실제 대한민국 2025~2026 표준인 **0.23%** (세금 0.23% + 수수료 0.015% 편도)로 동기화.
  * [persistence_manager.py](file:///c:/Users/stone/projects/nocodequant/app/services/managers/persistence_manager.py#L332): P&L 산출식의 추정 비용 기준을 왕복 수수료 및 거래세를 합산한 **0.245%**로 교정하여 실거래 수익 정합성 확보.

### 4. HTS 잔고 동기화 시 매입단가 SSoT 강제
* **패치 내역**:
  * [state_manager.py](file:///c:/Users/stone/projects/nocodequant/app/services/managers/state_manager.py#L955): 앱 재기동 등으로 HTS와 잔고 동기화(`sync_positions`) 시, 로컬 캐시 가격이 아닌 HTS 서버의 실제 매입 평단가(`avg_price`)를 최우선 단일 진실 공급원(SSoT)으로 지정하여 가격 왜곡을 차단.
  * [main_window.py](file:///c:/Users/stone/projects/nocodequant/app/ui/main_window.py#L1692): 실시간 실현손익을 HTS 서버로부터 정기 호출하는 요청 주기 파이프라인 추가.

---

## 🕒 실시간 주도주 차단 원인 분석 (2026-06-10)

* **배경 상황**: 
  * 오늘 시장은 **급하강 장세(급락장)**를 보였습니다.
  * `Trend Only Mode` 적용으로 인해 오직 `TREND` 전략만 단독 가동되었으며, `BREAKOUT` / `VOLUME` 등은 임계값 100 셋팅으로 비활성화되었습니다.
* **차단 로그 Forensics (`shadow_guard_log.jsonl` 분석)**:
  * **수정 전 (오전)**: 1% 이격 장벽에 막힘.
  * **수정 및 재시작 후 (오후)**: 완화된 3.0% 이격 제한이 정상 적용되었으나, 장중 포착된 종목들이 시장과 반대로 장초반에 거래량을 동반하여 비정상적으로 가파르게 폭발하며 5분봉 이평선을 아득히 초과함.
    * `10:50` | 테크윈(`001740`) : 이격률 **5.94%** (3.0% 초과) ➔ ❌ 차단
    * `10:50` | 후성(`093370`) : 이격률 **7.87%** (3.0% 초과) ➔ ❌ 차단
    * `10:55` | 랩지노믹스(`084650`) : 이격률 **4.70%** (3.0% 초과) ➔ ❌ 차단
    * `12:35` | SK오션플랜트(`100090`) : 이격률 **5.47%** (3.0% 초과) ➔ ❌ 차단
    * `12:45` | 이화산업(`006220`) : 이격률 **6.43%** (3.0% 초과) ➔ ❌ 차단
    * `14:15` | 이화산업(`006220`) : 이격률 **3.41%** (3.0% 초과) ➔ ❌ 차단
* **분석 결과**:
  * 오늘 포착된 주도주들은 장초반에 **시초가 휩소(Bull Trap)성 급등**을 보였고, 이후 시장 하강세에 동조하여 **대부분 고점 대비 -6% ~ -9% 이상 급락**하였습니다 (예: SK오션플랜트 18,570 ➔ 16,890원).
  * 주가가 다시 이평선에 도달하여 이격을 줄여 나갔을 때는 하락 모멘텀에 의해 진입 점수(`entry_score`)가 가드 기준선인 60점 미만으로 동시에 깎였습니다.
  * 따라서, **이격 제한 가드가 휩소 상투 잡기를 정확하게 차단하여 자산을 방어한 이상적인 흐름**으로 귀결되었습니다.
  * 15:00 이후에는 정책 엔진의 `TimeGate` 보호막이 작동하여 안전하게 장을 마감했습니다.

---

## 🔮 향후 계획
- **Regime-Adaptive 이격 완화 실거래 검증 지속**:
  * `UPTREND` 3.0% 및 일반 장세 1.0% 가드가 시장 국면 변화에 잘 적응하는지 오더 매니저 로깅을 통해 실시간 검증합니다.
  * HTS 단가 동기화 기능이 추가되었으므로 평단 왜곡에 의한 트레일링 스톱 오작동이 없는지 모니터링합니다.

---

## 📊 감사 1 (False Negative 분석) & V28.0 청산/진입 개선

> **[거버넌스 준수]**
> * **작업 등급**: L4 (핵심 전략 파일 직접 수정) → `PROTECTED:L4_STRATEGY_CORE`
> * **참조 문서**: [`docs/analysis/2026-06-10_Audit1_FalseNegative_and_V28_ko.md`](file:///c:/Users/stone/projects/nocodequant/docs/analysis/2026-06-10_Audit1_FalseNegative_and_V28_ko.md)
> * **영향 받는 모듈**: `config_engine.py`, `_engine_entry.py`, `_engine_exit.py`
> * **불변성 위반 여부**: 위반 없음

### 1. 감사 1: False Negative 분석 결론

**의뢰**: "진입 필터가 수익을 놓치는 False Negative를 찾고 기대수익 감소량을 추정하라."

**데이터**: `forensics.db` 38.3만 건 신호 + `trades.db` SELL 체결 1,036건 (2026-05-01~06-10, 47일)

**결론: FN 가설 기각 — 진입 필터는 합리적**

차단된 신호 전수를 가상 진입 시뮬레이션(SL -3%, 트레일링 2%, 비용 0.5% 차감)한 결과, 전략/국면/거래량비/MA20이격/대금/신뢰도/점수/시간대 8개 축 **모든 세그먼트에서 기대수익이 음수**였다.

| FN 유형 | 가상 거래 | 평균 순수익 | 승률 |
|---|---|---|---|
| 거래대금 5천만원 가드 | 202 | -2.03% | 13% |
| 전략 임계값 100 차단 | 1,485 | -0.85% ~ -1.45% | 22~29% |
| HOLD_HEAT 게이트 차단 | 715 | -0.67% ~ -1.35% | 20~33% |

필터 완화 시 회피 손실 추정액: **~50M–86M KRW** (6주 기준). 비활성 전략(th=100) 유지 결정.

**진짜 수익 누수는 청산 단계에 있다:**
- 체결 거래 평균 **-0.38%/건**, 승률 27%
- **누수 승자(Leaked Winner)**: MFE +1.0% 도달 345건 중 **134건(39%)이 손실 마감**, 평균 -2.22%
- STOP_LOSS 107건: 손절 전 평균 고점 **+2.14%** 찍고 반납
- 0~3봉(≤15분) 단타 608건(59%): 평균 -0.52% — 봉 초반 노이즈 진입이 주범
- MANUAL 청산 +0.79% vs NORMAL 청산 -0.42%: 사람이 자동 엔진을 능가

---

### 2. V28.0 적용 개선 3건 (모의투자 즉시 적용)

#### 2-1. L0 조기 수익 잠금 래칫
- **파일**: `config_engine.py` (`RATCHET_L0_PEAK_TRIGGER = 0.010`, `RATCHET_L0_SL = 0.003`), `_engine_exit.py`
- **내용**: 고점(peak) +1.0% 터치 시 SL 플로어를 +0.3%로 상향. 기존 L1~L5 래칫(종가 기준)과 달리 **유일하게 peak 기준** 발동. 반납폭 0.7%p로 의도적으로 넓게 유지(러너 보존).
- **근거**: 누수 승자 134건(-2.22%). 백테스트상 평균손익 -0.372% → -0.053%.

#### 2-2. 봉 성숙도 진입 게이트
- **파일**: `config_engine.py` (`ENTRY_BAR_MATURITY_SEC = 240`), `_engine_entry.py`
- **내용**: 현재 진행 중인 5분봉이 240초(4분) 경과 후에만 진입 확정. `elapsed = now - bar_start ∈ [0, 300)` 구간만 차단. 과거봉(백테스트)은 elapsed ≥ 300s → 자동 통과.
- **근거**: 0~3봉 608건(59%, -0.52%). `ENTRY_MIN_VALUE_5M` 봉 초반 대금 미적립 오차단도 동반 해소.

#### 2-3. 시간대 보수 가중
- **파일**: `config_engine.py` (`ENTRY_TH_HOUR_ADD = {10: 5, 12: 5}`), `_engine_entry.py`
- **내용**: 10시·12시 진입 허들 +5pt 소프트 가중(차단이 아닌 임계값 상향).
- **근거**: 10시 -0.77%(n=194), 12시 -0.74%(n=113).

---

### 3. 정리 & 커밋

| 작업 | 내용 |
|---|---|
| 임시 분석 파일 삭제 | `data/_fn_scan*.py`, `data/_fn_*.json` 5건 |
| 스크립트 정규 이동 | `scripts/audit1_false_negative_threshold.py`, `scripts/audit1_false_negative_segments.py` |
| 거버넌스 문서 생성 | `docs/analysis/2026-06-10_Audit1_FalseNegative_and_V28_ko.md` |
| 커밋 & 병합 | `feat/v28-exit-entry-improvements` → `main` fast-forward, 커밋 `3f22d0a`, `e57c7d0` |
| GitHub 푸시 | `ece4860..e57c7d0`, `origin/main` 완전 동기화 |

---

### 4. V28.0 효과 모니터링 자동화

- `scripts/monitor_v28_effect.py`: PRE(베이스라인)/POST(V28.0 후) KPI 비교 스크립트
- `scripts/run_v28_monitor.ps1`: conda `trade_exe_v2` 래퍼, `logs/v28_monitor/<날짜>.log` 기록
- `scripts/register_v28_monitor_task.ps1`: Task Scheduler 재등록 스크립트
- **Windows 작업 스케줄러 `NCQ_V28_Monitor`**: 평일 15:40 자동 실행, Status Ready

PRE 베이스라인 확정 (n=1,036, 47일): ① 누수 승자 134건(39%) ② 0~3봉 비중 59% ③ 평균 손익 -0.38%.

---

## 🔮 향후 계획 (갱신)

- **V28.0 효과 관찰**: 매일 15:40 자동 로그 확인. 목표: ①누수 승자↓ ②0~3봉 비중↓ ③평균 손익 → 0.
- **entry_score 지도학습 재보정 (보류)**: V28.0 POST 데이터 약 1주일 누적 후 착수. score≥90 구간이 +0.11%에 불과해 점수가 수익을 예측하지 못하는 문제. forensics.db 38.3만 건으로 core/volume/price/risk 가중치 재학습 예정.
- **L0 파라미터 점검**: 대형 승자(>+3%) 비중이 눈에 띄게 감소하면 `RATCHET_L0_PEAK_TRIGGER`를 0.012~0.015로 상향 조정.
