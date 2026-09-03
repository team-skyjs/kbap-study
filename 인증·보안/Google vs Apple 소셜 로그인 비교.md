---
tags: [oauth, oidc, google, apple, 소셜로그인, 인증]
생성일: 2026-07-08
상태: 완료
---

> [!info] 관련 노트
> 🏠 [[🏠 홈]] · 📂 주제: 인증·보안
> ⬅️ 먼저 읽기: [[OAuth 2.0과 OIDC — 소셜 로그인의 원리]] (등장인물·코드 플로우·검증을 모른다면 거기부터)
> 🔗 같은 원리의 일반화: [[S3 presigned URL과 클라이언트 시크릿]] — "client secret 서명은 서버에서"가 파일 업로드(AWS 키)에서도 똑같이 반복된다

# Google vs Apple 소셜 로그인 비교

> 맥락: 둘 다 OIDC인데 실제로 붙여보면 애플 쪽에서 예상 밖의 함정이 많다.
> 목표: 공통 골격 위에서 **뭐가 다르고, 애플의 함정 3개가 뭔지** 미리 알고 시작하기.

---

## 0. 한 줄 요약
**둘 다 표준 OIDC라 큰 흐름(코드 → 토큰 → ID 토큰 검증)은 같다.** 차이는 디테일에 있다 — 구글은 "OIDC 모범생"이라 문서대로 하면 되고, 애플은 **①이름·이메일을 딱 첫 동의 때만 주고 ②client_secret을 직접 서명해 만들어야 하며 ③가짜 릴레이 이메일**이 올 수 있다. 이 셋을 모르고 시작하면 반드시 한 번씩 데인다.

---

## 1. 공통 골격 (둘 다 이렇다)

[[OAuth 2.0과 OIDC — 소셜 로그인의 원리|개념 노트]]의 인가 코드 플로우 그대로:
```
authorize URL로 보냄 → 사용자 동의 → code 수신 → 뒷채널에서 토큰 교환
→ id_token 5체크(서명·iss·aud·exp·nonce) → (iss, sub)로 유저 식별 → 세션 발급
```
- 둘 다 JWKS 공개: 구글 `googleapis.com/oauth2/v3/certs` · 애플 `appleid.apple.com/auth/keys`
- 둘 다 `sub`가 유저 불변 ID (이메일로 식별하지 말 것 — 특히 애플은 릴레이 이메일 때문에 더더욱)

---

## 2. 핵심 비교표

| | **Google** | **Apple** |
|---|---|---|
| 프로토콜 | 표준 OIDC (discovery 문서 제공) | OIDC 기반이나 **비표준 디테일 많음** |
| `iss` | `https://accounts.google.com` | `https://appleid.apple.com` |
| 이메일 | 항상 옴 (`email_verified` 포함) | 옴 — 단 **가짜 릴레이 주소일 수 있음** |
| **이름** | 항상 옴 (`name` 클레임) | **첫 동의 때 딱 1번**, 그것도 토큰이 아닌 별도 `user` JSON으로 |
| client_secret | 콘솔에서 발급받는 고정 문자열 | **없음 — 내가 p8 키로 JWT를 서명해서 만듦** (최대 6개월 유효) |
| 리프레시 토큰 | 표준 (첫 동의 시 발급) | 발급되지만 용도 제한적 (토큰 갱신 검증용) |
| 계정 연동 해제 감지 | 토큰 폐기 API | 서버 알림(Server-to-Server Notification) 별도 구독 |
| 개발자 등록 | 무료 (Google Cloud Console) | **유료** (Apple Developer $99/년) |

---

## 3. Google — 모범생이라 특이사항이 적다

- **Discovery 문서**(`/.well-known/openid-configuration`)가 있어 엔드포인트·지원 기능을 기계가 자동으로 읽음. 라이브러리가 알아서 다 함.
- ID 토큰 검증용 공식 라이브러리 제공 (직접 JWKS 파싱할 필요 없음).
- `email_verified: true` 확인 습관 — 구글 워크스페이스 등 일부 케이스에서 미검증 이메일 존재 가능.
- 참고: 웹에선 "One Tap" 같은 간편 UI도 제공하지만, 뒤에서 도는 건 결국 같은 OIDC.

---

