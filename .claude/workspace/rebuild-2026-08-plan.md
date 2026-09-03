# kbap 백엔드 공부 노트 재구축 계획 (2026-08-26)

> 대상 레포: `/Users/yejinkim/dev/kfood/kbap-server` (읽기 전용, 실측 기준일 2026-08-26)
> **결정된 전제**: 기존 `kbap 백엔드/` 21노트 + 지도는 **폴더째 `kbap 백엔드 (old)/` 로 이동**(파일명 절대 변경 금지). 새 노트는 **`kbap 백엔드 (new)/` 에 번호 1부터 전부 새로** 쓴다. 개작이 아니라 신규 집필이다.
> 이 문서는 계획서다 — 이번 실행에서 노트·지도·레포는 하나도 건드리지 않았다.

---

## 0. 실측 요약 (새 노트가 근거로 삼을 사실)

| 사실 | 근거 |
|------|------|
| 모듈 3개 `:common`·`:api`·`:batch`, 루트 직속 | `settings.gradle.kts`, `CLAUDE.md:13` (ADR-0016·0018) |
| `*UseCase*` 클래스 0개 — Controller / `*Api`(springdoc) / Facade / Service / Repository(public) | `grep -r UseCase` 0건, `api/scan/ScanFacade.kt`, `api/member/MemberService.kt`, ADR-0014·0017 |
| 엔티티가 곧 도메인 모델 (toDomain/from 폐지) | `common/domain/food/model/Food.kt`, `CLAUDE.md:118` |
| **JPA 연관관계가 레포 전체에 0개** — `@OneToMany`/`@ManyToOne`/`@OneToOne`/`@ManyToMany` 를 ArchUnit이 금지, 참조는 `Long` id 컬럼 | `api/src/test/.../architecture/ModuleBoundaryTest.kt:157-160`, 프로덕션 grep 0건 |
| `join fetch` 사용처 0개 | `grep -r "join fetch"` 0건 |
| 2단계 키셋 페이징은 생존 (id만 커서 조회 → `findByIdIn`) | `common/domain/food/FoodJpaRepository.kt:107-118` |
| JWT는 여전히 서블릿 필터 | `api/core/auth/JwtAuthenticationFilter.kt`(`OncePerRequestFilter`), `api/core/config/WebConfig.kt:69-80` |
| Redis 용도 2개 — refresh 토큰 + 스캔 예약. `@Cacheable`·`@EnableCaching` 0개 | `api/infra/redis/{RedisRefreshTokenStore,RedisScanReservationStore}.kt` |
| 랭킹 = 원자적 카운트 UPDATE + 조회 시점 계산 **+ 리뷰 이벤트 원장** | `api/member/MemberService.kt:90`, `common/domain/member/model/{Ranking,MemberRankingEvent}.kt`(unique `uq_member_ranking_event`) |
| 배치는 **Spring Batch** — JDBC JobRepository·Reader/Processor/Writer·HTTP 트리거(202+폴링)+인앱 스케줄러 | `batch/config/BatchJdbcJobRepositoryConfig.kt`, `batch/trigger/*`, `batch/{outbox,vector}/*`, `batch/src/main/resources/application.yml` |
| LLM 3모델 fan-out·consensus 소멸, 음식 콘텐츠 채움 kbap-langchain 이관 | `CLAUDE.md:53` (KB-301·KB-320) |
| 마이그레이션 49개(전부 timestamp), specs 디렉터리 125개 | `api/src/main/resources/db/migration`, `specs/` |
| 가상 스레드 미사용 — `spring.threads.virtual` 설정 없음 | `api/src/main/resources/application.yml` grep 0건 |
| 프로필 4종(local/dev/staging/prod)이되 **api·batch 각각 4개 = yml 8개 + 프로필 없는 테스트용 2개 = 총 10개**. batch 프로필 파일은 datasource+JPA 7줄이 전부 | `api/src/main/resources/application-*.yml`, `batch/src/main/resources/application-*.yml`, 각 모듈 `src/test/resources/application.yml` |
| **batch 의 flyway "off" 는 설정이 아니라 의존성 부재** — `batch/build.gradle.kts` 에 flyway 0줄(api 는 3줄). batch resources 의 flyway 언급은 주석 한 줄뿐 | `batch/build.gradle.kts`, `api/build.gradle.kts`, `batch/src/main/resources/application.yml` |
| **`@ConfigurationProperties` 를 단 클래스는 레포 전체에 2개뿐** — `FoodVectorProperties`·`LlmModelProperties`. `JwtTokenProperties` 는 **애너테이션 없는 순수 data class** 이고 `AuthConfig` 가 `@Value` 로 조립한다 | `common/domain/food/vector/FoodVectorProperties.kt`, `common/infra/llm/config/LlmModelProperties.kt`, `api/infra/auth/AuthConfig.kt:18-23` |
| **JWT 인증 필터는 전역이 아니다** — `addUrlPatterns` **19개 경로에만** 붙는다. `/api/home`·`/api/foods/*`(단 `/api/foods/scanned` 는 포함)·`/api/auth/{login,refresh,logout}`·`/api/app-version` 은 **필터가 돌지 않고**, 인자 리졸버 `@AuthMemberIdOrNull` 이 헤더를 직접 재파싱한다 | `api/core/config/WebConfig.kt:80-100` |
| **게스트 예외는 2단계 장치** — 필터가 *도는* 경로 중 GET 3개만 통과: `^/api/community/posts$`, `^/api/community/posts/\d+$`, `^/api/reviews$` | `api/core/config/WebConfig.kt`, `api/core/auth/JwtAuthenticationFilter.kt`(`shouldNotFilter`) |
| **`ApiPaths.API = "/api"` — 버전리스**(`/api/v1` 아님) | `api/core/ApiPaths.kt` |
| **refresh 재사용 탐지 없음** — `consume` 이 Redis GETDEL 이라 옛 토큰은 죽지만, 재사용이 감지돼도 살아 있는 세션 계보를 끊지 않고 401 만 던진다. ADR·스펙에 이 선택의 근거 문장이 없다 | `api/auth/AuthService.kt:56-57`, `api/infra/redis/RedisRefreshTokenStore.kt` |
| **`@AuthMemberIdOrNull` 은 이름과 다르다** — 만료·위조 토큰은 null 이 아니라 **예외** | `api/core/auth/AuthMemberIdOrNullArgumentResolver.kt:24-29` |

