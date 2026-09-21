# dev/release 서버를 물리적으로 분리

- **날짜**: 2026-09-20
- **상태**: 확정
- **관련**: [DEPLOYMENT.md](../DEPLOYMENT.md)

## 배경

개발 환경과 배포 환경을 따로 두어야 하는데, 가진 자원은 오라클 Always Free 인스턴스 1개와 자택 맥미니 1대다.

2026년 6월 오라클이 Always Free ARM(Ampere A1) 할당량을 **4 OCPU·24GB → 2 OCPU·12GB로 축소**했다. 예고 없이 줄어든 것이라 기존 계획(한 서버에 dev/release 동시 운영)을 다시 검토해야 했다.

## 선택지

1. 오라클 1대에 dev/release를 컨테이너·포트로 분리해 동시 운영
2. 오라클 계정을 추가로 만들어 무료 인스턴스 확보
3. **dev = 맥미니 / release = 오라클로 물리 분리**

## 결정

3번. `dev` 브랜치는 맥미니로, `main` 브랜치는 오라클로 배포한다.

## 근거

- 1번: MySQL·Qdrant·Spring이 두 벌씩 떠서 12GB를 넘긴다. 또 dev에서 대량 임베딩을 돌리면 release 응답 속도가 그대로 느려진다 — 환경 분리의 목적 자체가 무너진다.
- 2번: Always Free는 **1인 1계정**이 정책이다. 위반이라 선택지에서 제외.
- 3번: 맥미니는 이미 상시 가동 중이라 추가 비용이 0이고, 리소스 경쟁이 물리적으로 발생할 수 없다.

## 트레이드오프

두 환경의 하드웨어·아키텍처가 다르다(맥미니 = Apple Silicon, 오라클 = ARM Ampere). 둘 다 ARM64라 컨테이너 이미지 아키텍처는 맞지만, 사양 차이 때문에 **dev에서 잡히지 않는 성능·메모리 문제가 release에서 나타날 수 있다.** 컨테이너 `mem_limit`을 오라클 기준으로 맞춰 dev에서도 같은 상한으로 돌린다.

## 영향

- GitHub Actions 워크플로가 브랜치별로 갈린다: `dev` → 맥미니 SSH 배포, `main` → GHCR 푸시 후 오라클 SSH 배포
- release에서는 빌드를 돌리지 않는다. 2 OCPU에서 Gradle·Next.js 빌드를 하면 서비스 중인 컨테이너가 느려지므로, 빌드는 Actions 러너가 하고 서버는 pull과 재시작만 한다
- OAuth redirect URI를 dev/release 두 벌 등록해야 한다 (Google·Kakao·Notion 각각)
- 오라클은 임베딩·추론을 전부 외부 API로 위임하므로 I/O 중계 역할이고, 실측 예상 메모리는 ~2.5GB로 12GB 안에서 충분하다
