# 배포 & 인프라

> **상태: 설계 기준 (2026-09-21).** `docker-compose.yml`, `nginx.conf`, GitHub Actions 워크플로는 아직 커밋되지 않았습니다.

---

## 서버 구성

| 환경 | 호스트 | 사양 | 트리거 |
|---|---|---|---|
| **dev** | 맥미니 (자택 상시 서버) | 기존 장비 재활용 | `dev` 브랜치 push |
| **release** | Oracle Cloud A1.Flex (Always Free, ARM Ampere) | **2 OCPU · 12GB** | `main` 브랜치 push |

### 왜 dev/release를 물리적으로 분리했는가

한 서버에 dev/release를 같이 올리는 방식(포트·컨테이너만 분리)을 먼저 고려했지만, **오라클 Always Free 할당량이 그걸 감당하지 못합니다.**

2026년 6월, 오라클이 Always Free ARM 할당량을 **4 OCPU·24GB → 2 OCPU·12GB로 축소**했습니다. 한 서버에 두 환경을 올리면 MySQL·Qdrant·Spring이 두 벌씩 떠서 12GB를 넘기고, dev에서 돌린 대량 임베딩 작업이 release 응답 속도에 그대로 영향을 줍니다.

계정을 추가로 만들어 무료 리소스를 늘리는 방법은 **Always Free 1인 1계정 원칙 위반**이라 선택지에서 제외했습니다. 대신 dev를 이미 갖고 있던 맥미니로 옮겨서 **리소스 경쟁 자체를 없앴습니다.** 추가 비용도 0입니다.

### 2 OCPU·12GB로 충분한 이유

이 프로젝트는 임베딩과 LLM 추론을 **전부 외부 API로 위임**합니다. 모델을 로컬에서 돌리지 않으므로 오라클 서버는 사실상 I/O 중계 역할입니다.

| 컨테이너 | 메모리 |
|---|---|
| Nginx + Frontend | ~200MB |
| Backend (Spring) | ~1GB |
| MySQL | ~800MB |
| Qdrant | ~500MB |
| **합계** | **~2.5GB / 12GB** |

CPU도 문서 파싱 시점에만 잠깐 튀고 평시에는 대기입니다.

---

## Docker Compose

```yaml
networks:
  internal:

services:
  nginx:
    ports: ["80:80", "443:443"]     # 외부에 노출되는 유일한 서비스
    networks: [internal]
  frontend:
    networks: [internal]            # 호스트 포트 미노출
  backend:
    networks: [internal]            # 호스트 포트 미노출
    mem_limit: 4g
  mysql:
    networks: [internal]
    mem_limit: 2g
  qdrant:
    networks: [internal]
    mem_limit: 2g
```

**Nginx만 호스트 포트를 물고 나머지는 내부 네트워크 전용**입니다. MySQL·Qdrant가 호스트에 포트를 열지 않으므로 방화벽에서 따로 막을 대상이 없습니다 — 차단 규칙을 유지하는 대신 노출 자체를 만들지 않는 쪽을 택했습니다.

`mem_limit`은 12GB 안에서 컨테이너 하나가 폭주해 서버 전체를 끌어내리는 것을 막기 위한 상한입니다.

> ⚠️ **미반영**: 캐싱 설계에는 Redis가 들어가는데([COST_OPTIMIZATION.md](COST_OPTIMIZATION.md)) 위 서비스 목록에 아직 없습니다. 구현 시 `redis` 서비스를 internal 네트워크에 추가해야 합니다(예상 ~200MB).

기본 명령:

```bash
docker compose up -d --build
docker compose ps
docker compose logs -f backend
docker compose down
```

---

## HTTPS

```
방문자 ──HTTPS──> Cloudflare ──HTTPS──> Nginx (오라클)
         (Cloudflare 자동)   (Origin Certificate)
                              SSL/TLS 모드: Full(strict)
```

- Cloudflare DNS를 **Proxied** 모드로 씁니다. 방문자↔Cloudflare 구간 인증서는 Cloudflare가 자동 처리하고, 오리진 IP도 가려집니다.
- Cloudflare↔오리진 구간은 **Origin Certificate**를 씁니다. **유효기간 15년이라 갱신 크론이 필요 없습니다** — Let's Encrypt였다면 90일마다 갱신 작업을 유지해야 했습니다.
- SSL/TLS 모드는 **Full(strict)**. Flexible(오리진 구간 평문)이나 Full(인증서 검증 안 함)과 달리, Cloudflare가 오리진 인증서를 실제로 검증합니다.

---

## CI/CD (GitHub Actions)

```
dev 브랜치 push
  └─> 빌드 · 테스트 ─> 맥미니로 SSH 배포 ─> docker compose up -d

main 브랜치 push
  └─> 빌드 · 테스트 ─> 이미지 GHCR 푸시 ─> 오라클로 SSH 배포 ─> docker compose pull && up -d
```

dev는 서버에서 바로 빌드하고, release는 **GHCR에 올린 이미지를 받아 띄웁니다.** release에서 빌드를 돌리지 않는 이유는 2 OCPU에서 Gradle 빌드와 Next.js 빌드를 돌리면 서비스 중인 컨테이너의 응답이 느려지기 때문입니다. 빌드는 Actions 러너가, 서버는 pull과 재시작만 합니다.

### 필요한 Secrets (예정)

| 구분 | 키 |
|---|---|
| 배포 | `DEV_SSH_HOST`, `DEV_SSH_KEY`, `PROD_SSH_HOST`, `PROD_SSH_KEY` |
| 레지스트리 | `GHCR_TOKEN` |
| 앱 | `OPENAI_API_KEY`, `JWT_SECRET`, `ENCRYPTION_KEY`, `MYSQL_ROOT_PASSWORD` |
| OAuth | `GOOGLE_CLIENT_*`, `KAKAO_CLIENT_*`, `NOTION_CLIENT_*` |

OAuth redirect URI는 dev/release가 다르므로 **제공사 콘솔에 두 환경 URI를 모두 등록**해야 합니다(Google·Kakao·Notion 각각).

---

## 운영 시 확인 항목

- [ ] Nginx 외에 호스트 포트를 무는 컨테이너가 없는지 (`docker compose ps`)
- [ ] MySQL 볼륨과 Qdrant 볼륨이 영속 볼륨으로 마운트됐는지 (컨테이너 재생성 시 데이터 유실 방지)
- [ ] PDF 원본 저장 경로(`Document.file_path`)가 볼륨 안에 있는지
- [ ] Cloudflare SSL 모드가 Full(strict)인지
- [ ] 각 컨테이너 `mem_limit` 합이 12GB를 넘지 않는지 (Redis 추가 후 재확인)

---

## 관련 문서

- [ARCHITECTURE.md](ARCHITECTURE.md#서버-내부-구조) — 컨테이너 간 통신 구조
- [decisions/2026-09-20-dev-release-server-split.md](decisions/2026-09-20-dev-release-server-split.md) — 결정 기록
