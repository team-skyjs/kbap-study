---
tags: [expo, react-native, eas, 모바일개발]
생성일: 2026-07-02
상태: 완료
---

> [!info] 관련 노트
> 🏠 [[🏠 홈]] · 📂 주제: Expo & React Native
> 🔗 함께 보기: [[로컬 빌드 vs EAS 클라우드 빌드 — K-Bap vs winemine]] (Dev Build를 만드는 2가지 방법) · [[iOS 앱 용량 — 네이티브 의존성과 ML Kit 모델]] (OTA 불가한 네이티브 변경의 실사례)

# EAS Dev Build vs Expo Go — 개념 + 셋업 정리 (다음에도 쓰는 용)

> 작성: 2026-07-02 · 맥락: K-Bap(kbap-fe, Expo SDK 56) 실기기 OCR 테스트 환경 구축하며 정리.
> 목표: "이거 왜 했고, 다음 프로젝트에서 또 어떻게 하지?"를 30분 뒤의 나도 바로 재현.

---

## 0. 한 줄 요약
**Expo Go는 남이 만든 껍데기 앱에 내 JS만 얹어 보는 것**이고, **Dev Build는 내 네이티브 코드까지
넣어 내가 직접 만든 개발용 앱**이다. **커스텀 네이티브 모듈(예: 온디바이스 ML Kit OCR)을 쓰면
Expo Go로는 안 되고 Dev Build가 필수.** 그래서 Apple 유료 계정으로 기기 등록 → EAS로 dev build를
만들어 폰에 깔았다. 이후 **JS 변경은 재빌드 없이 OTA(eas update)** 로 폰에 바로 반영.

---

## 1. 3가지 앱 형태 (이게 핵심 개념)

| | Expo Go | **Dev Build (우리가 쓴 것)** | Production Build |
|---|---|---|---|
| 정체 | 앱스토어의 **범용 컨테이너** 앱 | **내 앱**의 개발자 버전(내 번들ID) | 출시용 최종 앱 |
| 네이티브 모듈 | Expo SDK 내장분만 | **아무 네이티브 모듈 OK**(ML Kit 등) | 좌동 |
| 개발 편의(dev menu·핫리로드) | 있음 | 있음 | 없음 |
| 설치 방법 | 앱스토어에서 다운 | EAS로 빌드 → 링크/USB로 설치 | 앱스토어/TestFlight |
| 서명(Apple 계정) | 불필요 | **필요**(기기 등록 or 배포 인증서) | 필요 |
| 언제 | 순수 JS 프로토타입 | **커스텀 네이티브 필요할 때** | 출시 |

**왜 우리는 Dev Build였나:** `@react-native-ml-kit/text-recognition`(온디바이스 OCR)은
**네이티브 코드**라 Expo Go의 껍데기엔 안 들어 있음 → Expo Go에선 절대 안 돌아감 → Dev Build 필수.
(iOS 시뮬레이터도 ML Kit는 불안정 → **실기기** dev build로 감.)

> 비유: Expo Go = "공용 렌터카에 내 짐(JS)만 싣고 시승". 네이티브 모듈 = "엔진 개조".
> 개조는 렌터카에 못 하니 **내 차(dev build)** 를 만들어야 한다.

---

## 2. 사전 준비물 (1회)
- **Apple 유료 개발자 계정**($99/년) — 실기기 설치에 필요.
- **Expo 계정** — EAS 빌드 클라우드. (로그인 팁: GitHub 이메일이 이미 Expo에 연결돼 있으면
  GitHub "회원가입"이 아니라 **이메일+비번으로 로그인**. 안 그러면 로그인 루프에 빠짐.)
- **EAS CLI**: `npm i -g eas-cli` → `eas login`.
- 프로젝트에 `eas.json` (없으면 `eas build:configure`가 만들어줌).

우리 `eas.json`의 dev 프로필(핵심 두 줄):
```json
"development": { "developmentClient": true, "distribution": "internal" }
```
- `developmentClient: true` → dev 메뉴/핫리로드 되는 **개발용** 빌드.
- `distribution: "internal"` → 앱스토어 안 거치고 **링크/USB로 내부 설치**.

---

## 3. 셋업 절차 (우리가 실제로 한 순서)

### 3-1. 기기 등록 (ad-hoc: 이 아이폰에 설치 허용)
```
eas device:create
```
→ 나오는 **URL/QR을 폰에서 열어 프로비저닝 프로파일 설치** → 이 기기가 "이 앱 깔아도 되는 기기"로 등록됨.

> ⚠️ 실제로 막혔던 지점: 프로파일 설치가 "Safari could not install a profile / unknown error"로 실패.
> 원인 = 아이폰 **Stolen Device Protection(도난 기기 보호)** 이 보안 지연을 걸어서였음.
> 해결 = 결국 **USB 케이블로 기기 등록**해서 우회. (SDP 끄기/1시간 대기/익숙한 위치 모두 실패했음.)

### 3-2. Dev Build 빌드 (클라우드)
```
eas build --profile development --platform ios
```
- 클라우드에서 빌드(수 분~십수 분). 처음이면 **Apple Distribution 인증서 생성? → Y**,
  **EAS project 등록? → Y**, **Export compliance(암호화) → 보통 yes**(우리 앱은 표준 https만 →
  app.json에 `ITSAppUsesNonExemptEncryption: false`로 처리).
