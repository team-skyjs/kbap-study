---
tags: [kbap-backend, pr읽기, admin, 소프트삭제, 아웃박스]
생성일: 2026-09-01
상태: 완료
PR: 216
브랜치: (open)
---

> [!info] 관련 노트
> 🏠 [[🏠 홈]] · 📂 주제: [[🗺️ kbap 백엔드 공부 지도 (2026-08)]]
> ⬅️ 이 PR 이 되돌리는 동작: [[PR 201 — 어드민 음식 상세·수정·재수집·삭제 REST]] (소프트삭제)
> 🔗 개념: [[8. 영속성 컨텍스트와 엔티티 매핑 — 엔티티가 곧 도메인 모델]] · [[27. 아웃박스와 SQS — 다른 시스템에 일을 넘기는 법]] · [[28. 임베딩과 벡터 검색 — 뜻으로 음식 찾기]]

# PR 216 — 삭제 음식 조회·복원 REST와 벡터 큐 대칭

> 작성: 2026-09-01 · 맥락: [[PR 201 — 어드민 음식 상세·수정·재수집·삭제 REST|PR 201]] 이 소프트삭제를 열었는데, **삭제된 음식을 다시 볼 방법이 없었다.** `@SQLRestriction` 때문에 모든 조회에서 사라지기 때문. 실수로 지웠을 때 손쓸 수가 없다.
> 목표: **"안 보이게 만든 것을 다시 보는 법"**(native 쿼리로 매핑 필터 우회)과, **삭제·복원이 비동기 큐와 어떻게 짝을 맞춰야 하는지** 이해하기.

