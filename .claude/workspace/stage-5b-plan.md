# 5단계 둘째 묶음(5b) 계획서 · 실코드 흐름 — JWT 인증(kb-118)

- **대상 독자**: FE(React Native/Expo)만 해온 개발자. 1~4단계 개념 노트 + 5a(온보딩·스캔) 이수. **5단계 규칙: 새 개념 설명 금지 — 1~4단계·인증 개념 노트로 `[[링크]]`.**
- **담당**: `code-flow-writer` (2개 노트 모두).
- **노트 골격**(code-flow-writer 정의): ① 이 API가 하는 일(사용자 관점 한 줄) ② 흐름 표(계층 | 파일:줄 | 하는 일) ③ 계층 경계마다 "여기서 모듈이 바뀌는 이유"(4단계와 연결) ④ 요청/응답 JSON ⑤ 관련 테스트가 보장하는 것 ⑥ spec·ADR 링크.
- **제약**: kbap-server 읽기 전용. 모든 단계에 `경로:줄` 근거. 상상 금지. 완성 노트는 iCloud 볼트 `kbap 백엔드/`에만.
- **근거 파일 전부 실재 확인 완료**(2026-07-15). spec `specs/kb-118-firebase-jwt-auth/`(spec·plan·research·data-model·quickstart·tasks·contracts) 실존 확인.

## 묶음 구성 근거 (한 줄)
**JWT 인증(kb-118) 단독 2노트로 간다. 지도 ④(홈보다 JWT 먼저)가 아니라 진도요약 "다음 묶음: JWT"를 따른다** — JWT는 spec US1이 말하는 "서비스의 문"(로그인 없으면 회원 도메인 전체가 실사용 불가)이고, 무엇보다 `@AuthMemberId` 횡단 인증 계층은 5a 온보딩·스캔 노트가 이미 "5b JWT 흐름 예고"로 **빚(forward-ref)을 남겨둔 선행지식**이라 홈(kb-111)보다 먼저 갚아야 세로 슬라이스가 닫힌다. 홈 조회(kb-111)는 5c로.

## 왜 2노트인가
kb-118은 로그인·재발급·로그아웃 3개 API + 매 요청 인증 필터라는 횡단 계층까지 폭이 넓다(개념 3개 이상). "신원 확립"과 "세션 유지·검증"으로 자연 분할한다.
- **노트 1 = 처음 어떻게 로그인되나** (Firebase 토큰 → 자체 JWT 발급, US1)
- **노트 2 = 로그인 후 매 요청에서 어떻게 인증이 유지되고 세션이 어떻게 끝나나** (필터의 매 요청 검증 + rotation 재발급 + 로그아웃, US2·US3 + 횡단) — 5a가 남긴 `@AuthMemberId` 빚을 갚는 노트.

---

> [!note] 실측 발견 1 — "쿠키" 아님, 응답 **본문** Bearer (스펙 본문과 코드는 일치, 제목·지도만 낡음)
> spec **제목**(`…자체 JWT 쿠키 발급…`)과 지도 ④ 설명("쿠키 발급/재발급")은 초기 결정의 잔재다. spec **본문은 이미 개정**됨(`research.md` R5 "개정 2026-07-11 쿠키→응답 본문", `spec.md` FR-005). **코드도 본문 방식**: `AuthController`는 `ResponseEntity.ok(BaseResponse.ok(...))`로 토큰을 JSON 본문에 담을 뿐 `Set-Cookie`를 쓰지 않고(`AuthController.kt:23-42`), 필터는 `Authorization: Bearer` 헤더에서 읽는다(`JwtAuthenticationFilter.kt:35-41`). → 노트는 "토큰은 응답 본문, 매 요청은 `Authorization: Bearer` 헤더"로 **코드=개정 스펙 기준**으로 쓴다. `> [!warning] 스펙과 다름`이 아니라 `> [!note]`(제목/지도만 낡음)로 표기. **이유(왜 앱은 쿠키 안 씀)는 `[[쿠키·세션 vs 토큰 인증 — 왜 앱은 JWT인가]]`로 링크**(research.md R5 Rationale: 모바일 앱은 쿠키 자동 전송 없음, HttpOnly 이점도 앱엔 무의미).

