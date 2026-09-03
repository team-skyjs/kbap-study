---
tags: [kbap-backend, pr읽기, admin, 멤버, 대시보드, 집계]
생성일: 2026-09-01
상태: 완료
PR: 217
브랜치: (open)
---

> [!info] 관련 노트
> 🏠 [[🏠 홈]] · 📂 주제: [[🗺️ kbap 백엔드 공부 지도 (2026-08)]]
> ⬅️ 같은 기법을 먼저 쓴 PR: [[PR 216 — 삭제 음식 조회·복원 REST와 벡터 큐 대칭]] (native 쿼리로 상태 필터 우회)
> 🔗 개념: [[9. 연관관계를 쓰지 않는 설계 — N+1을 애초에 만들지 않기]] · [[15. 온보딩 프로필 저장 흐름 — 요청이 DB에 닿기까지]] · [[22. 리뷰 작성 흐름 — 쓰기 경로와 랭킹 원장]] · [[24. 주문 흐름 — 스캔 결과가 주문 이력이 되기까지]]

# PR 217 — 멤버 목록·상세·활동 3종과 대시보드 메트릭 REST

> 작성: 2026-09-01 · 맥락: 어드민 리빌드 에픽의 **마지막 큰 조각**. 음식(201·214·215·216)에 이어 **회원 쪽**을 REST 로 연다. 목록·상세·활동(리뷰·스캔·주문)·대시보드 지표.
> 목표: **회원 한 명의 활동을 서버가 어떻게 긁어 모으는지**, 그리고 **대시보드 숫자를 쿼리 몇 번으로 만드는지** 읽기.

