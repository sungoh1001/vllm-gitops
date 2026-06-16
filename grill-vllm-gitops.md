# vLLM GitOps — Shadow IT 인프라 셀프 서비스 설계 보고서

> 작성일: 2026-06-16
> 출처: /grill-me 세션 — 요구사항 구체화 및 기술 방향 결정

---

## 1. 배경 및 목표

### 1.1 현재 상황

| 항목 | 상태 |
|------|------|
| 하드웨어 | Dell R740xd, NVIDIA A100 80GB PCIe × 2장 |
| GPU 연결 | PCIe 4.0 ×16 (NVLink 없음) |
| 현재 LLM 런타임 | Ollama (호스트 직접 설치, 단일 사용자) |
| vLLM 경험 | 없음 |
| 실험 기록 | 없음 |
| 재현성 | 없음 (호스트 직접 설치, Git 미사용) |
| 목표 | vLLM으로 10명 동시 서빙 최적 조건 탐색 |

### 1.2 네트워크 환경

- 서버: **Private Air-gapped Network** (외부 인터넷 직접 접근 불가)
- GitHub Enterprise Server (GHE) — 내부망에서 Git 접근 가능
- Nexus Repository — Docker 이미지 프록시 (NVIDIA Registry 미러링)
- Hugging Face Mirror — 내부망에서 모델 다운로드 가능

### 1.3 핵심 제약

- GPU 간 NVLink 없음 → Tensor Parallelism(TP2)은 PCIe 병목으로 비효율적
- Air-gap → 외부 서비스 직접 접근 불가, 내부 미러 경유 필수
- SSH 접속을 완전히 제거하여 Git-first 워크플로우 강제

---

## 2. GPU 사용 전략

### 2.1 GPU 점유 정책

| 정책 | 값 |
|------|-----|
| 기본 모드 | **하이브리드** — GPU당 독립 실행, 필요 시 전체 요청 가능 |
| NVLink 부재 | TP2 대신 **독립 실행** (GPU당 별도 vLLM 인스턴스) |
| Dev/Ops 분리 | code-server: GPU 불필요. vLLM dev: GPU0. vLLM ops: GPU1 (Dev 미사용 시 GPU0도 사용) |

### 2.2 GPU 할당 테이블

| 시나리오 | Dev (GPU0) | Ops (GPU1) | Ops (GPU0) |
|----------|------------|------------|------------|
| Dev 실행 중, Ops 운영 중 | vLLM dev 점유 | vLLM ops 점유 | 미사용 |
| Dev 종료, Ops 운영 중 | free | vLLM ops 점유 | **Ops가 추가 확보** |
| Dev만 실행 중 | vLLM dev 점유 | free | free |
| 모두 미사용 | free | free | free |

---

## 3. Git-first Control Plane 아키텍처

### 3.1 구성 요소

| 레이어 | 도구 | 역할 | 설치 방식 |
|--------|------|------|-----------|
| 웹 IDE | **code-server** | 브라우저에서 run.sh 편집, git commit/push | `docker run codercom/code-server` |
| GitOps 엔진 | **Dockhand** | Webhook 수신 → 자동 git pull → docker compose up | `docker run dockhand/dockhand` |
| Git 저장소 | GHE (기존) | Git remote, webhook 발신 | 기존 인프라 |
| Docker 이미지 | Nexus (기존) | Docker Registry Proxy (NVIDIA → 내부) | 기존 인프라 |
| 모델 다운로드 | HF Mirror (기존) | Hugging Face 모델 내부 캐시 | 기존 인프라 |
| 컨테이너 런타임 | Docker Engine | GPU 컨테이너 실행 | 호스트 기본 설치 |

### 3.2 아키텍처 다이어그램

