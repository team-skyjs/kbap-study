---
tags: [aws, s3, presigned-url, 보안, 시크릿, iam, cors, 업로드, kbap]
생성일: 2026-07-16
상태: 완료
---

> [!info] 관련 노트
> 🏠 [[🏠 홈]] · 📂 주제: 백엔드
> 🔗 같은 원리(비밀 서명은 서버로): [[Google vs Apple 소셜 로그인 비교]] · [[16. 소셜 로그인 흐름 — Firebase 토큰이 자체 JWT가 되기까지]]
> 🔗 이 업로드 결과가 저장되는 곳: [[15. 온보딩 프로필 저장 흐름 — 요청이 DB에 닿기까지]] (`profileImageUrl` 그릇)

# S3 presigned URL과 클라이언트 시크릿 — 왜 앱은 AWS 키를 직접 못 쓰나

> 작성: 2026-07-16 · 맥락: K-Bap 이미지 업로드(온보딩 프로필 사진)를 presigned URL 방식으로 확정(7/16 회의). 이전 경험은 "앱에 access key 내장 → aws-sdk로 직접 PUT" — 데모에선 됐는데 스토어 배포 앱에선 왜 안 되는지가 논점. 실제로 ipa를 unzip해서 번들 속 문자열이 그대로 보이는 걸 확인함.
> 목표: "왜 서버가 서명해줘야 하는가"를 원리(서명·신뢰 경계)로 이해하고, 발급 API·IAM·CORS·키 운영까지 실전 체크리스트로.

---

## 0. 한 줄 요약
**앱 번들에 들어간 모든 문자열은 공개된 것이다** (`unzip app.ipa` + `strings` 한 줄이면 끝). 그래서 AWS secret은 서버에만 두고, 서버가 자기 권한을 **"이 경로에, 이 메서드로, 이 시간까지"로 좁혀 서명한 URL(presigned URL)**을 FE에 빌려준다 — 본질은 **권한의 임시 위임**이고, 유출돼도 피해가 그 서명의 범위(1개 키·1개 메서드·만료 전)로 제한된다.

---

## 1. 핵심 개념

### 1-1. 왜 앱에 시크릿을 못 넣나 — "배포된 클라이언트는 유리집"

| | 서버 코드 | 앱 번들 (ipa/apk) |
|---|---|---|
| 누가 갖나 | 우리만 (서버 안) | **모든 사용자가 사본을 소유** |
| 읽을 수 있나 | 침입해야 | `unzip` + `strings`면 끝 (권한·탈옥 불필요) |
| 시크릿 두면 | OK | **공개 게시와 동일** |

- 실측: `unzip app.ipa && strings Payload/*.app/main.jsbundle | grep AKIA` → access key가 평문으로 나온다. RN은 JS 번들이라 더 쉽고, 네이티브 컴파일이어도 문자열 리터럴은 바이너리에 그대로 남는다.
- **컴파일·난독화가 못 지키는 이유**: 앱은 실행될 때 그 키를 *사용*해야 한다 = 복원 로직과 재료가 전부 사용자 손안의 기기에 있다. 난독화는 자물쇠가 아니라 **시간 벌기**일 뿐. 열쇠와 자물쇠를 같은 상자에 넣어 배포하는 구조라 원리적으로 불가능하다.
- 데모에서 됐던 건 "기술적으로 동작"과 "안전"이 다른 문제라서다. AWS 키는 GitHub 공개 저장소 기준 **노출 후 수 분 내에 봇이 수집**하는 대표 표적 — 배포 앱도 규모만 다를 뿐 같은 공개 채널이다.

> 비유: 앱 번들은 **모든 손님에게 나눠주는 전단지**다. 전단지에 금고 비밀번호를 인쇄해놓고 "글씨를 꼬아놨으니 괜찮다"(난독화)고 할 수는 없다. 되는 방법은 하나 — 금고는 본사(서버)에 두고, 손님에겐 **"오늘 오후 3시까지, 3번 사물함에만 넣을 수 있는 일회용 접수증"**(presigned URL)을 써주는 것.

### 1-2. presigned URL의 본질 — 서명 = 권한의 임시 위임

presigned URL은 "서버(정확히는 서버가 가진 IAM 자격증명)가 할 수 있는 일 중 딱 한 조각"을 URL에 서명해 위임한 것이다. **SigV4 서명에 들어가는 것**:

