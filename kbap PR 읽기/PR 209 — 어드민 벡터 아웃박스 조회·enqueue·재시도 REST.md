---
tags: [kbap-backend, pr읽기, admin, rest, 아웃박스, 벡터, 페이징]
생성일: 2026-08-31
상태: 완료
PR: 209
브랜치: feat/kb402-admin-outbox-rest
머지: 2026-08-31 (squash `58b91567` → develop)
---

> [!info] 관련 노트
> 🏠 [[🏠 홈]] · 📂 주제: [[🗺️ kbap 백엔드 공부 지도 (2026-08)]]
> ⬅️ 같은 에픽의 앞 PR: [[PR 201 — 어드민 음식 상세·수정·재수집·삭제 REST]]
> 🔗 이 PR 을 읽는 데 쓰인 개념: [[27. 아웃박스와 SQS — 다른 시스템에 일을 넘기는 법]] · [[28. 임베딩과 벡터 검색 — 뜻으로 음식 찾기]] · [[9. 연관관계를 쓰지 않는 설계 — N+1을 애초에 만들지 않기]] · [[페이징 — Offset vs No-Offset(Keyset·Cursor)]]

# PR 209 — 어드민 벡터 아웃박스 조회·enqueue·재시도 REST

> 작성: 2026-08-31 · 맥락: 어드민 리빌드 에픽(KB-384)의 두 번째 조각. [[PR 201 — 어드민 음식 상세·수정·재수집·삭제 REST|PR 201]] 이 **음식 자체**를 REST 로 열었다면, 이 PR 은 그 음식이 **벡터로 동기화되는 파이프라인의 운영 창구**를 연다.
> 목표: 아웃박스라는 큐가 **운영자 눈에 어떻게 보여야 하는지**를 코드로 읽고, 이 PR 이 고른 페이징 방식이 사용자 API 와 왜 다른지 설명할 수 있게 되기.