### 레포 문서 ↔ 코드 불일치 (새 노트는 **코드 기준**으로 쓴다)

1. `CLAUDE.md:117` — "읽기 전용 연관 허용(현재 유일: `Food`→성분 `@OneToMany(EAGER)`)". **코드엔 그 연관이 없고 ArchUnit이 전면 금지**한다(`ModuleBoundaryTest.kt:157`).
2. `CLAUDE.md:151` — "`*V2Controller`·`*V2Api` 를 만들지 않는다". **`api/scan/ScanV2Controller.kt`·`ScanV2Api.kt` 가 존재**한다(`@RequestMapping(..., version = "2.0+")`). 노트는 "경로는 같고 헤더로 가르되, 스캔만 클래스가 갈라져 있다"로 실측 기술.
3. `batch/src/main/resources/application.yml` 의 주석·`logging.level` 이 아직 해체된 `:infra:llm`·`com.kbap.infra.llm` 좌표를 가리킨다. 인용 금지.
4. `common/port/{scan,exchange}` 는 `CLAUDE.md` 의 포트 목록에 없다. 포트 열거는 실제 디렉터리 기준.
5. `.env.example` 머리말은 "Spring Boot 는 `.env` 를 자동으로 읽지 않는다 — spring-dotenv 가 필요하다"고 하지만, `api/src/main/resources/application.yml:17-18` 의 `spring.config.import: optional:file:.env[.properties]` 로 **지금은 읽힌다.**
6. `CLAUDE.md:13`·`CLAUDE.md:30` 이 외부 시스템·어댑터 목록에 **`Kakao`** 를 적지만 **코드에 카카오는 없다** — `api/infra/place/` 는 `GooglePlaceSearchClient`·`GooglePlacesApi`·`GoogleGeocoder` 다(`PlaceConfig.kt:17` 이 `kbap.google.places-api-key` 를 읽고, 레포 전체 `kakao` 문자열 0건). KB-350(커밋 `921a70cb`)에서 **카카오 로컬 → Google Places API (New)** 로 전환하며 `KakaoLocalApi.kt`·`KakaoPlaceSearchClient.kt` 가 삭제됐고 문서만 남았다. **포트 열거·어댑터 열거는 디렉터리가 정본.**

---

## 1. 주제 승계 매트릭스 — 옛 21노트를 새 커리큘럼에서 어떻게 처리하나

판정 축: **재작성**(주제를 새 노트로 다시 씀) / **주제 승계**(주제가 다른 새 노트에 흡수·분할되어 들어감) / **주제 폐기**(새 커리큘럼에 자리 없음).
전부 신규 집필이므로 "부분개작"은 없다. 마지막 칸은 **옛 노트에서 살려 쓸 자산**(비유·FE 대비·구성)이다.

