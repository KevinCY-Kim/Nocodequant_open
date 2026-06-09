# NoCodeQuant — Antigravity Rules 거버넌스 안내서

> **Version:** 2026-02-27 | **헌장:** [AI Work Constitution v1.2](governance/AI_Work_Constitution_and_Reference_Map.md)

---

## 구조 개요

```
Antigravity_rules/
├── GOVERNANCE_MANIFEST.md    ← 문서 등록부 (Tier 0~3 전체 목록)
├── README.md                 ← 본 문서
│
├── governance/               ← Tier 0: 최상위 헌법
│   ├── AI_Work_Constitution_and_Reference_Map.md
│   └── Forensic_DB_Immutability_Rule.md
│
├── rules/                    ← Tier 1: 강제 규칙 (엔진 실행 전제)
│   ├── core/                 │   ├── market/
│   ├── guards/               │   ├── Signal_Gating_Rules.md
│   ├── Failure_Patterns_and_Guards_ko.md
│   └── Order_Lifecycle_Spec.md
│
├── policy/                   ← Tier 2: 적응형 정책
│   ├── Smart_Heat_Index_Spec.md
│   ├── Reason_Code_Architecture.md
│   └── Audit_Grade_Operational_Standard.md
│
├── architecture/             ← Tier 3: 설계 설명
│   └── (8개 문서)
│
└── _archive/kodex_legacy/    ← 범용 설계 DNA 보존
    └── (8개 KevinCY-Kodex 원본)
```

---

## 문서 분류 기준 (헌장 v1.2 §8)

| 단계 | 질문 | YES → | NO → |
|:----:|:-----|:------|:-----|
| **①** | 실행 경로에 직접 영향? | Rules 후보 | Docs |
| **②** | 변경 시 자본 손실 가능? | 최소 Tier 1 | Tier 2 또는 Docs |
| **③** | 엔진 코드가 이 문서를 전제로 동작? | Rules/Policy | Docs |

---

## 작업 시 필수 참조

1. 코드 변경 전 → [GOVERNANCE_MANIFEST.md](GOVERNANCE_MANIFEST.md) 확인
2. 파라미터 수정 → [Parameter_Invariants.md](rules/core/Parameter_Invariants.md) (L2 등급)
3. 데이터 수정 시도 → [Forensic_DB_Immutability_Rule.md](governance/Forensic_DB_Immutability_Rule.md) 사전 선언 의무

---

## `_archive/kodex_legacy/` 안내

초기 `KevinCY-Kodex` 범용 설계 문서 8개를 보존하고 있습니다.
NCQ 실행 엔진에 영향을 주지 않으며, 향후 플랫폼 확장(LLM Governance 등) 시 참고 자산으로 활용 가능합니다.

> 이 폴더의 문서는 NCQ 거버넌스 등록부에 포함되지 않습니다.