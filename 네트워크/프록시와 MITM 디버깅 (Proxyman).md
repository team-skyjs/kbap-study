---
tags: [네트워크, 프록시, https, tls, 인증서, proxyman, 디버깅, ios]
생성일: 2026-08-21
상태: 완료
---

> [!info] 관련 노트
> 🏠 [[🏠 홈]] · 📂 주제: 네트워크
> 🔗 같은 LAN 이야기: [[Metro 연결 실패 — Mac·폰 IP 불일치 (link-local·서브넷)]] — 폰이 맥을 IP로 찾아가는 건 여기도 똑같다 (`192.168.0.100`이 왜 그 값인지, 왜 바뀌는지)
> 🔗 같은 "서명한 놈이 누구냐" 계열: [[Android 구글 로그인 — SHA-1·OAuth 클라이언트·DEVELOPER_ERROR]] — 거긴 앱 서명 대조, 여긴 인증서 서명 대조
> 🔗 프록시로 들여다보면 **평문으로 보이는 것**들: [[OAuth 2.0과 OIDC — 소셜 로그인의 원리]] (Bearer 토큰) · [[클라이언트 저장소 — AsyncStorage·SecureStore·메모리 (K-Bap 실전)]] (그 토큰의 보관 위치)

# 프록시와 MITM 디버깅 (Proxyman)

> 작성: 2026-08-21 · 맥락: kbap-fe(Expo dev client)를 아이폰에서 돌리며 `dev.kbap.site` API 요청/응답을 맥 Proxyman으로 보려다, ①아이폰 트래픽이 아예 안 잡히고 ②SSL Proxying을 켜니 전부 `999` + 앱이 "You're offline"이 된 사건.
> 목표: **"프록시가 대체 뭘 하는 물건이고, HTTPS인데 왜 남이 읽을 수 있으며, 왜 인증서를 '설치'만으론 부족한가"** 를 원리로 이해하고, 다음에 5분 안에 다시 세팅하기.

---

## 0. 한 줄 요약

프록시는 **내 요청을 대신 보내주는 중간 서버**다. HTTPS는 원래 중간에서 못 읽지만, Proxyman 같은 디버깅 프록시는 **"내가 그 서버인 척하는 가짜 인증서를 즉석에서 발급"** 해서 TLS 연결을 **두 개로 쪼개** 읽는다(= MITM). 이게 통하려면 **기기가 Proxyman의 루트 CA를 신뢰**해야 하고, iOS는 ①프로파일 설치 ②`Certificate Trust Settings`에서 **완전 신뢰 ON** 이 **별개의 두 단계**다. ②를 안 하면 아이폰이 핸드셰이크를 끊어버리고, Proxyman 목록엔 HTTP 상태 코드가 아닌 내부 코드 **`999`** 가 뜬다.

---

## 1. 프록시란 무엇인가 — 세 종류 비교

> 비유: **대리인**. 내가 직접 가게에 가지 않고 심부름꾼을 보낸다.
> - **포워드 프록시** = 내가 고용한 심부름꾼. 가게는 심부름꾼만 본다. (누가 시켰는지 모름)
> - **리버스 프록시** = 가게가 고용한 접수원. 손님은 접수원만 본다. (뒤에 창고가 몇 갠지 모름)
> - **투명 프록시** = 길목에 숨어 있는 검문소. 나는 심부름꾼을 쓴 줄도 모른다.

| | **포워드 프록시 (forward)** | **리버스 프록시 (reverse)** | **투명 프록시 (transparent)** |
|---|---|---|---|
| 누구 편인가 | **클라이언트** 쪽 | **서버** 쪽 | 네트워크 운영자 쪽 |
| 클라이언트가 설정하나 | **예** (프록시 주소를 명시) | 아니오 (그냥 도메인으로 접속) | 아니오 (모르게 가로챔) |
| 대표 사례 | 사내 게이트웨이, 광고 차단, **디버깅 프록시** | Nginx·ALB·CloudFront, API 게이트웨이 | 통신사·학교망 필터링, 캐시 |
| 목적 | 감시·필터링·캐시·**관찰** | 로드밸런싱·TLS 종료·캐시·보호 | 정책 강제 |
| kbap 맥락 | **Proxyman이 여기** | `dev.kbap.site` 앞단이 여기일 것 | 해당 없음 |