| 옛 # | 옛 노트 | 처리 | 새 번호 | 살려 쓸 자산 / 버릴 것 |
|------|---------|------|---------|------------------------|
| 1 | 서버는 켜두는 프로그램 | 재작성 | **1** | 살림: "켜두고 기다리는 프로그램" 세계관 전환, 진입점 설명. 버림: `:app:api:boot` 좌표 |
| 2 | 요청 하나에 스레드 하나 | 재작성 | **2** | 살림: **카페 바리스타 200명 비유(경계 명시까지)**, 톰캣 기본값 실측 문단. 버림: "가상 스레드 — LLM fan-out" 절 전체(fan-out 제거, KB-320) |
| 3 | JVM 위의 Kotlin | 재작성 | **3** | 살림: TS 대비 널 안전·컴파일 안전망 서술. 버림: `:core:*`·`:infra:*` 모듈 나열 → 3모듈 + version catalog·buildSrc로 교체 |
| 4 | IoC 컨테이너와 Bean | 재작성 | **4** | 살림: React Provider 조립 대비. 예시 클래스만 `MemberService` 로 교체 |
| 5 | DI와 계층 구조 | 주제 승계(분할) | **4**+**5** | DI 개념은 4로, 계층은 5로. **버림: Controller/UseCase/도메인 3계층 서술 전체** — UseCase 계층이 존재하지 않는다(ADR-0017) |
| 6 | 설정과 프로필, 트랜잭션 | 재작성 | **6** | 살림: 프로필 4종·yml 구조. 추가: 트랜잭션 규약 강화(전 public 메서드 명시·readOnly·**격리수준 손대지 않음**, `CLAUDE.md:121-122`) |
| 7 | 영속성 컨텍스트와 엔티티 매핑 | 재작성 | **8** | 살림: **React Query 캐시 ↔ 영속성 컨텍스트 비유**, "FE 응답 타입 vs 엔티티" 대비, flush≠commit 콜아웃, BaseEntity·소프트삭제 절. **버림: 6절 "도메인↔엔티티 변환 — 도메인은 JPA를 모른다" 전체**(ADR-0014로 반대가 됨) |
| 8 | LAZY 로딩과 N+1 문제 | 주제 승계(재정의) | **9** | 살림: **장바구니 비유 + FE 워터폴(useEffect 개별 fetch) 대비**, 키셋 2단계 페이징 절. **버림: "모든 연관을 LAZY로 고정 + fetch join 규약"이라는 결론 전체** — 연관관계가 0개고 ArchUnit이 금지한다. 새 결론: N+1을 설계로 제거했다 |
| 9 | Flyway 마이그레이션 | 재작성 | **10** | 살림: **Git 커밋 로그 비유**(수정 금지·checksum 동결), 스키마 owner, 테스트도 같은 마이그레이션. 갱신: 49개, 19번 링크 → 새 26번 |
| 10 | 클린 아키텍처와 ports & adapters | 주제 승계(축소) | **12** | 살림: 의존성 역전 개념. **버림: `:persistence`·`:core:kernel` 서사, "도메인은 ORM을 모른다"** — 포트는 이제 **외부 시스템 seam에만** 남았다(ADR-0012·0016·0018) |
| 11 | 모듈러 모놀리스와 DDD | 주제 승계(분할) | **11**+**13** | 살림: 바운디드 컨텍스트 개념, 모노레포 패키지 대비. **버림: 8개 모듈 나열 전체** → 3모듈 + `common.domain.<context>` 패키지(11), 경계 강제는 ArchUnit 전용 노트(13)로 분리 |
| 12 | SDD와 TDD | 재작성 | **14** | 살림: 헌법 원칙 I 인용 구조, SpecKit 4단계. **버림: 멀티에이전트 TDD 하네스 서술**(2026-08-14 제거, `CLAUDE.md:7`). 갱신: specs 125개 |
| 13 | 온보딩 프로필 저장 흐름 | 재작성 | **15** | 살림: "가장 얇은 세로 슬라이스" 포지션. 버림: `MemberProfileUseCase` 경로 전체. 추가: 같은 경로 두 버전 핸들러(1.0 / 1.1+) |
| 14 | 메뉴판 스캔 판정 흐름 | 주제 승계(분할) | **19**+**20** | 살림: 위험도 4단계 판정 로직 설명. 버림: `ScanUseCase`. 분할 근거: v1(클라 OCR 수신)과 v2(서버 비전 LLM + 티켓 + Redis 예약)가 다른 흐름 |
| 15 | 소셜 로그인 흐름 | 재작성 | **16** | 살림: Firebase 검증→자체 JWT 시퀀스, FE 인증 노트와의 접점. 갱신: `AuthService`, `common.port.auth`, `api.infra.auth.{firebase,token}` |
| 16 | JWT 인증 필터와 세션 생명주기 | 재작성 | **17** | 살림: 필터=횡단 계층 설명, 회전·로그아웃 시퀀스. 추가: 게스트 예외(`shouldNotFilter`), MDC·`RequestLoggingFilter` 순서, 역할 속성 |
| 17 | 홈 화면 조회 흐름 | 재작성 | **18** | 살림: fan-in 조합 설명 뼈대. 갱신: `HomeService` 의존 4개(member·food·scan·ingredient) |
| 18 | 회원 랭킹 흐름 | 재작성 | **23** | 살림: **원자적 UPDATE로 lost update 막기** 설명(2번 노트와 상호 링크), 조회 시점 계산. 추가: `MemberRankingEvent` 원장·unique 제약·티어 |
| 19 | 배치 실행 골격 | **주제 폐기** | — (**26**이 새로 씀) | "조사 대기열 순회" 구조 자체가 없음. 두 번째 bootJar라는 사실만 26으로 넘어간다 |
| 20 | LLM 스코어링 파이프라인 | **주제 폐기** | — (**28**이 자리 대체) | 3모델 fan-out·consensus·가상스레드 병렬 전부 제거(KB-320), 콘텐츠 채움은 kbap-langchain 이관(KB-301). 살릴 자산 없음 |
| 21 | 캐싱과 키-값 저장소 | 재작성 | **29** | 살림: **AsyncStorage 비유(경계 명시)**, "정직하게: 여기서 Redis는 캐시가 아니다" 절 구성, 캐시의 세 질문. 추가: `RedisScanReservationStore` |
| 🗺️ | 공부 지도 | 새로 작성 | 새 지도 | 단계 구분 서문("아래에서 위로 쌓는다")은 살린다 |

집계: **재작성 12 · 주제 승계 5 · 주제 폐기 2** (+지도). 옛 노트 중 **자산을 통째로 버리는 건 19·20 둘뿐**이다.

---

## 2. 새 커리큘럼 — 최종 노트 목록 (파일명 확정안)

경로: `kbap 백엔드 (new)/`. 담당 `C`=concept-writer, `F`=code-flow-writer.
**파일명 = 제목**. 옛 폴더의 21개 파일명과 **한 글자도 겹치지 않음을 확인**했다(같은 번호 자리의 1~21번 제목이 전부 다름) — 볼트 안 동명 파일이 생기면 위키링크가 모호해지므로 이 제약은 집필 시에도 유지한다.

