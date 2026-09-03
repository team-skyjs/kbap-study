---
tags: [android, google-signin, oauth, firebase, eas, 코드서명, react-native, 트러블슈팅]
생성일: 2026-07-20
상태: 완료
---

> [!info] 관련 노트
> 🏠 [[🏠 홈]] · 📂 주제: Expo & React Native
> ⬅️ 먼저 읽기: [[OAuth 2.0과 OIDC — 소셜 로그인의 원리]] (idToken·accessToken이 뭔지부터) · [[Google vs Apple 소셜 로그인 비교]] (구글 로그인의 서버 쪽 골격)
> 🔗 연결: [[로컬 빌드 vs EAS 클라우드 빌드 — K-Bap vs winemine]] (코드서명 — 이 노트에선 서명이 '설치 허가'를 넘어 '로그인 신원'이 된다) · [[16. 소셜 로그인 흐름 — Firebase 토큰이 자체 JWT가 되기까지]] (idToken이 BE에 도착한 이후 구간) · [[EAS Dev Build vs Expo Go]] (JS-only 수정을 OTA로 내보낸 경로)

# Android 구글 로그인 — SHA-1·OAuth 클라이언트·DEVELOPER_ERROR

> 작성: 2026-07-20 · 맥락: K-Bap 첫 Android 빌드(EAS preview apk) 스모크 QA에서 **구글 로그인만** 실패. 같은 코드가 iOS(TestFlight)에선 멀쩡. 원인이 **두 겹**이라 한 번 고치고도 또 막혔다.
> 목표: "iOS는 되는데 Android만 안 됨"이 **어느 검증 단계에서 왜 갈라지는지**를 신뢰 체인으로 이해하고, 다음에 같은 증상이 오면 표만 보고 바로 좁히기.

---

## 0. 한 줄 요약

**Android 구글 로그인은 iOS에 없는 검증을 두 군데서 더 한다.** ① 구글 서버가 호출 앱의 **(패키지명 + 서명 SHA-1)** 을 등록된 Android OAuth 클라이언트와 대조 — 없으면 `code 10 (DEVELOPER_ERROR)`. ② Firebase **Android** SDK는 자격증명의 빈 문자열 accessToken을 거부(iOS SDK는 허용) — RNFB 브리지 버그와 겹쳐 `accessToken cannot be empty`. 두 벽 다 "**같은 JS 코드라도 플랫폼별 네이티브 계약은 다르다**"는 한 뿌리에서 나온다.

---

## 1. 무슨 일이 있었나 — 두 겹의 벽

| | 1차 벽 (서버측) | 2차 벽 (클라이언트측) |
|---|---|---|
| 증상 | 계정 선택 **직후** "로그인이 완료되지 않았어요" | 1차 해결 후, 버튼 누르자마자 실패 |
| 에러 | `ApiException: A non-recoverable sign in failure occurred` = **code 10 DEVELOPER_ERROR** | `[auth/unknown] Exception in HostFunction: accessToken cannot be empty` |
| 원인 | 구글 클라우드에 **Android OAuth 클라이언트(type 1)가 없음** | RNFB 브리지가 accessToken 부재를 **빈 문자열 `''`로** Android 네이티브에 전달 |
| iOS가 됐던 이유 | iOS는 SHA 대조 자체를 안 함 | iOS Firebase SDK는 빈 accessToken을 허용 |
| 해결 | Cloud Console에서 Android 클라이언트 **수동 생성** (재빌드 불필요) | `getTokens()`로 진짜 accessToken을 받아 함께 전달 (JS-only → OTA) |

같은 "iOS만 됨"이어도 **원인 지점이 완전히 달랐다** — 하나는 구글 서버의 앱 신원 대조, 하나는 기기 안 네이티브 SDK의 인자 검증.

---

## 2. 원리 ① — Android 구글 로그인 신뢰 체인 (어디서 끊기면 뭐가 나오나)

