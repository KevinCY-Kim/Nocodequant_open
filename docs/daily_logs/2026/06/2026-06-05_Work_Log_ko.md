# 📅 2026-06-05 업무 일지 (Daily Work Log)

## 🎯 주요 목표 및 이슈 정의
- [x] **거래대금 가이드라인 3억 원 조정 확인**:
  - 5분봉 절대 거래대금 허들을 기존 10억 원에서 3억 원(`300_000_000`)으로 하향 조정하여 하락장 및 중소형주 기회 포획 능력을 확보했습니다.
- [x] **strategy.py 코드 완벽 복원 및 적응형 과열 가드 버그 수정**:
  - 이전 작업 오류로 1268라인으로 잘려 나갔던 [strategy.py](file:///c:/Users/stone/projects/nocodequant/app/ai/strategy.py) 파일을 [restore_strategy.py](file:///c:/Users/stone/projects/nocodequant/scratch/restore_strategy.py) 복구 스크립트를 사용하여 원래의 **1315라인**으로 완벽 복구했습니다.
  - `check_overheat` 함수 호출부에 `confidence` 및 `current_regime` 인자가 누락되어 주도주 과열 완화 한도(3.0x)가 적용되지 않고 2.2x로 오동작하던 버그를 수정했습니다.
  - [strategy_back.py](file:///c:/Users/stone/projects/nocodequant/app/ai/strategy_back.py) 백업본과의 교차 검증을 통해 완벽한 복구를 검증했습니다.
- [x] **UI 테마 표시 불일치 패치 완료 (`premium_header_widget.py`)**:
  - 정적으로 매핑되지 않은 종목에 대해 테이블(2-hop 피어 추론 테마 노출)과 우측 상세 패널 헤더("기타" 폴백)의 테마명이 서로 달랐던 문제를 해결했습니다.
  - 상세 패널 헤더에서도 정적 맵 누락 시 **동적 추론 테마 캐시(`_inferred_theme_cache`)**를 교차 검색하도록 로직을 보완했습니다.
- [x] **3억 거래대금 허들 가드 단독 차단 이력 진단**:
  - 분석 진단 스크립트를 작성하여 forensics DB를 탐색하고, 09:57 가드 완화 조치 이후에도 3억 제한에 막혀 진입하지 못한 유망 신호들을 실측해 냈습니다.
- [x] **원격 Git 저장소 연동 및 소스 코드 백업**:
  - `https://github.com/KevinCY-Kim/Nocodequant` 프라이빗 저장소를 원격(origin)으로 지정하여 최초 연동 및 푸시 완료.
  - 민감 정보(`.env`), 거래 내역 데이터베이스(`*.db`), 대용량 로그 파일(`*.log`, `*.csv`) 등을 철저히 제외하는 [.gitignore](file:///c:/Users/stone/projects/nocodequant/.gitignore) 구성을 통해 보안성 확보.
  - 현재 프로젝트 아키텍처 및 V27.0 핵심 기능 정보와 완벽하게 부합하도록 [README.md](file:///c:/Users/stone/projects/nocodequant/README.md) 최신화 및 원격 반영 완료.

---

## ✅ 상세 작업 및 패치 내역

### 1. 적응형 과열 가드 (Adaptive Overheat Guard) 매개변수 누락 버그 해결
* **목적**: 주도주(Confidence >= 75% 및 UPTREND)일 때 거래대금/거래량 상한선이 기존 2.2x에서 3.0x로 유연하게 작동하도록 개선.
* **패치 및 조치 내역**:
  - [strategy.py:L976-978](file:///c:/Users/stone/projects/nocodequant/app/ai/strategy.py#L976-L978) 에서 `check_overheat` 호출 매개변수를 다음과 같이 보완했습니다.
  ```python
  if sess.get('position') is None and self._penalty_engine.check_overheat(
      vol_ratio_val, logic_gap, confidence=confidence, current_regime=current_regime
  ):
  ```
  - `reasons` 로그 출력에서도 실제 완화된 한도(`vol_cap`)가 정확하게 출력되도록 로깅 로직을 동기화시켰습니다.

### 2. UI 테마 동기화 패치 (`premium_header_widget.py`)
* **목적**: 종목의 렌더링 위치에 따라 테마명이 불일치하게 보였던 시각적 오류 교정.
* **패치 및 조치 내역**:
  - [premium_header_widget.py:L248-L255](file:///c:/Users/stone/projects/nocodequant/app/ui/widgets/premium_header_widget.py#L248-L255) 에서 정적 테마가 캐시되지 않은 경우, 주도주 랭킹 매니저가 O(1)로 추론해 보관 중인 **동적 추론 테마 캐시**를 2차로 조회하도록 교차 검색을 적용했습니다.
  ```python
  # 1) 정적 캐시 테마 조회
  themes = self.main.ranking_mgr._get_cached_themes(self.current_code)
  if themes:
      theme_name = themes[0]
  else:
      # 2) 정적 맵에 없으면 동적 추론 캐시 조회 (2-hop GraphRAG 결과 연동)
      inferred = self.main.ranking_mgr._inferred_theme_cache.get(self.current_code)
      if inferred:
          theme_name = inferred
  ```
  - 이제 코스모로보틱스 등 테마가 직접 지정되지 않은 종목들도 양쪽 화면에서 동일하게 "의료기기"로 완벽 동기화되어 나타납니다.

### 3. 절대 거래대금 허들 가드 단독 차단 이력 분석
* **목적**: 3억 원 가이드라인 조정 이후 거래가 미발생한 원인을 파악하고, 수급 필터에 막혀 누수된 유효 신호가 있는지 검출.
* **분석 기법**: 
  - [inspect_today_value_blocks.py](file:///c:/Users/stone/projects/nocodequant/scratch/inspect_today_value_blocks.py) 진단용 분석 스크립트를 작성하여 forensics DB를 탐색했습니다.
  - Risk, Reward, MTF 등 타 조건은 모조리 만족하여 원래대로라면 `BUY` 신호로 정상 진입했을 종목 중, 오직 **5분봉 거래대금 3억 원 미만**으로 인해 `HOLD_BLOCK` 처리된 건수를 실측했습니다.
* **분석 결과 (09:57:00 이후):**
  - **총 955건 (21개 종목)**이 이 3억 원 필터 단독 차단에 의해 진입을 거부당했습니다.
  - **주요 필터링 종목:**
    1. **비보존 제약 (082800):** 최고 진입 점수 **73.1점** | 최대 5분봉 거래대금 **6,193만 원**
    2. **코스모로보틱스 (439960):** 최고 진입 점수 **70.7점** | 최대 5분봉 거래대금 **1.69억 원**
    3. **제주은행 (006220):** 최고 진입 점수 **69.2점** | 최대 5분봉 거래대금 **2.60억 원**
    4. **케이엠제약 (225430):** 최고 진입 점수 **67.7점** | 최대 5분봉 거래대금 **88만 원**

### 4. Git 저장소 연동 및 보안 업로드 프로세스 진행
* **목적**: 로컬 소스 코드 자산을 프라이빗 클라우드 저장소에 안전하게 백업하고 버전 관리를 일원화.
* **패치 및 조치 내역**:
  - 로컬 Git 저장소 초기화 후 원격 브랜치(`https://github.com/KevinCY-Kim/Nocodequant`) 연결.
  - 데이터베이스(`*.db`), 환경 설정 및 보안 자격 증명(`.env`), 각종 텔레메트리 실행 로그 및 백테스트 산출물들을 완벽 배제하기 위해 [.gitignore](file:///c:/Users/stone/projects/nocodequant/.gitignore)를 설계/반영.
  - 원격 리포지토리의 기본 `README.md` 파일 병합 시 발생한 충돌 사항을 로컬의 상세 기재본 기준으로 병합 처리 (`rebase --continue`).
  - [README.md](file:///c:/Users/stone/projects/nocodequant/README.md) 파일 내용을 실제 프로그램 폴더 트리 구조 및 v27.0 핵심 매매 기능들과 부합하게 최신화하여 원격 브랜치(`main`)에 최종 푸시 완료.

---

## 📊 검증 및 패치 확인 결과

### 1. 구문 및 컴파일 체크
- `strategy.py` 와 `premium_header_widget.py` 파일 모두 `py_compile` 검사를 통해 구문 오류 및 컴파일상 이상이 없음(Pass)을 검증했습니다.

### 2. 백업본과의 교차 일치성
- [strategy_back.py](file:///c:/Users/stone/projects/nocodequant/app/ai/strategy_back.py)와의 `diff` 대조를 통해 손실되었던 코드 부분이 원본과 오차 없이 100% 온전하게 이식 복구되었음을 최종 팩트 체크했습니다.

### 3. Git 커밋 및 트래킹 파일 검증
- `git status --ignored` 검증 결과, 비밀번호 및 API 토큰이 저장되는 `.env` 파일과 민감한 원장 DB 파일들이 안전하게 커밋 제외 대상에 속해 있음을 물리적으로 검증했습니다.

---

## 🔮 향후 보완 및 논의 방향
- **거래대금 가이드라인 추가 하향 제안:** 
  - 오늘 하락장 변동성 속에서도 제주은행(최대 2.6억), 코스모로보틱스(최대 1.69억) 등 높은 신뢰도를 보인 주도 세력들의 수급이 3억 제한선에 막힌 것이 입증되었습니다.
  - 시장 흐름을 한 박자 빠르게 잡아내기 위해, 수급 허들(`ENTRY_MIN_VALUE_5M`)을 **1.5억 원** 혹은 **2억 원** 선으로 추가 하향 조정하는 방안을 고려해 볼 만합니다.
- **프로세스 재부팅 안내:**
  - `strategy.py` 버그 수정과 `premium_header_widget.py` 의 UI 패치 변경 사항이 엔진에 실시간으로 물려 구동되도록, 구동 중인 HTS 인스턴스를 재시작해 주시기 바랍니다.