### 1단계 · 백엔드 기초
| # | 파일명 | 담당 | 한 줄 | 근거 파일 |
|---|--------|------|-------|-----------|
| 1 | `1. 서버는 켜두는 프로그램 — 프로세스·포트·두 개의 실행 파일` | C | 서버 프로세스·포트·진입점, bootJar 2개 | `settings.gradle.kts`, `api/.../KbapApiApplication.kt`, `batch/.../KbapBatchApplication.kt` |
| 2 | `2. 요청 하나에 스레드 하나 — JVM 동시성과 Node 이벤트루프` | C | 스레드풀 vs 이벤트루프, 블로킹이 왜 괜찮나 | `api/src/main/resources/application.yml`(스레드 설정 없음 = 톰캣 기본) |
| 3 | `3. JVM 위의 Kotlin — 컴파일·널 안전·Gradle 3모듈` | C | Kotlin 2.3/Java 21, version catalog·buildSrc 컨벤션 플러그인 | `gradle/libs.versions.toml`, `buildSrc/`, `CLAUDE.md:57-69` |

### 2단계 · Spring 핵심
| # | 파일명 | 담당 | 한 줄 | 근거 파일 |
|---|--------|------|-------|-----------|
| 4 | `4. IoC 컨테이너와 의존성 주입 — 부품을 직접 만들지 않는다` | C | 컨테이너가 만들고 생성자로 꽂아준다 | `api/core/config/*.kt`, `api/member/MemberService.kt` |
| 5 | `5. kbap의 계층 — 컨트롤러에서 리포지토리까지` | C | Controller(+`*Api` 인터페이스)→Facade(선택)→Service→public Repository. UseCase는 왜 없나 | `api/scan/{ScanController,ScanApi,ScanFacade,ScanService}.kt`, ADR-0014·0017 |
| 6 | `6. 설정과 프로필, 트랜잭션 경계` | C | yml·프로필 4종, `@Transactional` 규약(readOnly·격리수준 불간섭) | `api/src/main/resources/application*.yml`, `CLAUDE.md:121-122` |
| 7 | `7. API 계약의 모양 — 공통 봉투·에러 코드·헤더 버저닝` | C | 모든 응답이 `BaseResponse<T>`, `ErrorCode` 채번, `X-API-Version` 필수 | `api/core/BaseResponse.kt`, `common/core/error/ErrorCode.kt`, `api/core/config/WebConfig.kt:30-40`, `api/member/MemberController.kt:19-63` |

### 3단계 · JPA·영속성
| # | 파일명 | 담당 | 한 줄 | 근거 파일 |
|---|--------|------|-------|-----------|
| 8 | `8. 영속성 컨텍스트와 엔티티 매핑 — 엔티티가 곧 도메인 모델` | C | 객체↔행 매핑, 변경 감지, BaseEntity·소프트삭제 | `common/domain/BaseEntity.kt`, `common/domain/food/model/Food.kt` |
| 9 | `9. 연관관계를 쓰지 않는 설계 — N+1을 애초에 만들지 않기` | C | N+1이 뭔지 → kbap은 연관 대신 `Long` id, 키셋 2단계 페이징 | `ModuleBoundaryTest.kt:150-165`, `FoodJpaRepository.kt:107-120`, `CLAUDE.md:117` |
| 10 | `10. Flyway 마이그레이션 — 스키마를 코드로 관리하기` | C | timestamp 버전 규칙, 스키마 owner=api, 테스트도 같은 마이그레이션 | `api/src/main/resources/db/migration/`(49) |

### 4단계 · 아키텍처·방법론
| # | 파일명 | 담당 | 한 줄 | 근거 파일 |
|---|--------|------|-------|-----------|
| 11 | `11. 모듈러 모놀리스와 DDD — 3모듈과 도메인 패키지` | C | 모듈 3개로 다이어트, 바운디드 컨텍스트는 `common.domain.<context>` | `settings.gradle.kts`, `common/domain/`(19 컨텍스트), ADR-0016 |
| 12 | `12. 포트와 어댑터 — 외부 시스템을 갈아끼우는 자리` | C | 계약은 `common.port.*`, 구현은 소비 모듈, 조립은 config. **비고: 핵심 실증 = KB-350 카카오→Google 전환**(커밋 `921a70cb`) — 컨트롤러·서비스는 **무변경**이고 포트 `common/port/place/PlaceSearchClient.kt` 만 +4/−7 로 수술됐다(카카오식 `searchPage`·`PlaceSearchPage` 제거, 구글의 `placeId`·`lang` 추가). "포트는 변경을 없애는 게 아니라 **번지는 범위를 가둔다**" | `common/port/**`, `common/infra/{llm,mq}/`, `api/infra/{auth,redis,storage,place,exchange}/`, ADR-0018 |
| 13 | `13. 경계를 지키는 테스트 — ArchUnit이 강제하는 규칙들` | C | 커널 Spring-free·도메인→포트 금지·`@Entity` 위치·연관관계 금지 | `api/src/test/kotlin/com/kbap/api/architecture/ModuleBoundaryTest.kt` |
| 14 | `14. SDD와 TDD — 스펙과 테스트로 개발하기` | C | SpecKit 4단계, 헌법 원칙 I, Kotest BehaviorSpec | `.specify/memory/constitution.md:156-170`, `specs/`(125), `CLAUDE.md:7,107` |