```
[앱: com.rocher.kbap + 서명 SHA-1 D2:F7:…:A3:35]
  │  ① "webClientId(웹 클라이언트)로 idToken 주세요"
  ▼
[Google Play Services (기기 내 시스템 앱)]
  │  ② 호출한 앱의 패키지명 + 서명 인증서 지문을 자동 첨부해 구글 서버로
  ▼
[구글 OAuth 서버]
  │  ③ 그 프로젝트에서 (패키지명, SHA-1)이 일치하는
  │     Android OAuth 클라이언트(type 1)를 조회
  │     ├─ 없음/불일치 ──▶ ✗ code 10 DEVELOPER_ERROR   ← 1차 벽
  │     └─ 일치 ──▶ ④ idToken 발급 (aud = webClientId)
  ▼
[앱 JS: GoogleAuthProvider.credential(idToken, accessToken)]
  │  ⑤ RNFB 브리지 → Android 네이티브 → Firebase Android SDK
  │     └─ accessToken 자리가 '' ──▶ ✗ "accessToken cannot be empty"  ← 2차 벽
  ▼
[signInWithCredential 성공 → Firebase 세션]
  │  ⑥ Firebase idToken → POST /auth/login → BE가 Admin SDK로 검증 → 자체 JWT
  ▼
[[16. 소셜 로그인 흐름 — Firebase 토큰이 자체 JWT가 되기까지]] 구간
```

> 비유: **아파트 출입.** iOS는 초대장의 이름(번들 ID)만 보고 들여보내는 곳이고, Android는 경비실(Play Services)이 **방문자의 지문(서명 SHA-1)과 동호수(패키지명)를 서버 명부(Android OAuth 클라이언트)와 대조**하는 곳이다. 명부에 내 지문이 등록 안 돼 있으면 초대장(webClientId)이 진짜여도 못 들어간다 — 그게 code 10.

### iOS와 뭐가 다른가

| | Android | iOS |
|---|---|---|
| 앱 신원 | 패키지명 + **서명 인증서 SHA-1** | 번들 ID + reversed client ID URL scheme |
| 서버측 대조 | Play Services가 **매 로그인마다** 구글 서버와 대조 | 없음 (SHA 같은 빌드별 지문 개념 자체가 없음) |
| 신원이 바뀌는 순간 | **서명 키가 바뀔 때마다** (debug↔EAS↔Play 재서명) | 사실상 없음 (번들 ID는 고정) |

iOS 번들 ID는 debug나 release나 같지만, Android는 **어떤 keystore로 서명했느냐가 곧 신원**이라 빌드 경로마다 지문이 다를 수 있다 — 여기서 온갖 함정이 나온다(§4).

### OAuth 클라이언트 타입 3형제 (google-services.json의 `client_type`)

| type | 종류 | 식별 방법 | 우리 코드가 참조하나? |
|---|---|---|---|
| 1 | Android | **패키지명 + SHA-1** 쌍 | ✗ — **존재하기만 하면 됨** (서버가 대조용으로만 씀) |
| 2 | iOS | 번들 ID | ✗ (plist 쪽) |
| 3 | Web | 클라이언트 ID 문자열 | ✅ `webClientId` — idToken의 수신자(`aud`) |

- **왜 webClientId(웹 타입)를 넘기나**: idToken의 `aud`(수신자)가 이 웹 클라이언트 ID로 찍혀야 Firebase/BE가 "우리 앱용 토큰"임을 검증할 수 있다 ([[OAuth 2.0과 OIDC — 소셜 로그인의 원리|OIDC 노트]]의 aud 체크). Android 클라이언트 ID를 넘기면 안 된다.
- **왜 json에 Android 클라이언트가 없어도 되나**: 검증은 json이 아니라 **구글 서버**가 한다. 실제로 K-Bap의 `google-services.json`엔 지금도 type 3 하나뿐이고, Android 클라이언트는 클라우드에만 생성해서 해결됐다(재빌드·json 재다운로드 불필요).

---

## 3. status code 표 — 에러 숫자만 보고 좁히기

`com.google.android.gms.common.api.ApiException`의 statusCode. (GoogleSignInStatusCodes는 CommonStatusCodes를 상속)

| code | 이름 | 뜻 | 전형적 원인 → 판별법 |
|---|---|---|---|
| **10** | **DEVELOPER_ERROR** | **설정이 잘못됨 (복구 불가)** | ① SHA-1 미등록/불일치 ② webClientId가 Web 타입이 아님 ③ Android OAuth 클라이언트 미생성 ④ 생성 직후 전파 지연 ⑤ 엉뚱한 프로젝트의 클라이언트 |
| 12500 | SIGN_IN_FAILED | 로그인 실패 (원인 불특정) | Play Services 버전·구성 문제 등 포괄 |
| 12501 | SIGN_IN_CANCELLED | 사용자가 취소 | 에러 아님 — 조용히 무시 (K-Bap 코드도 silent 처리) |
| 12502 | SIGN_IN_CURRENTLY_IN_PROGRESS | 이미 진행 중 | 중복 호출 방지 |
| 7 | NETWORK_ERROR | 네트워크 오류 | 재시도 가능 |
| 8 | INTERNAL_ERROR | 내부 오류 | 일시적일 수 있음 |
| 15 | TIMEOUT | 시간 초과 | |
| 16 | CANCELED | 작업이 취소됨 | 12501과 별개 (API 레벨 취소) |
| 17 | API_NOT_CONNECTED | API 미연결 | Play Services 버전/구성 |

