# Mission: Git-first vLLM Control Plane 구축

## Why
Dell R740xd 서버(A100 80GB PCIe × 2, 1.5TB RAM)에서 vLLM 기반 로컬 LLM 실험과 서빙을 **Git-first + 웹 UI + Docker 컨테이너 격리** 방식으로 운영할 수 있는 Control Plane을 직접 구축한다. 실험 설정이 Git 히스토리로 자동 기록되고, SSH 없이 브라우저만으로 모든 제어가 가능한 환경을 만든다.

## Success looks like
- Clean OS에서 시작해 Docker + NVIDIA Container Toolkit이 설치되고 GPU 컨테이너가 실행된다
- GPU별로 실험(vLLM dev)과 서빙(vLLM ops)을 분리해 컨테이너로 실행할 수 있다
- vLLM 실행 설정을 Git에 커밋하고, webhook이 자동으로 컨테이너를 갱신하는 파이프라인이 동작한다
- 5명의 팀원이 각자 브라우저로 code-server에 접속해 실험을 수행할 수 있다

## Constraints
- Air-gap 네트워크. 내부 Docker Registry / APT / HF / GHE 미러에 의존
- NVLink 없음 — GPU 2장은 독립 사용
- Docker/Git 개념은 알고 있으나 CLI 실사용 경험 적음
- 개념 이해를 먼저, 이후 실습

## Out of scope
- Kubernetes, ArgoCD (서버 1대)
- vLLM 내부 추론 엔진 동작 원리
- LLM 모델 파인튜닝
- 외부 인터넷 서비스 연동
