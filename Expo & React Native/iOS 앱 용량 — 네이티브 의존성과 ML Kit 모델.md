---
tags: [expo, react-native, ios, 앱용량, mlkit, cocoapods, patch-package, 네이티브빌드]
생성일: 2026-07-14
상태: 완료
---

> [!info] 관련 노트
> 🏠 [[🏠 홈]] · 📂 주제: Expo & React Native
> 🔗 함께 보기: [[로컬 빌드 vs EAS 클라우드 빌드 — K-Bap vs winemine]] (빌드가 어디서 도는지) · [[OCR 메뉴 스캔 — 박스 그룹핑]] · [[OCR 라인 분류 — 휴리스틱의 한계와 대안]] (이 ML Kit을 쓰는 기능)

# iOS 앱 용량 — 네이티브 의존성과 ML Kit 모델

> 작성: 2026-07-14 · 맥락: K-Bap TestFlight 다운로드가 **~450MB**. 범인은 `@react-native-ml-kit/text-recognition`이 ML Kit 텍스트 인식 모델 **5종(Latin·Chinese·Devanagari·Japanese·Korean)을 전부** 바이너리에 박아 넣는 구조였음. 우리는 `ocr.ts`에서 KOREAN 하나만 씀.
> 목표: "왜 한 번도 안 부르는 코드가 앱을 450MB로 만드는가"를 링커 수준에서 이해하고, patch-package 해법이 EAS 빌드에서도 살아남는 이유를 재현 가능하게 정리.

---

## 0. 한 줄 요약
**JS의 tree-shaking 직관은 네이티브 레이어에서 통하지 않는다.** RN 라이브러리 하나가 podspec에 하드코딩한 pod 5개를 통째로 끌고 오고, Obj-C 클래스는 리플렉션 가능성 때문에 링커가 못 버리며, ML 모델은 애초에 코드가 아니라 **리소스**라 데드 코드 스트리핑 대상도 아니다. 공식 선택 옵션이 없으면 **patch-package로 podspec + 네이티브 코드 2파일을 패치**하는 게 출구.

---

## 1. 핵심 개념

### 1-1. 앱 "용량"은 한 개가 아니다 — 3가지 사이즈

| 구분 | IPA (업로드) 사이즈 | 다운로드 사이즈 | 설치 사이즈 |
|---|---|---|---|
| 정체 | 개발자가 업로드하는 universal 바이너리 (모든 기기·아키텍처 포함) | 사용자 기기로 실제 내려받는 양 (thinning + 압축 후) | 압축 풀고 기기에 차지하는 양 |
| 누가 줄여주나 | 아무도 (내 책임) | Apple의 **app thinning**(slicing)이 기기별 변형본 생성 | — |
| 어디서 보나 | Xcode Organizer / EAS 빌드 산출물 | **App Store Connect → 앱 사이즈 리포트** | 〃 |

- **App thinning(slicing)**: Apple이 업로드된 universal IPA를 기기별로 쪼개서(불필요한 아키텍처·에셋 제거) 각 기기엔 필요한 것만 배달하는 것.
- **TestFlight도 thinning은 적용된다** — 다만 TestFlight 빌드에는 테스트용 부가 데이터가 붙고 스토어 배포본과 패키징이 달라서, TestFlight에서 본 숫자가 스토어 최종 다운로드 사이즈보다 부풀려 보일 수 있다. **진실은 App Store Connect의 app size report**로 확인한다. (브리프에는 "TestFlight는 thinning 미적용에 가까움"이라 적혀 있었는데, 리서치 결과 thinning 자체는 적용되는 게 맞다 — 부풀림의 원인은 부가 데이터·패키징 차이.)
- 단, thinning이 줄여주는 건 아키텍처·에셋 슬라이싱뿐. **5종 모델처럼 "모든 기기에 다 들어가는" 짐은 thinning이 못 덜어낸다.** 이건 빌드에서 빼는 수밖에 없다.

### 1-2. 왜 "한 번도 안 부르는" 네이티브 의존성이 바이너리에 남는가

