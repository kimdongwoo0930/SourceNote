# 데이터베이스

> **상태: 설계 기준 (2026-09-21).** 실제 DDL·마이그레이션은 아직 없습니다. 아래는 확정된 논리 스키마입니다.

저장소는 두 개로 나뉩니다.

| 저장소 | 담당 |
|---|---|
| **MySQL** | 사용자·인증·과목·문서 메타데이터·문제·사용량 로그 |
| **Qdrant** | chunk 벡터와 검색용 payload |

원문 chunk 텍스트와 벡터는 Qdrant에만, 그 외 모든 관계형 데이터는 MySQL에만 둡니다. 두 저장소는 `document_id`로 이어집니다.

---

## MySQL

```sql
User (id, email, name, created_at)

OAuthIdentity (id, user_id, provider, provider_user_id, created_at)
-- unique(provider, provider_user_id)

NotionConnection (id, user_id, access_token, workspace_name, connected_at)

Subject (id, user_id, name, created_at)

Document (id, user_id, subject_id, source_type, title,
          notion_page_id, notion_last_edited_time, file_path,
          status, ingested_at)

Question (id, user_id, subject_id, question_text, choices JSON,
          answer_index, explanation, source_chunk_ids JSON, created_at)

QuizAttempt (id, user_id, question_id, selected_index, is_correct, attempted_at)

Settings (id, user_id, llm_provider, llm_api_key, llm_model, updated_at)

LlmUsageLog (id, user_id, subject_id, operation_type, model_name,
             input_tokens, output_tokens, estimated_cost,
             cache_hit, cache_type, created_at)
```

### 관계

```
User 1 ──── N OAuthIdentity        (Google·Kakao 여러 개가 한 계정에 붙음)
User 1 ──── 1 NotionConnection
User 1 ──── 1 Settings
User 1 ──── N Subject
              └── N Document       (PDF | Notion)
              └── N Question ── N QuizAttempt
User 1 ──── N LlmUsageLog
```

### 테이블별 설계 의도

| 테이블 | 핵심 |
|---|---|
| **User** | 계정의 단일 주체. 이메일이 통합 키 역할을 합니다 |
| **OAuthIdentity** | provider별 신원을 `User`에서 분리했습니다. 덕분에 같은 사람이 Google·Kakao 양쪽으로 로그인해도 계정이 하나로 유지됩니다. `unique(provider, provider_user_id)`가 중복 연결을 막습니다 |
| **NotionConnection** | `access_token`은 **암호화 저장**. 사용자가 연결을 해제하면 행을 삭제합니다 |
| **Document** | `source_type`으로 PDF/Notion을 구분합니다. Notion만 `notion_page_id`·`notion_last_edited_time`을 쓰고(변경분 재동기화 판단), PDF만 `file_path`를 씁니다. `status`는 비동기 처리 상태로 프론트가 폴링하는 대상입니다 |
| **Question** | 생성된 **객관식 문제**입니다. `choices`·`source_chunk_ids`는 JSON. `source_chunk_ids`를 남기는 이유는 Phase 2의 자체검증 에이전트가 원본 chunk와 대조해야 하기 때문입니다 |
| **QuizAttempt** | 채점 결과 이력. Phase 2 약점 분석의 입력 데이터입니다. 과목별 오답률은 `question_id`로 `Question`을 조인해 구합니다 |
| **Settings** | 사용자당 1행. `llm_api_key`는 **암호화 저장**하고 조회 응답에서는 마스킹합니다 |
| **LlmUsageLog** | 비용의 단일 진실 원천. `cache_hit`·`cache_type`을 같이 남겨 캐싱 전후 절감액을 수치로 뽑습니다 → [COST_OPTIMIZATION.md](COST_OPTIMIZATION.md) |

### 인덱스 (구현 시 기준)

| 테이블 | 인덱스 | 용도 |
|---|---|---|
| `User` | unique(`email`) | 계정 통합 매칭 |
| `OAuthIdentity` | unique(`provider`, `provider_user_id`) | 로그인 조회 |
| `Document` | (`subject_id`), (`user_id`, `notion_page_id`) | 목록 조회, 재동기화 판단 |
| `Question` | (`subject_id`) | 과목 문제 목록 |
| `QuizAttempt` | (`user_id`, `attempted_at`), (`question_id`) | 오답률 집계 |
| `LlmUsageLog` | (`user_id`, `created_at`), (`user_id`, `subject_id`), (`operation_type`) | 대시보드 집계 |

### 네이밍 주의

