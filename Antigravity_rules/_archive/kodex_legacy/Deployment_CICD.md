# KevinCY-Kodex — Deployment & CI/CD Guideline

**Version:** 2025.11.27 (Universal Standard)
**Scope:** FastAPI · Docker (CPU/GPU) · GitHub Actions · Model Ops

---

## 1. CI/CD 파이프라인 개요 (CI/CD Pipeline Overview)

`KevinCY-Kodex`의 CI/CD 파이프라인은 코드 변경사항을 안정적이고 자동화된 방식으로 사용자에게 전달하는 것을 목표로 합니다.

**전체 흐름**:
`Git Push` (feature branch) → `Pull Request` → **[CI]** `Lint & Test` 실행 → `Code Review` & `Merge` (main branch) → **[CD]** `Docker Build & Push` → `Deploy` (staging/production)

---

## 2. 브랜치 전략 (Branching Strategy)

**Git-Flow**의 간소화된 버전을 따릅니다.

-   `main`: 항상 배포 가능한 상태를 유지하는 메인 브랜치.
-   `develop`: 다음 릴리즈를 위해 개발 중인 기능들이 통합되는 브랜치 (선택 사항).
-   `feature/<feature-name>`: 새로운 기능을 개발하는 브랜치. `main` 또는 `develop`에서 분기합니다.
-   `fix/<bug-name>`: 버그를 수정하는 브랜치.

모든 작업은 `feature` 또는 `fix` 브랜치에서 시작하며, `Pull Request`를 통해 `main` 브랜치로 병합됩니다.

---

## 3. CI (Continuous Integration)

**GitHub Actions**를 사용하여 CI 파이프라인을 구축합니다.

-   **트리거**: `main` 브랜치로의 `push` 또는 `pull_request` 이벤트 발생 시.
-   **주요 작업 (Jobs)**:
    1.  **System Dependencies**: 음성/AI 처리에 필요한 시스템 패키지(예: `ffmpeg`, `build-essential`)를 먼저 설치합니다.
    2.  **Linting**: `ruff` 또는 `flake8`을 사용하여 코드 스타일을 검사합니다.
    3.  **Testing**: `pytest`를 실행하여 모든 테스트가 통과하는지 확인합니다 (`Testing_Strategy.md` 준수).
-   **목표**: `main` 브랜치에 병합되는 모든 코드가 최소한의 품질 기준을 통과했음을 보증합니다.

**예시 워크플로우 (`.github/workflows/ci.yml`)**:
```yaml
name: CI

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    - name: Set up Python
      uses: actions/setup-python@v3
      with:
        python-version: '3.11'
    - name: Install System Deps (Audio/AI)
      run: |
        sudo apt-get update
        sudo apt-get install -y ffmpeg
    - name: Install dependencies
      run: |
        python -m pip install --upgrade pip
        pip install -r requirements.txt
    - name: Lint with ruff
      run: |
        pip install ruff
        ruff check .
    - name: Test with pytest
      run: |
        pip install pytest pytest-asyncio pytest-mock httpx
        pytest
```

---

## 4. Dockerization 전략 (AI Optimized)

-   **`Dockerfile` 표준**:
    -   **Base Image Strategy**:
        -   **CPU Mode (API/Web):** `python:3.11-slim` (경량화 우선)
        -   **GPU Mode (Local LLM/Whisper):** `nvidia/cuda:12.x.x-base-ubuntu22.04` (CUDA 지원 필수)
    -   **System Dependencies**: `Voice_Pipeline` 처리를 위해 `ffmpeg` 패키지 설치를 필수로 포함합니다.
    -   **Multi-stage Build**: 빌드 도구(`gcc` 등)와 실행 환경을 분리하여 최종 이미지 크기를 최소화합니다.
    -   **Non-root User**: 보안을 위해 `appuser` 계정을 생성하여 애플리케이션을 실행합니다.
-   **Model Weights Handling (중요)**:
    -   수 GB에 달하는 LLM/Embedding 모델 파일은 **Docker 이미지에 절대 포함하지 않습니다.**
    -   런타임 시 **Volume Mount**를 통해 호스트의 모델 폴더를 연결하거나, 컨테이너 시작 스크립트에서 S3/Cloud Storage로부터 다운로드 받습니다.
-   **`.dockerignore`**: `.git`, `__pycache__`, `.env`, `data/`, `static/tts/` 등 불필요하거나 임시적인 파일은 반드시 제외합니다.

---

## 5. CD (Continuous Deployment)

**GitHub Actions**를 사용하여 CD 파이프라인을 구축합니다.

-   **Staging 환경**:
    -   **트리거**: `main` 브랜치에 `push`될 때마다 자동 배포됩니다.
    -   **프로세스**: Docker 이미지 빌드 & Push (Registry) → Staging 서버에서 `docker pull` & 서비스 재시작.
-   **Production 환경**:
    -   **트리거**: **수동 실행(workflow_dispatch)** 또는 **Git 태그(Tag)** 생성 시 배포됩니다. (예: `v1.0.0`)
    -   **프로세스**: 검증된 특정 버전의 Docker 이미지를 배포하며, 서비스 중단을 최소화하기 위해 **무중단 배포(Rolling Update)** 전략을 권장합니다.

---

## 6. Secret 관리 (Secret Management)

API 키, DB 접속 정보 등 민감 정보는 코드에 절대 포함하지 않습니다.

-   **Local 환경**: `.env` 파일을 사용하며, 이 파일은 `.gitignore`에 추가하여 버전 관리에서 제외합니다.
-   **CI/CD 환경**: **GitHub Secrets**를 사용하여 워크플로우(`yaml`) 내에서 환경 변수로 주입합니다.
-   **배포 환경 (Staging/Production)**:
    -   서버의 시스템 환경 변수로 설정하거나, 배포 시점에 보안된 경로의 `.env` 파일을 로드합니다.
    -   가능하다면 클라우드 제공업체의 **Secret Manager** 서비스 사용을 권장합니다.

---

## 7. 모니터링 및 롤백 (Monitoring & Rollback)

-   **모니터링**: 배포 후 애플리케이션의 상태(Error Rate, Latency, CPU/GPU Usage)를 지속적으로 모니터링합니다. (`Architecture.md` 및 `Voice_Pipeline.md`의 로깅 규칙 참조)
-   **롤백 (Rollback)**: 배포 후 치명적인 문제가 발생할 경우, 즉시 이전 버전의 Docker 태그(예: `v1.0.0` -> `v0.9.9`)로 되돌릴 수 있는 **Revert 스크립트**를 사전에 준비합니다.

---

## 8. Agent Protocol (Antigravity Deployment Guardrails)
*에이전트가 배포 관련 스크립트(Dockerfile, YAML)를 작성할 때 따르는 절대 수칙입니다.*

1.  **No Hardcoding:** API Key나 Password를 스크립트 내부에 텍스트로 박아넣지 마십시오. 반드시 `${ENV_VAR}` 형식을 사용하십시오.
2.  **Volume Check:** 배포 설정 시, `data/` (벡터DB) 및 `static/` (TTS 파일) 폴더가 **영구 스토리지(Volume)** 에 연결되어 있는지 확인하십시오. (컨테이너 재시작 시 데이터 유실 방지)
3.  **Dry Run:** 배포 명령을 실행하기 전, 사용자에게 대상 서버와 버전을 보여주고 승인을 요청하십시오.

End of KevinCY-Kodex Deployment & CI/CD Guideline