> [!note] 실측 발견 2 — refresh 저장소는 Redis(3단계 JPA/MySQL과 다른 인프라)
> refresh 토큰의 서버 세션은 MySQL이 아니라 **Redis**에 jti→memberId로 TTL과 함께 저장된다(`RefreshTokenRedisAdapter.kt`, `application-*.yml`의 `data.redis`). `RefreshTokenStore` port는 `core/member`에, 구현만 `infra/persistence/auth`에(4단계 ports&adapters 그대로). 3단계 개념 노트가 Redis를 다루지 않으므로 **짧은 콜아웃으로만 소개**하고(키-값 저장소·TTL 자동 만료·`consume`=원자적 getAndDelete), 응답 말미 "개념 노트 후보: Redis/키-값 저장소"로 보고. 새 개념 노트를 이번 묶음에서 쓰지는 않는다.

> [!note] 실측 발견 3 — LoginUseCase에는 `@Transactional`이 없다
> 5a 온보딩·스캔 UseCase는 `@Transactional`이었지만 `LoginUseCase`·`RefreshUseCase`·`LogoutUseCase`에는 없다(`@Service`만). member 저장은 1건 단발이고 refresh 저장은 Redis(RDB 트랜잭션 밖)라 DB 트랜잭션 경계가 불필요하기 때문. → "모든 UseCase가 @Transactional은 아니다"를 `[[설정과 프로필, 트랜잭션 첫걸음]]` 링크와 함께 한 줄로 짚는다(트랜잭션 개념 오해 방지).

---

## 노트 1 — 소셜 로그인 흐름 — Firebase 토큰이 자체 JWT가 되기까지

- **파일명(=제목)**: `소셜 로그인 흐름 — Firebase 토큰이 자체 JWT가 되기까지.md`
- **담당**: code-flow-writer
- **API**: `POST /api/v1/auth/login` (`AuthApi.kt:38`, `ApiPaths.V1 + "/auth"`)
- **한 줄**: 앱이 구글·애플 로그인으로 받은 Firebase ID 토큰을 보내면, 서버가 그 토큰의 진위를 검증해 신원(플랫폼·원본 id·이메일)을 꺼내 기존 회원이면 로그인·처음이면 가입시키고, **우리 서비스 자체** access·refresh JWT를 만들어 응답 본문으로 돌려준다.