| 레이어 | JS (Metro 번들러) | 네이티브 (링커) |
|---|---|---|
| 제거 단위 | `import` 그래프 기준 모듈/함수 | pod 전체 (있거나 / 없거나) |
| 안 쓰면 | tree-shaking으로 번들에서 빠짐 | **그대로 남음** (아래 두 이유) |
| 결정 시점 | 번들 타임 (정적 분석) | `pod install` 타임 (podspec 의존성 그래프) |

안 빠지는 이유 두 가지 — 이게 이 노트의 핵심 원리:

1. **Obj-C 클래스는 데드 코드 스트리핑이 사실상 불가.** Obj-C는 `NSClassFromString(@"MLKChineseTextRecognizerOptions")`처럼 **문자열로 클래스를 런타임에 불러낼 수 있는 언어**다(리플렉션). 링커 입장에선 "정적으로 참조가 없다 ≠ 안 쓴다"라서 함부로 버릴 수 없고, RN 생태계가 관례로 쓰는 `-ObjC` 링커 플래그는 아예 "정적 라이브러리의 Obj-C 심볼을 전부 실어라"는 뜻이다.
2. **ML 모델 파일은 코드가 아니라 리소스다.** 데드 코드 스트리핑은 말 그대로 *코드*(심볼) 대상이다. 인식 모델 같은 데이터 파일은 pod이 앱 번들에 복사해 넣는 **에셋**이라, 링커의 심판대에 올라가지도 않는다. pod이 들어오면 모델도 무조건 들어온다.

> 비유: **JS import는 마트 낱개 구매, 네이티브 pod은 코스트코 묶음 판매.** 번들러(마트 직원)는 내가 담은 것만 계산해 주지만, podspec에 적힌 의존성은 5개들이 묶음이라 하나만 골라 살 수 없다. 그리고 Obj-C 클래스는 "사번(문자열)만 대면 아무나 호출할 수 있는 직원"이라 링커가 "안 쓰이는 것 같으니 해고"를 못 하고, 모델 파일은 애초에 직원이 아니라 **창고 짐짝**이라 해고 심사 대상조차 아니다.

### 1-3. 이번 사건의 숫자 (ML Kit Text Recognition v2, iOS)

- 공식 문서: **스크립트당 SDK 약 38MB**, 에셋(모델)은 **빌드 타임에 정적 링크** — iOS엔 나중에 내려받는 옵션이 없다.
- 5종 전부 = 약 190MB가 순수 인식기 몫. 우리 앱은 Korean 1종만 쓰므로 4종(~150MB)이 데드웨이트였다.
- 한국어 인식기는 **라틴 문자도 같이 읽으므로** Korean만 남겨도 기능 손실 없음 (메뉴판의 영문 병기까지 커버).
- 패치 주석에는 "스크립트당 ~80–100MB"로 적어뒀는데 이는 추정치. 공식 문서 수치는 38MB/스크립트이고, **실제 절감은 다음 빌드에서 실측**해야 한다 (열린 질문 ①).

### 1-4. Android는 같은 문제를 어떻게 푸나

| 구분 | iOS | Android |
|---|---|---|
| 모델 전달 | 정적 링크만 (앱에 내장) | **bundled**(내장) 또는 **unbundled**(Google Play Services가 최초 사용 전 다운로드) |
| 사이즈 | 스크립트당 ~38MB | bundled: 스크립트당 ~4MB/아키텍처 · unbundled: **~260KB** |
| 추가 수단 | — | dynamic feature module로 기능 자체를 온디맨드 배달 가능 |

iOS에는 Play Services 같은 공용 모델 배달부가 없어서 전부 앱이 짊어진다. **같은 라이브러리인데 iOS만 뚱뚱해지는 이유.**

---

## 2. 절차 / 방법 — 실제로 어떻게 뺐나