> code 10의 메시지가 하필 "A non-recoverable sign in failure occurred"라 숫자를 모르면 검색이 산으로 간다. **`ApiException` + statusCode 숫자**로 읽는 습관.

### 이번 사건의 소거법 (code 10 원인 ①~⑤ 중 뭐였나)

1. 설치된 apk의 **실제 서명 SHA-1을 직접 추출**(§6) → Firebase 등록값과 일치 → ① 배제
2. webClientId는 type 3 확인 + **iOS가 같은 ID로 동작** → ② 배제 (웹 클라이언트 유효의 방증)
3. Cloud Console 사용자 인증 정보에 **iOS·Web만 있고 Android 유형이 없음** → **③ 확정**
4. 함정: 처음 연 프로젝트가 이름만 비슷한 빈 "kbap"이었다. 프로젝트는 **표시 이름이 아니라 번호(44799256321)로 식별** — "iOS가 되는 이상 웹 클라이언트는 반드시 어딘가 있다"는 역추론으로 "잘못된 프로젝트를 보고 있음"을 잡아냄.

---

## 4. 원리 ② — SHA-1은 어떻게 등록되고, 어디서 어긋나나

### Firebase 콘솔 ↔ GCP 자동 생성

- Firebase 콘솔에 Android 앱을 추가하거나 SHA 지문을 등록하면, 링크된 GCP 프로젝트에 **Android OAuth 클라이언트가 자동 생성**되는 게 정상 동작.
- **자동 생성이 안 되는 알려진 경우**: 같은 (패키지명+SHA) 조합이 **다른 프로젝트에 이미 존재**하면 "An OAuth2 client already exists" 충돌로 실패. 삭제한 Firebase 프로젝트의 OAuth 클라이언트가 GCP에 잔존해 조용히 충돌하기도 한다. K-Bap처럼 **SHA 없이 앱을 선등록한 뒤 나중에 SHA만 추가**한 경우에도 자동 생성이 누락될 수 있다(실측).
- 등록/생성 직후엔 **전파 지연이 몇 분** 있을 수 있다(공식 FAQ: "allow a few minutes"). 생성했는데도 code 10이면 일단 몇 분 기다렸다 재시도.
- 자동 생성이 안 되면 **수동 생성**: GCP Console → API 및 서비스 → 사용자 인증 정보 → OAuth 클라이언트 ID 만들기 → Android → 패키지명 + SHA-1. (이번 해결책)

### 서명 키가 바뀌는 지점마다 SHA-1이 바뀐다

| 빌드 경로 | 서명하는 키 | Firebase에 등록할 SHA |
|---|---|---|
| 로컬 debug 빌드 | `~/.android/debug.keystore` | debug keystore의 SHA |
| **EAS 빌드 (지금 K-Bap)** | **EAS 관리 keystore** | EAS keystore의 SHA (`eas credentials`로 확인) |
| Play 스토어 출시 | **구글의 앱 서명 키로 재서명** (Play App Signing) | **앱 서명 키 SHA를 추가 등록** ← 최다 함정 |

> [!warning] Play 스토어 출시 때 재발 예약된 함정
> Play App Signing을 쓰면 업로드한 apk/aab를 **구글이 자기 키로 다시 서명**한다. 사용자 기기에 설치되는 건 그 재서명본이므로, **로컬·내부테스트는 되는데 스토어 배포본만 구글 로그인이 깨지는** 전형적 증상이 난다. Play Console → 앱 무결성(App integrity)에서 **앱 서명 키 SHA-1/SHA-256을 확인해 Firebase에 추가 등록**해야 한다. 지금은 EAS keystore 직배포라 무관하지만 출시 경로에선 반드시 만난다.

---

## 5. 원리 ③ — 2차 벽: "accessToken cannot be empty"의 진짜 인과