```
┌─────────────────── R740xd (Air-gap Network) ──────────────────┐
│                                                                │
│  ┌──────────────────────────────────────────────────────┐      │
│  │  Docker Engine                                        │      │
│  │                                                       │      │
│  │  ┌──────────────┐  ┌──────────┐  ┌──────────────────┐│      │
│  │  │  code-server   │  │ Dockhand  │  │  vLLM (ops)     ││      │
│  │  │  (웹 IDE)      │  │ (GitOps)  │  │  (10명 서빙)    ││      │
│  │  │                │  │          │  │                  ││      │
│  │  │  run.sh 편집   │  │ webhook  │  │  GPU1 (+GPU0)   ││      │
│  │  │  git commit    │  │ 수신     │  │  docker compose  ││      │
│  │  │  git push      │  │ git pull │  │  up              ││      │
│  │  └───────┬───────┘  └────┬─────┘  └──────────────────┘│      │
│  │          │               │                              │      │
│  └──────────┼───────────────┼──────────────────────────────┘      │
│             │               │                                      │
│             ▼               ▼                                      │
│      ┌────────────┐  ┌──────────┐                                 │
│      │    GHE     │  │  Nexus   │  ┌──────────────┐              │
│      │ (Git 저장소)│  │ (이미지   │  │  HF Mirror   │              │
│      │ webhook    │  │  프록시)  │  │  (모델 캐시)  │              │
│      │ 발신       │  │          │  │              │              │
│      └────────────┘  └──────────┘  └──────────────┘              │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

---

## 4. Git 리포지토리 구조

### 4.1 최소 구조 (P0)

```
vllm-experiments/
├── run.sh              # vLLM 실행 명령어 (전체 설정)
├── .env                # HF_TOKEN 등 시크릿 (gitignore)
└── .gitignore
```

### 4.2 run.sh 템플릿

```bash
#!/bin/bash
# === 실험 설정 (여기만 바꿔서 commit) ===
MODEL="meta-llama/Llama-3.1-8B-Instruct"
TP_SIZE=1
MAX_MODEL_LEN=8192
GPU_MEM_UTIL=0.90
DTYPE="bfloat16"
VLLM_IMAGE="nexus.internal.company.com/vllm/vllm-openai:v0.7.1"
GPU_DEVICES="device=0"

# === vLLM 실행 ===
docker run --gpus $GPU_DEVICES --rm \
  -e HF_TOKEN=$HF_TOKEN \
  $VLLM_IMAGE \
  vllm serve $MODEL \
    --tensor-parallel-size $TP_SIZE \
    --max-model-len $MAX_MODEL_LEN \
    --gpu-memory-utilization $GPU_MEM_UTIL \
    --dtype $DTYPE
```

---

## 5. 워크플로우

### 5.1 일일 실험 워크플로우

| 단계 | 작업 | 도구 | SSH 필요? |
|------|------|------|-----------|
| 1 | 브라우저로 code-server 접속 | 브라우저 → `http://server:8443` | ❌ |
| 2 | run.sh 수정 (모델, 파라미터 변경) | code-server VS Code UI | ❌ |
| 3 | git commit + git push | code-server Git GUI 또는 터미널 | ❌ |
| 4 | (자동) GHE webhook → Dockhand 수신 | Dockhand | ❌ |
| 5 | (자동) git pull + docker compose up | Dockhand | ❌ |
| 6 | vLLM 서빙 로그 확인 | code-server 터미널 또는 Dockhand UI | ❌ |
| 7 | 실험 결과 관찰 (수동) | code-server README 기록 | ❌ |
| 8 | git commit (결과 기록) | code-server | ❌ |

### 5.2 실험 전환 워크플로우

```
Dev (GPU0) 실험 중:
  code-server → run.sh 수정 → git push
    → Dockhand가 GHE에서 pull
    → GPU0으로 docker compose up
    → vLLM dev 인스턴스 재시작

Ops (GPU1) 운영 중:
  항상 GPU1에서 10명 서빙 유지
  Dev가 GPU0 사용 중이면 Ops는 GPU1만 사용
  Dev가 GPU0을 반납하면 Ops가 GPU0도 추가 확보
```

---

## 6. 의사 결정 요약