**흐름 (경유 파일:줄, 실재 확인)**
| 계층 | 파일 | 하는 일 |
|---|---|---|
| Controller | `app/api/.../auth/AuthController.kt:23-28` | `login(@RequestBody LoginRequest)` → `loginUseCase.login(request.idToken)` → `LoginResponse.from(result)`, `BaseResponse.ok` |
| 요청/응답 DTO | `app/api/.../auth/LoginRequest.kt`, `LoginResponse.kt` | 입력 `{idToken}`(`@NotBlank`) / 출력 `{newMember, accessToken, refreshToken}` (memberId 미포함 — 회원 식별은 토큰 sub) |
| UseCase | `application/client/.../auth/LoginUseCase.kt:27-56` | `verify → resolveMember(가입/조회) → issueRefreshToken → refreshTokenStore.save → issueAccessToken`. `resolveMember`는 `findByIdentity` 없으면 `saveNew(Member.signUp)`, `DUPLICATE_SOCIAL_IDENTITY` 경합은 재조회로 흡수(:43-56) |
| 검증 port | `application/client/.../auth/SocialTokenVerifier.kt` | `verify(idToken): SocialIdentity` 인터페이스(구현 모름 — 4단계 역전) |
| 검증 adapter | `application/client/.../auth/FirebaseTokenVerifier.kt`, `FirebaseClaimMapper.kt` | `FirebaseAuth.verifyIdToken`(서명·발급자·수신자·만료) → claims에서 provider·원본 uid·email 매핑. 실패는 `AuthException(INVALID_SOCIAL_TOKEN)` |
| verifier 조립 | `application/client/.../auth/AuthConfig.kt:26-38` | `@Bean socialTokenVerifier` — Firebase 자격증명 있으면 `FirebaseTokenVerifier`, 없으면 `UnavailableSocialAuth`(항상 401 폴백) |
| 토큰 발급 | `application/client/.../auth/TokenIssuer.kt:23-47` | jjwt `Jwts.builder` HS256 서명. access=`sub`+role+token_type+만료, refresh=`sub`+token_type+`jti`(UUID)+만료 |
| 세션 저장 port/adapter | `core/member/.../RefreshTokenStore.kt`(port), `infra/persistence/.../auth/RefreshTokenRedisAdapter.kt`(Redis 구현) | `save(jti, memberId, ttl)` — jti→memberId를 TTL과 함께 Redis에 |
| 회원 도메인/port | `core/member/.../Member.kt:35`(`signUp`), `SocialIdentity.kt`, `MemberRepository.kt`(`findByIdentity`·`saveNew`) | 순수 Kotlin. `signUp`은 프로필 비어있고 `onboardingCompleted=false`인 새 Member(→5a 온보딩으로 이어짐) |
| 설정값 | `app/api/.../resources/application.yml:37-49`, `AuthTokenProperties.kt`, `AuthConfig.kt:19-24` | `meogo.auth.jwt.secret`(32B+ 미설정 시 부팅 실패), `access-ttl: 30m`, `refresh-ttl: 14d`, firebase 자격증명 경로 |

**개념 노트 링크 맵** (각 구간 → 1~4단계·인증 노트)
- Controller가 3 UseCase 생성자 주입 / `BaseResponse` → `[[의존성 주입(DI)과 계층 구조 — 코드를 어떻게 나누나]]`, `[[IoC 컨테이너와 Bean — 부품을 직접 만들지 않는다]]`
- `AuthConfig`의 `@Bean`이 자격증명 유무로 다른 구현을 조립 → `[[IoC 컨테이너와 Bean — 부품을 직접 만들지 않는다]]`
- `SocialTokenVerifier` port에만 의존, Firebase 구현은 adapter / `RefreshTokenStore` port ↔ Redis adapter → `[[클린 아키텍처와 ports & adapters — 도메인을 순수하게 지키기]]`
- `Member.signUp`·`SocialIdentity` 순수 도메인, member 바운디드 컨텍스트 → `[[모듈러 모놀리스와 DDD — 경계를 코드로 지키기]]`
- `application.yml`·프로필·`AuthTokenProperties`(부팅 검증) / `@Transactional` 없음 → `[[설정과 프로필, 트랜잭션 첫걸음]]`
- Firebase가 소셜 IdP(구글·애플) 토큰을 대신 검증하는 층 → `[[OAuth 2.0과 OIDC — 소셜 로그인의 원리]]`, provider별 차이(애플 릴레이 이메일 시나리오)는 `[[Google vs Apple 소셜 로그인 비교]]`
- 자체 JWT를 왜 발급하나(소셜 토큰을 매번 안 쓰고) / 앱이 쿠키 대신 Bearer 저장 → `[[쿠키·세션 vs 토큰 인증 — 왜 앱은 JWT인가]]` **(필수 복리 — 아래 vault-keeper 지시)**
- 테스트·spec 근거 → `[[SDD와 TDD — 스펙과 테스트로 개발하기]]`

**FE 복리 연결** (역링크 append는 vault-keeper 몫)
로그인은 **FE가 Firebase SDK로 소셜 로그인 → idToken 획득 → `POST /api/v1/auth/login` 제출**의 뒷단이다. 응답 `newMember`로 FE가 온보딩/홈 분기. 노트의 "① idToken이 어디서 오나" 지점에서 인증 개념 노트로 링크(별도 FE 노트가 있으면 vault-keeper가 검색해 연결; 없으면 생략).

