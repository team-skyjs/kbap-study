---
tags: [kbap-backend, pr읽기, admin, rest, 동시성]
생성일: 2026-08-31
상태: 완료
PR: 201
브랜치: feat/kb397-admin-food-detail-rest
기준커밋: ee3df844 (2026-08-31 갱신)
---

> [!info] 관련 노트
> 🏠 [[🏠 홈]] · 📂 주제: [[🗺️ kbap 백엔드 공부 지도 (2026-08)]]
> 🔗 이 PR 을 읽는 데 쓰인 개념: [[5. kbap의 계층 — 컨트롤러에서 리포지토리까지]] · [[7. API 계약의 모양 — 공통 봉투·에러 코드·헤더 버저닝]] · [[8. 영속성 컨텍스트와 엔티티 매핑 — 엔티티가 곧 도메인 모델]] · [[23. 회원 랭킹 — 원자적 카운트와 조회 시점 점수 계산]]

# PR 201 — 어드민 음식 상세·수정·재수집·삭제 REST

> 작성: 2026-08-31 · 맥락: 어드민을 Thymeleaf 서버 렌더링에서 **SPA + REST** 로 갈아타는 큰 작업(에픽 KB-384)의 한 조각. 이 PR 은 음식 관리 4개 동작(상세·수정·재수집·삭제)을 JSON API 로 연다.
> 목표: **파일 7개가 각각 무슨 역할이고 왜 그렇게 짰는지**를 코드 단위로 따라가고, 이 PR 이 새로 들여온 패턴(동시 수정 방어 3겹)을 설명할 수 있게 되기.

