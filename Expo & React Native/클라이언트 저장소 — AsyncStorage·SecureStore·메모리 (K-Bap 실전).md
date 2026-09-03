---
tags: [expo, react-native, storage, asyncstorage, securestore, kbap]
생성일: 2026-07-09
상태: 완료
---

> [!info] 관련 노트
> 🏠 [[🏠 홈]] · 📂 주제: Expo & React Native
> 🔗 짝꿍: [[쿠키·세션 vs 토큰 인증 — 왜 앱은 JWT인가]] (cookie/session이 여기 없는 이유는 그쪽에서)
> 🔗 서버 반대편: [[29. 캐싱과 키-값 저장소 — kbap이 Redis를 쓰는 두 가지 방식]] (민감도 × 수명으로 저장소를 고르는 사고방식이 서버에서도 똑같이 반복된다)
> 형제: [[EAS Dev Build vs Expo Go]] · [[Google vs Apple 소셜 로그인 비교]]

# 클라이언트 저장소 — AsyncStorage·SecureStore·메모리 (K-Bap 실전)

> 맥락: KB-110 온보딩 중도이탈 재개를 AsyncStorage로 구현하다가 "저장소 종류가 왜 이렇게 많고 언제 뭘 쓰나"가 궁금해짐.
> 목표: K-Bap 실제 코드를 근거로, **민감도 × 수명**으로 저장소를 고르는 눈 만들기.

---

## 0. 한 줄 요약
클라이언트 저장소 선택은 **"이 데이터가 유출되면 얼마나 아픈가(민감도)" × "언제까지 살아야 하나(수명)"** 두 축이면 끝난다.
K-Bap의 실제 배치가 정확히 이 원리다: **메모리**(JWT — 짧고 민감) → **AsyncStorage**(온보딩 draft·최근검색 — 길고 무해) → **SecureStore**(OAuth 크리덴셜 — 길고 민감). cookie/session이 목록에 없는 건 브라우저 인프라가 앱에 없어서다 → [[쿠키·세션 vs 토큰 인증 — 왜 앱은 JWT인가]].

---

## 1. K-Bap 실사용 지도 (코드 검증 완료)

| 저장소 | 쓴 곳 | 파일 · 키 | 선택 이유 (코드로 확인) |
|---|---|---|---|
| **AsyncStorage** | 온보딩 중도이탈 draft | `onboarding/draft.ts` · `kbap.onboardingDraft.v1` | 평문이어도 무해한 UX 상태. 잃어도 "재개 지점이 살짝 옛것"일 뿐 |
| **AsyncStorage** | 최근 검색어 (최대 8개) | `data/useRecentSearches.ts` · `kbap.recentSearches.v1` | 〃 (BE 없이 로컬 전용, KB-21) |
| **SecureStore** | OAuth 크리덴셜 임시 보관 | `auth/credentials.ts` · `kbap.auth.pendingCredential.v1` | 토큰류 = 민감 → iOS Keychain / Android Keystore 암호화 |
| **메모리** | JWT 스켈레톤 (`setAuthToken`) | `api/client.ts` | 변수 보관, 앱 종료 시 소멸. BE 계약 확정 전 스텁 |
| **메모리 (TanStack Query)** | 서버 데이터 캐시 | `data/use*.ts` | "캐시"지 "저장"이 아님 — 진실은 서버, 앱은 사본 |
| **localStorage** | RN-web 폴백 | (웹 빌드 한정) | AsyncStorage 웹 어댑터의 구현체 |

시드의 추정이 코드 주석으로 실제 확인됨 — `draft.ts`의 저장이 일부러 **fire-and-forget**인 것부터가 "무해한 데이터" 판단의 증거:
```ts
export function saveOnboardingDraft(draft: OnboardingDraft): void {
  // fire-and-forget — a lost write only costs a slightly stale resume point.
  void AsyncStorage.setItem(KEY, JSON.stringify(draft)).catch(() => {});
}
```
> 실패해도 무시(`catch(() => {})`). 민감하거나 잃으면 안 되는 데이터였다면 이렇게 못 쓴다. **저장소 선택과 에러 처리 강도가 같은 판단에서 나온다**는 좋은 예.

---

## 2. AsyncStorage vs localStorage — 같은 key-value인데 뭐가 다른가

| | **AsyncStorage** (RN) | **localStorage** (웹) |
|---|---|---|
| API | **비동기** (Promise) | **동기** (즉시 반환) |
| 실제 저장 위치 | iOS: 앱 샌드박스 내 파일(manifest) / Android: **SQLite DB** | 브라우저가 관리하는 origin별 영역 |
| 용량 | Android 기본 **총 6MB, 레코드당 2MB** (플래그로 증설 가능) / iOS는 사실상 디스크 여유만큼 | 보통 5~10MB (origin당) |
| 격리 단위 | **앱** (샌드박스) | **origin** (`https://example.com`) |
| 지우는 주체 | 앱 삭제 시 OS | 사용자 "브라우저 데이터 지우기"로도 날아감 |