**Proxyman/Charles/mitmproxy = 포워드 프록시 + MITM 기능**을 얹은 개발자 도구다. "내가 내 기기에 직접 심부름꾼을 지정하고, 그 심부름꾼이 내용까지 읽게 허락해준 것" — 그래서 이건 해킹이 아니라 **내 기기에서 내가 승인한 자기 관찰**이다.

> [!note] 왜 처음엔 맥 트래픽만 잡혔나
> Proxyman은 켜지는 순간 **맥 자신의 시스템 프록시 설정만** 자기에게 돌려놓는다(끄면 원복). **원격 기기(아이폰)는 자동으로 알 수 없다** — 아이폰 Wi-Fi에 `맥IP:9090`을 수동으로 넣어줘야 비로소 아이폰이 "내 심부름꾼은 저기 저 맥"이라고 알게 된다. 이게 §6-1 단계다.

---

## 2. HTTPS인데 프록시가 어떻게 내용을 보나

### 2-1. SSL Proxying **끄면** — CONNECT 터널 (봉투만 보임)

HTTPS 요청이 프록시를 통과할 땐 평범한 GET이 아니라 **`CONNECT dev.kbap.site:443`** 이라는 특수 메서드로 시작한다. 프록시는 목적지까지 TCP를 열어주고 그 뒤로는 **바이트를 그대로 통과**시킬 뿐이다(RFC 9110의 표현대로 "blind forwarding"). 그래서 프록시가 볼 수 있는 건:

- ✅ **호스트명과 포트** (`CONNECT` 줄 + TLS ClientHello의 **SNI**)
- ✅ 시각, 데이터 양
- ❌ 경로(`/api/foods/detail`), 헤더, 바디, **Authorization 토큰** — 전부 암호화된 채 지나감

브리프 타임라인의 `02:49:57 CONNECT 200`이 정확히 이 상태다. **"연결은 잘 됐지만 내용은 못 본다."**

### 2-2. SSL Proxying **켜면** — TLS 세션을 두 개로 쪼갬 (MITM)

```
[SSL Proxying OFF] — 터널 1개, 암호는 폰↔서버 사이에서만 풀림
  iPhone ═════════ TLS (end-to-end) ═════════▶ dev.kbap.site
              (Proxyman은 통과만 시킴, 내용 못 읽음)

[SSL Proxying ON] — TLS 2개, 가운데서 평문이 됨
  iPhone ══ TLS #1 ══▶ Proxyman ══ TLS #2 ══▶ dev.kbap.site
            ▲                        ▲
   서버 인증서가 "Proxyman이     여기선 Proxyman이 평범한
   즉석 발급한 가짜 dev.kbap.site"  클라이언트로서 진짜 인증서를 검증
                    │
              평문 req/res를 여기서 읽어 화면에 표시
```

핸드셰이크 순서로 보면:

```
1. iPhone → Proxyman : CONNECT dev.kbap.site:443
2. iPhone → Proxyman : TLS ClientHello (SNI = dev.kbap.site)
3. Proxyman          : "아 dev.kbap.site를 원하는군" → 그 이름으로 서버 인증서를
                       **즉석 생성**하고 **자기 루트 CA로 서명**  ← 여기가 MITM 지점
4. Proxyman → iPhone : 그 가짜 인증서를 내밀며 ServerHello
5. iPhone            : 체인 검증 — "이 서명자(Proxyman CA)를 내가 신뢰하나?"
                       ├─ 신뢰 O → 핸드셰이크 성공 → 평문으로 요청 전송 ✅
                       └─ 신뢰 X → **핸드셰이크 즉시 중단** → Proxyman엔 999 ❌
6. (성공 시) Proxyman → dev.kbap.site : 별도의 진짜 TLS 연결로 요청 중계
```

> **핵심**: 프록시가 뚫는 건 암호가 아니라 **신뢰(trust)** 다. AES를 깨는 게 아니라, "너 나를 진짜 서버로 인정해줄래?"를 기기에게 **미리 허락받는** 것.

---

## 3. "가짜 인증서를 즉석 발급"의 정확한 의미 — 신뢰 체인

인증서 검증은 **체인 타고 올라가기**다: `서버 인증서 ← 이걸 서명한 중간 CA ← 이걸 서명한 루트 CA`. 기기는 맨 위 루트가 **자기 신뢰 저장소(trust store)** 에 있으면 통과시킨다.