| | |
|---|---|
| 링크 | [team-skyjs/kbap-server#216](https://github.com/team-skyjs/kbap-server/pull/216) · Jira KB-404 |
| 상태 | **OPEN** (2026-09-01 기준) · 작성자 rocher71 |
| 규모 | **8파일 · +329 / −0** (테스트 176줄, 시나리오 9개 추가 — 카탈로그 총 47개) |

---

## 0. 한 줄 요약

**삭제된 음식을 보는 창구 2개(목록·상세)와 복원 1개를 열었다.** 기술적 알맹이는 둘 — **native 쿼리로 소프트삭제 필터를 우회**하는 것, 그리고 **복원 시 벡터 큐를 삭제와 대칭으로 되돌리는 것**이다.

---

## 1. 문제 — 지운 걸 볼 수가 없다

[[8. 영속성 컨텍스트와 엔티티 매핑 — 엔티티가 곧 도메인 모델|8번 노트]]에서 본 대로, `BaseEntity` 에 이게 걸려 있다:

```kotlin
@MappedSuperclass
@SQLRestriction("status = 'ACTIVE'")
abstract class BaseEntity { … }
```

**하이버네이트가 모든 조회 SQL 에 `status='ACTIVE'` 를 자동으로 붙인다.** 편리하지만 대가가 있다 — **지운 걸 볼 방법도 같이 사라진다.** `findById(삭제된id)` 는 `null` 이고, [[PR 201 — 어드민 음식 상세·수정·재수집·삭제 REST|PR 201]] 의 상세 API 는 그래서 400 을 던진다.

> 실수로 삭제한 음식을 확인하려면 **DB 에 직접 붙는 수밖에 없었다.** 운영 도구가 없다는 뜻이다.

> [!warning] ⚠️ 이 노트에 `status` 가 두 개 나온다 — 헷갈리기 쉬움
> | | 무엇 | 값 | 삭제·복원과의 관계 |
> |---|---|---|---|
> | **`status`** | `BaseEntity` 의 **소프트삭제 플래그** | `ACTIVE` / `DELETED` | **삭제 시 `DELETED`, 복원 시 `ACTIVE` 로 되돌아간다** |
> | **`contentStatus`** | `Food` 의 **콘텐츠 생명주기** | `READY` · `PENDING_REVIEW` · `PENDING_IMAGE` … | **삭제·복원과 무관 — 삭제 전 값이 그대로 보존된다** |
>
> 이름이 비슷하지만 **완전히 다른 축**이다. "지워졌나"(status)와 "콘텐츠가 어디까지 준비됐나"(contentStatus)는 서로 독립이다.
> 아래에서 `@SQLRestriction("status = 'ACTIVE'")` 가 거르는 건 **앞쪽**이고, §5-2 에서 "복원이 안 바꾼다"고 하는 건 **뒤쪽**이다.

---

## 2. 해법 ① — native 쿼리는 필터를 안 받는다

```kotlin
@Query(
    value = "SELECT * FROM food WHERE status = 'DELETED' ORDER BY updated_at DESC, id DESC",
    countQuery = "SELECT count(*) FROM food WHERE status = 'DELETED'",
    nativeQuery = true,
)
fun findDeletedPage(pageable: Pageable): Page<Food>

@Query(value = "SELECT * FROM food WHERE id = :id AND status = 'DELETED'", nativeQuery = true)
fun findDeletedById(@Param("id") id: Long): Food?

@Query(value = "SELECT * FROM food WHERE id = :id", nativeQuery = true)   // 상태 무시
fun findAnyById(@Param("id") id: Long): Food?
```

**`@SQLRestriction` 은 하이버네이트가 SQL 을 만들 때 끼워넣는 장치**다. JPQL·메서드 이름 규약·Criteria 처럼 **하이버네이트가 SQL 을 생성하는 경로**에만 붙는다. `nativeQuery = true` 는 내가 쓴 SQL 을 **그대로 보내므로** 필터가 안 붙는다.

- **선례가 있다** — PR 본문이 `MemberBlockJpaRepository.findAnyByPair` 를 근거로 든다. 새 우회로를 뚫은 게 아니라 **레포에 이미 있는 패턴**을 따랐다.
- **`countQuery` 를 따로 준 이유** — native 페이징에선 하이버네이트가 총 건수 쿼리를 못 만들어준다. 직접 줘야 `Page` 가 `totalElements` 를 채운다.
- **정렬이 `updated_at DESC, id DESC`** — `updated_at` 이 `@UpdateTimestamp` 라 **소프트삭제 시점이 곧 마지막 수정 시각**이다. 즉 "최근에 지운 것부터". `id DESC` 는 같은 초에 여러 개가 지워졌을 때의 **tie-breaker** 다. → [[페이징 — Offset vs No-Offset(Keyset·Cursor)]]

> [!warning] 우회로가 세 개가 됐다
> 이제 음식을 id 로 찾는 방법이 셋이다 — `findById`(ACTIVE 만) · `findDeletedById`(DELETED 만) · `findAnyById`(상태 무시). **어느 걸 써야 하는지는 이름으로만 구분**되고 컴파일러는 안 도와준다. 실수로 `findAnyById` 를 일반 조회에 쓰면 **삭제된 음식이 사용자에게 노출**된다. 이름이 명확한 게 유일한 방어선이다.

---

## 3. 해법 ② — 복원은 벡터 큐도 되돌려야 한다

이 PR 에서 가장 생각할 거리가 많은 부분이다.

```kotlin
@Transactional
fun restoreFood(id: Long): AdminFoodRestoreResponse {
    val food = foodRepository.findAnyById(id) ?: throw BusinessException(ErrorCode.FOOD_NOT_FOUND)
    if (!food.isDeleted()) {
        return AdminFoodRestoreResponse(restored = false, contentStatus = food.contentStatus)  // 멱등
    }
    food.active()                                    // ← status: DELETED → ACTIVE (삭제 취소)
    if (food.isReady()) {                            // ← contentStatus 는 손대지 않는다(보존)
        cancelPendingVectorOutboxes(food.id, FoodVectorOutboxOperation.DELETE)   // ← 반대편 취소
        vectorOutboxRepository.enqueueIfAbsent(food.id, FoodVectorOutboxOperation.UPSERT)
    }
    return AdminFoodRestoreResponse(restored = true, contentStatus = food.contentStatus)
}
```

### 왜 "반대편 취소"가 필요한가

음식이 벡터 검색에 실리려면 아웃박스를 거친다(→ [[27. 아웃박스와 SQS — 다른 시스템에 일을 넘기는 법]]). 삭제·복원은 각각 반대 방향의 일감을 만든다:

```
삭제  → 벡터 DELETE 를 큐에 넣음
복원  → 벡터 UPSERT 를 큐에 넣음
```

문제는 **배치가 아직 안 돌았을 때**다. 삭제 직후 바로 복원하면 큐에 **DELETE 와 UPSERT 가 둘 다 PENDING** 으로 남는다. 배치가 어느 걸 먼저 집느냐에 따라 결과가 갈린다 — **UPSERT 를 먼저 처리하면 그다음 DELETE 가 방금 넣은 벡터를 지운다.** 복원했는데 검색에서 사라지는 것이다.

`cancelPendingVectorOutboxes` 가 그걸 막는다 — **복원할 때 아직 안 나간 DELETE 를 지우고** UPSERT 를 넣는다. 삭제 쪽도 대칭으로 PENDING UPSERT 를 지운다.

> [!tip] 이게 아웃박스 패턴의 전형적인 함정이다
> 큐는 **"일을 나중에 한다"**는 뜻이고, 그 사이 **원본이 또 바뀔 수 있다.** 그래서 큐에 넣을 때 **이미 들어 있는 반대 일감을 어떻게 할지** 정해야 한다. 여기선 "지운다"를 골랐는데, 다른 선택지도 있다 — 마지막 것만 남기기(compaction), 순서 보장(시퀀스), 또는 처리 시점에 현재 상태를 다시 읽기(**state-based**, 큐엔 "이 음식 재동기화" 만 넣고 UPSERT/DELETE 판단은 배치가). 마지막 방식이면 이 취소 로직 자체가 필요 없다.

### 멱등

이미 활성인 음식을 복원하면 **아무것도 안 하고 `restored=false`** 를 준다. [[PR 209 — 어드민 벡터 아웃박스 조회·enqueue·재시도 REST|PR 209]] 의 `retried=false`·[[PR 201 — 어드민 음식 상세·수정·재수집·삭제 REST|PR 201]] 의 `skipped=1` 과 **같은 어법**이다. 이 시리즈가 계속 지키는 규약이다.

---

## 4. 덤 — 목록에 영어 이름 병기

```kotlin
val englishName: String?,                                   // AdminFoodListItemResponse
englishName = food.nameTranslations[LanguageCode.EN.code],  // 없으면 null
```

- `nameTranslations` 는 JSON 컬럼(맵)이라 **키가 없으면 그냥 널**이다. 별도 조회가 없어 비용 0.
- PR 본문에 *"어드민 목록 컬럼 병기 요건(예진)"* 이라고 **요청자가 명시**돼 있다. 나중에 "이 필드 왜 있지?"를 묻지 않게 하는 좋은 습관.

---

## 5. 읽으며 눈에 띈 것 — 그리고 리뷰 결과

> 5건을 물었고 판정이 났다(2026-09-01, 커맨드 센터 경유). **반려 3 · 반영 2.**
> 반려 근거는 전부 다르고(중복 제거 / 의미론 / 감수), **반영 2건 중 하나는 실제 버그였다**(§5-5).

### ⬜ 1. 삭제 목록의 `updatedAt` 이 "삭제 시각"을 뜻한다

일반 목록과 **같은 DTO**(`AdminFoodListResponse`)를 재사용해서 **같은 필드가 화면에 따라 다른 의미**가 됐다. `deletedAt` 별칭 필드가 있으면 오해가 없지 않을까?

**→ 반려.** 이 API 의 **유일한 소비자(FE)가 이미 헤더를 다르게 처리**하고 있고 계약 문서에도 명시돼 있다. **같은 값을 담은 필드를 하나 더 내보내는 건 크러프트**(불필요한 잉여)다 — 두 필드가 생기면 "어느 쪽이 진짜냐"를 나중에 또 묻게 된다.

> 소비자가 **하나뿐**이라는 게 판단을 갈랐다. 공개 API 였다면 이름이 곧 문서라 별칭이 정당했을 수 있다.

### ⬜ 2. 복원이 `contentStatus`(콘텐츠 생명주기)를 안 바꾼다

> 먼저 오해를 막자 — **삭제 상태(`status`)는 당연히 풀린다.** `food.active()` 가 `DELETED` → `ACTIVE` 로 되돌린다(§3 코드). 여기서 말하는 건 **그 옆의 다른 필드**, 검수·이미지 준비 단계를 나타내는 **`contentStatus`** 다. **복원은 `status` 만 되돌리고 `contentStatus` 는 삭제 전 값 그대로 둔다.**

그래서 던진 질문: 삭제된 사이에 그 음식이 참조하던 것들(이미지·성분 카탈로그)이 바뀌었으면 **보존된 `contentStatus` 가 실제와 안 맞을 수 있다.** 복원 후 재검수를 유도할 필요는 없나?

**→ 반려(의도 확정).** **복원의 의미론은 "삭제 취소"** 다 — 지우기 전으로 되돌리는 것이지 새로 만드는 게 아니다. `contentStatus` 까지 손대면 그건 복원이 아니라 다른 동작이 된다.
참조가 의심되면 **그 용도의 도구가 이미 있다** — [[PR 201 — 어드민 음식 상세·수정·재수집·삭제 REST|PR 201]] 의 **재수집**(`POST …/recollect`). 한 동작에 여러 의미를 얹지 않고 **도구를 나눠 둔** 설계다.

### ⬜ 3. 취소된 아웃박스 행이 운영 화면에서 안 보인다

`cancelPendingVectorOutboxes` 가 소프트삭제를 쓰는데 [[PR 209 — 어드민 벡터 아웃박스 조회·enqueue·재시도 REST|PR 209]] 의 목록은 ACTIVE 만 본다.

**→ 반려(감수).** ①삭제 직후 복원은 **드문 경우**라 취소행 자체가 잘 안 생기고 ②큐잉 결과는 **토스트로 즉시 확인**되므로 목록에서 다시 찾을 일이 적다. 필요해지면 후속으로.

### ✅ 4. `findAnyById` 가 열렸다 — 오용 통로

일반 조회에 잘못 쓰면 삭제 데이터가 새는 통로다. ArchUnit 으로 호출처를 제한할 수 있지 않을까?

**→ 반영 완료 — ArchUnit 규칙으로**(커밋 `e80cd4c7`). 레포에 `ModuleBoundaryTest` 가 이미 있어서 **주석이 아니라 테스트로** 막았다: **상태 무시 조회를 `api.admin` 밖에서 호출하면 실패**. 메서드명 기반이라 [[PR 217 — 멤버 목록·상세·활동 3종과 대시보드 메트릭 REST|PR 217]] 의 멤버 쪽 `findPageAnyStatus`·`findAnyById` 도 **자동으로 커버**된다.

> [[13. 경계를 지키는 테스트 — ArchUnit이 강제하는 규칙들]] 의 **3층 구분**에서 **두 번째 층(테스트가 막는 것)** 으로 올라간 사례다. 처음엔 "사람이 지키는 것"(주석)으로 두려 했는데, **이미 있는 인프라를 쓰면 공짜**라 한 층 위로 갔다. 새 도구를 들이는 건 과하지만 **있는 도구를 쓰는 건 과하지 않다**는 판단.

### ✅ 5. 콘텐츠 수집 아웃박스는 대칭이 안 맞는다 — **실제 버그였다**

이 PR 은 `FoodVectorOutbox` 만 취소한다. 삭제 시 진행 중이던 `FoodContentOutbox` 는?

**→ 문제 확인, 수정됨**(커밋 `e80cd4c7`). 실제 거동을 추적했더니 **돈이 새고 있었다**:

```
음식 삭제 → 콘텐츠 아웃박스 PENDING 은 그대로 남음
          → 배치가 집어 SQS 로 발행
          → kbap-langchain 이 LLM 으로 콘텐츠 생성   ← 💸 이미 지운 음식에 비용 지출
          → ingest 콜백이 서버로 돌아옴
          → 그 음식이 없으니 FOOD-001 로 거절
          → SENT 행이 영구 표류 + 콜백 에러 로그 노이즈
```

**수정**: 삭제 시 **PENDING 콘텐츠 아웃박스도 소프트삭제로 취소**한다(§3 의 벡터 큐 대칭과 **같은 패턴**). 단 **SENT 는 이미 발행돼 취소할 수 없어** 콜백 거절을 그대로 둔다.

> [!warning] 이게 이 시리즈에서 나온 **가장 값진 발견**이다
> 증상이 조용하다 — 에러가 나긴 하는데 **로그 노이즈로만** 보이고, 기능은 멀쩡해 보인다. 그런데 그 뒤에서 **삭제된 음식에 LLM 비용이 계속 지출**되고 있었다.
> 배운 것: **"큐가 두 개인데 하나만 대칭을 맞췄다"는 비대칭 자체가 신호**다. 코드는 벡터 큐만 다루면서 아무 설명이 없었고, 그 침묵이 곧 누락이었다.
> 그리고 **"이미 나간 것은 못 되돌린다"**(SENT)는 아웃박스의 근본 한계다 — 취소는 **아직 안 나간 것에만** 통한다. → [[27. 아웃박스와 SQS — 다른 시스템에 일을 넘기는 법]]

---

## 6. 다음에 이 PR 을 다시 볼 때

0. **`status` 와 `contentStatus` 는 다른 축이다**(§1 경고 상자) — 전자는 삭제 플래그(복원 시 풀림), 후자는 콘텐츠 생명주기(복원해도 보존).
1. **`@SQLRestriction` 은 native 쿼리엔 안 붙는다** — 소프트삭제를 뚫는 표준 통로. 대신 `countQuery` 를 손으로 줘야 한다.
2. **비동기 큐가 있으면 "되돌리기"는 큐도 되돌려야 한다** — 반대편 PENDING 취소(§3). 아웃박스 패턴을 쓸 때 반드시 만나는 문제다.
3. 이 시리즈의 멱등 어법이 여기도 이어진다 — `restored=false`.
4. **리뷰 결과**(§5): **반려 3 · 반영 2.** 반려 근거가 각각 다르다 — 중복 제거(1) · 의미론(2) · 감수(3).
5. **§5-5 가 실제 버그였다** — 삭제된 음식에 LLM 비용이 계속 나가고 있었다. **리뷰 질문 하나가 비용 누수를 잡은 사례.**
6. 2026-09-01 기준 **OPEN** · 리뷰 반영은 `e80cd4c7` — 이후 변경은 [PR 216](https://github.com/team-skyjs/kbap-server/pull/216) 에서 확인.