### 5단계 · 실코드 흐름
| # | 파일명 | 담당 | 한 줄 | 근거 파일 |
|---|--------|------|-------|-----------|
| 15 | `15. 온보딩 프로필 저장 흐름 — 요청이 DB에 닿기까지` | F | 가장 얇은 세로 슬라이스 + 같은 경로 두 버전 핸들러. **비고: self-invocation 실물 자산** — `completeOnboarding`(`MemberService.kt:22`, `@Transactional`)이 같은 클래스 `getMember`(:130, `readOnly = true`)를 내부 호출해 프록시를 안 거치므로 readOnly 가 적용되지 않는다 | `api/member/{MemberController,MemberApi,MemberService,OnboardingRequest}.kt`, `common/domain/member/model/*` |
| 16 | `16. 소셜 로그인 흐름 — Firebase 토큰이 자체 JWT가 되기까지` | F | 소셜 검증→회원 조회/가입→토큰 발급 | `api/auth/{AuthController,AuthService,LoginResult}.kt`, `common/port/auth/*`, `api/infra/auth/{firebase,token}/` |
| 17 | `17. JWT 인증 필터와 세션 생명주기 — 매 요청 검증·재발급·로그아웃` | F | 필터라는 횡단 계층, 게스트 예외, refresh 회전 | `api/core/auth/JwtAuthenticationFilter.kt`, `api/core/config/WebConfig.kt`, `api/infra/redis/RedisRefreshTokenStore.kt` |
| 18 | `18. 홈 화면 조회 흐름 — 여러 서비스를 한 응답으로 모으기` | F | 읽기 fan-in 조합 | `api/home/{HomeController,HomeService,HomeResult,HomeResponse}.kt` |
| 19 | `19. 메뉴판 스캔 판정 흐름 — 메뉴명이 위험도가 되기까지` | F | 메뉴명→기피 성분 매칭→위험도(v1 경로) | `api/scan/{ScanController,ScanService,ScanResult}.kt`, `common/domain/{scan,ingredient}/` |
| 20 | `20. 스캔 v2와 사용량 티켓 — 서버 OCR·무료 한도·중복 요청 막기` | F | 티켓 검증→Redis 예약→비전 LLM→카운트, 실패 시 예약 해제 | `api/scan/{ScanV2Controller,ScanFacade,ScanTicketController,ScanTicketIssueService}.kt`, `common/port/scan/*`, `api/infra/redis/RedisScanReservationStore.kt` |
| 21 | `21. 이미지 업로드 흐름 — presigned URL과 업로드 완료 확정` | F | 서버가 서명한 URL로 클라가 직접 S3 업로드, 완료 콜백으로 확정 | `api/image/*`, `common/port/storage/*`, `api/infra/storage/S3PresignedUploadAdapter.kt`, `api/food/FoodImageBatchCollectService.kt:42-43`(ShedLock 콜아웃) |
| 22 | `22. 리뷰 작성 흐름 — 쓰기 경로와 랭킹 원장` | F | 리뷰 생성/삭제가 랭킹 이벤트를 남기고 unique 제약이 중복 가감을 막는다. 신고·차단 필터도 흡수 | `api/review/*`, `common/domain/review/`, `common/domain/member/MemberRankingEventJpaRepository.kt`, `api/{report,block}/*` |
| 23 | `23. 회원 랭킹 — 원자적 카운트와 조회 시점 점수 계산` | F | 원자적 UPDATE·값 객체 점수 계산·티어 | `api/member/{MemberService,MemberRankingResult}.kt`, `common/domain/member/model/{Ranking,RankingTier,MemberRankingEvent}.kt` |
| 24 | `24. 주문 흐름 — 스캔 결과가 주문 이력이 되기까지` | F | 주문 생성(스캔 이미지 연계·중복 이미지 충돌 처리)→커서 목록→상세. **연관관계 없이 썸네일을 배치 로딩하는 실물**(9번 노트의 실전판) | `api/order/{OrderController,OrderApi,OrderService,OrderCreateRequest,OrderResponses}.kt`, `common/domain/order/{OrderJpaRepository,OrderItemJpaRepository,model/{Order,OrderItem}}.kt`, `specs/kb-337-order-history/`, `specs/kb-376-order-detail-scan-image/` |
| 25 | `25. 커뮤니티 흐름 — 포스팅·댓글·커서 페이징` | F | 포스팅 CRUD + 대댓글(`parentCommentId`) + 커서 페이지 + 뷰어별 차단 필터(`viewerMemberId`). **비고: 조사 종결 — 의도다.** 게스트 댓글 401 은 `specs/kb-292-community-comments/spec.md:89` **FR-010** 의 명시 요구다 — 목록 조회는 인증 오류로 거부하고 **개수(`commentCount`)만 노출, 블러는 FE 책임**(`contracts/comments-api.md`: 댓글 4개 엔드포인트 전부 회원 전용). 코드만 보면(필터 URL 패턴 `community/posts/*` 에는 걸리는데 게스트 예외 정규식 `^/api/community/posts/\d+$` 에는 안 맞고 핸들러가 `@AuthMemberId` 필수, `CommunityController.kt:112-114`) **버그로 오인하기 쉬운 자리** — 노트에 이 대비를 그대로 쓴다. 문서-코드 불일치가 아니므로 §0 목록에 넣지 않는다 | `api/community/{CommunityController,CommunityApi,CommunityService,PostingCreateRequest,CommentCreateRequest,PostingListRequest}.kt`, `common/domain/community/{PostingJpaRepository,CommentJpaRepository,model/{Posting,Comment}}.kt`, `specs/kb-290-community-post-domain/`, `kb-291-community-feed-read/`, `kb-292-community-comments/` |
| 26 | `26. 배치 앱의 골격 — Spring Batch 잡·트리거·스케줄` | F | 두 번째 bootJar가 잡을 정의·기동·기록하는 법 | `batch/config/BatchJdbcJobRepositoryConfig.kt`, `batch/trigger/*`, `batch/schedule/BatchJobScheduler.kt`, `batch/src/main/resources/application.yml` |
| 27 | `27. 아웃박스와 SQS — 다른 시스템에 일을 넘기는 법` | F | DB 아웃박스 행 → 큐 발행 → kbap-langchain. 같은 앱 안 비동기(`@EventListener`+`@Async`)와 대비 | `batch/outbox/*`, `common/port/mq/*`, `common/infra/mq/SqsFoodContentEventPublisher.kt`, `api/metering/LlmCallCostEventListener.kt` |
| 28 | `28. 임베딩과 벡터 검색 — 뜻으로 음식 찾기` | F | 텍스트→256차원 벡터→문서 DB 적재→유사도 검색 | `batch/vector/*`, `common/domain/food/vector/*`, `common/port/llm/TextEmbeddingClient.kt`, `specs/kb-328-food-vector-outbox/` |