`Question` 테이블은 **퀴즈 문제**입니다. RAG 질의응답의 "질문"이 아닙니다(엔드포인트는 `POST /subjects/{id}/questions`로 이름이 겹칩니다). 또한 **RAG 질문·답변 이력을 저장하는 테이블은 현재 설계에 없습니다** — 질문하기의 흔적은 `LlmUsageLog`에만 남습니다.

---

## Qdrant

```
Collection: study_chunks        (단일 컬렉션, payload로 필터링)

vector:    1536차원 (text-embedding-3-small)
distance:  Cosine
index:     HNSW

payload: {
  user_id,       # 필수 필터 — 사용자 격리
  subject_id,    # 과목 범위 필터 ("전체 자료 검색"이면 제거)
  document_id,   # MySQL Document와 연결, 문서 삭제 시 일괄 삭제 키
  page_number,   # 출처 표시용
  chunk_index,   # 문서 내 순서
  ingested_at
}
```

검색은 `user_id` + `subject_id`로 필터링해 범위를 좁히고, **top-5 / 코사인 유사도 0.7 이상**만 근거로 채택합니다. 미달이면 "근거 없음"으로 처리하고 LLM을 호출하지 않습니다.

### 왜 컬렉션을 나누지 않았는가

사용자별 또는 과목별로 컬렉션을 쪼개는 방식을 먼저 검토했고, 단일 컬렉션으로 결정했습니다.

**1. 컬렉션 수가 곱으로 늘어납니다.** 과목별 컬렉션이면 컬렉션 수 = `사용자 × 과목`입니다. Qdrant는 컬렉션마다 별도 HNSW 인덱스와 세그먼트를 유지하므로, 작은 컬렉션이 수백 개면 벡터 총량은 그대로인데 인덱스 오버헤드와 메모리만 늘어납니다. 릴리즈 서버가 2 OCPU·12GB라 이 낭비를 감당할 여유가 없습니다.

**2. "전체 자료 검색"이 요구사항에 있습니다.** 단일 컬렉션이면 `subject_id` 필터를 빼는 것으로 끝나지만, 과목별 컬렉션이면 N개 컬렉션에 병렬 질의하고 결과를 직접 병합·재정렬해야 합니다.

**3. 컬렉션 라이프사이클을 애플리케이션이 관리해야 합니다.** 과목 생성마다 컬렉션 생성, 삭제마다 컬렉션 삭제가 붙습니다. MySQL 트랜잭션과 원자적으로 묶이지 않으므로 중간 실패 시 "과목은 있는데 컬렉션이 없는" 정합성 문제가 생깁니다. 단일 컬렉션에서는 과목 삭제가 `subject_id` 필터 삭제 한 번입니다.

**4. HNSW 필터 검색으로 성능이 충분합니다.** `user_id`·`subject_id`에 payload 인덱스를 걸면 필터가 인덱스 탐색에 반영됩니다. 개인 학습 자료 규모(사용자당 문서 수십 개, chunk 수만 개)에서 컬렉션 분리로 얻을 성능 이득은 사실상 없습니다.

**감수한 트레이드오프**: 물리적 격리가 아니라 **필터에 의한 논리적 격리**이므로, `user_id` 필터를 빠뜨린 질의 하나가 곧 다른 사용자 데이터 노출입니다. 따라서 검색 호출을 리포지토리 레이어 한 곳으로 모으고 거기서 `user_id` 필터를 강제 주입합니다. 컨트롤러·서비스에서 필터를 직접 조립하지 않습니다.

### 삭제 정합성

| 작업 | MySQL | Qdrant |
|---|---|---|
| 문서 삭제 | `Document` 행 삭제 | `document_id` 필터로 point 삭제 |
| 과목 삭제 | `Subject` + 하위 `Document`·`Question` 삭제 | `subject_id` 필터로 point 삭제 |
| 재동기화 | `notion_last_edited_time` 갱신 | 해당 `document_id` point 삭제 후 재삽입 |

두 저장소에 걸친 삭제라 한쪽만 성공할 수 있습니다. MySQL을 먼저 지우면 Qdrant에 고아 point가 남는데, 검색 결과에는 나오지만 출처를 그릴 문서 메타데이터가 없어 화면이 깨집니다. 따라서 **Qdrant를 먼저 삭제하고 MySQL을 나중에** 삭제합니다. 반대 순서의 잔여물(고아 point)보다 이쪽 잔여물(벡터 없는 문서 행)이 사용자에게 훨씬 안전합니다.

---

## 관련 문서

- [ARCHITECTURE.md](ARCHITECTURE.md) — 벡터가 만들어지고 쓰이는 흐름
- [decisions/2026-09-20-qdrant-single-collection.md](decisions/2026-09-20-qdrant-single-collection.md) — 결정 기록
