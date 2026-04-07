# 인증 및 토큰 정책

| 항목 | 내용 |
|------|------|
| Access Token 유효시간 | 30분. HTTP `Authorization: Bearer {token}` 헤더로 전달. |
| Refresh Token 유효시간 | 7일. HttpOnly Cookie로 저장 (XSS 방어). |
| 자동 재발급 | Axios interceptor에서 401 응답 감지 → `POST /api/auth/refresh` 호출. 성공 시 원래 요청 재시도. 실패 시 로그인 페이지 이동. |
| 로그아웃 | `POST /api/auth/logout` → 서버 Refresh Token 블랙리스트 등록 + 클라이언트 Cookie 삭제. |
| 비로그인 접근 | 인증 필요 페이지 접근 시 로그인 모달 표시. 로그인 완료 후 원래 URL로 redirect. React Router의 PrivateRoute 컴포넌트로 구현. |
| 소셜 로그인 플로우 | `GET /api/auth/login/{provider}` → OAuth 리다이렉트 → `GET /api/auth/callback/{provider}` → JWT 발급 → 홈으로 이동. provider: `google`\|`kakao` |
