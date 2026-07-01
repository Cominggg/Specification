# 인증 및 토큰 정책

| 항목 | 내용 |
|------|------|
| Access Token 유효시간 | 30분. HTTP `Authorization: Bearer {token}` 헤더로 전달. |
| Refresh Token 유효시간 | 7일. HttpOnly Cookie로 저장 (XSS 방어). |
| 자동 재발급 | Axios interceptor에서 401 응답 감지 → `POST /api/auth/refresh` 호출. 성공 시 원래 요청 재시도. 실패 시 로그인 페이지 이동. |
| 로그아웃 | `POST /api/auth/logout` → 서버 Refresh Token 블랙리스트 등록 + 클라이언트 Cookie 삭제. |
| 비로그인 접근 | 인증 필요 페이지 접근 시 로그인 모달 표시. 로그인 완료 후 원래 URL로 redirect. React Router의 PrivateRoute 컴포넌트로 구현. |
| 소셜 로그인 플로우 | `GET /api/auth/login/{provider}` → OAuth 리다이렉트 → `GET /api/auth/callback/{provider}` → Refresh Token 쿠키 설정 → `isNewUser` 파라미터로 분기. provider: `google`\|`kakao` |

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
