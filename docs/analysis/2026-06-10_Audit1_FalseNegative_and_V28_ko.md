# 감사 1: False Negative 분석 & V28.0 청산/진입 개선

- 작성일: 2026-06-10
- 대상: `app/ai/strategy.py`, `app/ai/_engine_entry.py`, `app/ai/_engine_exit.py`
- 데이터: `data/forensics.db` (신호 38.3만 건, 2026-05-01~06-10), `data/trades.db` (SELL 체결 1,036건)
- 재현 스크립트: `scripts/audit1_false_negative_threshold.py`, `scripts/audit1_false_negative_segments.py`, `scripts/monitor_v28_effect.py`

## 1. 의뢰: "필터 때문에 놓친 수익(False Negative)을 찾아라"

진입 필터에 막혀 수익 기회를 놓친 사례를 찾고 기대수익 감소량을 추정하는 것이 목표였다.

## 2. 결론: FN 가설 기각 — 진입 필터는 합리적

차단된 신호 전수를 가상 진입 시뮬레이션(SL -3%, 트레일링 2%, 거래비용 0.5% 차감)한 결과,
**모든 세그먼트에서 기대수익이 음수**였다. 필터로 인한 기대수익 감소량은 ≈ 0이며,
오히려 거래당 -0.7% ~ -2.0%의 손실을 회피하고 있었다.

| FN 유형 | 가상 거래 | 평균 순수익 | 승률 |
|---|---|---|---|
| 거래대금 5천만원 가드 차단 | 202 | -2.03% | 13% |
| 전략 임계값 100 차단(BREAKOUT/VOLUME/REVERSAL/MIXED) | 1,485 | -0.85% ~ -1.45% | 22~29% |
| 과열 게이트(HOLD_HEAT) 차단 | 715 | -0.67% ~ -1.35% | 20~33% |

전략/국면/거래량비/MA20이격/대금/신뢰도/점수/시간대 8개 축으로 분해해도 양(+)의
기대수익 세그먼트는 존재하지 않았다. → **진입 필터 완화는 근거 없음. 비활성 전략(th=100) 유지.**

## 3. 진짜 수익 누수는 청산 단계에 있다 (체결 1,036건 분석)

- 체결된 거래조차 **평균 -0.38%/건, 중앙값 -0.40%, 승률 27%**.
- **누수 승자(Leaked Winner): MFE +1.0% 이상 도달한 345건 중 134건(39%)이 손실 마감, 평균 -2.22%.**
- STOP_LOSS 107건은 손절 전 **평균 고점 +2.14%**까지 갔다가 반납.
- 0~3봉(≤15분) 단타 608건(59%)이 평균 -0.52% — 봉 초반 노이즈 진입이 휩소 손실 주범.
- 자동청산(NORMAL) -0.42% vs 수동청산(MANUAL) +0.79% — 사람이 엔진을 이김.

## 4. 적용한 개선 (V28.0)

> 모의투자 단계이므로 섀도우 모드 없이 즉시 적용.

| # | 변경 | 파일 | 근거 KPI |
|---|---|---|---|
| 1 | **L0 조기 수익 잠금 래칫**: 고점 +1.0% 터치 시 SL 플로어 +0.3%. 기존 래칫(종가 기준)과 달리 유일하게 peak 기준 발동. | `config_engine.py` `RATCHET_L0_*`, `_engine_exit.py` | 누수 승자 134건(-2.22%). 실거래 백테스트상 평균손익 -0.372% → -0.053% |
| 2 | **봉 성숙도 진입 게이트**: 진행 중 5분봉이 240초 경과한 뒤에만 진입 확정. 과거봉(백테스트)은 자동 통과. | `config_engine.py` `ENTRY_BAR_MATURITY_SEC=240`, `_engine_entry.py` | 0~3봉 608건(59%, -0.52%). 대금 가드 봉초반 오차단도 동반 해소 |
| 3 | **시간대 보수 가중**: 10시·12시 진입 허들 +5pt(차단 아닌 소프트). | `config_engine.py` `ENTRY_TH_HOUR_ADD`, `_engine_entry.py` | 10시 -0.77%(n=194), 12시 -0.74%(n=113) |

반납폭(L0: 0.7%p)은 러너 보존을 위해 의도적으로 넓게 유지. 기능 테스트는
`trade_exe_v2` 환경에서 실제 엔진 경로로 통과 확인(L0 발동/미발동, 게이트 차단/통과, 시간대 가중).

## 5. 모니터링 (B)

`python scripts/monitor_v28_effect.py` — PRE(베이스라인)/POST(적용 후) KPI 비교.
기대 방향: ①누수 승자 비중↓ ②0~3봉 비중↓ ③거래당 손익 0 근접 ④STOP_LOSS 비중↓.

**일일 자동 실행 (Windows 작업 스케줄러):**
평일 15:40(KRX 15:30 마감 후) `NCQ_V28_Monitor` 작업이 `scripts/run_v28_monitor.ps1`을
호출 → conda `trade_exe_v2` python으로 모니터 실행 → `logs/v28_monitor/<날짜>.log` 기록.

- 재등록: `powershell -ExecutionPolicy Bypass -File scripts/register_v28_monitor_task.ps1`
- 확인:   `schtasks /Query /TN NCQ_V28_Monitor /FO LIST`
- 즉시실행: `schtasks /Run /TN NCQ_V28_Monitor`
- 해제:   `schtasks /Delete /TN NCQ_V28_Monitor /F`

> 로컬 trades.db를 로컬 conda 환경으로 실행해야 하므로 클라우드 루틴이 아닌
> Windows 작업 스케줄러를 사용. 스케줄 등록은 머신 로컬 상태(repo 미포함).

## 6. 다음 단계 (보류)

**entry_score 지도학습 재보정** — score≥90 구간만 +0.11%로 점수가 수익을 예측하지 못함.
forensics.db를 학습 데이터로 core/volume/price/risk 가중치 재학습. 단, V28.0 효과
데이터가 모의투자에서 누적된 뒤 착수(청산 변경 효과와 점수 효과의 교란 방지).
