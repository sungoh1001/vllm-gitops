# Git-first vLLM Control Plane 학습 리소스

## Knowledge

### Docker & NVIDIA Container Toolkit

- [NVIDIA Container Toolkit 공식 문서](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/)
  GPU 컨테이너 실행을 위한 1차 자료. 설치 방법, `--gpus` 플래그, 환경 변수, MIG 설정. 이 워크스페이스에서 Docker GPU 설정의 기본 교재.

- [Docker Engine 설치 — Ubuntu (공식)](https://docs.docker.com/engine/install/ubuntu/)
  Ubuntu 24.04에 Docker Engine을 `apt`로 설치하는 공식 가이드. 에어갭 환경에서는 `apt` 소스를 내부 미러로 변경해 적용.

### Git & GitOps

- [Pro Git (한국어) — Scott Chacon, Ben Straub](https://git-scm.com/book/ko/v2)
  Git 공식 무료 서적. 2장(Git 기초), 7.8절(Git Hooks), 7.6절(Rewriting History)이 특히 관련됨.

- [GitOps — Operations by Pull Request — Weaveworks](https://www.weaveworks/blog/gitops-operations-by-pull-request)
  GitOps 개념을 만든 Weaveworks의 원본 블로그. 개념적 기초.

### Webhook & 자동화

- [GitHub Webhooks 공식 문서](https://docs.github.com/en/webhooks)
  GHE와 동일한 API 사용. Webhook 이벤트, 페이로드, 시크릿 검증.

- [Dockhand — Docker용 GitOps 엔진](https://github.com/dockhand/dockhand)
  Git push → webhook → git pull → docker compose up 파이프라인. 이 프로젝트의 GitOps 엔진 후보.

### 하드웨어

- [NVIDIA A100 PCIe 사양](https://www.nvidia.com/en-us/data-center/a100/)
  GPU 스펙, NVLink 호환성, MIG 지원 여부 확인.
- [Dell R740xd 기술 문서](https://www.dell.com/support/home/en-us/product-support/product/poweredge-r740xd/docs)
  GPU 라이저 구성, PCIe 슬롯 배치, BIOS 설정.

## Wisdom (Communities)

- [r/selfhosted](https://reddit.com/r/selfhosted)
  홈랩/셀프 호스팅 커뮤니티. code-server, Docker, GitOps 관련 실전 조언.
- [NVIDIA Developer Forum — CUDA](https://forums.developer.nvidia.com/c/gpu-accelerated-libraries/cuda-libraries/14)
  GPU 컨테이너, 드라이버, CUDA 관련 기술 질문.

## Gaps

- **에어갭 환경의 Docker 레지스트리 미러 설정**: 대부분의 Docker 문서는 공개 레지스트리를 가정. 내부 Nexus/Harbor 연동 참고자료 필요.
- **Dockhand 문서 부족**: 신생 프로젝트로, 공식 문서가 README 수준. 내부에서 직접 테스트하며 파악 필요.
- **vLLM 옵션 최적화 가이드**: throughput/latency/VEM 튜닝은 vLLM 자체 문서와 실험에 의존.