```
[정상]  dev.kbap.site 인증서 ← Let's Encrypt 등 ← 애플이 기본 탑재한 루트 CA  → 신뢰 ✅
[MITM]  dev.kbap.site 인증서(가짜) ← **Proxyman CA** ← 내가 아이폰에 설치·신뢰시킨 루트 → 신뢰 ✅
                                        └ 설치·신뢰 안 하면 → 신뢰 ❌ (999)
```

Proxyman은 **CA 개인키를 맥에 갖고 있으므로**, 어떤 도메인 이름이든 그 이름의 인증서를 만들어 **자기 손으로 서명**할 수 있다. 도메인 검증 같은 건 당연히 없다 — 원래 CA가 하는 일(신원 확인)을 생략하고 서명 권한만 쓰는 것이라, **그 CA를 신뢰한다는 건 "이 맥은 어떤 사이트로도 위장할 수 있다"를 허락하는 것**과 같다. (§10 보안 주의)

> [!warning] 예전에 설치해둔 인증서가 왜 소용없었나 — CA는 설치본마다 다르다
> 공식 문서(Proxyman): *"To generate a self-signed Root Certificate. **Proxyman does not use pre-generated or shared certificates.**"* / *"The Proxyman Certificate is a **self-signed certificate that is generated on your machine**."*
> → CA 개인키·인증서는 **그 맥에서 처음 켤 때 그 기기 이름으로 생성**되어 `~/Library/Application Support/com.proxyman.NSProxy/app-data/proxyman-ca.pem`에 저장된다. 그러니 **맥을 바꾸거나, 앱을 지웠다 깔거나, Factory Reset을 하면 CA가 새로 생긴다** → 아이폰에 남아 있던 옛 "Proxyman CA"는 **이름만 같은 남남**이고, 새 CA로 서명된 가짜 인증서를 검증하지 못한다. 증상은 처음 설치와 똑같이 999.
> **교훈: 999가 뜨면 "설치했었나?"가 아니라 "지금 이 맥의 CA를 신뢰 중인가?"를 확인할 것.**

---

## 4. iOS는 왜 "설치"와 "완전 신뢰"가 별개인가

애플 공식 문서(*Trust manually installed certificate profiles in iOS, iPadOS, and visionOS*)의 핵심:

> "If you manually install a profile that contains a certificate payload in iOS, iPadOS, and visionOS, **that certificate isn't automatically trusted for SSL**."
> → `Settings > General > About > Certificate Trust Settings`에서 **"Enable Full Trust For Root Certificates"** 를 직접 켜야 한다.

**왜 2단계인가 (설계 의도)**: 프로파일 설치는 "이 파일을 기기에 들여놓겠다" 정도의 행위지만, 루트 CA를 SSL에 신뢰시키는 건 **그 CA가 모든 HTTPS 사이트로 위장할 수 있게 허락**하는 것이다. 위험도가 차원이 다르기 때문에 애플은 ①설치 ②SSL 신뢰를 **다른 화면, 다른 스위치**로 분리했다. (심지어 ②는 `About` 안쪽 깊숙이 숨겨져 있고, 신뢰할 사용자 CA가 하나도 없으면 메뉴 자체가 안 보인다.) MDM으로 배포된 프로파일은 이 단계가 자동이지만, **직접 다운로드한 프로파일은 반드시 수동**이다.

**설치만 하고 신뢰를 안 켜면 무슨 일이 벌어지나**: 아이폰은 §2-2의 5번에서 체인 검증에 실패 → **애플리케이션 데이터를 한 바이트도 보내지 않고 핸드셰이크를 끊는다.** 서버는 요청이 온 줄도 모르고, Proxyman은 응답을 만들 재료가 없다. 앱 입장에선 그냥 "네트워크 실패" → K-Bap 화면의 **"You're offline"**.

---

## 5. `999`의 정체 — HTTP 상태 코드가 아니다

`999`는 **RFC에 없는 Proxyman 내부 코드**다. 메인테이너가 GitHub 이슈 #922에서 직접 답했다: *"yes, **999 is an internal code**. I suppose that I should remove it, and improve the wording for common SSL error from Swift NIO."*

읽는 법: **"요청을 서버까지 못 보냈거나 응답을 못 받았다"** = 대부분 **TLS 핸드셰이크 실패**.

