# SourceNote

전공 PDF·Notion 자료를 벡터DB에 인덱싱해 **출처 기반 질의응답**과 **객관식 문제 자동생성**을 제공하는 학습 도우미.

<br/>

## 핵심 기능

| 기능 | 설명 |
|---|---|
| **질문하기** | 과목 단위 RAG 검색 후 답변. 근거로 쓰인 문서명·페이지를 함께 반환하고, 유사도 threshold 미달이면 추측하지 않고 "근거 없음"으로 응답 |
| **문제 자동생성** | 과목의 chunk를 샘플링해 객관식 문제를 생성. 채점·오답 기록 저장 |
| **Notion 연동** | Notion OAuth로 계정 연결 → 페이지 다중 선택 가져오기. 재동기화는 `마지막 변경시간` 비교로 변경분만 |
| **사용량 대시보드** | 기능별(임베딩/질문답변/문제생성/Notion동기화)·과목별·일별 토큰 비용 집계 |

인증은 Google·Kakao OAuth 로그인 + 서버 발급 JWT. 이메일이 같으면 같은 계정으로 자동 통합됩니다.

<br/>

## 왜 이렇게 설계했는가

이 프로젝트의 목적은 "많은 사용자를 받는 서비스"가 아니라 **멀티유저·인증·비용 분리가 제대로 된 RAG 시스템**을 끝까지 만들어 보는 것입니다.

### 1. BYOK — 임베딩은 개발자 부담, 추론은 사용자 키

비용 성격이 다릅니다. 임베딩은 **문서를 넣을 때 한 번**이고 100페이지 전공서가 약 2원 수준이라 개발자 고정 키로 흡수해도 됩니다. 반면 질문답변·문제생성은 **사용자 수 × 호출 수만큼 무한히 누적**되고 고가 모델이면 질문 1회에 10원을 넘습니다.

그래서 추론만 사용자 본인 API 키(Claude / GPT 선택)로 분리했습니다. 부수 효과로 "요청마다 다른 provider·key·model로 LLM 클라이언트를 만든다"는 요구가 생깁니다. → [COST_OPTIMIZATION.md](docs/COST_OPTIMIZATION.md)

### 2. Qdrant 단일 컬렉션 (`study_chunks`) + payload 필터

사용자·과목마다 컬렉션을 나누면 컬렉션 수가 `사용자 × 과목`으로 선형 증가하고, 컬렉션마다 별도 HNSW 인덱스와 세그먼트를 들고 있어야 합니다. 2 OCPU·12GB 서버에서 감당할 구조가 아닙니다. 또 "전체 자료 검색" 기능이 요구사항에 있어서, 단일 컬렉션이면 **필터를 빼는 것만으로** 구현됩니다.