| | |
|---|---|
| 링크 | [team-skyjs/kbap-server#209](https://github.com/team-skyjs/kbap-server/pull/209) · Jira KB-402 (에픽 KB-384) |
| 상태 | **머지됨** (2026-08-31, squash → `develop`) · 작성자 KYJ |
| 규모 | **7파일 · +448 / −5** (테스트 218줄, 시나리오 10개) |

---

## 0. 한 줄 요약

**벡터 동기화 대기열(아웃박스)을 운영자가 들여다보고 손댈 수 있게 JSON API 3개를 열었다.** 목록 조회 · 밀린 것 일괄 넣기 · 실패한 것 재시도. 로직은 대부분 있던 것이고, **새로 한 일은 "조용히 아무것도 안 하던 자리"에 전부 결과값을 붙인 것**이다.

---

## 1. 이 PR 이 하는 일 (제품 관점)

먼저 배경 — **벡터 아웃박스가 뭔가**. 음식 정보가 바뀌면 그 뜻을 담은 벡터도 다시 만들어야 한다. 그런데 임베딩 API 호출은 느리고 돈이 든다. 그래서 **"이 음식 벡터 다시 만들어줘"를 DB 행으로 적어두고**, 배치가 나중에 집어간다. 그 적어두는 자리가 아웃박스다. → [[27. 아웃박스와 SQS — 다른 시스템에 일을 넘기는 법]] · [[28. 임베딩과 벡터 검색 — 뜻으로 음식 찾기]]

문제는 **그 줄이 밀리거나 실패했을 때 운영자가 볼 방법**이었다. 기존엔 Thymeleaf 페이지에 **실패 20건이 고정으로** 떠 있는 게 전부였다.

| 동작 | 엔드포인트 | 이 PR 전에는 |
|---|---|---|
| 대기열 보기 | `GET /api/admin/foods/vector-outboxes?page&status` | 실패 **20건 고정 뷰**만 — 페이지도 필터도 총 건수도 없음 |
| 밀린 것 일괄 넣기 | `POST …/vector-outboxes/enqueue` | 페이지에 버튼은 있었으나 **몇 건 넣었는지 안 알려줌** |
| 실패한 것 재시도 | `POST …/vector-outboxes/{id}/retry` | 버튼은 있었으나 **없는 id 를 눌러도 조용히 무시** |

**Thymeleaf 페이지는 무변경**(회귀 0). 서비스 메서드의 반환값만 늘렸는데, 페이지 컨트롤러는 그 반환을 안 쓰므로 영향이 없다.

> [!tip] 이 PR 의 성격을 한 문장으로
> **"동작은 그대로, 결과를 말하게 만들기."** 넣었으면 몇 건인지, 재시도했으면 됐는지 안 됐는지, 대상이 없으면 없다고 — 전부 **응답으로 대답**하게 바꿨다.

---

## 2. 바뀐 파일 한눈에

| 파일 | 증감 | 한 줄 역할 |
|---|---|---|
| `api/admin/AdminVectorOutboxApi.kt` | +79 (신규) | **문서 인터페이스** — Swagger 설명·응답코드 |
| `api/admin/AdminVectorOutboxController.kt` | +54 (신규) | **HTTP 입구** — 3개 매핑, 결과 enum → 응답/에러 번역 |
| `api/admin/AdminVectorOutboxResponse.kt` | +51 (신규) | **DTO 4종** — 페이지/항목/enqueue/retry 응답 |
| `api/admin/AdminFoodDashboardService.kt` | +41 −5 | **로직** — 목록 조회 신설, 기존 메서드 2개의 **반환값 확장** |
| `common/domain/food/FoodVectorOutboxJpaRepository.kt` | +4 | **DB 접근** — 상태별 페이지 조회 추가 |
| `common/core/error/ErrorCode.kt` | +1 | **FOOD-007** 채번 |
| `api/test/.../AdminVectorOutboxControllerTest.kt` | +218 | 테스트 10 시나리오 |

> [[PR 201 — 어드민 음식 상세·수정·재수집·삭제 REST|PR 201]] 과 **파일 구성이 판박이**다: `*Api`(문서) → `*Controller` → `*Response`(DTO) → `*Service` → `*JpaRepository` → `ErrorCode` → 테스트. 이 레포에서 **REST 엔드포인트 한 벌을 추가한다는 것이 정확히 이 7자리를 건드리는 일**임을 두 PR 이 함께 보여준다. → [[5. kbap의 계층 — 컨트롤러에서 리포지토리까지]]

---

## 3. 파일별로 무슨 코드가 왜 들어갔나

### 3-1. `AdminFoodDashboardService.kt` — 로직 (+41 −5)

**① 목록 조회 신설** — 이 PR 에서 가장 볼 게 많은 메서드다.

```kotlin
@Transactional(readOnly = true)
fun getVectorOutboxPage(page: Int, status: FoodVectorOutboxStatus?): AdminVectorOutboxPageResponse {
    val pageable = PageRequest.of(page - 1, VECTOR_OUTBOX_PAGE_SIZE, Sort.by(Sort.Direction.DESC, "id"))
    val result = when (status) {
        null -> vectorOutboxRepository.findAll(pageable)
        else -> vectorOutboxRepository.findByOutboxStatus(status, pageable)
    }
    val displayNames = foodRepository.findAllById(result.content.map { it.foodId }.distinct())
        .associate { it.id to it.displayName }               // ← id → 이름 사전을 한 번에
    return AdminVectorOutboxPageResponse(
        items = result.content.map { AdminVectorOutboxItemResponse.from(it, displayNames[it.foodId]) },
        page = page, totalPages = result.totalPages, totalCount = result.totalElements,
        hasPrev = page > 1, hasNext = page < result.totalPages,
    )
}
```

- **`page - 1`** — 사람은 1페이지부터 세고 스프링은 0부터 센다. 그 변환이 여기 한 줄로 갇혀 있다. 컨트롤러의 `page.coerceAtLeast(1)` 과 짝이다(0이나 음수를 넣어도 1로 눌린다).
- **N+1 을 이렇게 피한다** — 아웃박스 50건에 대해 음식 이름을 하나씩 조회하면 **51번** 쿼리다. 대신 `foodId` 를 모아 `distinct()` 하고 `findAllById` **한 번**으로 긁어 `Map` 을 만든 뒤 메모리에서 붙인다. **총 2번**. 이 레포엔 JPA 연관관계가 없으니 조합은 언제나 손으로 하는데, 그 표준 손동작이 바로 이것이다. → [[9. 연관관계를 쓰지 않는 설계 — N+1을 애초에 만들지 않기]] · [[24. 주문 흐름 — 스캔 결과가 주문 이력이 되기까지]]
- **`displayName: String?` 이 널인 경우** — 음식이 소프트삭제됐을 때다. `findAllById` 는 `@SQLRestriction("status = 'ACTIVE'")` 때문에 삭제된 행을 **아예 못 가져오고**, 그래서 `displayNames[foodId]` 가 널이 된다. **삭제된 음식의 아웃박스 행은 남아 있는데 이름만 비는** 상태가 화면에 그대로 보인다 — 운영자에겐 오히려 유용한 정보다(왜 이 줄이 안 나가는지 힌트). → [[8. 영속성 컨텍스트와 엔티티 매핑 — 엔티티가 곧 도메인 모델]]

**② `enqueueReadyFoodsForVectorSync` 반환 확장** — `Unit` → `Int`.

```kotlin
-    fun enqueueReadyFoodsForVectorSync() {
+    fun enqueueReadyFoodsForVectorSync(): Int {
         …
+        return targetIds.size
```

한 줄이지만 계약이 바뀌었다. 기존 Thymeleaf 컨트롤러는 반환을 **안 쓰므로** 컴파일도 동작도 그대로다 — 코틀린/자바에서 반환값 추가는 호출부에 영향이 없다.

**③ `retryVectorOutbox` 반환 확장** — `Unit` → `enum` 4갈래. **이 PR 의 핵심 설계**다.

```kotlin
-    fun retryVectorOutbox(outboxId: Long) {
-        val outbox = vectorOutboxRepository.findById(outboxId).orElse(null) ?: return   // ← 조용한 무시
-        if (outbox.outboxStatus == FoodVectorOutboxStatus.FAILED) { outbox.retry() }
+    fun retryVectorOutbox(outboxId: Long): AdminVectorOutboxRetryResult {
+        val outbox = vectorOutboxRepository.findById(outboxId).orElse(null)
+            ?: return AdminVectorOutboxRetryResult.NOT_FOUND
+        return when (outbox.outboxStatus) {
+            FoodVectorOutboxStatus.FAILED -> { outbox.retry(); AdminVectorOutboxRetryResult.RETRIED }
+            FoodVectorOutboxStatus.PENDING -> AdminVectorOutboxRetryResult.ALREADY_PENDING
+            FoodVectorOutboxStatus.COMPLETE -> AdminVectorOutboxRetryResult.ALREADY_COMPLETE
+        }
+    }
```

- **`?: return` 이 `?: return NOT_FOUND` 가 된 것** — 이 PR 전체를 요약하는 한 줄이다. **"없으면 조용히 아무것도 안 함"에서 "없다고 말함"으로.** 버튼을 눌렀는데 아무 일도 안 일어나면 운영자는 성공했는지 실패했는지 모른다.
- **`when` 이 enum 을 **빠짐없이** 받는다** — 코틀린은 enum 을 다루는 `when` 이 표현식으로 쓰이면 **모든 분기를 요구**한다(exhaustive). 나중에 상태가 하나 늘면 **컴파일이 깨져서** 여기를 고치게 만든다. → [[3. JVM 위의 Kotlin — 컴파일·널 안전·Gradle 3모듈]]
- **상태 3개가 각각 다른 답을 갖는다** — FAILED 만 실제로 되돌리고, PENDING·COMPLETE 는 **아무것도 안 하되 "안 했다"고 답한다.** 이게 멱등의 모양이다: 두 번 눌러도 안전하고, 두 번째엔 `retried=false` 가 온다. [[PR 201 — 어드민 음식 상세·수정·재수집·삭제 REST|PR 201]] 의 단건 재수집(`created=0, skipped=1`)과 **같은 기준**으로 맞춘 것.

### 3-2. `AdminVectorOutboxController.kt` — 번역기 (+54, 신규)

컨트롤러가 하는 일은 [[PR 201 — 어드민 음식 상세·수정·재수집·삭제 REST|PR 201]] 과 똑같다 — **enum 을 HTTP 로 옮기는 것**.

```kotlin
val payload = when (adminFoodDashboardService.retryVectorOutbox(id)) {
    RETRIED          -> AdminVectorOutboxRetryResponse(retried = true,  outboxStatus = PENDING)
    ALREADY_PENDING  -> AdminVectorOutboxRetryResponse(retried = false, outboxStatus = PENDING)
    ALREADY_COMPLETE -> AdminVectorOutboxRetryResponse(retried = false, outboxStatus = COMPLETE)
    NOT_FOUND        -> throw BusinessException(ErrorCode.VECTOR_OUTBOX_NOT_FOUND)
}
```

- **4갈래가 "200 세 개 + 400 한 개"로 접힌다.** 서비스는 HTTP 를 모르고, 무엇이 에러인지는 컨트롤러가 정한다.
- **응답이 `retried`(했나?) + `outboxStatus`(지금 뭐냐?) 두 필드**인 게 좋다. FE 는 `retried` 로 토스트를 띄우고 `outboxStatus` 로 행을 다시 그린다 — **다시 조회할 필요가 없다.**
- **`status` 를 enum 파라미터로 직접 받는다**(`FoodVectorOutboxStatus?`) — 스프링이 문자열을 enum 으로 변환해주고, 이상한 값이면 **컨트롤러 진입 전에** 400(COMMON-002)이 된다. 검증 코드를 안 써도 되는 자리.

### 3-3. `AdminVectorOutboxResponse.kt` — DTO 4종 (+51, 신규)

```kotlin
data class AdminVectorOutboxPageResponse(
    val items: List<AdminVectorOutboxItemResponse>,
    val page: Int, val totalPages: Int, val totalCount: Long,
    val hasPrev: Boolean, val hasNext: Boolean,
)
data class AdminVectorOutboxItemResponse(
    val id: Long, val foodId: Long, val displayName: String?,
    val operation: FoodVectorOutboxOperation,   // UPSERT / DELETE
    val outboxStatus: FoodVectorOutboxStatus,   // PENDING / COMPLETE / FAILED
    val attempts: Int, val lastError: String?,
    val createdAt: LocalDateTime, val updatedAt: LocalDateTime,
)
data class AdminVectorOutboxEnqueueResponse(val enqueued: Int)
data class AdminVectorOutboxRetryResponse(val retried: Boolean, val outboxStatus: FoodVectorOutboxStatus)
```

- **`attempts` 와 `lastError` 를 노출한다** — 운영 화면의 알맹이다. [[27. 아웃박스와 SQS — 다른 시스템에 일을 넘기는 법]] 에서 본 대로 벡터 아웃박스는 **5회 상한**(`MAX_ATTEMPTS = 5`)이 있고 실패 사유를 500자까지 보관한다. 그 둘이 화면에 나와야 "왜 안 나갔나"를 판단할 수 있다.
- **`hasPrev`/`hasNext` 를 서버가 계산해준다** — FE 가 `page < totalPages` 를 직접 계산하지 않게. 경계 계산은 한 곳에만 두는 게 안전하다.

### 3-4. `FoodVectorOutboxJpaRepository.kt` — 상태별 페이지 조회 (+4)

```kotlin
fun findByOutboxStatus(outboxStatus: FoodVectorOutboxStatus, pageable: Pageable): Page<FoodVectorOutbox>
```

메서드 **이름만으로** 쿼리가 만들어지는 Spring Data 규약(`findBy` + 필드명). 본문이 없다. → [[8. 영속성 컨텍스트와 엔티티 매핑 — 엔티티가 곧 도메인 모델]]

### 3-5. `ErrorCode.kt` — FOOD-007 채번 (+1)

```kotlin
VECTOR_OUTBOX_NOT_FOUND("FOOD-007", 400, …)
```

PR 201 이 FOOD-005·006 을 썼으니 그다음 번호. **400 이지 404 가 아니다** — 이 레포는 "없음"을 400 + `code` 로 표현하는 관행이다. → [[7. API 계약의 모양 — 공통 봉투·에러 코드·헤더 버저닝]]

### 3-6. 테스트 (+218, 10 시나리오)

| 묶음 | 확인하는 것 |
|---|---|
| 목록 | 필드 전체 · **`displayName` 채워짐** · `totalCount` · **id 내림차순 정렬** · 상태 필터 · 401 · 403 |
| enqueue | 건수 반환 · **재호출 시 0건**(멱등) |
| 재시도 | FAILED→성공 · **PENDING/COMPLETE 멱등**(`retried=false`) · 부재 400 |

권한 테스트(401·403)가 들어간 게 눈에 띈다 — 어드민 API 는 **누가 못 들어오는지**가 기능의 일부다.

---

## 4. 눈여겨볼 것 — 어드민은 왜 offset 페이징인가

이 PR 은 `PageRequest.of(page - 1, 50, Sort by id DESC)` 로 **진짜 offset 페이징**을 쓰고 `totalPages`·`totalCount` 를 응답에 담는다. 그런데 볼트의 [[페이징 — Offset vs No-Offset(Keyset·Cursor)]] 노트와 [[9. 연관관계를 쓰지 않는 설계 — N+1을 애초에 만들지 않기]] 는 이 레포가 **커서(keyset) 페이징**을 쓴다고 실측해뒀다. 모순일까?

**아니다. 사용자 API 와 어드민 API 가 다른 방식을 쓴다.**

| | 사용자 API (리뷰·주문·커뮤니티·북마크) | 어드민 API (이 PR) |
|---|---|---|
| 방식 | **커서(keyset)** | **offset** |
| 코드 모양 | `PageRequest.of(0, size + 1)` — 페이지는 **항상 0**, `+1` 로 다음 여부만 봄 | `PageRequest.of(page - 1, 50)` — 페이지 번호가 실제로 움직임 |
| 총 건수 | **안 준다**(세려면 전체 스캔) | `totalCount` **준다** |
| 왜 | 무한 스크롤 · 데이터가 크고 계속 늘어남 · 중간 삽입에도 안 밀림 | **페이지 번호·총 건수가 화면 요구사항** · 운영 데이터라 규모가 작음 · 관리자 수가 적음 |

> [!note] 같은 `PageRequest` 인데 쓰임이 다르다
> 사용자 API 의 `PageRequest.of(0, size + 1)` 은 **페이징 도구가 아니라 그냥 "limit"** 으로 쓰인 것이다. 진짜 커서는 `WHERE id < :cursor` 조건이 하고, `PageRequest` 는 개수만 자른다. 클래스 이름만 보고 "여기도 offset 이네"라고 읽으면 틀린다.

**offset 의 대가**를 이 PR 도 그대로 진다 — 뒤 페이지로 갈수록 DB 가 앞쪽 행을 세고 건너뛰어야 하고, 보는 사이 새 행이 들어오면 항목이 밀린다. 다만 **아웃박스는 관리자만 보고, 대개 앞 몇 페이지에서 끝나고, 정렬이 `id DESC` 라 새 행은 맨 앞에 붙는다** — 감수할 만한 트레이드오프다.

---

## 5. 기존 노트와 이어지는 곳

| 이 PR 에서 본 것 | 원리 노트 |
|---|---|
| 아웃박스 행 · `attempts`(5회 상한) · `lastError` · PENDING/COMPLETE/FAILED | [[27. 아웃박스와 SQS — 다른 시스템에 일을 넘기는 법]] |
| UPSERT/DELETE 오퍼레이션 · 벡터 동기화가 왜 큐를 타나 | [[28. 임베딩과 벡터 검색 — 뜻으로 음식 찾기]] |
| `findAllById` + `associate` 로 N+1 회피 (쿼리 2번) | [[9. 연관관계를 쓰지 않는 설계 — N+1을 애초에 만들지 않기]] · [[24. 주문 흐름 — 스캔 결과가 주문 이력이 되기까지]] |
| offset vs 커서 · `totalCount` 의 비용 | [[페이징 — Offset vs No-Offset(Keyset·Cursor)]] |
| 소프트삭제된 음식이 조회에서 자동 제외 → `displayName = null` | [[8. 영속성 컨텍스트와 엔티티 매핑 — 엔티티가 곧 도메인 모델]] |
| `BaseResponse` 봉투 · FOOD-007 채번 · 없음을 400 으로 | [[7. API 계약의 모양 — 공통 봉투·에러 코드·헤더 버저닝]] |
| 서비스는 enum, HTTP 는 컨트롤러가 정함 | [[5. kbap의 계층 — 컨트롤러에서 리포지토리까지]] |
| 어드민 경로 JWT·AUTH-008 | [[17. JWT 인증 필터와 세션 생명주기 — 매 요청 검증·재발급·로그아웃]] |
| 같은 에픽의 앞 PR — 파일 구성·멱등 기준이 동일 | [[PR 201 — 어드민 음식 상세·수정·재수집·삭제 REST]] |

---

## 6. 리뷰 질문과 답 — 5건 중 4건이 "수정 불요"

> 아래 5개를 물었고 답을 받았다. **1~4 는 설계 근거가 서 있어 수정하지 않았고, 5 만 FE 에서 처리**한다.
> 답을 옮겨 적는 이유: **"왜 안 고쳤나"가 나중에 같은 질문을 막아준다.** 코드에는 안 남는 판단이다.

### ⬜ 1. `enqueue` 의 동시 더블클릭 경합 — "단건은 막고 일괄은 감수"가 기준선인가?

**→ 수정 불요. 내 질문이 기준선을 잘못 잡았다.** 기준선은 **"단건이냐 일괄이냐"가 아니라 "잠금 비용 대비 피해 크기"** 다.

| | 단건 재수집([[PR 201 — 어드민 음식 상세·수정·재수집·삭제 REST|PR 201]]) | 일괄 enqueue(이 PR) |
|---|---|---|
| 잠금 비용 | **행 잠금 하나** — 싸다 | **집합 연산** — 범위가 커지고 데드락 위험 |
| 막지 않으면 피해 | 중복 아웃박스 행 | **중복 UPSERT = 같은 벡터를 두 번 씀 → 멱등, 실피해 0 수렴** |

- 싸게 막을 수 있으면 막고, 비싸게 막아야 하는데 피해가 0으로 수렴하면 감수한다 — **일관된 하나의 기준**이다.
- **DB 제약으로 막는 대안도 없었다**: MySQL 은 **부분 유니크 인덱스**(예: `WHERE status <> 'DONE'` 만 유니크)를 지원하지 않는데, DONE 행이 누적되므로 전체 유니크는 걸 수 없다. (PostgreSQL 이었다면 가능한 선택지다.)

### ⬜ 2. `totalCount` 의 `COUNT(*)` 비용

**→ 수정 불요.** 사용자가 **2인**이고 가끔 여는 운영 화면이라 수십만 행이 돼도 실질 문제까지 멀다. 그리고 "대략 건수"로 낮추는 건 **응답 계약을 안 바꾸고 서버 구현만 교체**하면 되는 일이라, **지금 여지를 열어둘 작업 자체가 없다.**

> 더 근본적인 건 **DONE 행 보존기간 정리(청소 배치)** 쪽이다 — 세는 비용이 아니라 **안 지우는 것**이 진짜 원인이다. 백로그 메모 가치. (2026-08-31 기준 이슈 미등록)

### ⬜ 3. 페이지 크기 50 고정

**→ 수정 불요.** 이유가 이미 서 있다 — 항목에 **`lastError`(최대 500자)** 가 들어가서 페이지가 커지면 응답이 급격히 부푼다. `size` 파라미터를 열면 **최대치 검증 등 표면적이 늘어나는데 현재 요구가 없다**(YAGNI). 요구가 생기면 **optional 파라미터 추가는 하위호환**이라 그때 붙이면 된다.

### ⬜ 4. `retry` 가 `attempts` 를 0 으로 되돌린다 — 무한 재시도 아닌가?

**→ 수정 불요. 두 재시도는 주체가 다르다.**

| | `attempts` 상한(5회) | retry 버튼 |
|---|---|---|
| 누가 | **백그라운드 워커**의 자동 재시도 | **운영자**의 명시적 판단 |
| 상한의 목적 | 사람이 안 보는 사이 무한히 도는 것을 멈추기 | — |
| 상한이 필요한가 | 필요 | **사람이 누르는 건 사람이 상한이다** |

- 반복 실패하면 `attempts` 가 **다시 쌓여 화면에 보이므로** 관찰도 된다(§3-3 의 `attempts`·`lastError` 노출).
- 기록은 서버 로그로 충분.

### ✅ 5. 소프트삭제된 음식의 아웃박스 행은 재시도해도 무의미

**→ 실질 지적으로 인정. 단 서버는 안 바꾼다 — FE 에서 처리.**

**서버 변경이 필요 없는 이유**: 그 행은 **`displayName` 이 널**이고(§3-1), FE 는 그 값을 이미 받고 있다. 즉 **판단 재료가 이미 응답에 있다.** 서버에 `retriable` 같은 필드를 새로 넣는 건 중복이다.

**FE 처리 방식**: `displayName == null` 인 행은 **재시도 버튼 비활성 + "삭제된 음식 — 재시도 무의미" 툴팁**. (2026-08-31 어드민 세션에 소액 발주 `P-A11` — 서버 계약 무변)

> [!tip] 이 다섯 문답에서 배울 것
> **"안 고침"에도 이유가 있고, 그 이유는 코드에 안 남는다.** 1번의 "잠금 비용 대비 피해 크기", 3번의 "요구가 없으니 YAGNI, 생기면 하위호환으로 추가", 4번의 "사람이 누르는 건 사람이 상한" — 전부 **다음에 같은 질문이 올라올 때 재론을 막아주는 기록**이다. PR 본문의 *"…같은 기준으로 감수한다"* 한 줄도 같은 역할을 한다.

---

## 7. 용어집

- **아웃박스(outbox)**: "이 일을 나중에 처리해달라"를 DB 행으로 적어두는 자리. 적는 쪽과 처리하는 쪽이 분리된다.
- **enqueue**: 큐(여기선 아웃박스 테이블)에 할 일을 넣는 것.
- **UPSERT / DELETE (operation)**: 벡터를 새로 쓰거나 갱신 / 벡터를 지움. 음식이 READY 가 되면 UPSERT, READY 에서 벗어나면 DELETE 가 쌓인다.
- **PENDING / COMPLETE / FAILED**: 대기 중 / 처리 완료 / 5회 시도 후 실패.
- **멱등(idempotent)**: 여러 번 실행해도 결과가 한 번 실행한 것과 같은 성질. 여기선 `retried=false` 로 "이미 그 상태"를 알려준다.
- **offset 페이징**: `LIMIT 50 OFFSET 100` 식. 페이지 번호로 건너뛴다. 총 건수를 알 수 있지만 뒤로 갈수록 느리고, 보는 사이 데이터가 들어오면 밀린다.
- **커서(keyset) 페이징**: 마지막으로 본 항목의 키를 들고 `WHERE id < :cursor` 로 이어 읽는다. 밀리지 않고 빠르지만 총 건수·페이지 번호가 없다.
- **exhaustive `when`**: 코틀린에서 enum 을 다루는 `when` 이 값을 돌려주면 모든 분기를 요구하는 성질. 상태가 늘면 컴파일이 깨져 알려준다.
- **Spring Data 메서드 이름 규약**: `findByOutboxStatus` 처럼 메서드 이름만으로 쿼리를 만들어주는 기능.

---

## 8. 다음에 이 PR 을 다시 볼 때

1. 한 문장 요약: **"조용히 아무것도 안 하던 자리에 전부 결과값을 붙였다."** `?: return` → `?: return NOT_FOUND` 가 그 상징.
2. 코드에서 배울 것 두 개: **N+1 회피 손동작**(`findAllById` + `associate`, §3-1)과 **멱등 응답 설계**(enum 4갈래 → `retried`+`outboxStatus`, §3-1③).
3. 헷갈리면 안 되는 것: 이 레포는 **사용자 API 는 커서, 어드민 API 는 offset** 이다(§4). `PageRequest` 가 보인다고 다 offset 이 아니다.
4. 이 노트는 **머지된 최종본**(`58b91567`) 기준이다. 머지·배포 기록은 §9.

---

## 9. 머지 기록

| | |
|---|---|
| 머지 | 2026-08-31 · squash → `develop` · `58b91567` |
| 리뷰 | 질문 5건 중 **4건 수정 불요**(설계 근거 확인, §6) · **1건은 FE 처리**(§6-5) — **서버 코드 무변경으로 머지** |

> 즉 이 노트가 설명하는 코드가 **최종본 그대로**다. §6 의 문답이 "왜 이 모양으로 남았나"의 기록이다.