> [!warning] "Possible Certificate Pinning" 힌트는 **추측**이다 — 이번 사건의 진짜 함정
> Proxyman은 999 옆에 흔한 원인으로 피닝을 제안하지만, 이건 근거 있는 진단이 아니라 **가장 흔한 케이스를 찍어주는 문구**다. 실제로 이번엔 피닝이 아니라 **CA 미신뢰**였다(K-Bap 앱엔 피닝이 없다 — §8 확인). 이 힌트를 곧이곧대로 믿었으면 "우리 앱에 피닝이 걸려 있나?"를 뒤지느라 시간을 날렸을 것.
> **감별법**: 같은 상태에서 **Safari로 그 도메인**에 들어가 본다. Safari도 실패 → **CA 신뢰 문제**(모든 앱 공통). Safari는 되는데 그 앱만 999 → **그 앱의 피닝**.

---

## 6. 절차 — 처음 한 번 (5단계)

**전제**: 아이폰과 맥이 **같은 Wi-Fi**여야 한다(같은 서브넷). VPN 앱은 전부 끈다 — 공식 문서가 명시적으로 경고: *"please ensure that you close all VPN apps, as they conflict with the HTTPS Proxy configuration."*

1. **맥**: Proxyman 실행 → `Certificate` 메뉴 → *Install Certificate on this Mac* (맥 자신도 신뢰시켜 둬야 시뮬레이터·맥 앱 디버깅이 된다)
2. **아이폰**: 설정 → Wi-Fi → 현재 네트워크 ⓘ → **Configure Proxy → Manual**
   - Server: **맥의 Wi-Fi IP** (`ipconfig getifaddr en0` → 예: `192.168.0.100`)
   - Port: **9090** · Authentication: **No**
3. **아이폰 Safari**(가급적 시크릿 탭)에서 **`http://proxy.man/ssl`** 접속 → 프로파일 다운로드
   - 이 주소는 Proxyman이 로컬에 띄운 HTTP 서버다. 안 열리면 → Wi-Fi 재접속 + Proxyman이 켜져 있는지 확인.
4. **설치**: 설정 → 일반 → **VPN 및 기기 관리** → 다운로드된 프로파일 → **설치**
5. **완전 신뢰**: 설정 → 일반 → **정보 → 인증서 신뢰 설정** → **Proxyman CA 스위치 ON** ← ⭐ **여기를 빼먹으면 999**
6. **맥 Proxyman**: 목록에서 `dev.kbap.site` **우클릭 → Enable SSL Proxying** (도메인 단위 또는 앱 단위)

**이후 루틴 (매일)**: 테스트 시작할 때 아이폰 Wi-Fi 프록시 **Manual ON** → 끝나면 **Off**. 인증서 신뢰는 그대로 둬도 되지만, 안 쓸 땐 지우는 게 원칙(§10).

---

## 7. 트러블슈팅 (실제로 막혔던 것 + 자주 나는 것)

| 증상 | 원인 | 해결 |
|---|---|---|
| **맥 앱 트래픽만 잡히고 아이폰은 하나도 안 잡힘** | Proxyman은 **맥의 시스템 프록시만** 자동 설정한다. 원격 기기는 스스로 알 수 없음 | 아이폰 Wi-Fi에 수동 프록시 `맥IP:9090` |
| **전부 `CONNECT 999` + 앱은 "You're offline"** | 아이폰이 Proxyman의 가짜 인증서를 **불신** → 핸드셰이크 중단 | 프로파일 설치 **+ 인증서 신뢰 설정 ON** (§6-4,5) |
| 예전에 인증서 깔았는데도 999 | **맥/재설치마다 CA가 새로 생성됨** — 옛 CA는 남남 | 옛 프로파일 삭제 → `proxy.man/ssl`에서 **다시** 받고 신뢰 |
| Safari는 되는데 **특정 앱만** 999 | 그 앱의 **certificate pinning** | 그 앱은 못 본다(§8). Bypass Proxy List에 넣어 노이즈 제거 |
| **아이폰 인터넷 전체 먹통** | 프록시를 켜둔 채 **맥 Proxyman 종료 / 맥 잠자기 / 맥이 다른 망으로 이동** → 심부름꾼이 사라짐 | 아이폰 Wi-Fi 프록시 **Off**. (이래서 "끝나면 Off"가 루틴) |
| 어제 되던 게 오늘 안 됨 | **DHCP로 맥 IP가 바뀜** (`192.168.0.100` → `.101`) | `ipconfig getifaddr en0`로 재확인 후 아이폰에 갱신. 자주 겪으면 공유기에서 맥 IP 고정 |
| `proxy.man/ssl`이 안 열림 | Proxyman이 꺼져 있거나, 프록시 설정이 안 먹은 상태 | Proxyman 실행 확인 → Wi-Fi 껐다 켜기(재접속) |
| 요청이 **일부만** 안 보임 | URLSession/Alamofire **캐시**로 실제 요청이 안 나감 | 캐시 비활성화 또는 Proxyman **No Caching**(⌥⌘N) |
| 트래픽이 튀는데 원인 불명 | VPN 앱이 프록시와 충돌 | VPN 종료 후 재시도 |