브리프 시점 해석은 "Android 네이티브는 accessToken도 요구한다"였는데, 리서치로 소스를 추적하니 **더 정확한 그림**이 나왔다. 요구한 것이 아니라 **'없음'이 '빈 문자열'로 바꿔치기된 것**이다.

### 검증된 인과 사슬 (설치된 v25.1.0 소스 실측)

```
① JS: GoogleAuthProvider.credential(idToken)          ← 스펙상 idToken만으로 OK
     (idToken·accessToken 둘 다 null일 때만 throw)
② RNFB 브리지 매핑: secret = rawNonce ?? secret ?? accessToken ?? ''
     → accessToken이 없으면 null이 아니라 ''(빈 문자열)로   ← 🐛 여기가 버그
     (lib/credentials/OAuthCredential.ts — v8991 TS 리팩토링에서 유입)
③ Android 네이티브: GoogleAuthProvider.getCredential(idToken, "")
     → 받은 걸 그대로 Firebase Android SDK에 전달
④ Firebase Android SDK: 빈 문자열 accessToken 거부
     → IllegalArgumentException "accessToken cannot be empty"
     → JS엔 [auth/unknown] Exception in HostFunction 으로 표면화
⑤ iOS: 같은 ''가 가도 FIRGoogleAuthProvider가 허용 → 통과   ← "iOS만 됨"의 정체
```

- **Firebase 검증 자체는 idToken만으로 충분**하다 — idToken이 신원 증명(JWT, `aud` 검증)이고 accessToken은 구글 API 접근용 발렛 키라는 [[OAuth 2.0과 OIDC — 소셜 로그인의 원리|OIDC 원칙]] 그대로. 문제는 토큰의 의미가 아니라 **브리지의 null 처리**였다.
- upstream도 이걸 버그로 인정: react-native-firebase **issue #9081** → 수정 커밋 `d800caf8`이 Android 모듈에서 **빈 문자열을 null로 변환**하도록 고침 ("Firebase Android rejects empty tokens; id-token-only … flows require null"). 즉 최신 버전에선 idToken-only도 Android에서 동작한다.
- **K-Bap의 수정(`getTokens()`로 진짜 accessToken을 받아 둘 다 전달)은 유효한 우회책** — 빈 문자열 대신 실제 값이 가므로 ④를 통과하고, iOS에도 무해하다. 라이브러리를 수정 버전으로 올리면 accessToken 없이도 되지만, 지금 코드를 되돌릴 이유는 없다. (`useSocialAuth.ts:72-73`에 반영 확인)

> 비유: "없음"을 전달해야 하는데 브리지가 **빈 봉투**를 만들어 건넸다. iOS 경비는 빈 봉투를 무시하고 통과시키는데, Android 경비는 "봉투가 왜 비었죠?"라며 퇴짜 놓는다. 교훈은 봉투(accessToken)가 필수라는 게 아니라, **다리(브리지)를 건너는 순간 null과 ''의 구분이 플랫폼마다 다르게 취급된다**는 것.

> [!tip] "RN 크로스플랫폼"의 실체
> 1차 벽(구글 서버의 SHA 대조)도 2차 벽(빈 문자열 거부)도 JS 코드에는 보이지 않는다. 같은 JS가 **플랫폼별 네이티브 SDK의 계약**(무엇을 검증하고 무엇을 거부하는가)을 만나 갈라진다. "iOS는 되는데 Android만 안 됨"을 만나면 한 번 고치고 끝이 아니라, **Android가 추가로 하는 검증마다 재발할 수 있다**고 가정하고 단계별로 좁히자.

---

## 6. 진단 절차 — 다음에 code 10을 만나면

1. **로그로 code 확정**: adb logcat에서 `ApiException` 찾기 (§7 치트시트).
2. **설치본의 실제 SHA-1 추출** → Firebase/GCP 등록값과 대조:
   - `apksigner verify --print-certs` 가 정석.
   - `keytool -printcert -jarfile`은 **v2-only 서명 apk에서 실패**한다 — v2/v3 서명은 인증서를 META-INF 파일이 아니라 ZIP Central Directory 앞의 **APK Signing Block**(매직 `APK Sig Block 42`, v2 ID `0x7109871a`) 안에 넣기 때문. keytool은 v1(JAR 서명)만 읽는다.
