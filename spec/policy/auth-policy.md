# 인증 및 토큰 정책

| 항목 | 내용 |
|------|------|
| Access Token 유효시간 | 30분. HTTP `Authorization: Bearer {token}` 헤더로 전달. 토큰 종류 claim `typ=access`인 토큰만 API 인증에 사용된다. |
| Refresh Token 유효시간 | 7일. HttpOnly Cookie로 저장 (XSS 방어). 토큰 종류 claim은 `typ=refresh`이며, `Authorization: Bearer`로 제출해도 인증되지 않는다 (`POST /api/auth/refresh`와 로그아웃 시 세션 식별에만 사용). |
| 자동 재발급 | Axios interceptor에서 401 응답 감지 → `POST /api/auth/refresh` 호출. 성공 시 원래 요청 재시도. 실패 시 로그인 페이지 이동. |
| 로그아웃 | `POST /api/auth/logout` → 현재 기기의 Access Token 블랙리스트 등록 + 현재 기기 세션의 Refresh Token 삭제 + 클라이언트 Cookie 삭제. 다른 기기 세션은 유지된다 (아래 "다중 기기 로그인 정책" 참고). |
| 비로그인 접근 | 인증 필요 페이지 접근 시 로그인 모달 표시. 로그인 완료 후 원래 URL로 redirect. React Router의 PrivateRoute 컴포넌트로 구현. |
| 소셜 로그인 플로우 | `GET /api/auth/login/{provider}` → OAuth 리다이렉트 → `GET /api/auth/callback/{provider}` → Refresh Token 쿠키 설정 → `isNewUser` 파라미터로 분기. provider: `google`\|`kakao` |

## 다중 기기 로그인 정책

> 2026-09-27 확정 (토큰 종류 구분·재사용 감지 포함).

| 항목 | 내용 |
|------|------|
| 동시 로그인 | 허용. 로그인할 때마다 기기(브라우저) 단위의 독립 세션이 생성된다. 세션은 Refresh Token의 세션 식별자(`jti`)로 구분한다. |
| 최대 세션 수 | 사용자당 5개. 6번째 로그인 시 가장 오래 전에 생성된 세션부터 제거한다. 제거된 기기는 Access Token 만료(최대 30분) 후 재발급이 거부되어 재로그인이 필요하다. |
| Refresh Token 회전 | `POST /api/auth/refresh` 호출마다 새 Refresh Token을 발급하고 기존 값은 즉시 무효화한다. 회전 시 세션 식별자는 유지된다. 검증과 교체는 원자적으로 처리한다. 이전 Refresh Token 유예 구간은 두지 않는다. |
| 재사용 감지 | 이미 회전되어 무효화된 Refresh Token이 다시 제출되면 탈취로 간주해 해당 세션을 즉시 폐기하고 `REFRESH_TOKEN_INVALID`를 반환한다. 해당 기기는 재로그인이 필요하며, 다른 기기 세션은 영향받지 않는다. 같은 Refresh Token으로 동시에 요청하는 경우도 재사용으로 판정되므로 클라이언트는 refresh 호출을 직렬화해야 한다. |
| 로그아웃 | 현재 기기의 세션만 종료한다. Refresh Token Cookie가 없거나 유효하지 않아도 로그아웃은 성공한다(해당 세션은 TTL 만료로 정리). |
| 회원 탈퇴 | 해당 사용자의 모든 세션을 삭제한다. |
| 전체 기기 로그아웃 | 제공하지 않는다. |

## PENDING 역할 정책

소셜 로그인 결과에 따라 아래와 같이 분기한다.

| 상황 | 처리 |
|------|------|
| 신규 소셜 로그인 | PENDING 역할 발급. `isNewUser=true` 로 FE 리다이렉트. |
| 탈퇴(INACTIVE) 후 재로그인 | 계정 재활성화(ACTIVE) 후 PENDING 역할 재발급. 기존 닉네임·가입 정보 초기화. `isNewUser=true` 로 FE 리다이렉트. |
| 기존 사용자 로그인 | USER/ADMIN 역할 유지. `isNewUser=false` 로 FE 리다이렉트. |
| 정지(SUSPENDED) 계정 로그인 시도 | OAuth2 인증 단계에서 차단. `USER_SUSPENDED` 에러 발생. 로그인 불가. |

PENDING 역할 사용자는 `POST /api/auth/register` (회원가입 완료) 호출 전까지 일반 기능 접근 불가.
회원가입 완료 후 USER 역할로 승격되며 USER 역할의 Access Token을 즉시 발급한다.
