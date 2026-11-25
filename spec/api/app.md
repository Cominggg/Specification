# 앱 API

앱(네이티브) 전용 엔드포인트. 웹은 사용하지 않습니다.

---

## GET /api/app/version

**용도**: 플랫폼별 최소 지원 versionCode를 조회합니다. 앱은 설치된 versionCode가 이 값보다 작으면 강제 업데이트 화면을 띄우고 스토어로 보냅니다.

### 요청

인증 불필요.

**Query Parameters**

| 이름 | 타입 | 필수 | 기본값 | 설명 |
|------|------|------|--------|------|
| `platform` | String | Y | — | `ANDROID` \| `IOS` (대소문자 구분) |

### 응답

| 필드 | 타입 | 설명 |
|------|------|------|
| `minVersionCode` | int | 해당 플랫폼의 최소 지원 versionCode |

```json
{ "minVersionCode": 10000 }
```

### 에러

| 코드 | 상태 코드 | 설명 |
|------|-----------|------|
| `INVALID_INPUT` | 400 | `platform` 누락, 또는 `ANDROID`·`IOS` 외 값 (소문자 `android` 등 포함) |

### 비고

- versionCode 규칙: `MAJOR*10000 + MINOR*100 + PATCH` (예: `1.0.0` → `10000`)
- 값은 서버 설정(환경 변수 `APP_MIN_VERSION_CODE_ANDROID` 기본 `10000`, `APP_MIN_VERSION_CODE_IOS` 기본 `0`)으로 관리하며, 코드 배포 없이 변경합니다. DB·관리자 화면은 없습니다.