3. **webClientId 타입 확인**: google-services.json `oauth_client`의 `client_type: 3`인지.
4. **GCP 사용자 인증 정보에서 Android 클라이언트(type 1) 존재 확인** — 이때 프로젝트를 **번호로** 식별할 것 (`google-services.json`의 `project_number`와 대조). 이름 검색은 동명 프로젝트에 낚인다.
5. 없으면 수동 생성(§4) → **몇 분 전파 대기** → 재시도. 재빌드 불필요.
6. code 10이 사라진 뒤의 에러는 **별개의 벽** — 메시지를 새로 읽는다 (이번엔 §5였다).

---

## 7. 치트시트

```bash
# 무선 디버깅으로 공기계 연결 (Android 11+, 케이블 없이)
adb pair 192.168.x.x:PAIRPORT     # 폰 "코드로 페어링" 팝업의 IP:포트 + 6자리 코드
adb connect 192.168.x.x:CONNPORT  # 무선 디버깅 메인 화면의 (다른) IP:포트
adb devices                       # "device"로 떠야 함

# 로그 버퍼 비우고 → 폰에서 로그인 1회 → 덤프
adb logcat -c
adb logcat -d | grep -iE "ReactNativeJS|ApiException|DEVELOPER_ERROR"

# 설치된 apk의 실제 서명 SHA-1
adb shell pm path com.rocher.kbap   # base.apk 경로
adb pull <경로> installed.apk
apksigner verify --print-certs installed.apk | grep -i SHA-1
# (apksigner 없으면: Android SDK build-tools 안에 있음.
#  keytool -jarfile은 v2-only 서명이라 실패 — §6 참고)

# EAS가 관리하는 keystore의 SHA 확인
eas credentials   # Android → keystore → SHA-1/SHA-256 표시
```

실측 기록: apk 서명 SHA-1 `D2:F7:2B:1A:F4:5C:C4:23:13:18:CB:98:D4:65:7D:6D:F8:2A:A3:35` = Firebase 등록값 일치 · 프로젝트 `k-bap-eb032`(번호 44799256321) · EAS keystore `3TdEsf5DKE` · 빌드 `00ba0336`(preview apk).

---

## 8. 용어집

- **DEVELOPER_ERROR (code 10)**: "앱 설정이 서버 등록과 안 맞는다"는 구글의 복구 불가 신호. 십중팔구 SHA-1/클라이언트 문제.
- **OAuth 클라이언트 타입 1/2/3**: Android(패키지명+SHA-1) / iOS(번들 ID) / Web(ID 문자열). 로그인 코드가 참조하는 건 3뿐.
- **webClientId**: idToken의 수신자(`aud`)가 될 웹 클라이언트 ID. Android 타입을 넣으면 안 됨.
- **서명 인증서 지문 (SHA-1)**: apk에 서명한 keystore 인증서의 해시 = Android에서 앱의 신원. 키가 바뀌면 신원도 바뀜.
- **Play App Signing**: 구글이 스토어 배포본을 자기 키로 재서명하는 제도. 업로드 키 ≠ 앱 서명 키 → SHA 추가 등록 필요.
- **APK Signing Block**: v2/v3 서명이 인증서·서명을 담는 ZIP 밖 블록. v1(META-INF 파일)과 달리 keytool로 못 읽음.
- **HostFunction 예외**: RN 브리지/TurboModule 경계에서 네이티브가 던진 예외가 JS로 표면화된 형태. "네이티브 쪽을 보라"는 신호.
- **idToken / accessToken**: 신원 증명 JWT("누구인가", `aud` 검증) / 자원 접근 토큰("무엇을 해도 되나"). 로그인의 본질은 idToken.
- **전파 지연 (propagation)**: OAuth 클라이언트 생성·변경이 구글 인프라에 퍼지는 데 걸리는 몇 분. 그동안 code 10이 잠깐 지속될 수 있음.

---

## 9. 다음에 빠른 재현 체크리스트

1. Android 로그인 실패 = 먼저 **logcat에서 status code 숫자** 확정 (§3 표).
2. code 10이면: 설치본 SHA-1 실측(apksigner) → webClientId 타입 → **GCP에 Android 클라이언트 존재** → 전파 대기, 순서로 소거.
3. GCP 프로젝트는 **이름 말고 번호로** 찾기 (`project_number` 대조).
4. Android OAuth 클라이언트는 **존재만 하면 됨** — json 재다운로드·재빌드 불필요.
5. "iOS는 되는데"는 단서지 결론이 아니다 — **Android의 추가 검증(서버 SHA 대조, 네이티브 인자 검증)마다** 따로 벽이 있을 수 있다.
6. `credential(idToken)`만 넘기는 코드 + 구버전 RNFB 조합은 Android에서 터진다 — accessToken을 함께 넘기거나(현 K-Bap) 수정 버전으로 업데이트.
7. Play 스토어 출시 시: **앱 서명 키 SHA를 Firebase에 추가 등록** (§4 warning) — 안 하면 스토어 빌드만 깨진다.

