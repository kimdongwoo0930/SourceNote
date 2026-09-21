# 아키텍처 & 파이프라인

> **상태: 설계 기준 (2026-09-21).** 코드 미착수. 구현 후 실제 클래스·설정과 대조해 갱신합니다.

- [문서 처리 파이프라인](#문서-처리-파이프라인)
- [검색(RAG) 파이프라인](#검색rag-파이프라인)
- [사용자 요청 경로](#사용자-요청-경로)
- [서버 내부 구조](#서버-내부-구조)
- [Spring 내부 처리 흐름](#spring-내부-처리-흐름-질문하기)

---

## 문서 처리 파이프라인

PDF 업로드와 Notion 페이지 가져오기가 같은 파이프라인을 탑니다. 업로드 요청은 즉시 `Document` 레코드를 만들고 **비동기로 처리**하며, 프론트는 `GET /documents/{id}`를 폴링해 상태를 확인합니다.

```
PDF 업로드 / Notion 페이지
        │
        ▼
 ① 텍스트 추출            Apache Tika / PDFBox (페이지 단위 보존)
        │
        ▼
 ② 페이지별 품질 체크      깨진 문자 비율·텍스트 분량 검사
        │
        ├── 통과 ─────────────────────────┐
        ▼                                 │
 ③ 저품질 페이지만 비전 모델 재처리        │   ← 수식·표 대응
        │                                 │
        └─────────────┬───────────────────┘
                      ▼
 ④ 토큰 기반 슬라이딩 윈도우 청킹   ~400 토큰 / 오버랩 ~80
                      │
                      ▼
 ⑤ 메타데이터 부착 → 임베딩 → Qdrant upsert
                      │
                      ▼
            Document.status = 완료
```

### 각 단계의 근거

| 단계 | 파라미터 / 판단 | 이유 |
|---|---|---|
| ② 품질 체크 | 페이지 단위 | 전공서는 **일부 페이지만** 깨집니다. 문서 전체를 한 덩어리로 판단하면 과잉 재처리가 됩니다 |
| ③ 비전 재처리 | 미달 페이지만 | 전 페이지를 비전 모델로 돌리면 비용이 자릿수로 뜁니다 |
| ④ 청킹 | ~400 토큰, 오버랩 ~80 | 400은 문단 1~2개 단위로 의미가 끊기지 않는 크기. 오버랩 80(20%)은 경계에 걸친 문장이 양쪽 chunk 어디에서도 검색되도록 보장 |
| ④ 토큰 기준 | 문자 수 아님 | 임베딩·프롬프트 비용의 단위가 토큰이라, 청킹도 같은 단위로 맞춰야 예측이 맞습니다 |
| ⑤ 임베딩 | `text-embedding-3-small` (1536d) | 개발자 고정 키. 중복 임베딩은 텍스트 해시 비교로 스킵 → [COST_OPTIMIZATION.md](COST_OPTIMIZATION.md) |

### Notion 재동기화

Notion 문서는 원본이 계속 바뀝니다. 매번 전체를 다시 임베딩하지 않고, 저장된 `Document.notion_last_edited_time`과 Notion API의 `last_edited_time`을 비교해 **변경된 페이지만** 다시 처리합니다.

---

## 검색(RAG) 파이프라인

```
사용자 질문
    │
    ▼
① 질문 임베딩          저장 때와 동일한 모델 (필수 — 다르면 벡터 공간이 안 맞음)
    │
    ▼
② Qdrant HNSW 근사 최근접 탐색
     filter: user_id = 나  AND  subject_id = 현재 과목
     ("전체 자료 검색" 옵션이면 subject_id 필터만 제거)
    │
    ▼
③ top-5 중 코사인 유사도 >= 0.7 만 선별
    │
    ├── 남은 chunk 0개 → "근거 없음"으로 응답, LLM 호출 안 함
    ▼
④ 프롬프트 조립         근거 chunk + 출처(문서명·페이지) + "근거 밖 추측 금지" 지시
    │
    ▼
⑤ LLM 답변 생성         사용자가 설정에서 고른 provider·key·model
    │
    ▼
답변 + 출처 목록(JSON) / LlmUsageLog 기록
```

**threshold 0.7의 의미**: 벡터 검색은 관련이 없어도 "가장 가까운 것"을 항상 돌려줍니다. 하한선이 없으면 자료에 없는 질문에도 엉뚱한 chunk를 근거처럼 붙여 답하게 됩니다. 0.7 미달은 근거로 인정하지 않고, 남은 chunk가 없으면 **LLM을 호출하지 않습니다**(비용도 절약).

**top-5**: 프롬프트에 들어가는 입력 토큰이 곧 비용입니다. 400토큰 chunk 5개면 약 2,000토큰으로, 답변 품질과 호출당 단가의 균형점입니다. → [COST_OPTIMIZATION.md](COST_OPTIMIZATION.md)

---

## 사용자 요청 경로

```
사용자 브라우저
    │  HTTPS (Cloudflare 인증서)
    ▼
Cloudflare  ── DNS Proxied, HTTPS 종단, 오리진 IP 은닉
    │  HTTPS (Origin Certificate, SSL 모드 Full(strict))
    ▼
Nginx  (오라클, :80 :443)  ── 라우팅만 담당
    ├── /              → frontend (Next.js)
    └── /api, /oauth2  → backend  (Spring Boot)
```

방문자↔Cloudflare 구간은 Cloudflare가 자동 처리하고, Cloudflare↔오리진 구간은 15년 유효 Origin Certificate를 씁니다(갱신 크론 불필요). 상세 → [DEPLOYMENT.md](DEPLOYMENT.md#https)

---

## 서버 내부 구조

```
┌──────────────── Docker network: internal ────────────────┐
│                                                          │
│   nginx        80:80, 443:443   ← 호스트 포트를 무는 유일한 컨테이너
│     │                                                    │
│     ├──→ frontend   (포트 미노출)                         │
│     └──→ backend    (포트 미노출)                         │
│              │                                           │
│              ├──→ mysql    (포트 미노출)                  │
│              ├──→ qdrant   (포트 미노출)                  │
│              └──→ redis    (포트 미노출)                  │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

- **Nginx만 외부에 노출**됩니다. MySQL·Qdrant·Redis는 호스트 포트를 열지 않으므로, 방화벽에서 따로 막을 대상 자체가 없습니다.
- **Backend가 데이터 저장소에 접근하는 유일한 통로**입니다. 프론트엔드는 어떤 경우에도 MySQL·Qdrant를 직접 조회하지 않습니다.
- Qdrant는 인증 없이 동작하지만 내부망 전용이라 외부에서 도달할 수 없습니다.

---

## Spring 내부 처리 흐름 (질문하기)

```
POST /subjects/{id}/questions
    │
    ▼
Controller          REST 매핑, DTO 검증, 인증 주체(userId) 추출
    │
    ▼
Service             과목 소유권 확인 → 사용자 Settings 조회(provider·key·model)
    │
    ▼
RetrievalAugmentationAdvisor      Spring AI 어드바이저
    │   ├── 질문 임베딩
    │   ├── Qdrant 검색 (user_id·subject_id 필터, top-5, threshold 0.7)
    │   └── 근거 chunk를 프롬프트에 주입
    ▼
ChatClient          사용자 설정으로 ChatModel을 요청 시점에 생성
    │
    ▼
응답 반환 (답변 + 출처) + LlmUsageLog 기록
```

### ChatModel을 고정 빈으로 두지 않는 이유

provider·API 키·모델이 **사용자마다 다르므로** 애플리케이션 시작 시점에 빈으로 고정할 수 없습니다. 요청마다 해당 사용자의 설정으로 생성합니다.

```java
ChatModel chatModel = switch (settings.provider()) {
    case ANTHROPIC -> AnthropicChatModel.builder()
            .anthropicApi(new AnthropicApi(settings.apiKey()))
            .defaultOptions(AnthropicChatOptions.builder().model(settings.model()).build())
            .build();
    case OPENAI -> OpenAiChatModel.builder()
            .openAiApi(new OpenAiApi(settings.apiKey()))
            .defaultOptions(OpenAiChatOptions.builder().model(settings.model()).build())
            .build();
};
```

임베딩 모델은 반대로 **서버 고정 키 하나**이므로 일반 빈으로 둡니다. 저장과 검색이 반드시 같은 모델이어야 하기 때문에 사용자가 바꿀 수 있어서도 안 됩니다.

---

## 관련 문서

- [DATABASE.md](DATABASE.md) — 스키마와 Qdrant payload 구조
- [API.md](API.md) — 엔드포인트 목록
- [DEPLOYMENT.md](DEPLOYMENT.md) — 서버 구성·CI/CD
- [COST_OPTIMIZATION.md](COST_OPTIMIZATION.md) — 캐싱과 비용
- [decisions/](decisions/) — 결정 기록