**forward 예고** (본문 말미 한 줄씩)
- 발급된 access 토큰이 매 요청에서 어떻게 쓰이나 → **노트 2(JWT 인증 필터)**.
- `Member.signUp`이 만든 빈 프로필이 채워지는 과정 → 5a `[[온보딩 프로필 저장 흐름 — 요청이 DB에 닿기까지]]`(back-link).

**관련 테스트가 보장하는 것** (⑤ 섹션 근거)
- `application/client/.../auth/LoginUseCaseTest.kt`(신규 가입/기존 재로그인/경합), `FirebaseClaimMapperTest.kt`(원본 uid 추출·provider 매핑·릴레이 이메일), `TokenTest.kt`(발급·서명), `app/api/.../auth/AuthControllerTest.kt`(login 엔드포인트·401 매핑, 페이크 verifier 빈).

**근거 spec/ADR**: `specs/kb-118-firebase-jwt-auth/`(spec US1·FR-005, research R1~R5, data-model, quickstart=자격증명 주입 경로, contracts). 아키텍처(도메인 순수·port 역전) → ADR-0001(멀티앱 모듈 레이아웃), `docs/architecture/meogo-conventions.md`, `docs/architecture/domains/member.md`.

---

## 노트 2 — JWT 인증 필터와 세션 생명주기 — 매 요청 검증·재발급·로그아웃

- **파일명(=제목)**: `JWT 인증 필터와 세션 생명주기 — 매 요청 검증·재발급·로그아웃.md`
- **담당**: code-flow-writer
- **API**: 매 요청 필터(엔드포인트 아님) + `POST /api/v1/auth/refresh` + `POST /api/v1/auth/logout`
- **한 줄**: 로그인 뒤 모든 보호된 요청은 `Authorization: Bearer {access}` 헤더를 필터가 가로채 검증하고 memberId를 컨트롤러에 꽂아준다(5a의 `@AuthMemberId`의 정체). access가 만료되면 refresh로 둘 다 새로 발급(rotation)하고, 로그아웃하면 서버의 refresh 세션을 폐기한다.

**흐름 A — 매 요청 인증 (횡단)**
| 계층 | 파일 | 하는 일 |
|---|---|---|
| 필터 등록 | `app/api/.../common/auth/WebMvcAuthConfig.kt:20-29` | `FilterRegistrationBean`으로 `members/*`·`scans`·`scans/*`·`auth/withdraw`에만 필터 적용(login/refresh/logout은 **비보호** — 문 앞이라 토큰이 아직 없음) |
| 필터 | `app/api/.../common/auth/JwtAuthenticationFilter.kt:19-48` | `OncePerRequestFilter`. `Authorization: Bearer` 추출 → `tokenParser.parseAccessToken` → memberId·role을 **request attribute**에 저장 → 체인 진행. `AuthException`이면 필터에서 401 JSON 직접 write |
| 파서 | `application/client/.../auth/TokenParser.kt:29-37` | 같은 HS256 키로 서명·만료·token_type 검증, `sub`→memberId·role 복원. 실패는 `EXPIRED_/INVALID_ACCESS_TOKEN` |
| 리졸버 등록 | `app/api/.../common/auth/WebMvcAuthConfig.kt:15-18` | `addArgumentResolvers`로 `AuthMemberIdArgumentResolver`·`AuthMemberIdOrNullArgumentResolver` 등록 |
| 리졸버 | `app/api/.../common/auth/AuthMemberIdArgumentResolver.kt:12-24`, `AuthMemberId.kt` | `@AuthMemberId Long` 파라미터에 필터가 넣어둔 request attribute(memberId) 주입. **5a 온보딩·스캔 노트가 "5b 예고"로 남긴 바로 그 조각** |