| | |
|---|---|
| 링크 | [team-skyjs/kbap-server#201](https://github.com/team-skyjs/kbap-server/pull/201) · Jira KB-397 (에픽 KB-384) |
| 상태 | **OPEN** (2026-08-31 기준 미머지) · 작성자 KYJ |
| 규모 | **7파일 · +803 / −3** (그중 테스트가 435줄 = 절반 이상, 시나리오 38개) |

---

## 0. 한 줄 요약

**기존에 Thymeleaf 화면만 쓰던 음식 관리 로직(`AdminFoodService`)을, 로직은 그대로 두고 JSON API 4개로 다시 열었다.** 새 비즈니스 로직은 거의 없고 — 진짜 새로운 건 **수정 API 의 동시 수정 방어**(비관적 락 + 버전 비교 + JPA 낙관 잠금 3겹)와 **에러 코드 2개 채번**이다.

---

## 1. 이 PR 이 하는 일 (제품 관점)

관리자가 어드민 화면에서 음식 하나를 눌렀을 때 필요한 것들:

| 동작 | 엔드포인트 | 이 PR 전에는 |
|---|---|---|
| 상세 보기 | `GET /api/admin/foods/{id}` | Thymeleaf 페이지로만 가능(HTML) |
| 수정 | `PUT /api/admin/foods/{id}` | 〃 |
| 콘텐츠 재수집 요청 | `POST /api/admin/foods/{id}/recollect` (단건)<br>`POST /api/admin/foods/recollect` (필터 일괄) | 일괄만 있었음 — **단건은 신규** |
| 소프트삭제 | `DELETE /api/admin/foods/{id}` | Thymeleaf 페이지로만 가능 |

**Thymeleaf 페이지는 한 줄도 안 건드렸다**(회귀 0). 같은 서비스 메서드를 두 입구(HTML 화면 / JSON API)가 나눠 쓰는 모양이 됐다.

> [!note] 왜 "REST 화" 인가
> 서버가 HTML 을 통째로 만들어 내려주던 방식(Thymeleaf)에서, **서버는 JSON 만 주고 화면은 브라우저에서 JS 가 그리는** 방식(SPA)으로 옮기는 중이다. FE 개발자에게 익숙한 그 구조로 어드민을 다시 짓는 것이고, 이 PR 은 그 SPA 가 먹을 데이터 창구를 여는 일이다.

---

## 2. 바뀐 파일 한눈에

| 파일 | 증감 | 한 줄 역할 |
|---|---|---|
| `api/admin/AdminFoodCatalogApi.kt` | +154 −1 | **문서 인터페이스** — Swagger 에 뜰 설명·예시·응답코드. 실행 코드 아님 |
| `api/admin/AdminFoodCatalogController.kt` | +75 | **HTTP 입구** — URL·메서드 매핑, 요청 DTO → 서비스 명령 변환, 결과 → 에러 코드 번역 |
| `api/admin/AdminFoodDetailResponse.kt` | +106 (신규) | **DTO 3종** — 상세 응답 / 수정 요청(+검증) / 재수집 응답 |
| `api/admin/AdminFoodService.kt` | +25 −2 | **비즈니스 로직** — 메서드 2개 추가, `updateFood` 에 버전 인자 추가 |
| `common/domain/food/FoodJpaRepository.kt` | +6 | **DB 접근** — 비관적 락 조회 메서드 1개 추가 |
| `common/core/error/ErrorCode.kt` | +2 | **에러 코드 2개 채번** (FOOD-005 · FOOD-006) |
| `api/test/.../AdminFoodCatalogControllerTest.kt` | +421 | 테스트 4묶음 |

> 계층 순서가 [[5. kbap의 계층 — 컨트롤러에서 리포지토리까지]] 그대로다: `*Api`(문서) → `Controller` → `Service` → `JpaRepository`. 이 PR 은 그 네 층을 한 번에 한 칸씩 늘린 셈이라 **계층 노트를 복습하기에 좋은 교재**다.

---

## 3. 파일별로 무슨 코드가 왜 들어갔나

### 3-1. `AdminFoodCatalogApi.kt` — 문서만 담당하는 인터페이스 (+154)

가장 많이 늘었는데 **실행되는 코드가 한 줄도 없다.** 전부 springdoc 애너테이션이다.

```kotlin
@Operation(
    summary = "음식 상세 조회",
    description = """
        - `ingredients` 는 미조사(null)와 조사 완료·해당 없음(빈 배열)을 구분해 내려간다 — 위험도 계산이 갈리는 도메인 구분이다.
        - 소프트삭제된 음식은 404 가 아니라 400(FOOD-001) 이다 — 조회는 ACTIVE 만 본다.
    """,
)
fun getFoodDetail(id: Long): ResponseEntity<BaseResponse<AdminFoodDetailResponse>>
```

- **왜 인터페이스로 빼나**: 컨트롤러 본문이 애너테이션 수십 줄에 파묻히지 않게 하려고. 컨트롤러는 `: AdminFoodCatalogApi` 를 구현하기만 하면 문서가 따라붙는다. → [[7. API 계약의 모양 — 공통 봉투·에러 코드·헤더 버저닝]]
- **읽는 값어치**: 이 설명문이 **팀의 계약 합의문**이다. "null 과 빈 배열을 구분한다" 같은 도메인 규칙이 코드보다 여기 먼저 적혀 있다.

### 3-2. `AdminFoodCatalogController.kt` — 번역기 (+75)

컨트롤러가 하는 일은 셋뿐이다. **비즈니스 판단을 하지 않는다.**

```kotlin
@PutMapping("/{id}")
override fun updateFood(
    @PathVariable id: Long,
    @Valid @RequestBody request: AdminFoodUpdateRequest,
): ResponseEntity<BaseResponse<AdminFoodDetailResponse>> {
    val command = UpdateFoodCommand(                       // ① 요청 DTO → 내부 명령
        koreanName = request.koreanName!!.trim(),
        nameTranslationsJson = request.nameTranslations?.let(objectMapper::writeValueAsString).orEmpty(),
        …
    )
    val result = try {
        adminFoodService.updateFood(id, command, expectedVersion = request.version)
    } catch (e: OptimisticLockingFailureException) {       // ② JPA 예외 → 우리 에러 코드
        throw BusinessException(ErrorCode.FOOD_VERSION_CONFLICT)
    }
    when (result) {                                        // ③ 결과 enum → HTTP 에러 코드
        AdminFoodUpdateResult.UPDATED -> Unit
        AdminFoodUpdateResult.NOT_FOUND -> throw BusinessException(ErrorCode.FOOD_NOT_FOUND)
        AdminFoodUpdateResult.INVALID_NAME,
        AdminFoodUpdateResult.INVALID_JSON,
        -> throw BusinessException(ErrorCode.INVALID_REQUEST)
        AdminFoodUpdateResult.DUPLICATE_NAME -> throw BusinessException(ErrorCode.DUPLICATE_FOOD_NAME)
    }
    return ResponseEntity.ok(BaseResponse.ok(adminFoodService.getFoodDetail(id)))  // ④ 반영본 재조회
}
```

- **① `!!` 가 왜 안전한가** — `@Valid` 가 먼저 돌아 `@NotBlank`·`@NotNull` 을 통과시킨 뒤에만 이 줄에 온다. 검증 실패는 여기 오기 전에 400(COMMON-002)으로 끝난다. (Kotlin 널 안전 → [[3. JVM 위의 Kotlin — 컴파일·널 안전·Gradle 3모듈]])
- **② JPA 예외를 우리 말로 번역** — `OptimisticLockingFailureException` 은 스프링 것이다. 이걸 그대로 밖으로 내보내면 500 이 된다. 우리 `ErrorCode` 로 갈아 끼워야 FE 가 `code` 로 분기할 수 있다.
- **③ 서비스는 `enum` 을 돌려주고 HTTP 는 컨트롤러가 정한다** — 서비스가 HTTP 를 몰라도 되게 하는 분업. `INVALID_NAME` 과 `INVALID_JSON` 이 **같은 400** 으로 합쳐지는 것도 여기서 결정된다.
- **④ 수정 후 다시 조회해서 돌려준다** — SPA 가 "저장 눌렀으니 화면도 이렇게 됐겠지"(낙관 업데이트)를 **서버 응답으로 확정**하게 하는 것. 서버가 정규화한 이름(`matchKey`)·자동 증가한 `version` 이 반영된 진짜 값이 내려간다.

> [!warning] 눈에 띈 것 — `objectMapper` 를 컨트롤러가 직접 만든다
> `private val objectMapper = jacksonObjectMapper()` 로 **필드에서 새로 생성**한다. 스프링이 이미 설정된 `ObjectMapper` 빈을 갖고 있는데 그걸 안 쓴 것이라, 나중에 전역 직렬화 설정(날짜 포맷·미지 필드 무시 등)을 바꿔도 이 자리만 안 따라간다. 동작에는 문제가 없지만 **주입받는 편이 관행**이다. → [[4. IoC 컨테이너와 의존성 주입 — 부품을 직접 만들지 않는다]]

### 3-3. `AdminFoodDetailResponse.kt` — DTO 세 개 (+106, 신규 파일)

**① `AdminFoodDetailResponse`** — 상세 응답. 엔티티를 그대로 내보내지 않고 API 모양으로 옮겨 담는다.

```kotlin
fun from(food: Food, imagePublicBaseUrl: String) = AdminFoodDetailResponse(
    koreanName = food.displayName(LanguageCode.KO),   // 정규화 키가 아니라 보여줄 이름
    imageRef = food.imageRef,                          // 저장된 S3 키
    imageUrl = ImageUrls.resolve(imagePublicBaseUrl, food.imageRef),  // 조회 시점 URL 조립
    ingredients = food.ingredients,                    // null 과 빈 배열을 구분해 그대로
    version = food.version,                            // ← 수정 요청에 다시 실어 보낼 값
    …
)
```

- **`imageRef` 와 `imageUrl` 을 둘 다 준다** — 저장은 키로, 노출은 URL 로. CDN 도메인이 바뀌어도 DB 가 안 썩는 설계다. → [[21. 이미지 업로드 흐름 — presigned URL과 업로드 완료 확정]]
- **`ingredients: List<FoodIngredient>?` 의 `?` 가 도메인 의미를 갖는다** — `null` = **아직 조사 안 함**, `[]` = **조사했고 해당 성분 없음**. 둘을 합치면 위험도 판정이 달라진다. → [[19. 메뉴판 스캔 판정 흐름 — 메뉴명이 위험도가 되기까지]]
- **`version` 을 응답에 넣는 이유** — 이게 §4 의 동시 수정 방어의 출발점이다.

**② `AdminFoodUpdateRequest`** — 수정 요청 + 검증 규칙.

```kotlin
data class AdminFoodUpdateRequest(
    @field:NotBlank @field:Size(max = 255) val koreanName: String? = null,
    @field:NotNull @field:Min(-1) @field:Max(10) val spiciness: Int? = null,
    val ingredients: List<FoodIngredient>? = null,
    @field:NotNull                                   // ← 2026-08-31 필수화 (§9)
    val version: Long? = null,
) {
    @AssertTrue(message = "ingredients 의 code 는 성분 카탈로그 코드여야 합니다")
    fun isIngredientCodesKnown(): Boolean = ingredients.orEmpty().all { it.code in KNOWN_INGREDIENT_CODES }
}
```

- **타입은 `String?` 인데 `@NotBlank`** — Kotlin 은 필드가 없으면 널이 되므로, "요청에 그 키가 아예 없음"을 널로 받아 검증기가 잡게 하는 흔한 관용구다. 타입을 `String`(널 불가)으로 두면 역직렬화 단계에서 다른 방식으로 터진다.
- **`@AssertTrue` 는 필드 하나로 판단할 수 없는 규칙**을 메서드로 검사한다. 여기선 "성분 코드가 카탈로그에 실재하는가". `IngredientCode` enum 을 진실의 출처로 삼는다.
- **`spiciness` 최솟값이 `-1`** — 0 이 아니다. **-1 = 미조사**라는 도메인 약속이 검증 범위에 박혀 있다.
- **전체 교체(PUT) 계약** — 번역 맵을 생략하면 "그대로 두기"가 아니라 **빈 맵으로 교체**된다. FE 가 부분 수정을 하려면 상세를 받아 통째로 다시 보내야 한다.

**③ `AdminFoodRecollectResponse`** — 단건·일괄이 **같은 응답 모양**(`requested/created/skipped/exceeded/max`)을 쓰도록 맞춘 DTO. SPA 가 두 버튼을 한 코드로 처리할 수 있다.

### 3-4. `AdminFoodService.kt` — 로직 (+25 −2)

새 메서드 둘, 기존 메서드 한 개에 인자 추가.

```kotlin
@Transactional(readOnly = true)
fun getFoodDetail(id: Long): AdminFoodDetailResponse =
    foodRepository.findById(id).orElse(null)
        ?.let { AdminFoodDetailResponse.from(it, imagePublicBaseUrl) }
        ?: throw BusinessException(ErrorCode.FOOD_NOT_FOUND)
```

- 기존에 `getFoodDetailOrNull`(Thymeleaf 용, 없으면 `null`)이 있는데 **REST 용을 따로 만들었다.** 화면은 null 을 받아 "없음" 페이지를 그리고, API 는 400 을 던져야 해서 **실패 표현 방식이 다르기 때문**이다.
- **소프트삭제된 음식이 왜 자동으로 걸러지나** — `BaseEntity` 에 `@SQLRestriction("status = 'ACTIVE'")` 가 걸려 있어 하이버네이트가 모든 조회에 `status='ACTIVE'` 를 자동으로 붙인다. 그래서 `findById` 가 삭제된 행을 **아예 못 본다** → FOOD-001. → [[8. 영속성 컨텍스트와 엔티티 매핑 — 엔티티가 곧 도메인 모델]]

```kotlin
@Transactional
fun requestRecollectForFood(id: Long): AdminFoodRecollectResult {
    val food = foodRepository.findByIdForUpdate(id) ?: throw BusinessException(ErrorCode.FOOD_NOT_FOUND)
    val alreadyPending = outboxRepository
        .findByFoodIdInAndOutboxStatus(listOf(food.id), FoodContentOutboxStatus.PENDING).isNotEmpty()
    if (alreadyPending) return AdminFoodRecollectResult(requested = 1, created = 0, skipped = 1)
    outboxRepository.save(FoodContentOutbox.pending(food.id, food.displayName))
    return AdminFoodRecollectResult(requested = 1, created = 1, skipped = 0)
}
```

- **멱등**하다 — 이미 PENDING 이면 새로 만들지 않고 `skipped=1`. 관리자가 버튼을 두 번 눌러도 큐에 중복이 안 쌓인다.
- 여기서도 **비관적 락으로 읽는다**(`findByIdForUpdate`). "확인하고 나서 넣는"(check-then-act) 사이에 다른 요청이 끼는 걸 막는 것 — [[20. 스캔 v2와 사용량 티켓 — 서버 OCR·무료 한도·중복 요청 막기]] 의 Redis Lua 와 **같은 문제, 다른 도구**다.
- 아웃박스에 행을 넣는 것이 곧 "일 시켜줘"라는 뜻이다. 그 뒤 배치가 집어간다 → [[27. 아웃박스와 SQS — 다른 시스템에 일을 넘기는 법]]

`updateFood` 는 시그니처만 바뀌었다(`expectedVersion: Long? = null` 추가, 기본값이 있어 **기존 Thymeleaf 호출부는 무변경**). 본문은 §4 참조.

### 3-5. `FoodJpaRepository.kt` — 비관적 락 조회 (+6)

```kotlin
@Lock(LockModeType.PESSIMISTIC_WRITE)
@Query("select f from Food f where f.id = :id")
fun findByIdForUpdate(@Param("id") id: Long): Food?
```

**이 6줄이 이 PR 에서 가장 중요한 코드**다. 실행되는 SQL 뒤에 `FOR UPDATE` 가 붙어, **이 행을 읽은 트랜잭션이 끝날 때까지 다른 트랜잭션은 같은 행을 못 읽고 기다린다.**

> 📌 [[23. 회원 랭킹 — 원자적 카운트와 조회 시점 점수 계산]] 에서 "kbap 은 비관적 락을 쓰지 않는다(`PESSIMISTIC`·`@Lock` 0건)"고 실측했었다. **이 PR 이 레포의 첫 비관적 락**이다. 랭킹은 초당 여러 번 일어나는 사용자 트래픽이라 락이 부담이지만, 어드민 수정은 **사람이 손으로 누르는 저빈도 작업**이라 성격이 다르다.

### 3-6. `ErrorCode.kt` — 코드 2개 채번 (+2)

```kotlin
DUPLICATE_FOOD_NAME("FOOD-005", 409, "이미 같은 이름의 음식이 있습니다"),
FOOD_VERSION_CONFLICT("FOOD-006", 409, "다른 관리자가 먼저 수정했습니다. 최신 내용을 다시 불러와 수정해 주세요"),
```

- 둘 다 **409 Conflict** — "요청은 멀쩡한데 지금 서버 상태와 충돌한다". 400(요청이 잘못됨)과 구분된다.
- 도메인-일련번호 채번 규약대로 `FOOD-` 다음 번호를 이어 붙였다. → [[7. API 계약의 모양 — 공통 봉투·에러 코드·헤더 버저닝]]
- FOOD-006 의 메시지가 **다음 행동을 알려준다**("다시 불러와 수정해 주세요"). 에러 메시지 작성 규약(kb-166)이 지켜진 예.

> [!note] PR 설명이 코드보다 낡았다
> PR 본문 "변경 사항"에는 **FOOD-005 채번만** 적혀 있는데, 실제 코드에는 **FOOD-006 도 함께** 들어갔다(버전 충돌). 최신 커밋(`7dc66de7` "무버전 수정 직렬화")까지 반영되지 않은 것으로 보인다. **코드가 정본**이다.

### 3-7. 테스트 (+421 — 전체의 절반)

Kotest `given` 4묶음이 늘었다. 각 묶음이 검증하는 것:

| 묶음 | 확인하는 것 |
|---|---|
| 상세 조회 | 필드 전체가 실리는지 · 없는 id → 400 · **소프트삭제 후 조회 거절** |
| 수정 | 반영된 상세가 돌아오는지 + **DB 실제 값 검증** · 이름 중복 409 · 빈 이름 400 · 없는 id 400 |
| 재수집 | 단건 생성 · **이미 PENDING 이면 스킵(멱등)** · 없는 id 400 · 필터 일괄 카운트 |
| 삭제 | 삭제 후 **목록에서 빠지는지** · 없는 id 400 |

성공 경로보다 **실패·경계 경로가 더 많다.** "응답만 맞으면 통과"가 아니라 **DB 를 다시 읽어 확인**하는 것도 눈여겨볼 점. → [[14. SDD와 TDD — 스펙과 테스트로 개발하기]]

---

## 4. 이 PR 의 핵심 — 동시 수정 방어 3겹

관리자가 둘인 상황을 생각해보자. A 와 B 가 같은 음식 상세를 열어두고, A 가 저장한 뒤 B 가 저장하면 **A 의 수정이 조용히 사라진다**(lost update). [[23. 회원 랭킹 — 원자적 카운트와 조회 시점 점수 계산]] 에서 본 그 문제인데, 여기선 **요청 사이에 사람의 시간이 낀다**는 점이 다르다.

이 PR 은 세 겹으로 막는다:

```
① 비관적 락 (findByIdForUpdate)      — 같은 순간에 들어온 두 PUT 을 줄 세운다
                                        (트랜잭션 안, 밀리초 단위)
② 명시적 version 비교                 — B 가 화면을 연 뒤 A 가 저장했는지 잡는다
   if (expectedVersion != food.version)  (요청과 요청 사이, 분 단위 — ①로는 못 막는다)
       throw FOOD_VERSION_CONFLICT
③ JPA @Version (Food.version)         — 위 둘을 빠져나가도 flush 시점에 최후 검사
                                        → OptimisticLockingFailureException → FOOD-006
```

- **①만으로는 왜 부족한가**: 락은 트랜잭션이 살아 있는 동안만이다. B 가 화면을 10분 열어둔 시간은 어떤 트랜잭션에도 속하지 않는다.
- **②의 `version` 은 어디서 오나**: 상세 응답(§3-3)에 실려 나간 값을 SPA 가 그대로 되돌려 보낸다. **화면이 본 시점의 스냅샷 번호**인 셈.
- **`version` 을 생략하면?** — **REST 경로에서는 이제 못 생략한다.** 처음엔 선택값이라 생략 시 무조건 덮어썼는데, 2026-08-31 커밋에서 `@NotNull` 로 **필수화**됐다(§9). 누락은 400(COMMON-002). 다만 **서비스 메서드의 기본값(`expectedVersion: Long? = null`)은 그대로**라 Thymeleaf 화면처럼 서비스를 직접 부르는 경로는 여전히 검사를 건너뛴다 — **탈출구가 사라진 게 아니라 HTTP 계약에서만 닫혔다.**
- **③이 살아 있는 이유**: `Food.version` 에 `@jakarta.persistence.Version` 이 붙어 있어 하이버네이트가 UPDATE 문에 `WHERE version = ?` 를 자동으로 붙이고, 안 맞으면 예외를 던진다. 컨트롤러의 `catch (OptimisticLockingFailureException)` 은 **죽은 코드가 아니라** 이 3번째 겹을 받는 자리다.

> [!tip] 하나만 가져간다면
> **락은 "동시"를 막고, 버전은 "그 사이"를 막는다.** 사람이 화면을 열어두는 시간이 끼는 작업에는 락만으로 부족하다.
> 그리고 셋이 **함께** 있어야 하는 이유가 하나 더 있다 — 같은 `version` 을 든 두 요청이 **동시에** 들어오면, ①이 둘을 줄 세워 준 덕분에 ②의 비교가 **결정적으로** 한쪽만 409 를 받는다. ①이 없으면 둘 다 통과한 뒤 ③에서 500 성 예외로 터질 수 있다.

---

## 5. 기존 노트와 이어지는 곳

| 이 PR 에서 본 것 | 원리 노트 |
|---|---|
| `*Api` 인터페이스 → Controller → Service → JpaRepository 네 층을 한 칸씩 늘림 | [[5. kbap의 계층 — 컨트롤러에서 리포지토리까지]] |
| `BaseResponse` 봉투 · `FOOD-005/006` 채번 · 409 vs 400 | [[7. API 계약의 모양 — 공통 봉투·에러 코드·헤더 버저닝]] |
| `@SQLRestriction` 으로 소프트삭제가 조회에서 자동 제외 · `@Version` | [[8. 영속성 컨텍스트와 엔티티 매핑 — 엔티티가 곧 도메인 모델]] |
| 아웃박스에 행을 넣어 배치에 일 넘기기 · 멱등 | [[27. 아웃박스와 SQS — 다른 시스템에 일을 넘기는 법]] |
| lost update · check-then-act · 락 vs 버전 | [[23. 회원 랭킹 — 원자적 카운트와 조회 시점 점수 계산]] · [[20. 스캔 v2와 사용량 티켓 — 서버 OCR·무료 한도·중복 요청 막기]] |
| READY 전이 시 벡터 아웃박스 enqueue | [[28. 임베딩과 벡터 검색 — 뜻으로 음식 찾기]] |
| `imageRef`(키) 저장 · `imageUrl`(URL) 조립 | [[21. 이미지 업로드 흐름 — presigned URL과 업로드 완료 확정]] |
| 어드민 경로라 JWT 인터셉터·AUTH-008 자동 적용 | [[17. JWT 인증 필터와 세션 생명주기 — 매 요청 검증·재발급·로그아웃]] |

---

## 6. 리뷰한다면 물어볼 것 — 그리고 반영 결과

> 아래 5개 중 **2개가 반영됐다**(2026-08-31, §9). ✅ = 반영 · ⬜ = 미반영(판단 대기)

⬜ 1. **`objectMapper` 를 컨트롤러가 직접 생성**한다(§3-2) — 스프링 빈을 주입받지 않는 이유가 있나?
⬜ 2. **`AdminFoodService` 에 `getFoodDetail` 과 `getFoodDetailOrNull` 두 개**가 생겼다 — 실패 표현만 다른 중복인데, 하나로 두고 컨트롤러가 번역하는 편이 낫지 않나?
⬜ 3. **PUT 전체 교체 계약** — 번역 맵을 생략하면 지워진다. SPA 가 항상 전체를 보내는 게 확실한가? 부분 수정(PATCH)이 필요해질 여지는?
✅ 4. **`version` 생략 시 무조건 덮어쓰기** — 어드민 SPA 는 항상 보내게 강제하는 게 안전하지 않나? → **필수화됨**(`ee3df844`)
✅ 5. **PR 설명에 FOOD-006 이 빠져 있다**(§3-6) → **본문 전면 갱신됨** — 에러 응답 요약표(FE 분기 기준)와 리뷰 반영 이력까지 추가됐다.

---

## 7. 용어집

- **REST 화**: 서버가 HTML 을 그려 주던 화면을 JSON API + 클라이언트 렌더링(SPA)으로 옮기는 작업.
- **Thymeleaf**: 서버에서 HTML 을 만들어 내려주는 자바 템플릿 엔진. 이 레포 어드민의 기존 방식.
- **`@Valid`**: 요청 DTO 의 검증 애너테이션을 실행시키는 스위치. 실패하면 컨트롤러 본문에 진입하기 전에 400.
- **`@AssertTrue`**: 필드 하나로 판단 못 하는 규칙을 메서드로 검사하게 하는 검증 애너테이션.
- **비관적 락(`PESSIMISTIC_WRITE`)**: "충돌이 날 것"이라 보고 **읽는 순간 행을 잠근다**(`SELECT … FOR UPDATE`). 확실하지만 대기가 생긴다.
- **낙관적 잠금(`@Version`)**: "충돌은 드물 것"이라 보고 잠그지 않되, **저장할 때 버전이 그대로인지 확인**해 틀리면 실패시킨다.
- **lost update**: 두 수정이 겹쳐 먼저 저장한 쪽 결과가 조용히 사라지는 것.
- **멱등(idempotent)**: 같은 요청을 여러 번 보내도 결과가 한 번 보낸 것과 같은 성질.
- **소프트삭제**: 행을 지우지 않고 `status='DELETED'` 로 표시만 하는 삭제. 조회에서 자동 제외된다.
- **아웃박스**: "이 일을 나중에 처리해달라"를 DB 행으로 적어두는 자리. 배치가 집어간다.
- **낙관 업데이트(FE)**: 서버 응답을 기다리지 않고 화면을 먼저 바꾸는 기법. 서버가 반영본을 돌려주면 그걸로 확정한다.

---

## 8. 다음에 이 PR 을 다시 볼 때

1. 이 PR 은 **로직 추가가 아니라 창구 추가**다 — `AdminFoodService` 의 기존 메서드는 거의 그대로다.
2. 진짜 새것은 **`findByIdForUpdate`(레포 첫 비관적 락)** 와 **에러 코드 2개**, 그리고 **동시 수정 3겹 방어**(§4).
3. 코드 절반이 테스트고, 그 절반이 실패 경로다.
4. 머지 여부·후속 변경은 [PR 201](https://github.com/team-skyjs/kbap-server/pull/201) 에서 확인 — 이 노트는 **2026-08-31 시점 브랜치 `feat/kb397-admin-food-detail-rest`(`ee3df844`)** 기준이다. 변경 이력은 §9.

---

## 9. 변경 이력 — 이 노트를 쓴 뒤 바뀐 것

### `ee3df844` — 음식 수정 `version` **필수화** (2026-08-31)

> `feat(admin)!:` — 느낌표는 **호환성을 깨는 변경**(breaking change) 표시다. 커밋 메시지 규약(Conventional Commits)에서 온 관례.

**무엇이 바뀌었나** — 딱 3파일, 실질은 한 줄이다.

| 파일 | 변경 |
|---|---|
| `AdminFoodDetailResponse.kt` | `version` 에 **`@field:NotNull` 추가** (+1줄) |
| `AdminFoodCatalogApi.kt` | Swagger 설명 "`version`(선택)…생략하면 무조건 덮어쓴다" → **"`version` 은 필수다…누락은 400(COMMON-002)"** |
| `AdminFoodCatalogControllerTest.kt` | 테스트 헬퍼에 `version` 기본값 추가 + **"version 을 누락하고 수정하면 400" 시나리오 신설** (+15줄) |

```kotlin
// 테스트가 검증하는 것 — 요청 바디에서 version 키를 빼고 보낸다
putUpdate(food.id, updateBody(koreanName = "무버전찌개") - "version").andExpect {
    status { isBadRequest() }
    jsonPath("$.code") { value("COMMON-002") }
}
```

**왜 바꿨나** (커밋 메시지 원문): *"소비자가 어드민 SPA 뿐이고 FE 가 이미 항상 전송하므로 서버가 필수로 강제해 **조용한 덮어쓰기(last-write-wins)를 구조적으로 차단**한다."*

**이 결정이 좋은 이유** — 원래 `version` 은 **선택**이라, 안 보내면 검사를 건너뛰고 덮어썼다. 문제는 그 경로가 **실수로도 열린다**는 것이다: FE 가 필드 하나를 빠뜨리거나, 누가 curl 로 급히 고칠 때 조용히 남의 수정을 지운다. 그리고 **아무 에러도 안 난다** — 사고가 났는지조차 모른다.
필수화하면 그 경로가 **타입 검증 단계에서** 닫힌다. "규율로 지키는 것"을 **"컴파일·검증이 지키는 것"으로 옮긴 것**이고, 이건 이 레포가 여러 곳에서 쓰는 방식이다 → [[13. 경계를 지키는 테스트 — ArchUnit이 강제하는 규칙들]] 의 "사람이 지켜야 하는 것을 기계가 지키게 만들기"와 같은 계열.

> [!note] 단, 탈출구가 완전히 사라진 건 아니다
> 서비스 메서드의 시그니처는 그대로다 — `updateFood(id, command, expectedVersion: Long? = null)`. **기본값이 살아 있어** Thymeleaf 화면처럼 서비스를 직접 호출하는 경로는 여전히 버전 검사를 건너뛴다. 즉 **닫힌 건 HTTP 계약 한 겹**이고, 서비스 계층은 두 호출자를 모두 받아준다. 나중에 Thymeleaf 가 걷히면 그 기본값도 없앨 수 있다.

### 그 앞의 리뷰 반영 이력 (PR 본문 기준)

이 PR 은 처음부터 이 모양이 아니었다. 본문에 리뷰 반영 이력이 정리돼 있는데, **§4 의 3겹 방어가 한 번에 설계된 게 아니라 리뷰를 거치며 쌓였다**는 게 읽힌다:

| 커밋 | 무엇이 붙었나 |
|---|---|
| `f46ec359` | Codex 리뷰 P1(성분 코드 검증)·P2(단건 재수집 원자성 → **비관적 락**) + FE 요청으로 낙관 잠금(`version`/FOOD-006) 도입 |
| `de7c0ad3` | version 검사 **경합 제거**(잠금으로 직렬화) + `OptimisticLockingFailure→FOOD-006` **벨트** 추가 |
| `7dc66de7` | `ingredients` null/[] 구분 보존 · spiciness 범위 검증 · 무버전 경로 잠금 통일 |
| `ee3df844` | **version 필수화** — REST 경로에서 last-write-wins 소멸 |

즉 **①비관적 락은 "재수집 원자성"을 고치다 들어왔고, ②version 비교는 FE 요청으로, ③JPA 낙관 잠금은 ②의 경합을 막는 벨트로** 각각 다른 이유로 붙었다. 결과적으로 3겹이 됐지만 처음부터 3겹을 노린 설계는 아니었다는 것 — 실제 코드는 대개 이렇게 자란다.

> 본문에 **감수하기로 한 것**도 명시돼 있다: *"일괄 재수집 check-then-insert 경합은 감수(기존 코드·비치명)"*. 모든 경합을 다 막지 않고 **비용 대비 위험으로 잘라낸 판단**을 기록에 남긴 좋은 예다.

### 아직 안 바뀐 것 (§6 의 ⬜ 3개)

`objectMapper` 직접 생성 · `getFoodDetail`/`getFoodDetailOrNull` 중복 · PUT 전체 교체 계약 — 셋 다 **동작 문제가 아니라 설계 취향**이라 남았다. 특히 마지막 둘은 **Thymeleaf 가 걷히면 자연히 정리될 것**들이라, 지금 손대면 두 번 고치게 된다.
