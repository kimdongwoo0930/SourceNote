# Notion 연동에 Public Integration OAuth 사용

- **날짜**: 2026-09-20
- **상태**: 확정
- **관련**: [API.md](../API.md#notion-연동)

## 배경

사용자의 Notion 페이지를 자료로 가져와야 한다. Notion API는 Internal Integration(토큰 직접 발급)과 Public Integration(OAuth) 두 방식을 제공한다.

## 선택지

1. Internal Integration — 사용자가 Notion에서 통합을 만들고 토큰을 복사해 붙여넣고, 쓰고 싶은 페이지마다 공유 설정
2. **Public Integration OAuth** — 앱이 인가 URL로 보내고 사용자는 동의 화면에서 공유할 페이지를 선택

## 결정

2번. Notion OAuth Public Integration으로 계정을 연결한다.

## 근거

- **동의 화면이 그대로 페이지 선택 화면 역할을 한다.** Notion 인가 화면에서 사용자가 공유할 워크스페이스·페이지를 직접 고르므로, 앱이 따로 "권한 주는 방법" 안내를 만들 필요가 없다.
- 1번은 사용자가 Notion 설정을 여러 단계 돌아야 하고, 토큰을 직접 다루게 된다.
- 로그인(Google·Kakao)과 외부 연동(Notion)을 **OAuth 2.0 한 가지 방식으로 통일**할 수 있다. 토큰 교환·저장·해제 흐름이 같아진다.

## 트레이드오프

Spring Security의 기본 OAuth 클라이언트 프리셋에 Notion이 없다. `application.yml`에 `authorization-uri`·`token-uri`를 직접 등록해야 하고, 토큰 교환 시 **Basic 인증 헤더와 `Notion-Version` 헤더**가 필요해 표준 흐름에서 벗어나는 처리가 붙는다.

## 영향

- `NotionConnection`에 `access_token`을 **암호화 저장**, `workspace_name`을 함께 저장해 연결 상태를 화면에 표시
- 연결 해제(`DELETE /notion/connection`)는 행 삭제로 처리
- 가져오기는 2단계: `GET /notion/pages?search=`로 목록을 보여주고 → `POST /subjects/{id}/documents/notion`으로 다중 선택 수집
- 재동기화는 `last_edited_time` 비교로 변경분만 처리