- **누가** (서명한 자격증명 — 이 권한 이상은 어차피 못 위임)
- **무엇을** (HTTP 메서드: PUT / GET / …)
- **어디에** (버킷 + 객체 key 경로 — *정확히 그 key 하나*)
- **언제까지** (만료 시각. SigV4 상한 **7일**; 서명에 쓴 게 임시 자격증명이면 **그 세션 만료가 먼저** 적용됨 — Lambda/ECS role로 서명하면 1시간짜리가 되는 흔한 함정)
- (선택) **어떤 헤더로** — 예: `Content-Type`을 서명에 포함하면 클라이언트가 다른 타입으로 못 올림

서명은 secret key 기반 HMAC이라 **secret 없이는 위조 불가**, URL의 어느 파라미터든 바꾸면 서명 불일치로 403. 그래서:

> [!important] 유출 피해 반경이 설계로 제한된다
> access key 유출 = **그 키의 모든 권한**이 영구히(폐기 전까지) 넘어감.
> presigned URL 유출 = **그 객체 1개에, 그 메서드 1번의 종류로, 만료 전까지**만. 프로필 사진 업로드 URL이 새어봤자 남이 내 프로필 사진 자리에 이미지를 넣을 수 있는 것이 최대 피해다. 이 비대칭이 이 패턴의 존재 이유.

### 1-3. presigned PUT vs presigned POST (policy)

| | **presigned PUT** | **presigned POST** |
|---|---|---|
| FE가 보내는 것 | URL에 **바이너리 body 하나** | multipart/form-data 폼 (필드 여러 개 + 파일) |
| Content-Type 제한 | 서명에 포함하면 가능 (정확히 일치 강제) | policy로 가능 (`starts-with $Content-Type image/` 같은 프리픽스도) |
| **크기 제한** | **불가** ← 유일하게 밀리는 부분 | **가능** (`content-length-range`) |
| 구현 난도 (모바일) | `fetch(url, {method:'PUT', body: blob})` 끝 | 폼 필드 조립 필요 |

- **모바일 앱은 보통 PUT이 단순** — RN에서 폼 조립 없이 fetch 한 줄. K-Bap도 PUT 방향.
- PUT의 크기 제한 공백은 보완책으로 막는다: BE가 발급 요청 단계에서 용도별 상한 정책을 갖고, 업로드 후 BE가 `HeadObject`로 실제 크기·타입 검증(§1-4), 버킷에 수명주기 규칙. 웹에서 불특정 다수 업로드를 받는다면 그때는 POST policy가 정답.

### 1-4. 업로드 후 신뢰 경계 — "FE가 보낸 key는 주장일 뿐"

⑤단계(BE가 `profileImageUrl` 저장)에서 **BE는 FE가 보낸 key를 반드시 검증**해야 한다. FE는 이미 신뢰 경계 바깥이라:

- **남의 key 제출**: 내가 발급받은 적 없는 `profile/다른멤버/x.jpg`를 내 프로필로 등록 시도
- **경로 조작**: `../` 류나 전혀 다른 프리픽스의 key
- **업로드 안 하고 key만 제출**: 깨진 이미지 URL이 저장됨

방어: ① key를 **FE가 짓지 않는다** — BE가 발급 시점에 `profile/{memberId}/{uuid}.jpg`로 생성해 내려주고, 저장 시점에 "이 멤버에게 발급된 key인가"(프리픽스 검증 또는 발급 기록 대조)를 확인. ② 필요하면 `HeadObject`로 실존·크기·Content-Type 확인. — [[OCR 라인 분류 — 휴리스틱의 한계와 대안]]의 "FE는 후보, BE가 단언"과 같은 원리다.

### 1-5. 일반화 — "비밀이 필요한 서명은 전부 서버로"

같은 패턴이 반복된다. 클라이언트는 *요청*하고, **비밀을 쥔 서버가 서명/검증**한다:

| 사례 | 클라이언트 몫 | 서버 몫 (비밀 보유) |
|---|---|---|
| S3 업로드 | 발급 요청 + PUT | AWS secret으로 presign |
| Apple 로그인 | identity token 획득 | client secret(서명 JWT)으로 애플과 검증 — [[Google vs Apple 소셜 로그인 비교]] |
| 결제 | 결제창 띄우기 | secret key로 승인(confirm) 호출 |
| 자체 인증 | 토큰 제시 | JWT 서명 키로 발급·검증 — [[17. JWT 인증 필터와 세션 생명주기 — 매 요청 검증·재발급·로그아웃]] |