| 결정 번호 | 주제 | 결정 사항 | 근거 |
|-----------|------|-----------|------|
| D1 | GPU 사용 방식 | **독립 실행** (TP2 안 함) | NVLink 없음, PCIe 병목 |
| D2 | GPU 점유 정책 | **하이브리드** | 기본 독립, 필요 시 전체 할당 |
| D3 | Ops GPU 전략 | GPU1 고정, Dev 미사용 시 GPU0 추가 사용 | 자원 활용률 + Dev 보호 |
| D4 | P0 (실험 히스토리) | **Git 커밋 = 실험 설정** (`run.sh` 단일 파일) | 가장 단순한 Git-first 시작점 |
| D5 | P1 (환경 재현성) | **Docker 컨테이너 전용** | 호스트 오염 방지, 버전 고정 |
| D6 | Git 서버 | **GHE (기존 활용)** | 이미 내부망에 구축됨. Gitea 불필요 |
| D7 | 컨테이너 관리 | **Dockhand** (GitOps 엔진) | Git-first 철학 내장, webhook 수신 자동화 |
| D8 | 웹 IDE | **code-server** | 브라우저 VS Code, Git 내장 |
| D9 | SSH 제거 | **완전 제거 목표** | Git-first 강제, 편의성 향상 |
| D10 | 모델 다운로드 | **HF Mirror (기존)** | Air-gap 문제 없음 |
| D11 | 이미지 다운로드 | **Nexus (기존)** | NVIDIA Registry 프록시 |
| D12 | Metrics | **Throughput, Latency(P50/P95/P99), VRAM 효율, 안정성** — 수동 수집 | 자동화는 P3 이후 |

---

## 7. 우선순위 로드맵

| 우선순위 | 기능 | 상태 | 비고 |
|----------|------|------|------|
| **P0** | Git 커밋 = 실험 설정 | ✅ 결정 완료 | `run.sh` 단일 파일, `git log` = 실험 이력 |
| **P1** | 환경 재현성 (Docker) | ✅ 결정 완료 | `vllm/vllm-openai` 이미지, Nexus 경유 |
| **P2** | Git push = 자동 실행 (Dockhand) | ✅ 결정 완료 | GHE webhook → Dockhand → git pull → compose up |
| **P3** | GPU 스케줄링 (다중 사용자) | 🔲 미정 | Docker GPU 락 메커니즘, 대기열 |
| **P4** | Metrics 자동 수집/시각화 | 🔲 미정 | 수동 수집으로 시작, 필요시 도입 |
| **P5** | K8s + ArgoCD 전환 | 🔲 미정 | 서버 규모가 커지면 검토 |

---

## 8. SSH 완전 제거 조건

SSH 없이 모든 작업이 가능하려면 다음이 서버에 설치/실행되어 있어야 합니다 (SSH로 단 1회 초기 설치):

```bash
# 1. Docker Engine (설치되어 있다고 가정)

# 2. code-server
docker run -d --name code-server \
  -p 8443:8080 \
  -v /home/$USER/vllm-experiments:/home/coder/project \
  codercom/code-server:latest

# 3. Dockhand
docker run -d --name dockhand \
  -p 9001:9001 \
  -v /home/$USER/vllm-experiments:/repo \
  -v /var/run/docker.sock:/var/run/docker.sock \
  dockhand/dockhand
```

설치 후 GHE에 Webhook 등록:
- URL: `http://server:9001/hooks/vllm-push`
- Trigger: Push events
- Secret: (선택)

이후 모든 작업은 브라우저(code-server)에서만 수행. SSH 재접속 불필요.

---

## 9. 참고 사항

- vLLM 경험이 없으므로, 첫 실험은 `meta-llama/Llama-3.1-8B-Instruct` 같은 작은 모델로 TP=1, GPU0에서 시작하는 것을 권장
- Dockhand의 정확한 설치/설정은 공식 문서 (GitHub `dockhand/dockhand`) 참조 필요
- Portainer CE는 GitOps 철학이 약하고 UI 버튼으로 Git 우회 가능하므로 채택하지 않음
- 향후 서버 증설이나 다중 사용자 환경이 되면 K8s + ArgoCD 전환을 검토할 수 있음