`★ 왜 RN에는 localStorage가 없나 ─────────────`
① localStorage는 **DOM API**다 — `window`가 있어야 하는데 RN엔 브라우저 엔진 자체가 없다.
② 더 근본적으로, localStorage는 **동기**라 읽고 쓰는 동안 JS 스레드를 멈춘다. 웹에서도 이게 잔렉 원인인데,
RN은 JS 스레드가 멈추면 **터치 응답·애니메이션이 통째로 멈춘다** → 스토리지는 반드시 비동기여야 했고,
그래서 이름부터 **Async**Storage다.
③ RN-web으로 빌드하면 네이티브 저장소가 없으니 AsyncStorage가 localStorage를 **껍데기로 빌려 쓴다**(비동기 API는 유지).
`──────────────────────────────────────`

---

## 3. AsyncStorage vs SecureStore — 암호화 말고도 다르다

| | **AsyncStorage** | **SecureStore** (expo-secure-store) |
|---|---|---|
| 암호화 | ❌ **평문** (루팅/백업 추출 시 그대로 노출) | ✅ iOS **Keychain** / Android **Keystore** 기반 |
| 크기 | 큼 (수 MB) | **작음 — 값당 2048바이트 권고** (일부 iOS에서 초과 시 실패 이력) |
| 속도 | 빠른 편 | 암호화 오버헤드로 상대적으로 느림 |
| 용도 | 상태·캐시·설정 | **비밀만**: 토큰, 크리덴셜, 키 |
| 웹 지원 | localStorage 폴백 | ❌ 없음 (K-Bap도 웹에선 메모리로 폴백) |

K-Bap의 실제 사용 — 애플/구글 로그인 성공 직후의 크리덴셜을 파킹:
```ts
// auth/credentials.ts
try {
  await SecureStore.setItemAsync(STORE_KEY, JSON.stringify(credential));
} catch {
  // SecureStore unavailable (web) — memory-only is fine for the stub.
}
```

> [!note] "토큰은 SecureStore, UX 상태는 AsyncStorage" 원칙 — 항상 맞나? (시드 Q2)
> **원칙의 근거는 위협 모델**이다: AsyncStorage는 기기 백업 추출·루팅 시 평문으로 읽힌다. 유출 시 계정 탈취로 이어지는 것(토큰·크리덴셜)만 SecureStore에 넣고, 나머지는 넣지 말라 — **역방향도 원칙이다**: ① 2048바이트 제한 때문에 큰 데이터는 물리적으로 못 넣고 ② 느려서 자주 읽는 상태 저장엔 부적합. 즉 "민감하면 SecureStore"가 아니라 **"작고 비밀이면 SecureStore, 그 외엔 쓰지 마라"** 가 정확한 문장.

---

## 4. 수명과 소멸 — 뭐가 언제 죽나 (시드 Q4)

| 이벤트 | 메모리 | AsyncStorage | SecureStore(iOS Keychain) | SecureStore(Android) |
|---|---|---|---|---|
| 앱 재시작 | ❌ 소멸 | ✅ 생존 | ✅ 생존 | ✅ 생존 |
| 앱 삭제 | ❌ | ❌ 삭제 | ⚠️ **생존할 수 있음!** | ❌ 삭제 |
| 재설치 (같은 번들ID) | ❌ | ❌ (새로 시작) | ⚠️ **옛 값이 되살아날 수 있음** | ❌ |
| 기기 변경 | ❌ | ❌ | 백업 복원 설정에 따라 이전 가능 | ❌ |

> [!warning] 시드의 "Keychain은 앱 삭제 후에도 남는다" — 검증 결과 **사실** (iOS 한정)
> Expo 공식 문서 기준: iOS에서 SecureStore 데이터는 **같은 번들 ID로 재설치하면 삭제 전 값이 남아있을 수 있다.** Android는 삭제 시 같이 지워짐.
> **실무 함정**: 재설치한 유저의 Keychain에 옛 크리덴셜이 남아 "로그인 안 했는데 토큰이 있는" 상태가 될 수 있다 → 첫 실행 감지 시(예: AsyncStorage에 설치 마커가 없으면) SecureStore를 한 번 비워주는 패턴이 정석.

---

## 5. 결정 트리 — "이 데이터 어디에 저장?" (시드 Q5)

```
Q1. 서버가 진실의 원본인가?
 └─ Yes → 저장 말고 캐시 (TanStack Query). 끝.
Q2. 앱을 껐다 켜도 살아있어야 하나?
 └─ No → 메모리 (변수/상태). 끝.
Q3. 유출되면 계정·보안 사고인가? (토큰, 크리덴셜, 키)
 └─ Yes → SecureStore. (단 2KB 이내로 — 크면 설계를 다시 볼 것)
 └─ No  → AsyncStorage. (잃어도 되는 정도에 맞춰 에러 처리 강도 결정)
```
K-Bap 대입: 음식 목록(Q1→캐시) · JWT 액세스 토큰(Q2→메모리, 추후 리프레시는 Q3→SecureStore) · OAuth 크리덴셜(Q3 Yes→SecureStore) · 온보딩 draft·최근검색(Q3 No→AsyncStorage). **전부 트리와 일치.**

