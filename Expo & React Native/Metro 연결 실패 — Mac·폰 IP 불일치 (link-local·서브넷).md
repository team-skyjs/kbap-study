---
tags: [expo, react-native, metro, networking, 트러블슈팅]
생성일: 2026-07-06
상태: 완료
---

> [!info] 관련 노트
> 🏠 [[🏠 홈]] · 📂 주제: Expo & React Native
> 형제: [[EAS Dev Build vs Expo Go]] · [[로컬 빌드 vs EAS 클라우드 빌드 — K-Bap vs winemine]]

# Metro 연결 실패 — Mac·폰 IP 불일치 (link-local·서브넷)

> 작성: 2026-07-06 · 맥락: K-Bap 실기기 테스트 중 `npx expo start --dev-client`로 폰을 Metro에 붙이려는데 계속 연결 실패.
> 목표: "왜 폰이 Mac의 Metro를 못 찾았나 / 다음에 또 나면 뭘 보나"를 30분 뒤의 나도 재현.
> 상태: 시드 노트를 **리서치로 검증 완료** — 아래 2·4·5가 채워진 부분.

---

## 0. 한 줄 요약
Mac에 **네트워크 인터페이스가 여러 개**(WiFi + Thunderbolt Bridge/iPhone USB/가상 어댑터)라, Metro가 IP를 자동 선택할 때 **폰이 도달할 수 없는 IP**(링크로컬 `169.254.x` 또는 다른 서브넷)를 골라 광고해서 실패했다. **진짜 WiFi IP를 `REACT_NATIVE_PACKAGER_HOSTNAME`으로 고정**하거나 **`--tunnel`** 을 쓰면 됨.

---

## 1. 무슨 일이 있었나 (증상)
- 명령: `npx expo start --dev-client` → 폰의 dev 런처에서 서버를 탭.
- 폰 화면 에러: **`Error loading app — Failed to connect to http://172.16.103.236:8081`**
- 런처 "DEVELOPMENT SERVERS"에 잡힌 주소: **`http://169.254.65.106:8081`** (← 링크로컬이라 폰이 못 씀)
- 실행할 때마다 Metro가 광고하는 IP가 바뀜: `172.16.102.238` → `172.16.103.236`, 런처엔 `169.254.x`.
- Mac 인터페이스 실측 (`ipconfig getifaddr en0`, `ifconfig -lu`):
  ```
  en0 : 192.168.0.100   ← 실제 WiFi, default route (폰이 닿을 수 있는 유일한 것)
  en9 : 169.254.65.106  ← 링크로컬 (Expo가 이걸 광고함 ❌)
  en4 : 169.254.216.95  ← 링크로컬
  en10: 169.254.128.208 ← 링크로컬
  ```
- 폰은 `192.168.0.x` WiFi에 있었음 → `169.254.x`나 `172.16.x`로는 도달 불가.

---

## 2. 진짜 원인 (검증된 메커니즘)

### 2-1. `169.254.x`는 뭐고 어디서 생기나 — Q1 답 ✅
**`169.254.0.0/16` = 링크로컬 = APIPA(Automatic Private IP Addressing).** DHCP 서버로부터 IP를 못 받은 인터페이스가 **스스로 부여**하는 임시 주소. 핵심 성질:
- **라우팅되지 않음** — 라우터/스위치가 이 주소로 온 패킷을 그냥 버림. 같은 물리 링크 안에서만 유효.
- 그래서 다른 기기(폰)가 이 IP로 Mac에 절대 못 붙음. macOS에선 "self-assigned IP address" 경고로도 뜸.