**24번 목차 초안**: 주문이 뭘 담나(스캔 이미지 + 항목) → `createOrder` 트랜잭션과 `imagePath` unique 충돌 폴백(`OrderService.kt:31-59`) → 커서 목록(`getOrderPage`) → 상세와 **썸네일 배치 로딩**(`loadFoodsById`/`resolveThumbnails`, `OrderService.kt:124-133`) → 연관관계 없이 N+1을 피하는 법(9번과 상호 링크).
**25번 목차 초안**: 포스팅 도메인(`Posting`) → 작성·수정·삭제와 소유자 검증(`CommunityService.kt:32-72`) → 피드 조회의 커서 페이징과 **뷰어별 차단 필터**(`getPostingPage(viewerMemberId, ...)`) → 댓글·대댓글 트리(`createComment(parentCommentId)`) → 22번(신고·차단)과의 접점.

### 확장
| # | 파일명 | 담당 | 한 줄 | 근거 파일 |
|---|--------|------|-------|-----------|
| 29 | `29. 캐싱과 키-값 저장소 — kbap이 Redis를 쓰는 두 가지 방식` | C | 캐싱 일반 → Redis는 여기서 캐시가 아니다(refresh·스캔 예약) | `api/infra/redis/*.kt` |

**지도**: `kbap 백엔드 (new)/🗺️ kbap 백엔드 공부 지도 (2026-08).md`
— 옛 지도와 **파일명이 달라야 한다**(동명 파일 2개면 `[[🗺️ kbap 백엔드 공부 지도]]` 링크가 모호해진다).

**합계 29노트 + 지도 1.**

---

## 3. 신규 커버 대상 — 취사선택

### 채택 (옛 커리큘럼에 없던 10주제)
| 대상 | 새 # | 왜 |
|------|------|-----|
| API 계약·헤더 버저닝 | 7 | 컨트롤러 아무거나 열면 첫 줄에 나온다. 모르면 5단계 전부가 안 읽힌다. `appversion` 강제 업데이트 게이트도 콜아웃으로 흡수 |
| ArchUnit 경계 테스트 | 13 | 모듈이 3개로 줄면서 경계를 지키는 **유일한 강제 수단**이 됐다 |
| scan v2 + 티켓 + Redis 예약 | 20 | 앱 핵심 가치 경로이자 한도·중복·실패 해제라는 동시성 실전이 한 파일에 있다 |
| 이미지 presigned 업로드 | 21 | FE가 직접 구현할 계약이고, 볼트에 `S3 presigned URL과 클라이언트 시크릿` 노트가 이미 있어 복리가 즉시 붙는다 |
| 리뷰 쓰기 + 랭킹 원장 | 22 | 옛 지도가 "리뷰 도입 시 후속"으로 예고한 것의 실현. 쓰기 트랜잭션·원장·unique 제약 |
| **주문 흐름** | 24 | **사용자 결정으로 채택** — 합류 직후 다룰 가능성. 덤으로 "연관관계 없이 썸네일 배치 로딩"이라는 9번 노트의 실전 사례가 여기 있다 |
| **커뮤니티 흐름** | 25 | **사용자 결정으로 채택** — 합류 직후 다룰 가능성. 대댓글 트리·뷰어별 차단 필터는 리뷰 흐름에 없는 요소다 |
| Spring Batch 골격 | 26 | 옛 19번 폐기의 대체. 배치 앱을 못 읽으면 레포 절반이 빈칸 |
| 아웃박스 + SQS | 27 | "왜 우리 배치는 LLM을 안 부르나"(kbap-langchain 이관)의 답이자 시스템 간 경계 |
| 임베딩·벡터 검색 | 28 | 옛 20번의 자리 대체. 배치→외부 저장소 파이프라인의 현재형 |

### 제외 (사유)
| 대상 | 뺀 이유 |
|------|---------|
| `report`/`block` | 조회 필터 한 겹 → 22번에 흡수(25번에서도 뷰어 필터로 재등장) |
| `bookmark` | 동형 CRUD |
| `place`(Google Places·Geocoding) | 외부 어댑터의 또 다른 예 → 12번에서 사례로 인용 |
| `admin`(7컨트롤러·Thymeleaf) | 서버사이드 렌더링은 별세계이고 합류 초기에 만질 일이 적다 |
| `appversion` 버전 게이트 | 7번 콜아웃 |
| `metering`(LLM 비용 계량) | 도메인 이벤트 개념만 값지다 → 27번에서 "같은 앱 안 비동기 vs 다른 시스템 큐" 대비로 흡수 |
| ShedLock | 21·26번 콜아웃(실제 위치는 `api/food/FoodImageBatchCollectService`) |
| Spring Boot 4 자체 | 버전 숫자는 노트감이 아니다. 실질 영향(네이티브 API 버저닝)은 7번이 다룬다 |

---

## 4. 실행 계획

### 4-1. 아카이브 처리 절차 (R0 — vault-keeper, 노트 집필 전 선행)