---

## 6. 용어집
- **AsyncStorage**: RN의 평문 비동기 key-value 저장소. iOS 파일 / Android SQLite.
- **SecureStore / Keychain / Keystore**: OS 차원의 암호화 비밀 저장소(expo-secure-store가 감싼 것).
- **fire-and-forget**: 결과를 기다리지도 실패를 처리하지도 않는 쓰기. 잃어도 되는 데이터의 신호.
- **샌드박스**: 앱마다 격리된 저장 공간. 다른 앱이 못 읽음 (암호화와는 별개!).
- **origin**: 웹 저장소의 격리 단위 (`프로토콜+도메인+포트`). 앱엔 이 개념이 없음.
- **캐시 vs 저장소**: 진실이 서버에 있으면 캐시(버려도 됨), 클라이언트에만 있으면 저장소(버리면 유실).

---

## 7. 다음에 빠른 재현 체크리스트
1. 판단 축 2개: **민감도 × 수명**. 결정 트리(§5) 4문답이면 끝.
2. SecureStore는 "작고 비밀인 것" 전용 — **2KB 권고**, 느림, 웹 미지원.
3. **iOS Keychain은 앱 삭제를 살아남는다** → 재설치 유저의 유령 크리덴셜 주의 (첫 실행 시 청소).
4. AsyncStorage Android 기본 한도 **총 6MB / 건당 2MB**.
5. 에러 처리 강도 = 데이터 중요도의 거울 (draft.ts의 fire-and-forget처럼).

---

## 8. 참고
- 실제 코드: `kbap-fe/src/lib/` — `onboarding/draft.ts` · `data/useRecentSearches.ts` · `auth/credentials.ts` · `api/client.ts`
- [expo-secure-store 공식 문서](https://docs.expo.dev/versions/latest/sdk/securestore/) (2048바이트·삭제 후 생존 명시)
- [AsyncStorage 6MB 이슈](https://github.com/react-native-community/async-storage/issues/83)

---

> [!note] 2026-08-26 · 서버 반대편의 "저장 구조 바꾸기" → [[10. Flyway 마이그레이션 — 스키마를 코드로 관리하기]]
> 여기 AsyncStorage 에 담는 값의 모양을 바꿀 때 앱이 하는 일(구버전 데이터 읽어 옮기기, 깨진 값 방어)을 서버에서는 **Flyway 마이그레이션 파일 한 장**으로 하는데, 갈림길은 되돌리기다 — 폰의 저장소는 사용자마다 한 벌이라 최악엔 지우고 다시 받으면 되지만, DB 는 서버 전체가 공유하는 **딱 한 벌**이라 잘못 적용하면 되돌릴 수 없다.

---

> [!note] 2026-08-27 · 서버 반대편의 저장소 선택 → [[29. 캐싱과 키-값 저장소 — kbap이 Redis를 쓰는 두 가지 방식]]
> 이 노트의 결론(**민감도 × 수명** 두 축이면 저장소 선택은 끝난다)이 서버에서도 그대로 반복된다. kbap 서버의 배치는 **MySQL**(주문·리뷰 — 영구, 잃으면 안 됨) → **Redis**(refresh 세션 14일 TTL·스캔 예약 짧은 TTL — 잃으면 재로그인/재시도로 감당) → **JVM 힙**(요청 처리 중에만 사는 값)이다. 축이 하나 늘 뿐이다 — 폰은 기기가 하나라 "누가 공유하나" 를 물을 일이 없었지만, 서버는 여러 대라 **프로세스 밖에 둬야 하는가**가 셋째 축이 된다. Redis 가 선택된 진짜 이유가 "빠르다" 가 아니라 거기에 있다.
>
> **여기서 잘 틀리는 자리**: Redis 라고 하면 캐시를 떠올리기 쉬운데, kbap 에는 캐시가 한 군데도 없다(`@Cacheable` 0건). 위 §1 표의 **TanStack Query 행만이 진짜 "캐시"** 다 — 진실이 서버에 따로 있으니까. Redis 의 refresh 세션은 원본이 따로 없어서 날아가면 전 사용자가 로그아웃된다. 정의상 캐시가 아니다.
>
> (2026-08-27 수정: 머리말의 옛 링크가 재구축 전 노트인 `21. 캐싱과 키-값 저장소`를 가리키고 있어 새 29번으로 갈아끼웠다. 함께 달려 있던 "FE는 토큰을 SecureStore에" 라는 설명도 **거짓이라 걷어냈다** — 위 §0·§1 대로 K-Bap 의 JWT 는 **메모리**에 있고, SecureStore 에 있는 건 OAuth 크리덴셜이다.)