---

## 10. 참고

- [Google — Android 클라이언트 인증 (패키지명+SHA-1)](https://developers.google.com/android/guides/client-auth)
- [GoogleSignInStatusCodes 레퍼런스](https://developers.google.com/android/reference/com/google/android/gms/auth/api/signin/GoogleSignInStatusCodes)
- [Firebase Android troubleshooting FAQ — SHA·OAuth 클라이언트·전파 지연](https://firebase.google.com/docs/android/troubleshooting-faq)
- [Firebase — "OAuth2 client already exists" 충돌](https://support.google.com/firebase/answer/6401008)
- [react-native-firebase — Google 소셜 로그인 가이드](https://rnfirebase.io/auth/social-auth)
- [react-native-firebase issue #9081 — idToken-only 크레덴셜이 Android에서 실패](https://github.com/invertase/react-native-firebase/issues/9081) · 수정 커밋 `d800caf8`
- [Android 공식 — APK Signature Scheme v2 구조](https://source.android.com/docs/security/features/apksigning/v2)
- [Firebase — SHA 지문 등록 (Play App Signing 포함)](https://support.google.com/firebase/answer/9137403)
- [OAuth.net — ID Tokens vs Access Tokens](https://oauth.net/id-tokens-vs-access-tokens/)

## 관련 파일 (프로젝트, 2026-07-20 실측)

- `kbap-fe/src/lib/auth/useSocialAuth.ts:72-73` — KB-196 수정 반영 확인 (`getTokens()` → `credential(idToken, accessToken)`)
- `kbap-fe/google-services.json` — oauth_client는 여전히 type 3(Web) 하나 (Android 클라이언트는 클라우드에만 존재하면 됨의 실증)
- `@react-native-firebase/auth@25.1.0` `lib/credentials/OAuthCredential.ts` — `secret = … ?? accessToken ?? ''` (버그 지점) · `android/.../ReactNativeFirebaseAuthModule.java:2049` · `ios/RNFBAuth/RNFBAuthModule.m:1524`
- 진행 기록: spec 레포 `bridge/QA.md` Q-14 · KB-196/P-020 (OTA 후 실기기 최종 확인은 spec 세션에서 종결)

---

> [!note] 같은 "서명한 놈이 누구냐" 구조 — TLS 인증서 편 (2026-08-21 추가)
> 이 노트의 신뢰 체인(구글 서버가 **패키지명+서명 SHA-1**을 대조 → 불일치면 앱이 아니라 **플랫폼이** 먼저 거절)과 똑같은 사고방식이 HTTPS에도 있다: 기기가 **서버 인증서를 누가 서명했나**를 신뢰 저장소와 대조해, 모르는 CA면 **데이터 전송 전에 핸드셰이크를 끊는다**. 디버깅 프록시는 자기 CA를 그 저장소에 넣어 통과하는 것 — iOS는 설치와 완전 신뢰가 별개 단계이고, 안드로이드는 한술 더 떠 **앱이 사용자 CA를 기본 불신**한다(`network_security_config` 필요) → [[프록시와 MITM 디버깅 (Proxyman)]]

> [!note] 이 노트가 끝나는 지점에서 서버 쪽 노트가 시작한다 (2026-08-27 추가)
> 이 노트의 종착점은 **앱이 마침내 `idToken` 을 손에 쥐는 것**이다(SHA-1 대조 통과 → `GoogleSignin.signIn()` 성공 → `signInWithCredential` 로 Firebase 세션). 서버 쪽 흐름 노트는 정확히 그 다음 줄 — `getIdToken()` 으로 꺼낸 토큰을 `POST /api/auth/login` 으로 던지는 순간 — 에서 시작한다(`kbap-fe/src/lib/auth/useSocialAuth.ts:50-55`). 이음매에서 헷갈리기 쉬운 것 하나: **서버로 가는 idToken 은 구글이 준 그 토큰이 아니라 Firebase 세션이 재포장해 발급한 토큰**이다. 구글 것을 그대로 보내면 서버의 `verifyIdToken` 이 실패한다 → [[16. 소셜 로그인 흐름 — Firebase 토큰이 자체 JWT가 되기까지]] §1