1. **폴더 이동**: `kbap 백엔드/` → `kbap 백엔드 (old)/`. 21개 파일 + 지도 **파일명은 그대로**. 폴더가 바뀌어도 위키링크는 파일명으로 해석되므로 **old 폴더 내부 상호 링크(총 100여 개)와 외부 인바운드 링크가 전부 살아 있다.**
2. **배너**: old 폴더 21노트 + 지도 상단(frontmatter 바로 아래)에 붙인다. 초안 —
   ```markdown
   > [!warning] 보관 노트 — 현재 코드와 다름 (2026-08-26 확인)
   > 2026-07 시점 kbap-server 를 기준으로 쓴 노트다. 이후 모듈 개편(ADR-0016·0018)·계층 개편(ADR-0017)·배치 재작성으로 **여기 적힌 구조·클래스 이름 상당수가 지금은 존재하지 않는다.** 기록으로만 남긴다.
   > 지금 기준 노트 → [[<새 노트 파일명>]] · 전체 목차 → [[🗺️ kbap 백엔드 공부 지도 (2026-08)]]
   ```
   - `<새 노트 파일명>` 자리는 §1 매트릭스의 "새 번호" 열에서 채운다. **주제 폐기(19·20)** 두 노트는 그 줄을 "대체 노트 없음 — 해당 구조가 제거됨(KB-301·KB-320)"으로 바꾼다.
   - **번호 재배치 반영**: 옛 1~18의 대체 번호는 §1 표 그대로 변동 없다. 바뀐 건 **옛 21(캐싱) → 새 29** 하나뿐이며, 참고용으로 언급되는 폐기 노트 대체 번호도 옛 19→새 26, 옛 20→새 28로 밀렸다.
   - frontmatter 에 `상태: 보관` 추가. (볼트 규칙 §5의 상태 값 확장 — 사용자 승인 필요.)
3. **옛 지도**: 파일명 유지 + 위 배너 + 서문 한 줄 교체("이 지도는 2026-07 커리큘럼의 기록이다. 현행 커리큘럼 → [[🗺️ kbap 백엔드 공부 지도 (2026-08)]]"). 체크박스·본문은 **손대지 않는다**(당시 진도 기록으로서의 값이 있다).
4. **홈(`🏠 홈.md`) 표기**: 현재 kbap 섹션은 지도 한 줄뿐이다(`🏠 홈.md:24-25`). 두 줄로 바꾼다 —
   ```markdown
   ### kbap 백엔드
   - [[🗺️ kbap 백엔드 공부 지도 (2026-08)]] — FE 개발자가 kbap-server(Kotlin·Spring)에 합류하기 위한 단계별 커리큘럼 (개별 노트는 지도에서 뻗어나감)
   - [[🗺️ kbap 백엔드 공부 지도]] — 📦 보관: 2026-07 시점 커리큘럼(21노트). 현재 코드와 다름
   ```
   old 지도를 홈에서 빼면 고아 노트가 되어 린트(규칙 §6)에 걸리므로 **보관 줄로 남긴다.**
5. **작업 로그**: `📜 작업 로그.md` 에 append 한 줄. 기존 줄(7번 노트 참조 2건)은 append-only 원칙상 **수정하지 않는다** — old 파일이 살아 있어 링크도 유효하다.

### 4-2. 역링크 이관 매핑 (전수 조사 결과)

볼트 전체를 옛 21노트 + 지도 파일명으로 grep한 결과, **kbap 폴더 밖에서 들어오는 링크는 9개 노트에서 15건**이다. 아카이브 이동 자체로는 **한 건도 깨지지 않는다**(파일명 유지). 아래는 새 노트 완성 후 **링크를 갈아끼울 계획**이며, 실제 수정은 vault-keeper 담당이다.

| 참조하는 외부 노트 | 현재 대상(옛) | 갈아끼울 대상(새) | 본문 수정 필요? |
|---------------------|---------------|-------------------|-----------------|
| `🏠 홈.md` | 🗺️ 지도 | 새 지도 + old 보관 줄 | 예 (§4-1-4) |
| `📜 작업 로그.md` ×2 | 7. 영속성 컨텍스트 | **교체하지 않음** | 아니오 (append-only 이력) |
| `Claude Code 작동원리/2. 행동을 다듬는 장치 …` | 12. SDD와 TDD | 14. SDD와 TDD | **예** — "kbap이 서브에이전트로 TDD를 굴린다"는 서술이 거짓(2026-08-14 하네스 제거) |
| `Claude Code 작동원리/5. 여러 에이전트의 협업` | 12. SDD와 TDD | 14. SDD와 TDD | **예** — "test-writer→implementer→reviewer 팀" 서술이 거짓 |
| `백엔드/S3 presigned URL과 클라이언트 시크릿.md` | 13 / 15 / 16 | 15 / 16 / 17 **+ 21(이미지 업로드) 신설 링크** | 예 — 직계 후속인 21번과 양방향 연결 |
| `백엔드/페이징 — Offset vs No-Offset(Keyset·Cursor).md` | 8. LAZY·N+1 | 9. 연관관계를 쓰지 않는 설계 | **예** — "fetch join과 충돌해 2단계로 결합(`FoodRepositoryAdapter.findFoodPage`)"이 거짓. 실제는 `FoodJpaRepository.findFoodPageIds` + `findByIdIn`, fetch join 없음 |
| `인증·보안/쿠키·세션 vs 토큰 인증 — 왜 앱은 JWT인가.md` | 15 / 16 | 16 / 17 | 아니오 (링크만) |
| `인증·보안/Google vs Apple 소셜 로그인 비교.md` | 15 | 16 | 아니오 |
| `인증·보안/OAuth 2.0과 OIDC — 소셜 로그인의 원리.md` | 15 | 16 | 아니오 |
| `Expo & React Native/Android 구글 로그인 — SHA-1·OAuth…md` ×2 | 15 | 16 | 아니오 |
| `Expo & React Native/OCR 라인 분류 — 휴리스틱의 한계와 대안.md` | 14 | 19 (+20 병기) | 예 — 서버 OCR(v2)이 생겨 FE OCR 결과 전송은 v1 경로임을 한 줄 명시 |
| `Expo & React Native/OCR 메뉴 스캔 — 박스 그룹핑.md` | 14 | 19 (+20 병기) | 예 — 위와 동일 |
| `Expo & React Native/클라이언트 저장소 — AsyncStorage·SecureStore·메모리 (K-Bap 실전).md` | 21. 캐싱·Redis | **29.** 캐싱과 키-값 저장소 | 아니오 |