> [!example] K-Bap 실전 확인 (2026-08-21, 실코드 대조)
> - `kbap-fe/.env` → `EXPO_PUBLIC_BE_BASE=https://dev.kbap.site` — Proxyman에서 **이 호스트 하나만** SSL Proxying 켜면 앱 API가 다 잡힌다.
> - `src/lib/api/client.ts`는 **`fetch` 기반 공용 클라이언트**다. RN의 `fetch`는 iOS에서 **NSURLSession/CFNetwork**로 내려가고, CFNetwork는 **시스템(=Wi-Fi) 프록시 설정을 기본으로 따른다** → 그래서 앱 코드를 **한 줄도 안 고치고** 트래픽이 Proxyman에 잡힌 것.
> - 피닝 관련 의존성·설정 **없음**(`trustkit`/`NSPinnedDomains`/`network_security_config` 모두 미사용) → 999의 원인은 피닝일 수 없었다.
> - 켜는 순간 `Authorization: Bearer <BE accessToken>`(KB-67 하이브리드 로그인)이 **평문으로 다 보인다**. 디버깅엔 최고, 스크린샷 공유엔 최악. → [[OAuth 2.0과 OIDC — 소셜 로그인의 원리]]

---

## 8. Certificate Pinning — 왜 이 방법으로도 못 보나 / 우리도 넣어야 하나

**피닝이란**: 앱이 "이 도메인의 인증서(또는 그 공개키)는 **반드시 이것**이어야 한다"를 **앱 안에 박아두고**, OS의 신뢰 저장소와 **무관하게** 직접 대조하는 것. 그래서 사용자가 CA를 아무리 설치·신뢰해도 **앱이 자체 판단으로 거부**한다. (App Store 앱을 Proxyman으로 못 보는 이유 — 메인테이너 답변: *"There is no solution to bypass the SSL Pinning unless you hold the Pinned certificate."*)

**우회가 어려운 이유 = 그게 설계 목표**다. 우회하려면 앱 바이너리를 뜯어 검증 코드를 무력화(탈옥+Frida 류)해야 하는데, 이는 **기기를 장악한 수준의 권한**을 전제한다. 즉 피닝은 "설정으로 뚫리는 방어"가 아니다.

**우리 앱에 넣는다면 (트레이드오프)**

| | 얻는 것 | 잃는 것 / 리스크 |
|---|---|---|
| 보안 | 악성 CA·사내 MITM·사용자가 속아서 깐 CA로부터 API 트래픽 보호 | — |
| **운영** | — | ⚠️ **서버 인증서 갱신 = 앱 전멸**. 애플 문서 경고: *"if the server deploys new certificates that alter the public keys, your app will refuse to connect."* Let's Encrypt처럼 **자동 갱신**하면 리프 피닝은 사실상 자살 |
| 개발 | — | **우리도 Proxyman으로 못 본다** (디버그 빌드 예외 분기 필요) |
| 구현 | iOS는 ATS 내장(`NSPinnedDomains` → `NSPinnedCAIdentities` / `SPKI-SHA256-BASE64`), Android는 `network_security_config`의 `pin-set` | Expo 앱이면 **네이티브 설정(config plugin)** 필요, 양 플랫폼 따로 |

**권고**: K-Bap 규모·단계에선 **아직 넣지 말 것.** 넣게 된다면 애플 권고대로 ①리프가 아니라 **CA 공개키를 핀** ②**백업 핀을 복수로** ③**핀 만료/복구 경로(강제 업데이트)** 를 먼저 설계. 그전에 값싼 방어(짧은 토큰 수명·refresh 로테이션·서버측 이상탐지)가 먼저다.