- 끝나면 **빌드 상세 페이지(expo.dev)** 에 QR/설치 링크가 뜸.

### 3-3. 폰에 설치
- 빌드 페이지의 **QR/링크를 폰에서 열어 설치**. (CLI 터미널에 QR이 항상 뜨진 않음 — 웹 페이지에서 받으면 됨.)
- 이제 홈화면에 **내 앱(K-Bap dev)** 아이콘이 생김. 이게 dev client.

### 3-4. 코드 붙여서 개발
두 가지 방법:
- **(집·같은 WiFi) 로컬 Metro**: `npx expo start --dev-client` → 폰의 dev 앱에서 서버 선택/QR → 핫리로드.
- **(밖·다른 네트워크) OTA**: 아래 4번.

---

## 4. 코드 변경을 폰에 반영하는 법 (제일 자주 쓰는 것)

**핵심 규칙: 무엇을 바꿨냐에 따라 다르다.**

| 바꾼 것 | 반영 방법 | 재빌드? |
|---|---|---|
| **JS/TS·화면·스타일** | `eas update` (OTA) 또는 로컬 Metro 리로드 | ❌ 불필요 |
| **네이티브**(새 native 패키지, app.json plugins/권한/번들ID, SDK 업) | `eas build` 다시 → 재설치 | ✅ 필요 |

### OTA (EAS Update) — 밖에서 폰만으로 최신 JS 보기
1회 셋업: `eas update:configure` (expo-updates 설치 + app.json에 updates/runtimeVersion + eas.json에 channel).
매번:
```
eas update --branch development --message "무엇 바꿨는지"
```
→ 폰의 dev 앱 **Updates 탭에서 방금 업데이트 선택** → 인터넷으로 받아 로드. (Mac 꺼져 있어도 됨.)

- **runtimeVersion이 빌드와 업데이트 간 일치**해야 폰이 받음. 우리는 **fingerprint 정책** 사용
  (네이티브 지문이 같으면 호환). `--branch`(=채널)가 빌드 채널과 일치해야 함(우린 `development`).
- **주의**: OTA는 **JS만** 갈아끼움. 네이티브 바뀌면 OTA로 안 되고 재빌드.
- 지금 폰에 깔린 빌드는 OTA 수신 가능(expo-updates 포함, runtime `8c9263af`).

---

## 5. 트러블슈팅 (실제로 겪은 것)
- **프로파일 설치 "unknown error"** → 아이폰 Stolen Device Protection 지연. **USB로 기기 등록** 우회.
- **빨간 화면 + MIME-type 에러** → dev 빌드가 **죽은 Metro 서버**를 부르는 중. Dismiss →
  앱 안 ⚙️(또는 폰 흔들기) → "Go to home" → **Updates 탭에서 원하는 업데이트 탭**해서 로드.
- **Expo 로그인 루프** → GitHub로 "회원가입" 말고 **이메일+비번 로그인**.
- **CLI에 QR 안 뜸** → expo.dev 빌드 상세 페이지에서 QR/링크 받기.

---

## 6. 명령어 치트시트
```
npm i -g eas-cli           # EAS CLI 설치
eas login                  # Expo 로그인 (이메일+비번)
eas device:create          # 이 아이폰 등록 (안되면 USB)
eas build --profile development --platform ios   # dev build (클라우드)
# → expo.dev 빌드페이지 QR/링크로 폰에 설치

npx expo start --dev-client   # 집: 로컬 Metro 연결(핫리로드)
eas update:configure          # (1회) OTA 셋업
eas update --branch development --message "..."   # 밖: OTA로 JS 배포
```

---

## 7. 용어집
- **Expo Go**: 앱스토어에 있는 범용 Expo 실행기. 커스텀 네이티브 ✗.
- **Dev Build / dev client**: 내 번들ID로 만든 개발용 앱. 네이티브 O + dev도구 O.
- **EAS**: Expo Application Services(클라우드 빌드/업데이트/제출).
- **OTA(eas update)**: 앱 재설치 없이 JS 번들만 무선 교체.
- **runtimeVersion / fingerprint**: 빌드와 OTA의 "네이티브 호환 키". 같아야 업데이트가 로드됨.
- **channel / branch**: OTA 배포 줄기(우린 `development`).
- **ad-hoc / provisioning profile**: "이 기기에 이 앱 설치 허용" 애플 서명 장치.
- **distribution: internal**: 앱스토어 안 거치고 링크/USB로 내부 배포.

---

## 8. 다음 프로젝트 빠른 재현 체크리스트
1. Apple 유료 계정 + Expo 계정 확인.
2. `eas login` → `eas build:configure`(eas.json 생성).
3. dev 프로필에 `developmentClient:true, distribution:"internal"` 확인.
4. `eas device:create`로 기기 등록(막히면 USB).
5. `eas build --profile development --platform ios` → 폰 설치.
6. 개발: 집=`expo start --dev-client`, 밖=`eas update:configure`(1회)+`eas update`.
7. 네이티브 바꾸면 재빌드, JS만 바꾸면 OTA.