### 1-6. 서버 없이 하려면 — Cognito identity pool (대안 비교)

Cognito identity pool은 앱에 **임시 AWS 자격증명**(수명 있는 STS 토큰)을 직접 내려주는 AWS 서비스다. 키 내장과 달리 자격증명이 단기이고 IAM 정책 변수(`${cognito-identity.amazonaws.com:sub}`)로 "자기 경로만" 잠글 수 있다.

| | 키 내장 (안티패턴) | **BE 발급 API (K-Bap 선택)** | Cognito identity pool |
|---|---|---|---|
| 보안 | ❌ 영구 키 공개 | ✅ 비밀이 서버 밖에 안 나감 | ✅ 단기 자격증명 (설계 자체는 안전) |
| 서버 필요 | 없음 | **발급 API 1개** | 없음 (서버리스 가능) |
| 업로드 정책 통제 | 없음 | **BE 코드로 자유** (용도·검증·기록) | IAM 정책 문법으로만 (표현력 제한) |
| 러닝커브 | — | 이미 아는 것(API 하나) | Cognito·IAM role·Amplify 학습 필요 |
| 적합한 때 | 없다 | **BE가 이미 있는 앱** | BE가 아예 없는 순수 서버리스 |

→ K-Bap은 BE(종한)가 이미 있으므로 발급 API가 가장 싸고 통제력도 최대. Cognito는 "서버가 없을 때의 정답"이지 서버가 있는데 도입하면 학습·운영만 는다.

---

## 2. 절차 / 방법

### 2-1. 전체 시퀀스 (확정안)

```
① FE → BE:  POST /files/presigned-url {용도, 확장자}     ← BE만 AWS secret 보유
② BE → FE:  { url: "https://bucket.s3...X-Amz-Signature=...", key: "profile/abc.jpg" }
③ FE → S3:  PUT <url>  (body = 이미지 바이너리)           ← 여기가 FE 몫
④ FE → BE:  PATCH /members/me/profile { profileImageUrl: key }
⑤ BE:       key 검증 후 저장                                ← §1-4 신뢰 경계
```

현황(2026-07-16 Swagger 재확인): `profileImageUrl` 그릇(KB-147)은 배포됨 — `OnboardingRequest`·`ProfileUpdateRequest`·`MyProfileResponse`에 존재. **①②의 발급 API는 미배포** (BE 대기). 콘솔 세팅·키 발급·FE 연결은 KB-149.

### 2-2. 발급 API 설계 베스트 프랙티스

- **만료는 짧게**: 업로드용은 **1~5분**이면 충분 (발급 직후 바로 PUT하는 흐름). 7일은 상한이지 권장이 아님.
- **key는 서버가 생성**: `profile/{memberId}/{uuid}.{ext}` — 네임스페이스로 소유가 경로에 새겨지고, uuid라 덮어쓰기·추측 불가. FE에게 파일명 결정권을 주지 않는다.
- **Content-Type을 서명에 포함**: 발급 요청의 확장자로 `image/jpeg` 등을 정해 서명 → FE가 html 따위를 못 올림 (XSS 방지 — 버킷이 CDN 뒤에 있으면 특히).
- **용도 화이트리스트**: `{용도: "profile"}`처럼 enum으로 받고 용도별 경로·타입·정책을 서버가 결정.
- **크기**: PUT은 서명으로 못 막으므로(§1-3) 업로드 후 `HeadObject` 검증 또는 필요 시 POST policy 전환.

### 2-3. IAM 최소 권한 — 이 프로젝트에 딱 맞는 JSON