**원칙**: 링크 교체는 **대상 새 노트가 완성된 회차 직후**에만 한다(존재하지 않는 노트를 가리키는 유령 링크 방지). old 링크는 교체 전까지 유효하므로 중간 상태에서도 볼트가 깨지지 않는다.

### 4-3. 회차 계획

**10회 실행 × 노트 1~4개 = 29노트 + 지도.**

| 회차 | 묶음 | 산출물 |
|------|------|--------|
| R0 | 아카이브 | 폴더 이동·배너·frontmatter·홈 2줄·옛 지도 서문·작업 로그 (vault-keeper) |
| R1 | 2단계 | 5·7·4·6 + 새 지도 초판(전 단계 골격 + 체크박스) |
| R2 | 3단계 | 9·8·10 |
| R3 | 4단계 | 11·12·13·14 (+`Claude Code 작동원리` 2노트 역링크 수리) |
| R4 | 5단계 읽기·인증 | 15·16·17·18 (+`인증·보안` 3노트, `Expo/Android 구글 로그인` 링크 교체) |
| R5 | 5단계 스캔·이미지 | 19·20·21 (+`Expo/OCR` 2노트, `백엔드/S3 presigned` 수리) |
| R6 | 5단계 쓰기 ① 리뷰·랭킹 | 22·23 |
| R7 | 5단계 쓰기 ② 주문·커뮤니티 | 24·25 |
| R8 | 5단계 배치·비동기 | 26·27·28 |
| R9 | 1단계 | 1·2·3 |
| R10 | 확장·마무리 | 29 + 지도 최종 + 홈·작업 로그 정리 + 린트(깨진 링크·고아·홈 미등록) |

**순서 근거**: 5번(계층)이 이후 모든 노트가 쓸 어휘를 정하므로 최우선이고, 7번(API 계약)이 확정돼야 5단계 흐름 노트의 인용이 흔들리지 않는다. 쓰기 계열은 R6(리뷰·랭킹)과 R7(주문·커뮤니티)로 쪼갠다 — 24번이 9번(연관관계 없는 설계)의 실전 사례를, 25번이 22번(신고·차단)을 되받으므로 **둘 다 앞 회차가 끝난 뒤**여야 인용이 성립한다. 1단계(1·2·3)는 개념이 여전히 참이고 좌표 표기만 갱신하면 되므로 R9로 미룬다. 역링크 수리는 대상 노트가 완성된 회차에 붙여 유령 링크를 만들지 않는다.

---

## 5. 리스크·미확인

1. **동명 파일 충돌** — old 폴더를 남기므로 새 노트 제목이 옛 제목과 한 글자라도 같으면 볼트 전체에서 위키링크가 모호해진다. §2 목록은 확인했으나 **집필 중 제목을 바꾸면 반드시 재확인**해야 한다. 새 지도 파일명에 `(2026-08)` 를 붙인 이유도 이것이다.
2. **`상태: 보관` frontmatter 값**은 볼트 규칙 §5(`진행중`·`완료`·`seed`)에 없는 확장이다 — 사용자 승인 필요.
3. **레포 문서가 코드보다 낡은 곳 4건**(§0 하단). 노트는 코드 기준으로 쓴다. 레포 `CLAUDE.md` 수정 제안은 별건(우리는 읽기 전용).
4. **ShedLock 위치가 사전 요약과 다름** — presigned 회수가 아니라 `api/food/FoodImageBatchCollectService`(OpenAI 이미지 배치 회수, 3시간 cron)에 붙어 있다. 21·26번 집필 시 재확인.
5. **specs 디렉터리 125개** (사전 요약 "30+"와 큰 차이). 14번의 규모 서술은 집필 시 다시 센다.
6. 배치 잡 2종(outbox·vector)은 **기본 off**(`EMBEDDING_ENABLED`·`VECTOR_ENABLED` false)라 로컬 실행 관찰이 어렵다 — 26·28번은 코드 읽기 중심으로 쓴다.
7. `order`·`community` 는 **사용자 결정으로 채택됨**(새 24·25). 애초 제외 사유였던 "리뷰 흐름과 동형"은 여전히 일부 사실이므로, 두 노트는 22번과 겹치는 부분을 다시 설명하지 말고 **차이점(주문=이미지 연계·썸네일 배치 로딩, 커뮤니티=대댓글 트리·뷰어 필터)에 지면을 쓴다.**
8. **refresh 재사용 탐지 부재는 보안 판단이 필요한 자리다**(§0 실측표). 노트는 사실만 기록했고, 근거 문장이 ADR·스펙 어디에도 없다 — 의도된 트레이드오프인지 누락인지 **사용자에게 별도로 알릴 후보**.
8. old 폴더의 21노트는 배너를 붙여도 **검색 결과에는 계속 뜬다**. 사용자가 옛 노트를 실수로 읽는 것을 완전히 막지는 못한다 — 배너를 frontmatter 직후 최상단에 두는 이유.