**en4/en9/en10이 링크로컬인 이유** = 각각 DHCP 없는 인터페이스라서:
- **Thunderbolt Bridge (`bridge0`)**: macOS 기본 인터페이스. DHCP 서버가 없어서 **설계상 항상 `169.254.x`** 를 자가 부여. (맥끼리 케이블 연결용)
- **iPhone USB 연결**: 아이폰을 케이블로 꽂으면 tethering용 `en` 인터페이스가 생기는데, 공유(핫스팟)가 아니면 DHCP를 못 받아 `169.254.x`.
- **가상화(UTM/VMware/VirtualBox/Parallels)**: 가상 네트워크 어댑터를 만듦. 자체 DHCP가 없으면 링크로컬. (VirtualBox 어댑터가 우선순위를 뺏는 건 유명한 함정.)
- VPN은 보통 `utun`(10.x/CGNAT)이라 이 케이스는 아님.

> [!note] 왜 매번 IP가 바뀌었나
> APIPA는 `169.254.x.x`를 **무작위로 골라 충돌 검사** 후 부여. 그래서 실행마다 `65.106`, `216.95`처럼 값이 달라졌던 것.

### 2-2. Metro는 왜 틀린 IP를 골랐나 + HOSTNAME이 뭘 덮나 — Q2 답 ✅
Metro/Expo는 dev server를 켤 때 폰에게 광고할 **호스트 IP를 자동 탐지**한다. 문제는 이 자동 탐지 로직이 **"인터페이스 목록에서 첫 번째 비-내부(non-internal) IPv4"** 를 고르는 식이라, 인터페이스가 여러 개면 **default route(en0=WiFi)가 아닌 엉뚱한 걸** 집을 수 있다. (VirtualBox·Thunderbolt Bridge가 순서상 앞서면 그걸 광고.)

- 이렇게 잘못 고른 IP가 **런처 URL, 번들(JS) URL, HMR(핫리로드) 소켓 주소**에 전부 그대로 박힘 → 폰이 그 IP로 접속 시도 → 실패.