---

## 9. Android는 왜 세팅이 한 단계 더 필요한가 (참고)

| | **iOS** | **Android 7(API 24)+** |
|---|---|---|
| 사용자가 설치한 CA를 앱이 신뢰? | **예** — 완전 신뢰만 켜면 **모든 앱**에 적용 | **아니오** — 기본은 **시스템 CA만** 신뢰 |
| MITM 디버깅에 필요한 것 | 프로파일 설치 + 완전 신뢰 ON | 위 + **앱에 `network_security_config.xml` 추가**해 `user` 저장소를 허용하도록 **재빌드** |
| 함정 | 끝나고 인증서 안 지우면 위험 | 그 설정을 **릴리스 빌드에 남기면 프로덕션이 MITM에 취약** (공식 문서 경고) |

→ 안드로이드에서 같은 디버깅을 하려면 "앱을 고쳐서 다시 빌드"가 필요하다. iOS가 특별히 관대한 게 아니라, **신뢰 결정 주체가 OS냐 앱이냐**의 차이. (안드로이드 서명·신뢰 이야기는 [[Android 구글 로그인 — SHA-1·OAuth 클라이언트·DEVELOPER_ERROR]]와 같은 사고방식이다.)

---

## 10. 곁가지 두 개

**① iPhone Mirroring은 왜 이 문제와 무관한가**
미러링은 **화면 픽셀과 입력 이벤트**를 맥으로 스트리밍하는 Continuity 기능이다. 아이폰의 HTTP 요청은 여전히 **아이폰의 네트워크 스택 → 아이폰의 Wi-Fi**로 나간다. 맥은 "화면을 보고 있을 뿐" 경로상 아무 데도 끼어 있지 않다. → **미러링을 켜도 트래픽은 안 잡히고, 프록시를 켜면 미러링 없이도 잡힌다.** 둘은 완전히 다른 층(표현 vs 네트워크).

**② 보안 주의 — 이건 진짜 위험한 스위치다**
공식 문서도 굵게 경고한다: *"Make sure that you **delete the certificate on your iPhone** when you're not debugging by Proxyman. If not, your HTTP/HTTPS requests can be intercepted and leak your sensitive data."*
- 신뢰된 루트 CA = **그 개인키를 가진 자는 어떤 사이트로도 위장 가능**. 맥이 털리면 아이폰 HTTPS가 통째로 열린다.
- 카페 Wi-Fi에서 프록시를 켜둔 채 은행 앱을 쓰지 말 것. 안 쓸 땐 **프록시 Off + 프로파일 삭제**.
- 캡처 화면/HAR 파일엔 **토큰·개인정보가 평문**이다. 이슈에 첨부하기 전 마스킹.

---

## 11. 대안 도구 — 언제 뭘 쓰나

| 도구 | 정체 | 언제 쓰나 | 비고 |
|---|---|---|---|
| **Proxyman** | macOS 네이티브 MITM 프록시 | **기본값.** 실기기·시뮬레이터 다 되고 UI가 제일 편함 | Map Local·Breakpoint(요청/응답 실시간 편집)·No Caching. 유료(무료 제한) |
| **Charles** | 원조 Java 기반 MITM 프록시 | 팀에 이미 세팅이 있거나 Windows/Linux 혼용 | UI가 낡음, 동작은 동일 원리 |
| **mitmproxy** | 오픈소스 CLI/웹 | **자동화**(스크립트로 요청 변조·기록), CI, 무료 | 파이썬 애드온으로 뭐든 가능, 학습곡선 |
| **RN DevTools의 Network 패널** | RN 내장(0.83+) | **앱이 보낸 fetch/XHR만** 빠르게 볼 때 — 인증서·프록시 세팅 **불필요** | kbap-fe는 **RN 0.85.3**이라 이미 있다. 단 WebSocket·모킹·throttling 미지원, **네이티브 레이어 요청은 안 보임** |
| ~~Flipper~~ | 구 RN 디버깅 플랫폼 | — | RN 0.74에서 기본 템플릿에서 **제거**됨. 신규 프로젝트는 고려 대상 아님 |