**흐름 B — 재발급 (rotation)**
| 계층 | 파일 | 하는 일 |
|---|---|---|
| Controller | `app/api/.../auth/AuthController.kt:30-35` | `refresh(@RequestBody RefreshRequest)` → `refreshUseCase.refresh(request.refreshToken!!)` → `TokenResponse.from` |
| DTO | `app/api/.../auth/TokenRequests.kt:6-22` | 입력 `{refreshToken}`(`@NotBlank`) / 출력 `{accessToken, refreshToken}` |
| UseCase | `application/client/.../auth/RefreshUseCase.kt:21-45` | `parseRefreshToken` → `refreshTokenStore.consume(jti)`(원자적 getAndDelete=구 토큰 즉시 폐기) → 회원 존재 확인 → 새 refresh 발급·save + 새 access. **만료 시**(:24-28) Redis jti 삭제 후 예외 재throw = 강제 로그아웃 |
| 세션 저장 | `infra/persistence/.../auth/RefreshTokenRedisAdapter.kt:16-21` | `consume`=`getAndDelete`(회전 원자성), `delete`(폐기) |

**흐름 C — 로그아웃**
| 계층 | 파일 | 하는 일 |
|---|---|---|
| Controller | `app/api/.../auth/AuthController.kt:37-42` | `logout(@RequestBody(required=false) LogoutRequest?)` → `logoutUseCase.logout(request?.refreshToken)`(멱등) |
| UseCase | `application/client/.../auth/LogoutUseCase.kt:11-17` | refresh 없으면 no-op, 있으면 jti 추출해 `refreshTokenStore.delete` — 이후 그 refresh로 재발급 불가 |
| 에러 코드 | `application/client/.../auth/AuthErrorCode.kt` | 401 6종(social/access/refresh × invalid/expired) — 필터·파서·재발급이 공유 |

**개념 노트 링크 맵**
- 필터가 컨트롤러보다 앞에서 요청을 가로채는 "횡단 계층" / `OncePerRequestFilter` → `[[의존성 주입(DI)과 계층 구조 — 코드를 어떻게 나누나]]`(계층 분리), 서버가 요청마다 실행하는 지점은 `[[요청 하나에 스레드 하나 — JVM 서버의 동시성과 Node 이벤트루프의 차이]]`
- `WebMvcAuthConfig`의 `@Bean`·`@Configuration`으로 필터/리졸버 조립 → `[[IoC 컨테이너와 Bean — 부품을 직접 만들지 않는다]]`
- `TokenParser`·`RefreshTokenStore`가 application/core에, 필터는 app 모듈 / port ↔ Redis adapter → `[[클린 아키텍처와 ports & adapters — 도메인을 순수하게 지키기]]`, `[[모듈러 모놀리스와 DDD — 경계를 코드로 지키기]]`
- access는 stateless(만료 전 무효화 불가)라 짧게, refresh는 Redis 폐기 가능이라 rotation — 왜 두 토큰인가/왜 앱은 JWT인가 → `[[쿠키·세션 vs 토큰 인증 — 왜 앱은 JWT인가]]` **(필수 복리)**
- `@Transactional` 없음(Redis는 RDB 트랜잭션 밖) → `[[설정과 프로필, 트랜잭션 첫걸음]]`
- 테스트·spec(US2 rotation·US3 폐기) → `[[SDD와 TDD — 스펙과 테스트로 개발하기]]`

**5a 빚 갚기 (back-link 이행 — vault-keeper 몫)**
5a 온보딩·스캔 노트의 "인증 리졸버(5b JWT 흐름 예고)" 지점에 이 노트로 **역링크 append**: `[[JWT 인증 필터와 세션 생명주기 — 매 요청 검증·재발급·로그아웃]]`("`@AuthMemberId`가 memberId를 어디서 얻는지 여기서 확인"). 기존 본문은 건드리지 말 것.