격리는 payload 인덱스(`user_id`, `subject_id`) 필터로 보장하고, `user_id` 필터는 리포지토리 레이어에서 강제 주입합니다. → [DATABASE.md](docs/DATABASE.md#qdrant)

### 3. dev(맥미니) / release(오라클) 물리 분리
 브랜치도 `dev` → 맥미니 | `main` → 오라클로 갈립니다. → [DEPLOYMENT.md](docs/DEPLOYMENT.md)

### 4. Notion은 Internal 토큰 붙여넣기 대신 Public Integration OAuth

Internal Integration 방식은 사용자가 Notion에서 직접 페이지를 공유 설정하고 토큰을 복사해 와야 합니다. Public Integration OAuth는 **동의 화면이 그대로 페이지 선택 화면 역할**을 해서, 연동 UX가 한 단계로 끝납니다. 로그인(Google/Kakao)과 외부 연동(Notion)을 전부 OAuth 2.0 한 가지 방식으로 통일한다는 점도 같습니다.

### 5. 저품질 페이지만 비전 모델로 재처리

전공서는 수식·표 때문에 텍스트 추출이 부분적으로 깨집니다. 전체를 비전 모델로 돌리면 비용이 자릿수로 뛰므로, **페이지 단위로 품질을 검사해 미달 페이지만** 재처리합니다. → [ARCHITECTURE.md](docs/ARCHITECTURE.md#문서-처리-파이프라인)

---

## 아키텍처

```
Browser
  │ HTTPS
  ▼
Cloudflare  (Proxied, HTTPS 종단)
  │ Origin Certificate / SSL 모드 Full(strict)
  ▼
┌───────────── Oracle Cloud A1.Flex (2 OCPU · 12GB) ─────────────┐
│                                                                │
│  Nginx  :80 :443   ← 유일한 외부 노출                           │
│    ├── /              → frontend  (Next.js)                    │
│    └── /api, /oauth2  → backend   (Spring Boot)                │
│                            ├── MySQL   메타데이터              │
│                            ├── Qdrant  study_chunks            │
│                            └── Redis   질문·시맨틱 캐시         │
│                                                                │
│  frontend·backend·mysql·qdrant·redis = Docker 내부망 전용       │
│  (호스트 포트 미노출 → 방화벽 규칙 자체가 불필요)                │
└────────────────────────────────────────────────────────────────┘
       │ 외부 API
       ├── OpenAI Embeddings   개발자 고정 키
       ├── Claude / GPT Chat   사용자 API 키 (BYOK)
       └── Notion API          사용자 OAuth 토큰
```

질문 1건이 처리되는 경로:

```
Controller → Service → RetrievalAugmentationAdvisor (Qdrant 검색 + 프롬프트 조립)
           → ChatClient (사용자 설정 provider·key·model로 즉석 생성) → 출처 포함 JSON
```

파이프라인 상세(청킹 파라미터, 품질 체크, threshold, 비동기 처리)는 → **[docs/ARCHITECTURE.md](docs/ARCHITECTURE.md)**

---

## 기술 스택

| 영역 | 사용 기술 | 비고 |
|---|---|---|
| 언어 | Java 21, TypeScript | |
| 백엔드 | Spring Boot, Spring AI, Spring Security | Spring AI의 `RetrievalAugmentationAdvisor`로 RAG 조립 |
| 프론트엔드 | Next.js | |
| RDB | MySQL | 사용자·과목·문서·문제·사용량 로그 |
| 벡터DB | Qdrant | 단일 컬렉션 `study_chunks`, 1536차원, Cosine, HNSW |
| 캐시 | Redis | 동일 질문 캐싱 + 시맨틱 캐싱 |
| 문서 파싱 | Apache Tika / PDFBox | 저품질 페이지는 비전 모델로 재처리 |
| 임베딩 | `text-embedding-3-small` (1536d) | 개발자 고정 키 |
| 추론 LLM | Anthropic Claude / OpenAI GPT | **사용자 API 키**, 모델은 사용자가 선택 |
| API 문서 | springdoc-openapi | 프론트-백엔드 계약 |
| 인프라 | Docker Compose, Nginx, Cloudflare | Nginx만 외부 노출 |
| CI/CD | GitHub Actions, GHCR | `dev`→맥미니, `main`→오라클 |
| 테스트 | JUnit, Testcontainers(Qdrant), WireMock(LLM API) | |

---

## API 문서

springdoc-openapi가 스펙을 생성합니다.

| 환경 | Swagger UI | OpenAPI JSON |
|---|---|---|
| 로컬 | http://localhost:8080/swagger-ui.html | http://localhost:8080/v3/api-docs |
| release | `https://<도메인>/swagger-ui.html` | `https://<도메인>/v3/api-docs` |

엔드포인트 목록·인증 방식은 → **[docs/API.md](docs/API.md)**

---

## 폴더 구조

```
SourceNote/
├── backend/                  # (예정) Spring Boot — Java 21
│   └── src/main/
│       ├── java/.../sourcenote/
│       │   ├── auth/         # OAuth 로그인, JWT 발급·검증
│       │   ├── subject/      # 과목
│       │   ├── document/     # PDF·Notion 수집, 파싱, 청킹, 임베딩
│       │   ├── rag/          # 검색 어드바이저, 프롬프트 조립, 질문하기
│       │   ├── quiz/         # 문제 생성·채점
│       │   ├── notion/       # Notion OAuth·API 클라이언트
│       │   ├── settings/     # provider·API 키·모델 설정
│       │   ├── usage/        # LlmUsageLog 기록·집계
│       │   └── common/       # 설정, 예외, 암호화
│       └── resources/application.yml
├── frontend/                 # (예정) Next.js + TypeScript
├── infra/                    # (예정) docker-compose.yml, nginx.conf
├── .github/workflows/        # (예정) dev.yml, release.yml
├── docs/
│   ├── ARCHITECTURE.md       # 파이프라인·요청 경로·내부 구조
│   ├── API.md                # 엔드포인트 목록
│   ├── DATABASE.md           # MySQL 스키마 + Qdrant 컬렉션
│   ├── DEPLOYMENT.md         # 서버 분리·Docker·HTTPS·CI/CD
│   ├── COST_OPTIMIZATION.md  # 캐싱 전략·비용 계산·BYOK 근거
│   └── decisions/            # 날짜별 설계 결정 기록
├── LICENSE
└── README.md
```

---

## 향후 계획

**Phase 1 (MVP)** — 위 핵심 기능 4종 + 인증 + 사용량 집계.

**Phase 2 — 에이전트 확장.** Spring AI 2.0 `ToolCallingAdvisor`(어드바이저 체인 기반 툴 콜링 루프)로 구현 예정. 진입 시점에 순서를 정합니다.

- **A. 문제 생성 자체검증** — 생성된 문제를 원본 chunk와 대조 검증, 기준 미달이면 재생성 (Reflection 패턴)
- **B. 학습 약점 분석 플래너** — 오답률 조회 → 약점 관련 chunk 재검색 → 맞춤 복습 문제·요약 생성 (가장 에이전트다운 기능)
- **C. 질문 라우팅** — 자료 기반/일반 지식/애매함을 스스로 판단해 검색 여부와 모델을 결정

아직 미정: 응답 DTO 세부 필드 검증 규칙, 메인(랜딩) 화면, 문제 생성 chunk 샘플링 가중치(초기엔 완전 랜덤).

<br/>

## 참고 링크

- 전체 설계 원문: [Notion — 소스노트](https://app.notion.com/p/3e10b2c80b4681f3a4eccbc103d8a18b)

## 라이선스

[Apache License 2.0](LICENSE)