| | |
|---|---|
| 링크 | [team-skyjs/kbap-server#217](https://github.com/team-skyjs/kbap-server/pull/217) · Jira KB-405 |
| 상태 | **OPEN** (2026-09-01 기준) · 작성자 rocher71 |
| 규모 | **13파일 · +958 / −0** — 이번 시리즈 **최대**. 테스트 377줄(13개 시나리오) |

---

## 0. 한 줄 요약

**엔드포인트 5개를 새로 열었다** — 멤버 목록·상세·리뷰/스캔/주문 활동·대시보드 지표. 기존 코드를 **한 줄도 안 고쳤고**(`−0`), 이 시리즈에서 배운 손동작들(native 조회·`q` 관례·N+1 회피)을 **그대로 재사용**한다.

---

## 1. 무엇이 열렸나

| 엔드포인트 | 무엇을 |
|---|---|
| `GET /api/admin/members?page&q` | 목록·검색 — **탈퇴 회원 포함**, 20건씩 |
| `GET /api/admin/members/{id}` | 상세 — 프로필·회피 설정·활동 수 3종 |
| `GET …/{id}/reviews` · `/scans` · `/orders` | 활동 목록 3종, 각 20건씩 |
| `GET /api/admin/dashboard/metrics` | 활성 회원·검수 대기·주간 스캔(+전주 비교)·최근 7일 시리즈 |

> [!note] `−0` 의 의미 — 충돌을 피하는 PR 분리
> PR 본문에 *"음식 영역 파일은 건드리지 않았고(#214·#215·#216 머지 대기 충돌 방지)"* 라고 적혀 있다. **네 PR 이 동시에 열려 있는 상황**이라, 겹치는 파일을 안 만지도록 **범위를 의도적으로 갈랐다.** 그래서 삭제 라인이 0이다.

---

## 2. 이 시리즈에서 배운 것을 그대로 쓴다

이 PR 이 공부에 좋은 이유는 **새 기법이 거의 없다는 것**이다. 앞 PR 들의 손동작이 관례로 굳어 재사용된다.

### ① native 쿼리로 상태 필터 우회 — [[PR 216 — 삭제 음식 조회·복원 REST와 벡터 큐 대칭|216]] 에서 배운 것

```kotlin
@Query(
    value = "SELECT * FROM member ORDER BY id DESC",
    countQuery = "SELECT count(*) FROM member",
    nativeQuery = true,
)
fun findPageAnyStatus(pageable: Pageable): Page<Member>
```

어드민은 **탈퇴 회원도 봐야 한다.** `Member` 도 `BaseEntity` 라 `@SQLRestriction("status='ACTIVE'")` 가 걸리는데, native 쿼리는 그 필터를 안 받는다. 216 이 음식에 쓴 기법이 **하루 만에 회원으로 번졌다** — 좋은 패턴은 이렇게 퍼진다.

> ⚠️ 216 에서 지적한 위험도 함께 온다: `findAnyById`·`findPageAnyStatus` 를 **일반 API 에서 잘못 부르면 탈퇴 회원이 노출**된다. 이제 음식·회원 양쪽에 우회 통로가 생겼다.

### ② `q` 검색 관례 — 세 번째 반복

```kotlin
memberRepository.searchPageAnyStatusByKeyword(keyword, keyword.toLongOrNull() ?: NO_MEMBER_ID, pageable)
```

```sql
WHERE id = :memberId
   OR nickname LIKE CONCAT('%', :keyword, '%')
   OR email    LIKE CONCAT('%', :keyword, '%')
```

**"부분 일치 + 숫자면 id 도"** — [[PR 214 — 벡터 아웃박스 목록 검색 q 추가|214]] 와 똑같은 모양이다(센티널 `-1` 까지). 음식 → 아웃박스 → 회원으로 **세 번 반복됐으니 이제 레포 관례**다. 어디서 검색하든 운영자가 같은 방식으로 쓸 수 있다는 게 이 일관성의 값어치다.

### ③ N+1 회피가 **헬퍼로 승격**됐다

```kotlin
private fun foodNamesOf(foodIds: List<Long>): Map<Long, String> =
    foodRepository.findAllById(foodIds.distinct()).associateBy({ it.id }, { it.displayName })
```

[[PR 209 — 어드민 벡터 아웃박스 조회·enqueue·재시도 REST|209]] 에서 한 번, [[24. 주문 흐름 — 스캔 결과가 주문 이력이 되기까지|24번 노트]]에서 또 한 번 본 그 손동작이 — **리뷰·스캔 두 목록이 같이 쓰니까 private 함수로 뽑혔다.** 연관관계가 없는 설계에서 조합은 늘 손으로 하는데, **세 번째부터는 추출한다**는 자연스러운 진화다. → [[9. 연관관계를 쓰지 않는 설계 — N+1을 애초에 만들지 않기]]

- **스캔은 `mapNotNull { it.foodId }`** — 스캔 이력의 `foodId` 는 **널일 수 있다**(카탈로그에 없는 메뉴를 스캔한 경우). 그래서 `map` 이 아니라 `mapNotNull`, 응답에선 `foodName = null`. → [[19. 메뉴판 스캔 판정 흐름 — 메뉴명이 위험도가 되기까지]]

### ④ 멱등·페이지 응답 모양도 그대로

`items / page / totalPages / totalCount / hasPrev / hasNext` — 209·216 과 동일. FE 가 페이지네이션 컴포넌트를 하나만 만들면 된다.

---

## 3. 활동 3종 — 회원 한 명을 세로로 훑기

```kotlin
fun getMemberReviewPage(memberId: Long, page: Int): AdminMemberReviewPageResponse {
    requireMemberExists(memberId)                                    // ① 존재 확인
    val result = reviewRepository.findByMemberId(memberId, activityPageable(page))   // ② 페이지 조회
    val foodNames = foodNamesOf(result.content.map { it.foodId })    // ③ 이름 배치 로딩
    return AdminMemberReviewPageResponse(items = …, totalCount = result.totalElements, …)
}
```

스캔·주문도 **글자 그대로 같은 모양**이다. 세 메서드가 ①②③ 구조를 공유한다.

- **`requireMemberExists` 를 먼저 부르는 이유** — 없는 회원의 활동을 조회하면 **빈 목록이 아니라 400(MEMBER-003)** 이어야 한다. "회원은 있는데 활동이 없음"과 "회원 자체가 없음"은 다른 상황이고, FE 는 후자에서 화면을 벗어나야 한다.
- **대가**: 활동 조회마다 **회원 조회 쿼리가 한 번 더** 나간다(총 3번: 회원 + 활동 + 음식 이름). 상세 화면에서 탭을 옮길 때마다 반복되는데, 어드민 트래픽이라 감수할 만하다.
- **`activityPageable` 이 `id DESC` 고정** — 최신순. 별도 정렬 옵션은 없다.

**활동별로 담는 것**(PR 본문 기준): 리뷰는 음식 이름·별점 3종·이미지 URL / 스캔은 음식 매칭 여부·가격 / 주문은 메뉴판 이미지 URL·도로명 주소. 각 도메인 노트([[22. 리뷰 작성 흐름 — 쓰기 경로와 랭킹 원장|22]]·[[19. 메뉴판 스캔 판정 흐름 — 메뉴명이 위험도가 되기까지|19]]·[[24. 주문 흐름 — 스캔 결과가 주문 이력이 되기까지|24]])에서 본 필드들이 어드민 화면으로 다시 나오는 셈이다.

---

## 4. 대시보드 — 숫자 다섯 개를 쿼리 몇 번으로 만드나

```kotlin
fun getMetricsSummary(): AdminDashboardMetricsResponse {
    val today = LocalDate.now()
    val dailyScans = scanHistoryRepository.countDailySince(today.minusDays(13).atStartOfDay())
        .associate { it.date to it.count }              // ← 14일치를 한 번에
    val thisWeek = (6L downTo 0L).map(today::minusDays)
    val prevWeek = (13L downTo 7L).map(today::minusDays)
    return AdminDashboardMetricsResponse(
        totalActiveMembers = memberRepository.countByMemberStatus(MemberStatus.ACTIVE),
        pendingReviewCount = foodRepository.countByContentStatus(FoodContentStatus.PENDING_REVIEW),
        weeklyScanCount   = thisWeek.sumOf { dailyScans[it] ?: 0L },
        prevWeekScanCount = prevWeek.sumOf { dailyScans[it] ?: 0L },
        weeklyScans       = thisWeek.map { AdminDailyCountResponse(it, dailyScans[it] ?: 0L) },
    )
}
```

**쿼리는 3번뿐**이다(활성 회원 수 · 검수 대기 수 · 일별 스캔 14일치). 나머지는 전부 메모리 계산이다.

- **핵심 아이디어: 14일치를 한 번에 긁고 7·7 로 자른다.** "이번 주"와 "전주"를 따로 쿼리하면 2번인데, **범위를 넓혀 한 번에 가져와 나누면 1번**이다. 집계 화면에서 자주 쓰는 손동작.
- **`?: 0L` 이 왜 필요한가** — `GROUP BY date` 는 **행이 있는 날짜만** 돌려준다. 스캔이 0건인 날은 결과에 아예 없다. 그대로 쓰면 차트에 그날이 빠지므로, **날짜 목록을 코드가 만들고 없는 날은 0으로 채운다.** 시계열 API 의 필수 처리다.
- **차트 계산을 클라이언트로 넘겼다** — PR 본문: *"차트는 클라이언트 계산, dayLabel/heightPct 제외"*. 서버는 `date`·`count` 원자료만 주고 **막대 높이(%)·요일 라벨은 FE 가 만든다.** 옳은 분리다 — 서버가 `heightPct` 를 계산하면 차트 디자인이 바뀔 때마다 서버를 고쳐야 한다.
- **LLM 비용은 뺐다** — *"Langfuse 연동 전이라 미포함"*. 못 넣는 이유를 PR 에 남긴 것.

---

## 5. 읽으며 눈에 띈 것 — 그리고 리뷰 결과

> 6건을 물었고 판정이 났다(2026-09-01, 커맨드 센터 경유). **반려 4 · 반영 2.**
> **6번(이메일 로그)은 실제 유출이었다** — 확인 후 마스킹 조치까지 갔다(§5-6).

### ⬜ 1. `LocalDate.now()` 를 서비스가 직접 부른다

시계가 코드에 박혀 있어 "어제 기준" 같은 테스트를 쓰기 어렵다. `Clock` 주입이면 고정할 수 있다.

**→ 반려(v1 기준).** 대시보드는 **대략치 메트릭**이라 시간 경계의 정확도 요구가 낮고, `Clock` 주입은 **레포 전체 관례를 바꾸는 일**이라 이 한 화면 때문에 도입하기엔 과하다.

> 판단 축이 "옳은가"가 아니라 **"이 화면의 요구 수준에 맞는가"** 였다. 정산·과금처럼 경계가 돈으로 직결되는 자리였다면 답이 달랐을 것이다.

### ⬜ 2. `sanctions` 가 항상 빈 배열

정지 모델이 없는데 필드를 미리 냈다 — YAGNI 와 부딪히지 않나?

**→ 반려(유지).** 이 필드가 지금 약속하는 건 **"배열이다"** 정도의 최소 계약뿐이라 **잘못된 모양을 굳힐 위험이 작다.** 반대로 지금 빼면 정지 모델이 들어올 때 **FE 왕복 비용**(응답 구조 변경 → 타입 수정 → 재배포)이 더 크다. **자리 표시가 발주 의도**였다.

> 내 지적의 전제("구조가 확정 안 됐으면 빼는 게 낫다")가 **약속의 크기를 과대평가**했다. 빈 배열은 필드 이름 하나만 약속하지 원소의 모양까지 약속하지 않는다.

### ⬜ 3. 활동 3종 응답 DTO 가 판박이

제네릭 `AdminPageResponse<T>` 로 묶으면 줄겠지만, **Swagger 스키마 이름이 뭉개지는** 문제가 있어 일부러 편 것 아닌가?

**→ 반려. 의도 확정 — 내 추정이 맞았다.** 두 가지 이유:
1. **springdoc 이 제네릭을 단일 스키마명으로 뭉갠다** — `AdminPageResponse` 하나로 합쳐져 **FE 코드젠에서 `items` 의 타입 구분이 소실**된다. 타입 안전을 잃는 대가가 코드 193줄보다 크다.
2. **레포 관례가 "기능별 명시 Response"** 다 — 여기만 제네릭으로 가면 일관성이 깨진다.

> **중복이 항상 나쁜 게 아니다.** 여기선 중복을 없애는 순간 **생성되는 클라이언트 타입이 뭉개진다** — 코드 재사용과 계약 표현이 충돌할 때 이 레포는 **계약 쪽**을 택했다.

### ✅ 4. `requireMemberExists` 가 회원을 통째로 로딩한다

`existsById` 가 더 싸 보이지만, 그건 `@SQLRestriction` 을 타서 **탈퇴 회원을 "없음"으로 오판**한다.

**→ 반영 완료 — 주석**(커밋 `d52a680c`). **내 결론이 정답**으로 확인됐다 — 지금 코드가 맞고, `existsById` 로 바꾸면 탈퇴 회원 조회가 깨진다. 다만 **왜 통째로 로딩하는지 이유를 주석으로** 남기기로 했다(안 그러면 다음 사람이 "최적화"하다 버그를 만든다).

> 이 시리즈에서 반복되는 교훈이다 — **코드가 맞아도 이유가 안 보이면 나중에 잘못 고쳐진다.** [[PR 215 — 검수 비대상 거절에 전용 코드 FOOD-008 채번|PR 215]] 의 비대칭 주석과 같은 처방.

### ⬜ 5. native 검색이 `LIKE '%…%'` 두 컬럼

**→ 반려.** [[PR 214 — 벡터 아웃박스 목록 검색 q 추가|PR 214]] 와 **동일한 판정**이다 — 어드민 저빈도 화면이라 수용하되 **"스케일하면 최적화 1순위"** 인식만 유지.

### 🔍 6. `email` 로 검색된다 — 로그 유출 가능성

어드민 전용이라 검색 자체는 정당하지만, **검색어가 요청 로그에 남으면 개인정보가 로그로 샌다.**

**→ 유출 확인, 조치 완료**(커밋 `d52a680c`). 조사 결과 **`RequestLoggingFilter` 가 쿼리스트링을 원문 그대로 로깅**하고 있었다 — 마스킹 대상은 **위경도뿐**이었다. 즉 어드민이 이메일로 검색할 때마다 **그 이메일이 서버 로그에 평문으로 쌓이고 있었다.**

**조치**: 마스킹 목록에 **`q` 를 일괄 등록** + `QueryMaskingTest` 추가.

> [!warning] 이 지적이 실제 개인정보 유출을 잡았다
> 나머지 5건이 설계 취향·성능인 데 반해 이건 **개인정보 취급** 문제였다. 로그는 **접근 통제가 API 와 다른 곳**(로그 수집기·보관 스토리지)에 쌓이므로, **API 로는 어드민만 볼 수 있는 데이터가 로그로는 더 넓게 퍼진다.**
> **트레이드오프도 기록됐다** — `q` 를 통째로 가리면 **음식 검색어 로그까지 가려진다.** 경로별로 분기해 어드민 `q` 만 가릴 수도 있었지만, **단순성을 택했다.** 로그 마스킹은 빠뜨리면 사고라 **넓게 잡는 쪽이 안전**하다.
> → [[17. JWT 인증 필터와 세션 생명주기 — 매 요청 검증·재발급·로그아웃]] 의 MDC 상관 로깅(KB-130)

---

## 6. 다음에 이 PR 을 다시 볼 때

1. **새 기법이 없다는 게 이 PR 의 특징**이다 — 216 의 native 우회, 214 의 `q` 관례, 209/24 의 N+1 회피가 전부 재사용된다. **에픽이 굳어가는 모습**을 보는 자료.
2. 집계 손동작 하나: **넓은 범위를 한 번에 긁어 메모리에서 자르고, 빈 날짜는 코드가 0으로 채운다**(§4).
3. **서버는 원자료, 차트 계산은 클라이언트** — 응답 설계의 좋은 경계.
4. **리뷰 결과**(§5): **반려 4 · 반영 2.** 가장 값진 건 6번 — **실제 개인정보 로그 유출을 잡았다.**
5. 3번의 교훈: **중복 제거가 항상 이득은 아니다.** 제네릭으로 묶으면 springdoc 스키마가 뭉개져 FE 타입 안전을 잃는다.
6. 2026-09-01 기준 **OPEN** · 리뷰 반영은 `d52a680c` — 이후 변경은 [PR 217](https://github.com/team-skyjs/kbap-server/pull/217) 에서 확인.