**관련 테스트가 보장하는 것**
- `app/api/.../common/auth/JwtAuthenticationFilterTest.kt`(헤더 없음·형식 오류·만료·정상 통과), `application/client/.../auth/RefreshUseCaseTest.kt`(rotation·구 토큰 거절·만료 강제 로그아웃·미존재 회원), `LogoutUseCaseTest.kt`(폐기·멱등), `infra/persistence/.../auth/RefreshTokenRedisAdapterTest.kt`(Redis Testcontainers 저장/consume/delete/TTL).

**근거 spec/ADR**: `specs/kb-118-firebase-jwt-auth/`(spec US2·US3, FR-005~, research R5 에러 매핑 표, data-model=Redis 세션). 아키텍처 → ADR-0001, `docs/architecture/meogo-conventions.md`.

---

## 작성자 공통 지침 (code-flow-writer가 따를 것)
- `kbap-repo-map`으로 탐색 시작, `study-note-style` 규약 준수. **모든 단계에 `경로:줄` 근거**, 상상 금지.
- **새 개념 설명 금지** — 위 링크 맵대로 1~4단계·인증 노트로 `[[링크]]`. Redis만 예외적으로 짧은 콜아웃(위 실측 발견 2) + 응답에 "개념 노트 후보: Redis" 보고.
- 스펙 제목/지도의 "쿠키"는 낡음 — 코드=개정 스펙(응답 본문 Bearer) 기준으로 쓰고 `> [!note]`로 한 줄 짚기(위 실측 발견 1). `> [!warning] 스펙과 다름`은 쓰지 않는다(본문·코드 일치).
- 발췌는 15줄 이내, 주석 금지 레포이므로 발췌 아래 줄 단위 풀이. (예외: `application.yml`·`AuthApi.kt`의 설명 주석/Swagger는 인용 가능 — 실제 문서화 주석이라.)
- frontmatter: `tags: [kbap-backend, 흐름]`, `생성일: 2026-07-15`, `상태: 완료`.
- 노트 1이 로그인·발급(신원 확립), 노트 2가 필터·재발급·로그아웃(세션 유지)의 주인. 두 노트가 `TokenIssuer`/`TokenParser`/`RefreshTokenStore`를 함께 스치면 본문 설명 주인은 위 배정대로, 나머지는 링크.

## vault-keeper 지시 (노트 완성 후 vault-bookkeeping)
1. **필수 복리 (양방향)**: `[[쿠키·세션 vs 토큰 인증 — 왜 앱은 JWT인가]]`(`인증·보안/`, 실재 확인) ↔ 두 흐름 노트 모두. 이 개념 노트 하단에 두 흐름 노트로 역링크 append("앱 JWT 개념이 kbap에서 실제로 어떻게 발급·검증되는지"). **2단계·5a 두 번 예약된 복리**를 여기서 이행.
2. **④ 예약 복리**: `[[OAuth 2.0과 OIDC — 소셜 로그인의 원리]]`(`인증·보안/`, 실재 확인) ↔ 노트 1(Firebase 검증 지점). 이 노트 하단에 노트 1로 역링크 append. 보너스: `[[Google vs Apple 소셜 로그인 비교]]`(실재 확인) ↔ 노트 1(애플 릴레이 이메일 시나리오, US1 AS5).
3. **5a 빚 갚기**: 5a 온보딩·스캔 노트의 "인증 리졸버 … 5b 예고" 줄 아래에 노트 2로 역링크 append(위 "5a 빚 갚기" 참조).
4. **지도 갱신**: `🗺️ kbap 백엔드 공부 지도.md` — ④ 체크박스 `[x]`로, ④ 설명의 "쿠키 발급/재발급"을 "응답 본문 Bearer 발급/재발급"으로 소폭 정정(초기 결정 잔재), 진도 요약을 "완료 노트 16 / 다음 묶음: 홈 화면 조회(kb-111)"로. 지도는 새로 만들지 말고 체크·정정만.
5. 이전 개념 노트 역링크·홈 등록·작업 로그·린트는 표준 절차대로.
