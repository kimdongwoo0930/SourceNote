# 이메일 기준으로 소셜 계정 자동 통합

- **날짜**: 2026-09-20
- **상태**: 확정
- **관련**: [DATABASE.md](../DATABASE.md#mysql), [API.md](../API.md#인증)

## 배경

Google과 Kakao 로그인을 둘 다 지원한다. 같은 사람이 이번엔 Google, 다음엔 Kakao로 로그인하면 계정이 두 개로 갈라져 업로드한 자료가 사라진 것처럼 보인다.

## 선택지

1. provider별로 완전히 별개 계정
2. 로그인 후 사용자가 수동으로 계정 연결
3. **이메일이 같으면 자동 통합**

## 결정

3번. `User`와 `OAuthIdentity`를 분리하고, 이메일을 통합 키로 쓴다.

```
1. OAuthIdentity에서 (provider, provider_user_id)로 조회 → 있으면 그 User로 로그인
2. 없으면 User를 email로 조회
   - 있으면: 기존 User에 새 OAuthIdentity 연결 (계정 통합)
   - 없으면: User + OAuthIdentity 신규 생성
3. 매칭/생성된 User로 JWT 발급
```

## 근거

- 1번은 사용자가 "자료가 없어졌다"고 인식한다. 개인 학습 도구에서 치명적이다.
- 2번은 화면과 플로우를 새로 만들어야 하는데, 1인 프로젝트 MVP 범위에서 비용 대비 효용이 낮다.
- 신원을 `OAuthIdentity`로 분리해 두면 provider가 늘어나도 `User`는 그대로다.

## 트레이드오프

**제공사가 알려준 이메일을 신뢰한다는 가정**이 깔린다. Google·Kakao 모두 검증된 이메일을 주므로 현재 범위에서는 성립하지만, 이메일 검증을 보장하지 않는 provider를 추가한다면 이 로직을 다시 봐야 한다(그 경우 자동 통합 대상에서 제외).

## 영향

- `OAuthIdentity`에 `unique(provider, provider_user_id)` — 중복 연결 차단
- `User.email`에 unique 인덱스
- refresh token은 **해시로 DB 저장**해 로그아웃 시 즉시 무효화 가능하게 한다
- 이후 요청은 서버 자체 JWT(`Authorization: Bearer`)로 인증하며, 제공사 토큰은 로그인 시점 프로필 조회에만 쓰고 보관하지 않는다