## 4. Apple — 함정 3개 (여기서 다 데인다)

### 함정 ① 이름·이메일은 "첫 동의 때 딱 한 번"
- 사용자가 **처음 동의하는 순간에만** 이름(별도 `user` JSON)과 이메일을 줌. **두 번째 로그인부터는 안 줌.**
- 이때 저장에 실패하면(서버 에러, 트랜잭션 롤백…) **그 유저의 이름을 영영 모른다.**
- 복구법: 사용자가 설정 → Apple 로그인 → 해당 앱 연동 해제 후 재동의해야만 다시 옴 (사용자에게 시킬 수 없는 UX).
> **교훈: 첫 응답의 이름·이메일은 무조건, 즉시, 안전하게 저장부터.** 가입 로직 실패와 분리해서라도.

> [!example] K-Bap 실전 (2026-07-09 확인)
> `kbap-fe/src/lib/auth/credentials.ts`가 정확히 이 함정을 코드로 대비 중 — 타입 주석에 "Apple sends fullName/email **only on the very first authorization** — forward them to the BE on first sign-in **or they're gone for good**"이라고 박아두고, 크리덴셜을 SecureStore에 파킹해 BE 계약 확정 전까지 보존한다. 저장 위치 판단은 [[클라이언트 저장소 — AsyncStorage·SecureStore·메모리 (K-Bap 실전)|저장소 노트]] 참고.

### 함정 ② client_secret이 "고정 문자열"이 아니다
- 구글은 콘솔에서 secret을 복사하면 끝. 애플은 **개발자가 직접 만든다**:
```
.p8 개인키 (Apple 콘솔에서 1회 다운로드, 재다운 불가!)
  → 이 키로 ES256 서명한 JWT를 만듦 = 이게 client_secret
  → 클레임: iss=Team ID, sub=클라이언트 ID, aud=https://appleid.apple.com
  → 만료 최대 6개월 → 주기적 재생성 필요 (만료되면 어느 날 갑자기 로그인 전멸)
```
> **교훈: secret 만료 6개월 — 자동 재생성 or 만료 알림을 시스템에 박아둘 것.** p8 파일은 안전한 곳에 보관(재다운 불가).

### 함정 ③ Private Email Relay (가짜 이메일)
- 동의 화면에서 사용자가 "이메일 숨기기"를 고르면 `xxxxx@privaterelay.appleid.com` 같은 **릴레이 주소**가 옴.
- 실제 메일은 전달되지만: 내 도메인을 애플에 등록해야 발송 가능, 사용자가 릴레이를 끊으면 연락 두절.
- **이메일로 기존 계정과 매칭하는 로직이 있다면 여기서 깨진다** (같은 사람인데 이메일이 다름) → 역시 `(iss, sub)` 식별이 답.

### 그 외 알아둘 것
- **scope 요청 시 응답이 `form_post`**: 결과가 GET 쿼리가 아니라 **POST body**로 옴 → 콜백 엔드포인트가 POST를 받아야 하고, 크로스사이트 POST라 쿠키가 안 실릴 수 있음(state를 쿠키에만 의존하면 깨짐).
- **App Store 심사 규정 4.8**: iOS 앱이 구글 등 서드파티 로그인을 제공하면 **비슷한 프라이버시 수준의 로그인 옵션(사실상 Apple 로그인)을 함께 제공해야** 심사 통과. "구글만" 넣으면 리젝.

---

## 5. 실무 결론 (설계에 반영할 것)

1. 유저 테이블: `provider(iss) + provider_user_id(sub)` 복합 식별. 이메일은 참고 정보.
2. 애플 첫 응답의 이름·이메일 → **최우선 저장** (실패 시 재시도 큐라도).
3. 애플 client_secret **6개월 만료 캘린더/자동화** 필수.
4. 한 사람이 구글·애플 둘 다로 가입하는 **계정 통합** 정책 미리 결정 (이메일 매칭? 수동 연동?) — 애플 릴레이 이메일 때문에 자동 매칭은 불완전.
5. iOS 출시 예정이면 처음부터 Apple 로그인 포함 설계 (심사 4.8).

---

