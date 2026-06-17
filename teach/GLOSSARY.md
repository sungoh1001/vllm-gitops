# Git-first vLLM Control Plane 용어집

서버 설정과 GitOps Control Plane 구축에 필요한 핵심 용어를 정의합니다.

---

## 인프라

**NVIDIA Container Toolkit**:
Docker가 호스트의 NVIDIA GPU를 컨테이너에 노출할 수 있게 해주는 도구 모음(`nvidia-container-runtime` 포함). `docker run --gpus` 플래그를 사용 가능하게 하는 필수 구성 요소.

**Docker Engine**:
컨테이너를 빌드·실행·관리하는 런타임. `dockerd` 데몬과 `docker` CLI로 구성됨. Ubuntu 24.04에서는 `apt`로 설치.

**Private Registry / Mirror**:
외부 인터넷 없이 Docker 이미지를 저장·배포하는 내부 서버. Nexus Repository, Harbor 등이 사용됨. `docker pull` 요청을 내부 저장소로 리디렉션.

**Air-gap Network**:
외부 인터넷에 직접 연결되지 않은 격리된 네트워크. 모든 소프트웨어와 데이터는 내부 미러를 통해 유입.

---

## GPU

**NVLink**:
GPU 간 고속 직접 연결 인터페이스(600GB/s). A100 PCIe 버전에서는 별도 NVLink Bridge 하드웨어 필요. 없으면 GPU 간 통신은 PCIe(32GB/s)로만 가능.

**Tensor Parallelism (TP)**:
LLM을 여러 GPU에 나눠서 연산하는 방식. 매 연산마다 GPU 간 데이터 전송 필요. NVLink가 없으면 PCIe 병목으로 비효율.

**VRAM**:
GPU의 전용 메모리. A100 80GB 기준. vLLM은 모델 가중치 + KV Cache를 여기에 올림.

**MIG (Multi-Instance GPU)**:
A100 이상에서 지원. 하나의 물리 GPU를 여러 개의 작은 GPU 인스턴스로 하드웨어 분할.

---

## 컨테이너

**Docker Image**:
컨테이너의 청사진. vLLM 공식 이미지는 `vllm/vllm-openai`로 제공되며, 내부 Registry에 미러링해서 사용.

**Docker Compose**:
여러 컨테이너 구성(`compose.yaml`)을 선언적으로 정의하고 한 번에 관리하는 도구.

**Volume Mount**:
호스트 디렉토리를 컨테이너에 연결. 컨테이너 삭제 후에도 데이터 보존. 모델 캐시와 실험 결과 저장에 필수.

---

## Git & GitOps

**GitOps**:
Git 저장소를 시스템의 단일 진실 공급원으로 삼고, Git 변경을 자동으로 시스템에 반영하는 운영 방식.

**Webhook**:
Git 이벤트(push 등) 발생 시 등록된 URL로 HTTP POST를 보내는 메커니즘. GitOps 파이프라인의 방아쇠.

**Declarative Configuration**:
"이렇게 하라"(how) 대신 "이 상태여야 한다"(what)를 기술하는 설정 방식. Docker Compose YAML이 대표적 예.

**Reconciliation**:
실제 상태를 원하는 상태로 지속적으로 맞추는 프로세스. GitOps의 핵심 루프.

---

## 도구

**code-server**:
브라우저에서 VS Code를 실행하는 오픈소스 웹 IDE. Git 내장. SSH 없이 코드 편집과 Git 작업을 웹에서 가능하게 함.

**Dockhand**:
Git webhook을 수신해 `git pull` + `docker compose up`을 자동 실행하는 경량 GitOps 도구.