> **실전 판단**: "JS에서 보낸 요청 파라미터가 궁금하다" → **RN DevTools Network 탭**(공짜, 30초). "네이티브가 보내는 것까지 포함해 실제 와이어를 보고 싶다 / 응답을 바꿔치기해 화면을 테스트하고 싶다(Map Local) / 실기기에서 재현되는 것만 잡고 싶다" → **Proxyman**.

---

## 12. 치트시트

```bash
# 맥의 진짜 Wi-Fi IP (아이폰 프록시에 넣을 값)
ipconfig getifaddr en0

# 맥의 현재 시스템 프록시 상태 확인 (Proxyman이 켜지면 여기가 바뀐다)
networksetup -getwebproxy "Wi-Fi"
networksetup -getsecurewebproxy "Wi-Fi"

# 프록시를 통해 서버 응답 확인 (터널만 / MITM 검증)
curl -x http://127.0.0.1:9090 https://dev.kbap.site/ -v

# 서버가 실제로 내미는 인증서 체인 보기 (누가 서명했나 = 진짜인가 Proxyman인가)
openssl s_client -connect dev.kbap.site:443 -servername dev.kbap.site </dev/null 2>/dev/null \
  | openssl x509 -noout -subject -issuer -dates

# Proxyman CA 파일 위치 (맥마다 다르게 생성됨)
ls -l ~/Library/Application\ Support/com.proxyman.NSProxy/app-data/proxyman-ca.pem
openssl x509 -in ~/Library/Application\ Support/com.proxyman.NSProxy/app-data/proxyman-ca.pem \
  -noout -subject -fingerprint -sha256   # 아이폰에 신뢰된 것과 지문 대조용
```

```text
아이폰 경로 3개 (외우면 편함)
  프록시  : 설정 → Wi-Fi → (i) → Configure Proxy → Manual
  설치    : 설정 → 일반 → VPN 및 기기 관리
  ⭐완전신뢰: 설정 → 일반 → 정보 → 인증서 신뢰 설정
```

---

## 13. 용어집

- **프록시(proxy)**: 클라이언트와 서버 사이에서 요청을 대신 전달하는 중간 서버.
- **포워드 / 리버스 / 투명 프록시**: 클라이언트 쪽 대리인 / 서버 쪽 접수원 / 몰래 끼어든 검문소.
- **HTTP CONNECT**: HTTPS를 프록시로 보낼 때 쓰는 메서드. 프록시에 "목적지까지 TCP 터널만 뚫고 내용은 그대로 흘려라"라고 지시. 이 상태에선 프록시가 **호스트·포트만** 안다.
- **MITM (Man-In-The-Middle)**: 통신 중간에서 양쪽 행세를 하며 내용을 읽는 것. 공격 기법이지만, **내가 내 기기에 허락하면** 디버깅 도구가 된다.
- **SSL Proxying**: Proxyman/Charles 용어로 "이 도메인은 MITM으로 복호화해서 보여줘" 설정.
- **루트 CA / 인증서 체인**: 인증서를 서명한 상위 기관들의 사슬. 맨 위 루트가 기기 신뢰 저장소에 있어야 검증 통과.
- **자체 서명(self-signed)**: 자기 개인키로 자기를 서명한 인증서. Proxyman CA가 이것 — 신뢰는 **기기가 수동으로 허락**해야 생긴다.
- **trust store(신뢰 저장소)**: OS가 들고 있는 "믿을 만한 루트 CA 목록". iOS는 시스템 것 + 사용자가 완전 신뢰한 것.
- **Certificate Trust Settings / Full Trust**: iOS에서 수동 설치한 루트 CA를 **SSL에 한해** 신뢰시키는 별도 스위치. 설치 ≠ 신뢰.
- **TLS 핸드셰이크**: 암호 통신 시작 전 신원 확인·키 합의 절차. 인증서 검증 실패 시 **데이터 전송 전에 끊긴다**.
- **SNI (Server Name Indication)**: ClientHello에 담기는 접속 대상 도메인 이름(암호화 전). 한 IP에 여러 사이트가 있을 때 서버가 알맞은 인증서를 고르는 근거이자, **프록시가 도메인을 아는 방법**.
- **certificate pinning (SSL pinning)**: 앱이 특정 인증서/공개키만 허용하도록 못박는 방어. OS 신뢰와 무관 → MITM 차단.
- **SPKI 해시**: 인증서의 공개키(Subject Public Key Info)를 SHA-256으로 요약한 값. 피닝의 단위(`SPKI-SHA256-BASE64`).
- **status 999**: HTTP 표준이 아닌 **Proxyman 내부 코드**. "응답을 못 받았다" ≒ 대부분 TLS 핸드셰이크 실패.
- **NSURLSession / CFNetwork**: iOS 네트워크 스택. **시스템 프록시 설정을 기본으로 따르기** 때문에 RN `fetch`가 자동으로 프록시를 탄다.
- **network_security_config**: 안드로이드에서 앱의 신뢰 정책(사용자 CA 허용·핀셋)을 정의하는 XML. 릴리스에 남기면 위험.
- **DHCP / 사설 IP**: 공유기가 기기에 IP를 자동 배정하는 방식 / `192.168.x.x` 같은 내부망 주소. → 자세히는 [[Metro 연결 실패 — Mac·폰 IP 불일치 (link-local·서브넷)]]