BE(presign용) IAM user/role에는 **딱 이만큼만**:

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "PresignUploadOnly",
    "Effect": "Allow",
    "Action": ["s3:PutObject"],
    "Resource": "arn:aws:s3:::kbap-uploads/profile/*"
  }]
}
```

- presigned URL은 **서명자 권한의 부분집합**만 위임 가능 → 서명자가 `PutObject` 하나면 무슨 짓을 해도 그 이상 못 넘어간다. 유출 시 최악 시나리오를 IAM에서 한 번 더 캡핑하는 것.
- `s3:*`, `Resource: "*"` 금지. 조회/삭제가 필요해지면 그때 Action을 하나씩 추가 (`GetObject`는 CDN/공개 정책 쪽에서 다루는 게 보통).

### 2-4. CORS — 모바일과 웹은 다르다

- **CORS는 브라우저 전용 메커니즘**이다. 네이티브 앱(RN 포함)의 fetch는 `Origin` 헤더를 안 보내고 브라우저 차단 로직도 없음 → **모바일 업로드는 CORS 설정과 무관하게 동작**한다.
- 즉 "모바일에서 되는데 웹에서 CORS 에러"는 버그가 아니라 기본값. 나중에 웹(어드민 등)에서 같은 업로드를 하려면 그때 버킷 CORS에 추가:

```json
[{
  "AllowedOrigins": ["https://admin.example.com"],
  "AllowedMethods": ["PUT"],
  "AllowedHeaders": ["Content-Type"],
  "MaxAgeSeconds": 3000
}]
```

- 웹의 단골 삽질: CORS 에러처럼 보이는 것의 절반은 **진짜 CORS(preflight에 응답 없음)**, 나머지 절반은 **403(서명 문제)이 CORS 에러로 위장**된 것 — 브라우저는 CORS 헤더 없는 403을 CORS 에러로 보고한다. 구분법: 같은 URL을 curl로 쏴보기 (curl은 CORS 무시 → 403이면 서명 문제).

### 2-5. 키 전달·운영 — "한 번이라도 커밋된 키는 죽은 키"

- **전달은 비-git 채널로**: 1Password 공유·시크릿 매니저 등. Slack DM도 차선일 뿐 (검색 가능한 영구 기록).
- **왜 커밋 삭제로 부활 안 되나**: ① git 이력·리플로그에 SHA로 영구 보존 ② 이미 clone한 모든 사람·CI 러너에 사본 ③ 포크에 전파 ④ GitHub raw CDN 캐시는 별도 (지워도 남음) ⑤ 공개 레포면 **봇이 수 분 내 수집** (10분 내 악용 실사례). → 이력 정리는 부차 작업이고, **폐기(revoke)가 1순위·즉시**다.
- GitHub에 AWS 키가 노출되면 AWS가 자동 감지해 `AWSCompromisedKeyQuarantineV2` 정책을 붙이지만(EC2 생성 등 차단), 이건 응급 처치지 폐기를 대신 안 한다.
- **로테이션**: IAM user는 access key를 2개까지 가질 수 있음 → 새 키 발급 → 배포 전환 → 옛 키 비활성 → 무중단 교체. 정기 로테이션보다 "노출 의심 시 즉시"가 본질.

---

## 3. 트러블슈팅 (예상 403 목록 — 만나면 여기부터)

- **`SignatureDoesNotMatch`** → 서명에 포함된 것과 실제 요청 불일치: 서명 시 `ContentType: image/jpeg`를 넣었는데 FE가 `Content-Type` 헤더를 안 보냄/다르게 보냄(가장 흔함), 또는 FE가 임의 헤더 추가, URL 인코딩 변형.
- **`Request has expired`** → 만료 지남. 발급→업로드 사이가 길어지는 UX(사진 편집 등)면 만료를 그만큼만 늘리거나 업로드 직전 발급.
- **`AccessDenied` (만료 전인데)** → 서명자 IAM에 해당 Action/Resource 권한이 없거나, 임시 자격증명 세션이 먼저 만료(§1-2), 버킷 정책이 별도로 거부.
- **웹에서만 실패** → §2-4. preflight(OPTIONS) 응답 확인, curl 대조로 CORS인지 403인지부터 판별.
- **PUT은 됐는데 이미지가 깨짐** → body에 FormData나 base64 문자열을 넣은 경우 — presigned PUT의 body는 **날 바이너리** 하나여야 한다.

---

## 4. 치트시트

```bash
# 번들 속 시크릿 확인 (이번에 직접 해본 것)
unzip app.ipa && strings Payload/*.app/main.jsbundle | grep -E "AKIA|secret"

# presigned PUT 동작 확인 (FE 붙이기 전에 URL만 검증)
curl -v -X PUT --upload-file test.jpg -H "Content-Type: image/jpeg" "<presigned-url>"

# 업로드 결과 확인 (BE 검증과 동일한 시선)
aws s3api head-object --bucket kbap-uploads --key profile/abc.jpg

# 노출 키 응급 대응 순서
aws iam update-access-key --access-key-id AKIA... --status Inactive  # ① 즉시 무력화
aws iam create-access-key                                            # ② 새 키
aws iam delete-access-key --access-key-id AKIA...                    # ③ 확인 후 삭제
```

---

## 5. 용어집
- **presigned URL**: 자격증명 보유자가 특정 요청(메서드+경로+만료)을 미리 서명해둔 URL. 소지자는 그 요청 하나를 서명자 권한으로 실행 가능.
- **SigV4 (AWS Signature Version 4)**: AWS 요청 서명 규격. secret key로 요청 정보를 HMAC 서명 — secret 없이 위조 불가, 파라미터 변조 시 무효.
- **access key / secret access key**: IAM 자격증명 쌍. 앞은 식별자(`AKIA...`, 공개돼도 그 자체론 무해), 뒤가 진짜 비밀.
- **STS / 임시 자격증명**: 수명이 있는 AWS 자격증명. role·Cognito가 발급. 이걸로 presign하면 URL 수명도 세션에 캡핑.
- **IAM least privilege**: 필요한 Action·Resource만 허용하는 원칙. 유출 시 피해 상한을 정책으로 설정.
- **POST policy**: presigned POST에 동봉되는 조건문서(크기 범위·타입 프리픽스 등). PUT엔 없는 통제력의 원천.
- **CORS**: 브라우저가 타 출처 요청을 제한하는 **브라우저 전용** 규칙. 서버/네이티브 앱엔 적용 안 됨.
- **`AWSCompromisedKeyQuarantine`**: GitHub 등에 노출된 키에 AWS가 자동으로 붙이는 격리 정책.
- **key rotation**: 자격증명을 새것으로 교체하는 운영. IAM user는 키 2개 슬롯으로 무중단 교체.

---

## 6. 다음에 빠른 재현 체크리스트
1. 클라이언트에 시크릿이 들어갈 것 같으면 → **번들은 전단지**다. 서명이 필요하면 서버로.
2. 업로드 설계 기본형: **BE 발급(만료 수 분, key는 서버 생성 `용도/{소유자}/{uuid}`, Content-Type 서명 포함) → FE PUT(날 바이너리) → BE key 검증 후 저장.**
3. FE가 보낸 key는 주장 — **프리픽스(소유) 검증 + 필요시 HeadObject.**
4. 크기 제한이 꼭 필요하면 PUT이 아니라 **POST policy** (`content-length-range`).
5. presign용 IAM은 **단일 버킷 프리픽스 `PutObject`만.**
6. 모바일은 CORS 무관 — 웹 추가할 때만 버킷 CORS. "CORS 에러"는 curl로 서명 403과 구분부터.
7. 키가 git에 닿았다? **즉시 Inactive → 새 키 → 삭제.** 이력 청소는 그다음.

---

## K-Bap 적용 현황 (2026-07-16)
- 방식 확정: presigned URL (7/16 회의) · 콘솔 세팅·키 발급·FE 연결 = 예진(KB-149) · 발급 API = BE 종한
- Swagger 재확인: `profileImageUrl` 필드(KB-147) 배포됨 (`OnboardingRequest`·`ProfileUpdateRequest`·`MyProfileResponse`), **발급 API 미배포** — FE는 ③④단계 코드를 미리 준비 가능 (curl 치트시트로 URL만 먼저 검증)

---

## 참고 링크
- [AWS 공식 — Download and upload objects with presigned URLs](https://docs.aws.amazon.com/AmazonS3/latest/userguide/using-presigned-url.html) — 만료 상한(SigV4 7일)·임시 자격증명 캡핑
- [AWS 공식 — POST Policy 구성](https://docs.aws.amazon.com/AmazonS3/latest/API/sigv4-HTTPPOSTConstructPolicy.html) — `content-length-range`·`starts-with` 조건
- [Differences between PUT and POST S3 signed URLs — Advanced Web Machinery](https://advancedweb.hu/differences-between-put-and-post-s3-signed-urls/)
- [S3 Uploads — Proxies vs Presigned URLs vs Presigned POSTs — Zac Charles](https://zaccharles.medium.com/s3-uploads-proxies-vs-presigned-urls-vs-presigned-posts-9661e2b37932)
- [AWS 공식 — S3 CORS](https://docs.aws.amazon.com/AmazonS3/latest/userguide/cors.html) · [Deep dive into CORS configs on S3 — AWS 블로그](https://aws.amazon.com/blogs/media/deep-dive-into-cors-configs-on-aws-s3-how-to/)
- [AWS 공식 — Cognito identity pool 자격증명](https://docs.aws.amazon.com/cognito/latest/developerguide/getting-credentials.html) — 서버 없는 대안
- [Deleting leaked API keys isn't a solution — Truffle Security](https://trufflesecurity.com/blog/remediate-leaked-api-keys-with-key-rotation) · [AKIA 노출 대응 실증 — Chris Farris](https://www.chrisfarris.com/post/akia-response/) (10분 내 악용 사례)
- 프로젝트: Jira KB-149·KB-147 · BE Swagger `https://meogo.handev.site/swagger-ui`

---

> [!note] 이 발급 정책이 서버 어디에 적혀 있나 (2026-08-26 추가)
> 이 노트가 "만료 몇 분·어떤 Content-Type·몇 바이트까지"를 **설계 원칙**으로 정했다면, kbap-server에서 그 세 값은 코드가 아니라 `application.yml` 의 `kbap.upload.allowed-content-types`·`max-bytes`·`upload-ttl` 에 적혀 `ImageUploadProperties` 한 그릇으로 묶여 들어온다 — 정책을 바꾸는 데 재빌드가 필요 없다는 뜻이다 → [[6. 설정과 프로필, 트랜잭션 경계]]

> [!note] 서버가 S3 를 아는 자리는 어디까지인가 (2026-08-26 추가)
> 이 노트의 결론은 "클라가 S3 로 직접 올리되 서명은 서버가 한다"였다. kbap-server 에서 그 **서명하는 코드는 파일 하나**다 — `PresignedUploadPort` 라는 3줄짜리 순수 인터페이스가 계약이고, AWS SDK 를 import 하는 어댑터는 그 뒤에 갇혀 있어서 업로드를 쓰는 서비스는 자기가 진짜 S3 를 부르는지 테스트용 페이크를 부르는지 모른다(로컬은 자격증명이 없어 아예 빈이 안 뜨고 페이크가 그 자리를 채운다). 스토리지를 갈아끼울 때 고쳐야 할 파일 수를 미리 정해 두는 설계 → [[12. 포트와 어댑터 — 외부 시스템을 갈아끼우는 자리]]

> [!note] 업로드가 끝난 뒤 그 경로가 DB 에 어떤 모양으로 앉나 (2026-08-27 추가)
> 이 노트는 presigned URL 로 파일이 스토리지에 올라가는 데까지다. 그 결과가 회원 프로필로 저장될 때 서버는 **objectKey 만** 남긴다 — `MemberProfile.validatedImagePath` 가 앞의 `/` 를 떼고, 빈 문자열·512자 초과와 함께 **`http(s)://` 로 시작하는 값을 아예 거절**한다(`MEMBER-008`). 즉 CDN 도메인이 붙은 완전 URL 을 그대로 보내면 400 이다. 도메인은 조회할 때 `ImageUrls.resolve(...)` 가 다시 붙인다. 저장은 상대 경로, 응답은 절대 URL — CDN 을 갈아끼워도 DB 를 손대지 않으려는 배치다 → [[15. 온보딩 프로필 저장 흐름 — 요청이 DB에 닿기까지]] §3-5

> [!note] 그 "발급 API 미배포"가 배포됐다 — 3단계 계약의 실물 (2026-08-27 추가)
> 위 「K-Bap 적용 현황」이 *"발급 API 미배포 — FE는 ③④단계 코드를 미리 준비 가능"* 으로 끝나 있는데, 그 API 가 지금 레포에 있다. 이 노트가 **원리**(왜 서버가 서명하는가)라면 그 구현은 → [[21. 이미지 업로드 흐름 — presigned URL과 업로드 완료 확정]].
> 이 노트의 설계가 코드에서 달라진 곳은 하나다 — **2단계가 아니라 3단계**다. 발급(`POST /api/images/upload-url`) → S3 직접 PUT → **확정(`POST /api/images/complete`)**. 서버는 서명만 해주고 업로드 성공을 모르기 때문에(그게 이 노트 §0 의 "권한의 임시 위임"이 치르는 대가다), 클라가 끝났다고 신고해야 `uploaded_image` 행이 생긴다. 그 행이 곧 **소유 증명**이 되어 리뷰·커뮤니티·주문이 "이거 네 사진 맞냐"를 확인한다.
> 파생 사실 둘: ① 서버는 발급 시점에 **아무 원장도 남기지 않는다** — 그래서 ②만 성공하고 ③이 안 오면 S3 에 **고아 객체**가 남고 회수 잡도 수명주기 규칙도 없다(의도된 미구현, `specs/kb-145-presigned-url/research.md:80`). ② 파일 이름은 **서버가 짓는다** — 앱이 key 를 못 정하는 게 경로 규약이자 보안 장치다.
