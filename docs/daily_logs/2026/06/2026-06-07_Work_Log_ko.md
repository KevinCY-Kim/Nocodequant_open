# 📅 2026-06-07 업무 일지 (Daily Work Log)

## 🎯 주요 목표 및 이슈 정의
- [x] **시나리오 F (1.5억 절대 가드 + 아침 1.5x 동적 수급 가드) 공식 적용**:
  * 3거래일(2026-06-02 ~ 2026-06-04) 교차 검증 백테스팅 결과, 기존 3억 절대 가드 대비 누적 손익이 **213.33% → 344.95%**로 대폭 향상되고 거래당 평균 손익 역시 **0.35% → 0.55%**로 극대화됨을 확인하여 프로덕션 코드에 정식 이식했습니다.
- [x] **단위 테스트 작성 및 무결성 검증**:
  * `tests/test_volume_gating.py` 파일에 신규 동적 수급 가드 검증 테스트 3종(아침 미달 차단, 아침 충족 진입, 오후 미적용 패스)을 추가 작성하고 전체 테스트의 100% 합격을 완료했습니다.
- [x] **장중 상시 동적 가드(시나리오 J)와의 트레이드오프 분석 및 문서화**:
  * 상시 동적 제어 시 승률과 평균 손익은 향상되나 기회 비용(진입 횟수 80% 감소로 누적 손익 344% -> 161% 급감)이 큼을 실측하여 [dynamic_volume_guard_analysis_ko.md](file:///C:/Users/stone/projects/nocodequant/docs/design/dynamic_volume_guard_analysis_ko.md)로 영구 보존했습니다.

---

## ✅ 상세 작업 및 패치 내역

### 1. 설정 파라미터 업데이트 (`app/core/config_engine.py`)
* **변경 목적**: 절대 거래대금 완화 적용 및 아침 동적 거래량 배수 파라미터 신설.
* **패치 내역**:
  ```python
  ENTRY_MIN_VALUE_5M = 150_000_000 # [V27.9.8] 절대 거래대금 5분봉 허들 가드 (1.5억 원)
  MORNING_VOLUME_GUARD_MULT = 1.5   # [V27.9.8] 아침 9:30 이전 직전 5봉 평균 대비 거래량 배수 가드 (1.5배)
  ```

### 2. 신호 판정 엔진 이식 (`app/ai/_engine_entry.py`)
* **변경 목적**: 9시 30분 이전 장초반에 한해, 직전 5분봉의 평균 거래량 대비 1.5배 이상의 볼륨 스파이크가 발생했는지 동적 필터링 적용.
* **패치 내역**:
  ```python
  # ── [V27.9.8] 아침 09:30 이전 동적 수급 가드 ──────────────────────────
  if not abort_buy:
      volume_guard_mult = ACTIVE_CONFIG.get("MORNING_VOLUME_GUARD_MULT", 0.0)
      if volume_guard_mult > 0:
          curr_dt = curr.get('datetime')
          if curr_dt is not None:
              try:
                  import pandas as pd
                  _ts = pd.Timestamp(curr_dt)
                  if _ts.hour == 9 and 0 <= _ts.minute <= 30:
                      if len(df) >= 2:
                          # 직전 최대 5봉 평균 계산 (5봉 미만 시 존재하는 봉 수만큼)
                          start_idx = max(0, len(df) - 6)
                          end_idx = len(df) - 1
                          prev_vols = df['volume'].iloc[start_idx:end_idx]
                          avg_vol = prev_vols.mean()
                          if avg_vol > 0:
                              curr_vol = float(curr.get('volume', 0.0))
                              ratio = curr_vol / avg_vol
                              if ratio < volume_guard_mult:
                                  abort_buy = True
                                  result.reasons.append(
                                      f"매수 보류: 아침 동적 수급 부족 "
                                      f"(ratio={ratio:.2f} < limit={volume_guard_mult:.1f})"
                                  )
              except Exception:
                  pass
  ```

### 3. 단위 테스트 보완 (`tests/test_volume_gating.py`)
* **변경 목적**: 새로운 동적 가드가 회귀 오류 없이 규격대로 작동하는지 방어막 구축.
* **추가된 테스트 케이스**:
  * `test_morning_volume_guard_blocks_low_volume`: 09:30 이전, 거래량 1.2배 미달 시 `DecisionLabel.HOLD_BLOCK` 및 사유 체크.
  * `test_morning_volume_guard_allows_high_volume`: 09:30 이전, 거래량 1.6배 충족 시 정상 `DecisionLabel.BUY` 진입 검증.
  * `test_morning_volume_guard_inactive_afternoon`: 11:30 (장중) 시점에는 1.2배 미달이더라도 가드가 해제되어 정상 `DecisionLabel.BUY` 처리 검증.

---

## 📊 검증 및 패치 확인 결과

### 1. 구문 컴파일 체크
* `_engine_entry.py` 및 `config_engine.py` 두 모듈 모두 컴파일 검사 결과 이상이 없음(Pass)을 검증했습니다.

### 2. 단위 테스트 실행 결과
* `python -m unittest tests/test_volume_gating.py` 실행 결과, 총 5개 테스트가 모두 성공(OK)하여 무결성을 입증했습니다.

---

## 🔮 향후 보완 및 제안
- 이번 패치 적용으로 1.5억 ~ 3억 원대 중소형 우량 주도주 진입율이 대폭 높아졌으므로, 실시간 체결 시 슬리피지 방지용 동적 슬리피지 모델과의 호환 상태를 지속 관찰할 필요가 있습니다.
- 다음 실거래 세션 운영 전, HTS 클라이언트를 재부팅하여 `config_engine.py` 변경값을 동기화해 주시기 바랍니다.