라이브러리에 공식 "스크립트 선택" 옵션이 없다(podspec에 5종 하드코딩, 업스트림 PR #28에서 유래). 그래서 **patch-package로 2파일 패치**:

1. **`RNMLKitTextRecognition.podspec`** — 의존성을 `GoogleMLKit/TextRecognitionKorean`만 남기고 4종 삭제.
2. **`ios/TextRecognition.m`** — 제거된 4종 모듈의 `@import`와 인식기 분기 삭제. **podspec만 고치면 컴파일 에러** — 네이티브 코드가 5종 클래스를 무조건 참조하기 때문. (의존성 제거 ≠ 코드 참조 제거, 둘 다 해야 한다.)
3. `package.json`에 `"postinstall": "patch-package"` 등록 → EAS 클라우드 빌드(fresh install)에서도 자동 재적용.
4. 검증: `npx expo prebuild --clean` 후 `ios/Podfile.lock`에 Korean 서브스펙만 남았는지 확인.

```bash
# 패치 만들기 (node_modules를 직접 고친 뒤)
npx patch-package @react-native-ml-kit/text-recognition

# 검증
npx expo prebuild --clean
grep TextRecognition ios/Podfile.lock   # Korean만 나와야 정상
```

실코드: `kbap-fe/patches/@react-native-ml-kit+text-recognition+2.0.0.patch`, `src/lib/scan/ocr.ts:96` (`TextRecognitionScript.KOREAN` 단일 호출), 커밋 45d36ad. ✅ 2026-07-14 기준 패치 적용·Podfile.lock 검증 완료.

> [!warning] 네이티브 변경 = OTA 불가
> podspec·.m 변경은 네이티브 바이너리를 바꾸므로 EAS Update(OTA)로 배포할 수 없다. **재빌드 + 스토어/TestFlight 재배포** 필요. ([[EAS Dev Build vs Expo Go]]의 OTA 경계 참고.)

### 왜 Podfile `post_install` 훅으로는 안 되나

CocoaPods의 의존성 해석(resolution)은 **analysis 단계**에서 끝나고, `post_install`은 그 결과로 만들어진 Xcode 프로젝트를 저장 직전에 만지는 훅이다. 즉 **훅이 돌 때는 이미 5종 pod이 설치 그래프에 확정된 뒤**라, 거기서 의존성을 빼는 건 불가능하다. 의존성을 바꾸려면 그래프의 입력인 **podspec 자체**를 고쳐야 하고, 그래서 patch-package가 정답이 된다.

### patch-package가 Expo CNG(prebuild)에서 살아남는 메커니즘

- Expo CNG는 `ios/` 디렉토리를 **일회용 산출물**로 취급한다 — `prebuild`가 매번 `node_modules`의 podspec들을 읽어 재생성.
- patch-package는 `ios/`가 아니라 **`node_modules` 쪽을 패치**하고, `postinstall`에 걸려 있어 `npm install` 직후(= prebuild보다 먼저) 재적용된다.
- 따라서 순서가 항상 `install → 패치 적용 → prebuild가 패치된 podspec을 읽음` 이 되어 살아남는다. EAS 클라우드 빌드도 fresh `npm install`부터 시작하므로 동일.
- 반대로 **생성된 `ios/` 디렉토리 자체를 고친 패치**는 prebuild가 갈아엎는다 — 그 용도는 별도 도구 `patch-project`(Expo config plugin) 몫. 헷갈리지 말 것.

### 대안 비교 — patch-package vs fork vs config plugin

| | patch-package | fork (내 레포로 복제) | Expo config plugin |
|---|---|---|---|
| 개입 지점 | `node_modules` 파일 diff | 패키지 소스 자체 | prebuild 때 네이티브 프로젝트 변형 |
| 유지보수 | 라이브러리 **버전 올리면 패치 무효** → 재생성 필요 (파일명에 버전 박혀 있어 실패가 시끄럽게 드러남 = 장점) | 업스트림 변경을 수동 머지해야 함 (조용히 뒤처짐) | 라이브러리가 아닌 **생성물**을 고치므로 podspec 의존성 제거엔 부적합 |
| 적합한 경우 | **이번처럼 2파일 소규모 수정** | 수정이 크고 지속적일 때 | `Info.plist`·`AndroidManifest` 등 프로젝트 설정 변경 |
| 탈출구 | 업스트림에 subspec 선택 옵션 PR (열린 질문 ③) | PR 보내고 머지되면 폐기 | — |

---

## 3. 트러블슈팅 (실제로 막혔던 것)

- **증상**: podspec에서 4종 의존성만 지우니 빌드 실패 → **원인**: `TextRecognition.m`이 `@import MLKitTextRecognitionChinese;` 등 5종 모듈을 무조건 import + 분기에서 클래스 참조 → **해결**: .m에서 import·분기도 함께 삭제하고, 지원 안 하는 스크립트 요청 시 `"Unsupported script (Korean-only build)"`로 명시적 reject.
- **증상**: "patch-package가 EAS에서도 먹힐까?" 불안 → **원인**: 클라우드는 fresh install이라 로컬에서 고친 node_modules가 없음 → **해결**: `postinstall` 스크립트 등록. 이거 없으면 로컬에서만 되고 클라우드 빌드는 5종 원복.
- **증상**: 패치 후에도 TestFlight 숫자가 기대만큼 안 줄어 보임 → **원인 후보**: TestFlight 부가 데이터로 부풀려 보임 / ML Kit 외 다른 짐(2차 후보) → **해결**: App Store Connect app size report 기준으로 판단할 것.

---

## 4. 치트시트

```bash
# 지금 뭐가 링크되는지 — 서브스펙 확인
grep -A2 "GoogleMLKit" ios/Podfile.lock

# 패치 생성/재생성 (라이브러리 버전 올린 뒤엔 필수)
npx patch-package @react-native-ml-kit/text-recognition

# CNG 재생성 + 패치 생존 검증
npx expo prebuild --clean && grep TextRecognition ios/Podfile.lock
```

**용량 진단 도구 (2차 조사용, 열린 질문 ②):**
- App Store Connect **app size report** — 기기별 다운로드/설치 사이즈의 공식 출처
- Xcode Organizer → App Thinning Size Report (로컬 아카이브 시)
- `Assets.car` 분석(`assetutil`) — 이미지 에셋 비대 확인
- **linkmap** (`LD_GENERATE_MAP_FILE=YES`) — 바이너리에서 어느 라이브러리가 몇 바이트 먹는지 심볼 단위 추적
- Emerge Tools 같은 사이즈 분석 서비스

---

## 5. 용어집
- **app thinning / slicing**: Apple이 universal 바이너리를 기기별 변형본으로 쪼개 필요한 것만 배달하는 스토어 측 최적화.
- **subspec**: CocoaPods 패키지 안의 하위 패키지 단위 (`GoogleMLKit/TextRecognitionKorean`처럼 `/` 뒤가 서브스펙). 골라 담기의 단위지만, 골라 담을지는 **podspec 작성자**가 결정한다.
- **데드 코드 스트리핑 (dead code stripping)**: 링커가 정적으로 도달 불가능한 심볼을 최종 바이너리에서 제거하는 것. Obj-C 리플렉션·`-ObjC` 플래그 때문에 Obj-C에선 거의 무력.
- **정적 링크 (static linking)**: 라이브러리 코드를 빌드 타임에 앱 바이너리 안으로 복사해 넣는 방식. iOS ML Kit 모델이 이 방식 + 리소스 동봉.
- **patch-package**: `node_modules` 수정본과 원본의 diff를 `patches/*.patch`로 저장했다가 `postinstall`마다 재적용하는 도구.
- **CNG (Continuous Native Generation)**: Expo가 `ios/`·`android/`를 소스가 아닌 **재생성 가능한 산출물**로 취급하는 워크플로우. 네이티브 커스텀은 config plugin이나 node_modules 패치로 "재생성의 입력"에 넣어야 살아남는다.
- **OTA (EAS Update)**: JS 번들만 무선 교체하는 배포. 네이티브 바이너리가 바뀌면 불가.

---

## 6. 다음에 빠른 재현 체크리스트
1. 앱이 뚱뚱하다? → 감으로 찍지 말고 **App Store Connect app size report / linkmap**부터 (TestFlight 숫자는 부풀려 보일 수 있음).
2. RN 라이브러리가 범인이면 → 그 라이브러리 **podspec의 `dependency` 목록**을 열어본다 (JS에서 하나만 import해도 pod은 전부 딸려온다).
3. 필요한 서브스펙만 남기는 공식 옵션이 있는지 README/이슈 확인 → 없으면 **patch-package로 podspec + 참조하는 네이티브 코드 함께** 패치.
4. `postinstall: patch-package` 등록 확인 (EAS 생존 조건).
5. `expo prebuild --clean` → `Podfile.lock` 검증 → **재빌드** (OTA 불가).
6. 라이브러리 버전 올릴 땐 패치 재생성 각오.

---

## 열린 질문 (다음 빌드에서 확인)
1. **실측**: Korean-only 후 몇 MB? (문서 기준 ~150MB 절감 예상, 패치 주석 추정은 그보다 큼 — 둘 다 실측으로 판정)
2. 그래도 200MB+면 2차 후보 진단: Firebase 불필요 모듈(`Podfile.lock`에서 사용 모듈 대조), 미사용 폰트/이미지(`Assets.car` 분석), linkmap으로 상위 점유 라이브러리 추적.
3. 업스트림(a7medev/react-native-ml-kit)에 **스크립트 선택 설치 옵션 PR**? 현재 관련 이슈·PR 없음(2026-07 기준) — 5종 하드코딩은 PR #28에서 유래. 머지되면 패치 유지보수에서 탈출 가능. Podfile 환경변수나 `$RNMLKitTextRecognitionScripts` 같은 opt-in 방식이 관례.

---

## 참고 링크
- [ML Kit Text Recognition v2 — iOS](https://developers.google.com/ml-kit/vision/text-recognition/v2/ios) — "About 38 MB per script SDK", 정적 링크 명시
- [ML Kit Text Recognition v2 — Android](https://developers.google.com/ml-kit/vision/text-recognition/v2/android) — bundled 4MB vs unbundled 260KB
- [Apple Developer Forums — App slicing and universal app size](https://developer.apple.com/forums/thread/93493) · [TestFlight 사이즈 관련 스레드](https://developer.apple.com/forums/thread/108370) — TestFlight 부가 데이터로 사이즈 부풀림
- [CocoaPods Podfile Syntax — post_install 훅의 위치](https://guides.cocoapods.org/syntax/podfile.html) · [Podspec Syntax — subspec](https://guides.cocoapods.org/syntax/podspec.html)
- [Expo — Continuous Native Generation](https://docs.expo.dev/workflow/continuous-native-generation/) · [patch-project (생성물 패치용, patch-package와 다름)](https://docs.expo.dev/config-plugins/patch-project/)
- [patch-package (npm)](https://www.npmjs.com/package/patch-package)
- [a7medev/react-native-ml-kit — 업스트림 레포](https://github.com/a7medev/react-native-ml-kit) (PR #28: Non-latin 지원 추가 = 5종 하드코딩의 기원)

> [!note] 서버가 OCR 을 하면 이 문제의 뿌리가 사라진다 (2026-08-27 추가)
> 이 노트는 ML Kit 모델 5종을 **1종으로 줄이는** 해법(patch-package)까지 갔다. 그 사이 서버가 다른 답을 냈다 — 스캔 v2 는 **앱이 글자를 아예 읽지 않는다.** 사진만 올리면 서버가 비전 LLM 으로 메뉴명·가격을 뽑는다. 온디바이스 OCR 이 필요 없어지면 `@react-native-ml-kit/text-recognition` 의존성 자체가 빠지고, **Korean 하나로 줄인 그 38MB 도 0 이 된다.**
> 다만 공짜는 아니다 — 오프라인이 안 되고(§6 의 "오프라인/속도" 칸), 호출마다 LLM 비용이 나가고, 그래서 무료 3회 한도·티켓·중복 차단이라는 장치가 통째로 붙는다. **앱 용량을 서버 비용과 맞바꾼 것**이다. 어느 쪽이 싼지는 사용량이 정한다.
> 위 「열린 질문」 1번(Korean-only 후 몇 MB)은 v2 로 완전히 넘어가면 **질문 자체가 없어지는** 종류다 — 다만 스토어에 남은 구버전 앱은 여전히 v1 을 쓰므로 당분간 둘 다 유효하다. → [[20. 스캔 v2와 사용량 티켓 — 서버 OCR·무료 한도·중복 요청 막기]]
