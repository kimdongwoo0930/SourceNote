# API

<br/>

## Swagger UI

springdoc-openapi가 컨트롤러·DTO에서 OpenAPI 3 스펙을 생성합니다. 별도 문서를 손으로 유지하지 않고, 이 파일은 전체 지도 역할만 합니다.

| 환경 | Swagger UI | OpenAPI JSON |
|---|---|---|
| 로컬 | http://localhost:8080/swagger-ui.html | http://localhost:8080/v3/api-docs |
| release | `https://<도메인>/swagger-ui.html` | `https://<도메인>/v3/api-docs` |

프론트엔드는 바이브코딩으로 별도 진행되므로, **엔드포인트와 DTO를 먼저 확정하고 코드를 시작하는 API-first** 방식을 씁니다. 이 스펙이 프론트-백엔드 계약입니다.

## 인증

- 로그인은 OAuth 2.0(Google / Kakao)으로 시작하고, 서버가 **자체 JWT(access + refresh)** 를 발급합니다.
- 이후 모든 보호된 요청은 `Authorization: Bearer <access token>` 헤더를 붙입니다.
- refresh token은 해시로 DB에 저장하므로 로그아웃 시 즉시 무효화됩니다.
- 아래 표에서 **Auth 시작 엔드포인트를 제외한 전부가 인증 필요**입니다.

```http
POST /subjects/1/questions HTTP/1.1
Authorization: Bearer eyJhbGciOi...
Content-Type: application/json

{ "question": "TCP 3-way handshake 과정 설명해줘" }
```

---

## 엔드포인트

### Auth / User

| Method | Path | 설명 |
|---|---|---|
| GET | `/oauth2/authorization/{google\|kakao}` | 소셜 로그인 시작 (리다이렉트) |
| POST | `/auth/refresh` | access token 재발급 |
| POST | `/auth/logout` | refresh token 무효화 |
| GET | `/users/me` | 내 계정 정보 조회 |

이메일이 같으면 provider가 달라도 같은 계정으로 통합됩니다(계정 매칭 로직 → [decisions/2026-09-20-oauth-account-merge.md](decisions/2026-09-20-oauth-account-merge.md)).

### Subject (과목)

| Method | Path | 설명 |
|---|---|---|
| GET | `/subjects` | 내 과목 목록 |
| POST | `/subjects` | 과목 생성 |
| GET | `/subjects/{id}` | 과목 상세 |
| DELETE | `/subjects/{id}` | 과목 삭제 (문서·chunk 함께 정리) |

### Document (자료)

| Method | Path | 설명 |
|---|---|---|
| GET | `/subjects/{id}/documents` | 과목의 자료 목록 |
| POST | `/subjects/{id}/documents/pdf` | PDF 업로드 (`multipart/form-data`) — **비동기 처리 시작** |
| GET | `/notion/pages?search=` | Notion 페이지 검색 (가져오기 모달용) |
| POST | `/subjects/{id}/documents/notion` | 선택한 Notion 페이지 가져오기 (다중) |
| GET | `/documents/{id}` | 처리 상태 조회 — 업로드 후 **폴링** |
| POST | `/documents/{id}/resync` | 재동기화 (Notion: 변경분만) |
| DELETE | `/documents/{id}` | 문서 삭제 + Qdrant chunk 삭제 |

업로드는 즉시 `Document`를 만들고 202 성격의 응답을 돌려준 뒤 백그라운드에서 파싱·임베딩합니다. 프론트는 `GET /documents/{id}`의 `status`를 폴링합니다. → [ARCHITECTURE.md](ARCHITECTURE.md#문서-처리-파이프라인)

### 질문하기 (RAG)

| Method | Path | 설명 |
|---|---|---|
| POST | `/subjects/{id}/questions` | 질문 전송 → RAG 검색 + 답변 생성 |

- 응답에는 **답변 + 근거 출처(문서명·페이지)** 가 포함됩니다.
- 유사도 0.7 이상 chunk가 없으면 LLM을 호출하지 않고 "근거 없음"으로 응답합니다.
- "전체 자료 검색" 옵션은 `subject_id` 필터를 제거합니다.
- 사용자 Settings에 API 키가 없으면 실패합니다(BYOK).

> ⚠️ 이름 주의: 이 엔드포인트의 `questions`는 **RAG 질문**이고, DB의 `Question` 테이블은 **퀴즈 문제**입니다. 서로 다른 개념입니다 → [DATABASE.md](DATABASE.md#네이밍-주의)

### 문제풀기 (Quiz)

| Method | Path | 설명 |
|---|---|---|
| POST | `/subjects/{id}/quiz/generate` | 과목 chunk 샘플링 → 객관식 문제 생성 |
| GET | `/subjects/{id}/quiz` | 생성된 문제 목록 조회 |
| POST | `/quiz-questions/{id}/attempts` | 답안 제출 및 채점 |

### Settings

| Method | Path | 설명 |
|---|---|---|
| GET | `/settings` | 내 설정 조회 (API 키는 마스킹) |
| PUT | `/settings/api-keys` | provider·API 키·모델 저장 |
| GET | `/settings/llm/models?provider=` | 제공사별 모델 목록 (최근 10개) |

모델 목록은 서버가 **사용자가 입력한 키로** 제공사 `/v1/models`를 호출해 가져옵니다(Anthropic: `x-api-key` + `anthropic-version`, OpenAI: `Bearer`). 채팅 가능 모델만 필터링해 최신순 10개를 반환합니다. 키 유효성 검증을 겸합니다.

### Notion 연동

| Method | Path | 설명 |
|---|---|---|
| GET | `/oauth2/authorization/notion` | Notion 계정 연결 시작 (동의 화면에서 공유 페이지 선택) |
| DELETE | `/notion/connection` | 연결 해제 |

### Usage (사용량)

| Method | Path | 설명 |
|---|---|---|
| GET | `/usage/summary?range=` | 기간 총액 + 기능별·과목별 분해 |
| GET | `/usage/daily?range=` | 일별 추이 |

전부 `LlmUsageLog` 집계 쿼리로 산출합니다. → [COST_OPTIMIZATION.md](COST_OPTIMIZATION.md#사용량-대시보드)

---

## 규약

| 항목 | 규칙 |
|---|---|
| 소유권 검사 | 모든 리소스 조회·변경은 JWT의 `userId`로 소유권을 확인합니다. 남의 리소스는 404로 응답해 존재 여부를 노출하지 않습니다 |
| 벡터 검색 격리 | Qdrant 질의에는 `user_id` 필터가 **항상** 들어갑니다(리포지토리 레이어에서 강제) |
| API 키 노출 | 사용자 LLM API 키와 Notion 토큰은 **어떤 응답에도 원문으로 실리지 않습니다**(마스킹) |
| 비용 기록 | LLM·임베딩을 호출하는 모든 경로는 `LlmUsageLog`를 남깁니다(캐시 히트 포함) |
| 에러 형식 | *(미확정)* 구현 시 공통 에러 응답 형태를 확정하고 여기에 기록 |