**`REACT_NATIVE_PACKAGER_HOSTNAME`** = 이 자동 탐지를 **수동으로 덮어쓰는** 환경변수. 여기 넣은 값이 Metro가 광고하는 **모든 URL의 호스트 부분**(dev server URL + 번들 URL + HMR 웹소켓 호스트)에 일괄 적용됨. 그래서 진짜 WiFi IP를 넣으면 폰이 도달 가능한 주소로 광고됨.
```
REACT_NATIVE_PACKAGER_HOSTNAME=192.168.0.100 npx expo start --dev-client
```
> ⚠️ 과거 Expo엔 이 변수를 일부 URL이 안 따르는 버그 이력이 있었음(expo-cli #262). 최신 Expo에선 대체로 정상. 안 먹으면 아래 3-B(수동 입력)로.

### 2-3. 서브넷/방화벽 변수 (IP가 맞아도 막히는 경우)
- **다른 서브넷/게스트망·AP isolation**: 폰과 Mac이 같은 공유기라도 게스트 WiFi거나 AP isolation이 켜져 있으면 LAN 직접 연결 차단 → 올바른 IP라도 실패. → `--tunnel`.
- **macOS 방화벽**: 8081 인바운드를 막으면 실패. (시스템 설정 → 네트워크 → 방화벽 확인.)

---

## 3. 해결책 (실전, 위→아래 순서로)
- **A. 올바른 IP로 고정 (근본)**:
  ```
  REACT_NATIVE_PACKAGER_HOSTNAME=192.168.0.100 npx expo start --dev-client
  ```
- **B. 폰 런처에서 수동 URL 입력**: `http://192.168.0.100:8081` (폰이 같은 `192.168.0.x` WiFi일 때)
- **C. mDNS 이름으로 입력(IP 변동 회피)**: 런처에 `http://<맥이름>.local:8081` 입력. iOS는 Bonjour(mDNS)를 지원해서, IP가 바뀌어도 이름으로 찾음. (단 같은 네트워크 + mDNS 미차단 조건. 회사망은 mDNS를 막는 경우 있음.)
- **D. 터널 (네트워크/방화벽 무관, 최후)**:
  ```
  npx expo start --dev-client --tunnel
  ```
- 공통 전제: A/B/C는 **폰과 Mac이 같은 WiFi/서브넷**. 게스트망·AP isolation이면 D.

---

## 4. `--tunnel`은 어떻게 방화벽/서브넷을 우회하나 — Q3 답 ✅
`--tunnel`은 **ngrok**(`@expo/ngrok`)을 써서 로컬 dev server를 **공용 URL**(예: `https://xxxx.bacon.8081.exp.direct`)로 노출한다.

**원리 (아웃바운드 터널):**
1. Mac이 ngrok **서버로 바깥으로 나가는(outbound) 연결**을 하나 연다.
2. 방화벽/NAT는 **나가는 연결은 기본 허용**하므로, 인바운드 포트를 안 열어도 통로가 생김.
3. 폰은 로컬 IP가 아니라 **인터넷의 공용 URL**로 접속 → ngrok 서버가 그 요청을 아까 열린 터널로 Mac에 전달.
4. 그래서 **폰·Mac이 다른 네트워크여도, 서브넷이 달라도, 방화벽이 인바운드를 막아도** 됨. (LAN 직접연결 불필요)

**왜 느린가:** 모든 번들 요청·핫리로드가 **인터넷 → ngrok 서버 → 다시 Mac** 을 왕복. LAN 직결(1~수ms)과 달리 원거리 홉이 끼어, 큰 프로젝트는 첫 번들 로드가 30초+ 걸리기도. 그래서 "될 때는 A/B, 안 될 때만 D".

---

## 5. 용어집
- **링크로컬 / APIPA (`169.254.0.0/16`)**: DHCP 실패 시 인터페이스가 자가 부여하는 주소. **라우팅 안 됨**(같은 물리 링크 내에서만). ↔ 사설 IP(`192.168/16`·`10/8`·`172.16/12`)는 **라우팅 가능**(공유기가 서브넷 간 전달).
- **사설 IP (private IP)**: 내부망용 IP 대역. 공유기가 NAT로 인터넷과 연결. 폰이 닿을 수 있는 건 이쪽.
- **서브넷 / 서브넷 마스크**: 두 기기가 "같은 LAN(직접 통신 가능)"인지 판정하는 기준. 마스크 밖이면 라우터를 거쳐야 함.
- **default route (기본 게이트웨이)**: 목적지가 로컬에 없을 때 내보내는 기본 통로(여기선 en0=WiFi). **Metro의 IP 자동선택은 default route를 안 따르고 인터페이스 순서로 골라서** 틀린 걸 집었음 — 이게 문제의 핵심.
- **DHCP vs static vs link-local**: IP를 받는 3경로 — 서버가 자동 배정 / 수동 고정 / DHCP 실패 시 자가부여(169.254).
- **Thunderbolt Bridge (`bridge0`)**: 맥끼리 케이블 연결용 기본 인터페이스. DHCP 없어 항상 링크로컬.
- **`REACT_NATIVE_PACKAGER_HOSTNAME`**: Metro가 광고할 호스트를 수동 지정하는 환경변수. dev URL·번들 URL·HMR 소켓 호스트에 일괄 적용.
- **HMR (Hot Module Replacement)**: 코드 저장 시 앱을 통째로 재시작 않고 바뀐 모듈만 교체하는 핫리로드. 별도 웹소켓 주소를 씀 → 이것도 IP가 맞아야 함.
- **mDNS / Bonjour (`*.local`)**: IP 대신 이름으로 로컬 기기를 찾는 프로토콜. iOS 지원. IP 변동 회피용.
- **ngrok / `--tunnel`**: 아웃바운드 터널로 로컬 서버를 공용 URL에 노출. 방화벽/서브넷 우회, 대신 느림.
- **AP isolation**: 같은 WiFi에 붙은 기기끼리 통신을 막는 공유기 옵션(게스트망 흔함). 켜져 있으면 LAN 직결 불가.

---

## 6. 다음에 빠른 재현 체크리스트
1. `ipconfig getifaddr en0` → Mac의 **진짜 WiFi IP** 확인 (`169.254` 아님).
2. 폰 WiFi가 **같은 서브넷(예: `192.168.0.x`)** 인지 확인.
3. `REACT_NATIVE_PACKAGER_HOSTNAME=<그 IP> npx expo start --dev-client` 로 고정.
4. 안 되면 런처에서 `http://<그 IP>:8081` **수동 입력**.
5. IP가 계속 바뀌어 귀찮으면 `http://<맥이름>.local:8081` (mDNS).
6. 그래도 안 되면 `--tunnel` (다른 망·방화벽 우회).
7. 방화벽(시스템 설정 → 네트워크 → 방화벽) 8081 인바운드 확인.
8. (선택) 안 쓰는 가상 어댑터/Thunderbolt Bridge를 끄면 자동선택 오작동이 줄어듦.

---

## 7. 참고 링크
- [Thunderbolt Bridge & 169.254 (Astropad)](https://astropad.com/blog/thunderbolt-bridge/)
- [QR/Network URL이 링크로컬을 광고하는 이슈 (listhen #225)](https://github.com/unjs/listhen/issues/225)
- [Expo가 REACT_NATIVE_PACKAGER_HOSTNAME을 안 따르던 이슈 (expo-cli #262)](https://github.com/expo/expo-cli/issues/262)
- [Expo CLI 문서 (tunnel 등)](https://docs.expo.dev/more/expo-cli/)
- [@expo/ngrok](https://www.npmjs.com/package/@expo/ngrok)

---

> [!note] 같은 IP 지식이 또 쓰인 곳 — 폰 트래픽을 맥에서 훔쳐보기 (2026-08-21 추가)
> §1의 `ipconfig getifaddr en0`(맥의 진짜 Wi-Fi IP)·"같은 서브넷이어야 함"·"DHCP라 IP가 바뀐다"가 **Proxyman 디버깅 프록시** 세팅에 그대로 재활용된다 — 거기선 그 IP를 아이폰 Wi-Fi의 수동 프록시 주소(`192.168.0.100:9090`)로 넣는다. 다만 그다음 벽은 IP가 아니라 **인증서 신뢰**(설치 ≠ 완전 신뢰)다 → [[프록시와 MITM 디버깅 (Proxyman)]]

> [!note] 같은 그림의 반대편 — 이번엔 **내가 그 서버를 짠다** (2026-08-27 추가)
> 이 노트에서 애먹은 것들(포트 `8081` 을 한 프로그램이 점유한다 / 폰 입장의 `localhost` 는 **폰 자신**이라 맥의 Metro 에 안 닿는다 / 그래서 맥의 LAN IP 를 써야 한다)은 Metro 의 사정이 아니라 **서버라는 프로그램의 일반 사정**이다. 같은 세 가지가 kbap 백엔드에서 그대로 반복된다 → [[1. 서버는 켜두는 프로그램 — 프로세스·포트·두 개의 실행 파일]] §2-3·§7-1.
> 구체적으로 겹치는 자리: **① 포트 점유** — `:api` 와 `:batch` 가 둘 다 8080 이라 같이 띄우면 `Port 8080 was already in use` 로 죽는다(`lsof -i :8080` 으로 범인을 찾는 것까지 같다). **② 폰의 `localhost`** — 시뮬레이터/실기기에서 `http://localhost:8080` 으로 백엔드를 부르면 안 닿는다. 원인이 이 노트 §1 과 정확히 같다. **③ 기다리는 프로세스** — `npx expo start` 를 켜 두면 터미널 커서가 멈춘 채 안 돌아오는 그 상태가, 백엔드에서 말하는 "서버는 켜 두는 프로그램" 의 대기 상태 그 자체다. **Metro 를 띄워 본 적이 있다면 이미 서버를 켜 본 것**이다.