---

## 14. 다음에 빠른 재현 체크리스트

1. 맥·아이폰 **같은 Wi-Fi**? VPN 다 껐나?
2. `ipconfig getifaddr en0` → 아이폰 Wi-Fi 프록시 **Manual = 그 IP:9090** (Auth No)
3. 트래픽이 잡히나? (`CONNECT 200`이면 연결은 성공 — 여기까진 내용 안 보이는 게 정상)
4. 내용 보려면 → 해당 도메인 **우클릭 → Enable SSL Proxying**
5. **999 뜨면 → 인증서 신뢰**: `proxy.man/ssl` 재설치 → 설정 → 일반 → **정보 → 인증서 신뢰 설정 ON** (맥 바꿨거나 재설치했으면 **무조건 다시**)
6. 그래도 999인데 **Safari는 정상** → 그 앱의 **피닝** (포기하거나 Bypass 목록에)
7. 끝나면 **프록시 Off** (안 그러면 아이폰 인터넷 먹통 + 보안 위험), 장기 미사용이면 프로파일 삭제
8. 가벼운 확인이면 애초에 **RN DevTools Network 탭**으로 끝낼 수 있는지 먼저 판단

---

## 15. 참고

- [Proxyman Docs — iOS Device 설정](https://docs.proxyman.com/debug-devices/ios-device) (프록시 설정·`proxy.man/ssl`·완전 신뢰·VPN 경고·인증서 삭제 권고)
- [Proxyman Docs — macOS 인증서 설치](https://docs.proxyman.com/debug-devices/macos) ("self-signed certificate that is generated on your machine", CA 파일 경로)
- [Proxyman Docs — Android 설정](https://docs.proxyman.com/debug-devices/android-device-emulator) (`network_security_config`, 릴리스 빌드 주의)
- [Proxyman Issue #922 — "999 is an internal code"](https://github.com/ProxymanApp/Proxyman/issues/922) (메인테이너 답변 + 피닝 케이스)
- [Apple — Trust manually installed certificate profiles in iOS, iPadOS, and visionOS](https://support.apple.com/en-us/102390) ("isn't automatically trusted for SSL")
- [Apple — Identity Pinning: How to configure server certificates for your app](https://developer.apple.com/news/?id=g9ejcf8y) (`NSPinnedDomains`·`SPKI-SHA256-BASE64`·로테이션 경고)
- [Android — Network security configuration](https://developer.android.com/privacy-and-security/security-config) (사용자 CA·pin-set)
- [RFC 9110 §9.3.6 — CONNECT](https://www.rfc-editor.org/rfc/rfc9110#name-connect) (터널·blind forwarding)
- [React Native — DevTools Network 패널](https://reactnative.dev/docs/react-native-devtools) (0.83+, fetch/XHR/Image, WebSocket·모킹 미지원)

---

> [!note] 프록시를 펼쳐야만 보이는 것 하나 (2026-08-26 추가)
> kbap 서버는 API 계약 버전을 URL이 아니라 `X-API-Version` **요청 헤더**로 가른다. 그래서 경로만 봐서는 이 요청이 어느 계약으로 처리됐는지 알 수 없고, 이 노트의 SSL Proxying으로 헤더를 펼쳐 보는 것이 사실상 유일한 확인 수단이다(헤더를 빼먹으면 400 `COMMON-002`) → [[7. API 계약의 모양 — 공통 봉투·에러 코드·헤더 버저닝]]