## 6. 용어집
- **Private Email Relay**: 애플이 실주소를 숨기고 발급하는 전달용 주소 (`@privaterelay.appleid.com`).
- **p8 키**: 애플이 주는 PKCS#8 개인키 파일. client_secret JWT 서명용. 1회만 다운로드 가능.
- **ES256**: 애플 client_secret 서명에 쓰는 타원곡선 서명 알고리즘.
- **form_post**: 인가 응답을 GET 리다이렉트가 아닌 POST body로 보내는 방식.
- **Team ID / Services ID**: 애플 개발자 계정 식별자 / 웹용 클라이언트 ID.
- **심사 규정 4.8 (Login Services)**: 서드파티 로그인 제공 시 프라이버시 보호 옵션 동반 의무.

---

## 7. 다음에 빠른 재현 체크리스트
1. 둘 다 OIDC — 흐름·검증은 [[OAuth 2.0과 OIDC — 소셜 로그인의 원리|개념 노트]] 그대로.
2. 애플 3함정: **이름은 첫 회만 · secret은 서명 JWT(6개월) · 이메일은 릴레이일 수 있음.**
3. 식별은 언제나 `(iss, sub)`. 이메일 매칭 과신 금지.
4. 애플 scope 쓰면 콜백은 **POST 수신** 준비.
5. iOS + 소셜 로그인 = Apple 로그인 의무 (4.8).

---

## 8. 참고
- [Sign in with Apple — client secret 생성 (Scott Brady)](https://www.scottbrady.io/openid-connect/implementing-sign-in-with-apple-in-aspnet-core)
- [p8로 client_secret 만들기](https://karser.dev/how-to-generate-client-secret-for-apple-sign-in-from-p8-certificate/)
- [Curity — Sign in with Apple 동작 정리](https://curity.io/docs/idsvr/latest/authentication-service-admin-guide/authenticators/sign-in-with-apple.html)
- Google: [OpenID Connect 문서](https://developers.google.com/identity/openid-connect/openid-connect)

---

> [!note] kbap-server 실제 구현 — 릴레이 이메일·(iss, sub) 식별을 코드에서 어떻게 다루나 (2026-07-15 추가)
> 함정 ③(Private Email Relay)과 §5의 `(iss, sub)` 식별 원칙이 kbap 백엔드 로그인에 그대로 나타난다: 서버는 Firebase claims에서 **원본 provider uid**(구글·애플이 발급한 식별자, Firebase 내부 uid가 **아님**)를 꺼내 회원을 식별하고, 이메일은 nullable로 둔다(애플이 이메일을 안 주거나 릴레이 주소일 수 있으므로). 실코드 → [[16. 소셜 로그인 흐름 — Firebase 토큰이 자체 JWT가 되기까지]]

> [!note] "구글은 모범생"의 예외 — Android 클라이언트 등록 (2026-07-20 추가)
> §3의 "문서대로 하면 된다"는 **서버(뒷채널) 쪽** 얘기다. Android **앱** 쪽은 (패키지명+서명 SHA-1)을 구글 서버가 대조하는 별도 등록 모델이 있어, Android OAuth 클라이언트가 없으면 같은 코드가 iOS만 되고 Android는 `DEVELOPER_ERROR(code 10)`로 떨어진다. K-Bap 실사고와 신뢰 체인 → [[Android 구글 로그인 — SHA-1·OAuth 클라이언트·DEVELOPER_ERROR]]

> [!note] §5-1 의 "복합 식별" 원칙이 DDL 과 테스트로 못 박힌 자리 (2026-08-27 추가)
> 위 2026-07-15 콜아웃이 원리로 적어 둔 것이 kbap 스키마에 그대로 서 있다 — `UNIQUE KEY uk_member_provider_uid (provider, provider_uid)`, `email varchar(255) DEFAULT NULL`(unique 아님). 한 가지가 더 있다: 저장되는 `provider_uid` 는 **Firebase 내부 uid 가 아니라 구글·애플이 발급한 원본 sub** 이고, 그걸 픽스처 문자열 `"firebase-uid-should-not-be-stored"` 로 못 박은 테스트가 있다(`FirebaseClaimMapperTest.kt:40-52`). 나중에 Firebase 를 걷어내도 회원 테이블이 살아남는 이유다 → [[16. 소셜 로그인 흐름 — Firebase 토큰이 자체 JWT가 되기까지]] §4-4
