---
tags: [kbap-backend, pr읽기, admin, 아웃박스, 집계, llm비용]
생성일: 2026-09-02
상태: 완료
PR: 219
브랜치: feat/kb407-admin-thymeleaf-parity (open)
---

> [!info] 관련 노트
> 🏠 [[🏠 홈]] · 📂 주제: [[🗺️ kbap 백엔드 공부 지도 (2026-08)]]
> ⬅️ 대칭 구조의 앞 PR: [[PR 209 — 어드민 벡터 아웃박스 조회·enqueue·재시도 REST]] · [[PR 214 — 벡터 아웃박스 목록 검색 q 추가]]
> 🔗 개념: [[27. 아웃박스와 SQS — 다른 시스템에 일을 넘기는 법]] · [[9. 연관관계를 쓰지 않는 설계 — N+1을 애초에 만들지 않기]]

# PR 219 — 콘텐츠 아웃박스·이미지 배치·LLM 비용 레포트 REST

> 작성: 2026-09-02 · 맥락: **Thymeleaf 화면을 전수 대조**해 "SPA 에 아직 없는 것"을 찾아낸 PR. 4종이 나왔고 그중 2종은 **이미 REST 가 있었으며**, 진짜 없는 3종을 신설했다.
> 목표: **같은 아웃박스인데 왜 하나엔 재시도가 있고 하나엔 없는지**를 통해 **"큐마다 책임이 다르다"**를 이해하기.

| | |
|---|---|
| 링크 | [team-skyjs/kbap-server#219](https://github.com/team-skyjs/kbap-server/pull/219) · Jira KB-407 |
| 상태 | **OPEN** (2026-09-02 기준) · 작성자 rocher71 |
| 규모 | **14파일 · +643 / −0** (테스트 279줄, 시나리오 12개) |

---

## 0. 한 줄 요약

**옛 화면(Thymeleaf)과 새 화면(SPA)을 한 줄씩 대조해 빠진 걸 메운 PR.** 신설 3종 — 콘텐츠 아웃박스 조회 · 이미지 배치 목록 · LLM 비용 레포트. 기존 코드 **수정 0줄**(`−0`)이다.

---

## 1. 이 PR 이 특이한 점 — "무엇을 만들까"가 아니라 "무엇이 빠졌나"

앞의 PR 들(201·209·216·217·218)은 **필요한 기능을 새로 설계**했다. 이건 다르다 — **옛 화면을 목록으로 놓고 하나씩 대조**해서 구멍을 찾았다.

그 결과가 재밌다:

| Thymeleaf 에 있던 것 | 실측 결과 |
|---|---|
| 콘텐츠 아웃박스 조회 | ❌ REST 없음 → **신설** |
| 이미지 배치 **목록** | ❌ REST 없음 → **신설** |
| LLM 운영 지표 | ❌ REST 없음 → **신설** |
| 음식 **시드** 등록 | ✅ **이미 있었다** (`POST /api/admin/foods`) |
| 이미지 배치 **제출** | ✅ **이미 있었다** (`POST /api/admin/foods/images`) |

> [!tip] 만들기 전에 세어 보는 것의 값어치
> 4종을 다 만들 뻔했는데 **실코드 대조로 2종이 이미 있음**을 확인했다. 있는 걸 또 만들면 **엔드포인트가 둘로 갈려** 나중에 어느 쪽이 정본인지 싸우게 된다. PR 본문이 *"실코드 대조 결과"* 라고 근거를 남긴 게 좋다.

---

## 2. 콘텐츠 아웃박스 — 벡터 아웃박스와 **대칭이되 셋이 다르다**

[[PR 209 — 어드민 벡터 아웃박스 조회·enqueue·재시도 REST|PR 209]] 의 벡터 아웃박스 화면과 **응답 모양이 같다**(`items/page/totalPages/totalCount/hasPrev/hasNext`). 그런데 뜯어보면 **세 군데가 다르고, 그 차이가 전부 의미 있다.**

### 차이 ① `displayName` 이 자체 컬럼이라 **조인이 아예 없다**

```kotlin
data class AdminContentOutboxItemResponse(
    val id: Long, val foodId: Long,
    val displayName: String,        // ← 아웃박스 행이 직접 들고 있다 (널 아님!)
    val outboxStatus: FoodContentOutboxStatus,
    val attempts: Int,
    val createdAt: LocalDateTime, val sentAt: LocalDateTime?,
) {
    fun from(outbox: FoodContentOutbox) = AdminContentOutboxItemResponse(
        displayName = outbox.displayName,   // ← 그냥 읽는다. 음식 테이블을 안 본다.
        …
    )
}
```

[[PR 209 — 어드민 벡터 아웃박스 조회·enqueue·재시도 REST|209]] 에서는 이름을 얻으려고 **`findAllById` 로 배치 로딩해 `Map` 을 만들었고**(N+1 회피), [[PR 214 — 벡터 아웃박스 목록 검색 q 추가|214]] 에서는 검색하려고 **JPQL 상관 `exists` 서브쿼리**까지 썼다. **여기선 그 복잡도가 통째로 없다.**

```sql
-- 219 의 검색 — 자기 테이블만 본다
where (:foodId is not null and o.foodId = :foodId)
   or o.displayName like concat('%', :keyword, '%')
```

`FoodContentOutbox` 가 **수집 요청 시점의 이름을 스냅샷으로 복사**해 두기 때문이다.

> [!note] 그리고 이 스냅샷이 [[PR 218 — 표시 이름 편집·어드민 반려 유형·삭제 시 이름 반납|PR 218]] 의 조사와 정확히 맞물린다
> 218 에서 **"삭제 시 매치키 개명이 다른 데이터를 깨지 않나"** 를 물었고, 답 중 하나가 *"영속 스냅샷은 **표시용**이지 조인 키가 아니다"* 였다. **그 스냅샷이 바로 이 `displayName`** 이다.
> 그래서 음식이 개명되거나 삭제돼도 **이 화면은 "그때 뭘 요청했는지"를 그대로 보여준다** — 운영 이력으로서는 오히려 그게 맞다.
>
> **트레이드오프**: 지금 이름과 다를 수 있다. 음식 이름을 바꾸면 **옛 아웃박스 행은 옛 이름으로 남는다.** 209 의 조인 방식은 항상 최신이지만 삭제되면 널이 되고, 219 의 스냅샷 방식은 항상 값이 있지만 낡을 수 있다 — **"그때의 사실"과 "지금의 사실" 중 무엇을 보여줄지의 선택**이다.

### 차이 ② **재시도(retry)가 없다** — 이 PR 의 핵심 질문

209 의 벡터 아웃박스에는 `POST …/{id}/retry` 가 있다. 여기엔 없다. **왜?**

| | 벡터 아웃박스 | 콘텐츠 아웃박스 |
|---|---|---|
| 상태 | PENDING · COMPLETE · **FAILED** | PENDING · **SENT** · COMPLETE |
| 실패하면 | `attempts` 5회 후 **FAILED 로 멈춘다** | — **모델에 FAILED·retry 개념이 없다** |
| 밀린 걸 다시 굴리는 주체 | **운영자**(retry 버튼) | **배치**(PENDING 을 자동 재발행) |
| 새 작업을 넣으려면 | enqueue API | **recollect API**(→ [[PR 201 — 어드민 음식 상세·수정·재수집·삭제 REST]]) |

**즉 "재시도"라는 일 자체가 다른 곳에 있다.** 벡터 큐는 **멈춘 것을 사람이 다시 민다**. 콘텐츠 큐는 **PENDING 이면 배치가 알아서 계속 재발행**하고, 새로 시키고 싶으면 **재수집 요청**을 넣는다.

> [!tip] 같은 이름의 패턴이라도 책임은 다를 수 있다
> 둘 다 "아웃박스"지만 **운영 모델이 다르다.** 209 를 읽고 나서 "아웃박스에는 retry 가 있는 것"이라고 외웠다면 여기서 **"왜 안 만들었지?"** 하고 버그로 오해했을 것이다.
> **PR 이 그걸 미리 적어 둔 게 좋다** — *"조회 전용 … PENDING 재발행은 배치 자동·신규 수집 요청은 recollect API 담당이라 재시도/enqueue 는 만들지 않았다."* **안 만든 것에 이유를 남기면 그건 누락이 아니라 결정이 된다.** → [[27. 아웃박스와 SQS — 다른 시스템에 일을 넘기는 법]]

### 차이 ③ 검색 코드가 214 의 **최종형**을 그대로 쓴다

```kotlin
status == null -> outboxRepository.searchByKeyword(keyword, keyword.toLongOrNull(), pageable)
else           -> outboxRepository.searchByKeywordAndStatus(keyword, keyword.toLongOrNull(), status, pageable)
```

- **메서드 이름이 `searchByKeyword…`** — [[PR 214 — 벡터 아웃박스 목록 검색 q 추가|214]] 리뷰에서 *"`@Query` 가 붙으면 이름 파싱이 안 되니 규약형 이름은 오해를 부른다"* 고 판정돼 리네임된 그 관례를 **처음부터 따랐다.**
- **센티널 `-1` 이 아니라 `toLongOrNull()` 널 그대로** — 역시 214 리뷰 반영분(`(:foodId is not null and …)`)을 처음부터 적용했다.

> **리뷰 한 번이 다음 PR 의 출발점을 바꿨다.** 214 에서 고친 두 가지가 219 에서는 **고칠 필요 없이 처음부터 그 모양**이다.

---

## 3. LLM 비용 레포트 — 217 이 못 넣은 것을 다른 방식으로

[[PR 217 — 멤버 목록·상세·활동 3종과 대시보드 메트릭 REST|PR 217]] 의 대시보드는 **LLM 비용을 뺐다** — *"Langfuse 연동 전이라 미포함"*. 219 는 그걸 **다른 출처로** 채운다: **서버 자체 미터링**.

```kotlin
fun getLlmCostReport(days: Int): AdminLlmCostReportResponse {
    require(days in 1..MAX_REPORT_DAYS) { "days 는 1..$MAX_REPORT_DAYS 여야 합니다: $days" }
    val today = LocalDate.now()
    val sumsByDate = llmCallCostRepository
        .sumDailyByModelSince(today.minusDays(days - 1L).atStartOfDay())    // ← 한 번에 긁고
        .groupBy(DailyModelCostSum::date)
    val dailyCosts = (days - 1L downTo 0L).map(today::minusDays).map { date ->
        val models = sumsByDate[date].orEmpty()                              // ← 없는 날은 빈 목록
            .sortedByDescending { it.costUsd }
        AdminLlmDailyCostResponse(
            date = date,
            callCount = models.sumOf { it.callCount },
            costUsd = models.fold(BigDecimal.ZERO) { acc, m -> acc + m.costUsd },
            models = models,
        )
    }
    return AdminLlmCostReportResponse(days = dailyCosts)
}
```

- **데이터 출처가 `LlmCallCost`** — [[27. 아웃박스와 SQS — 다른 시스템에 일을 넘기는 법|27번 노트]]에서 본 **`LlmCallCostIncurred` 도메인 이벤트**가 쌓아 온 그 테이블이다. *"같은 앱 안 비동기(`@EventListener`)"* 로 기록해 둔 게 여기서 **운영 지표로 회수**된다.
- **집계 손동작이 [[PR 217 — 멤버 목록·상세·활동 3종과 대시보드 메트릭 REST|217]] 과 같다** — **넓게 한 번 긁고 → 메모리에서 날짜별로 자르고 → 빈 날은 0/빈 목록으로 채운다.** `GROUP BY` 는 행이 있는 날만 주므로 **없는 날을 코드가 만들어야** 차트가 안 끊긴다.
- **모델별 상세를 비용 내림차순**으로 준다 — "어느 모델이 돈을 먹나"가 한눈에 보이게.
- **`BigDecimal` 로 합산** — 돈이므로 `Double` 이 아니다. `fold(BigDecimal.ZERO)` 로 누적한다.

> [!note] Langfuse vs 자체 미터링
> Langfuse 같은 외부 관측 도구는 **프롬프트·응답·지연까지** 보지만 연동 비용이 든다. 자체 미터링은 **호출 수·비용만** 알지만 이미 우리 DB 에 있다. **"지금 필요한 질문(이번 달 얼마 썼나)에 답할 수 있으면 충분"** 이라는 판단이고, 217 이 비워 둔 자리를 **가진 데이터로** 메운 것이다.

---

## 4. 이미지 배치 목록 — 재사용만 한 케이스

```kotlin
fun getImageBatches(): ResponseEntity<BaseResponse<AdminImageBatchListResponse>>
```

Thymeleaf 의 *"최근 20건 + 항목 카운트(pending·done·failed)"* 뷰를 **기존 서비스(`AdminImageBatchQueryService.getRecentBatches`)를 그대로 불러** REST 로 감싼 것. **새 로직 0줄** — 이 시리즈에서 계속 반복된 *"로직은 그대로, 창구만 추가"* 의 가장 순수한 형태다.

---

## 5. 읽으며 눈에 띈 것 — 그리고 리뷰 결과

> 6건을 물었고 판정이 났다(2026-09-02, 커맨드 센터 경유). **반영 5 · 반려 1** — 이번엔 지적이 대부분 채택됐다.

### ✅ 1. `require(days in 1..30)` 이 사용자 입력 검증에 쓰였다

[[PR 215 — 검수 비대상 거절에 전용 코드 FOOD-008 채번|PR 215]] 가 정리한 대로 `require` 는 `IllegalArgumentException` → **범용 `COMMON-002`** 로 나간다. 그런데 이건 프로그래머 실수가 아니라 **사용자가 보낸 값**이다.

**→ 반영. `@Min`/`@Max` 로 검증 계층에 승격.** 그러면 ①400 이 **검증 단계에서** 나가고 ②**Swagger 에 범위가 문서화**되고 ③서비스는 값이 유효하다고 가정할 수 있다.

> [!tip] 노트가 정리한 교훈이 다음 PR 에 되돌아온 사례
> 215 에서 **"`require` = 내부 버그 신호 / `BusinessException` = 업무 규칙"** 이라는 분류를 정리했다. 219 는 그 분류의 **세 번째 칸**을 보여준다 — **사용자 입력은 둘 다 아니고 "검증"의 몫**이다.
>
> | 상황 | 도구 | 결과 |
> |---|---|---|
> | 프로그래머가 잘못 불렀다 | `require` | `COMMON-002`(범용) |
> | 업무 규칙상 안 된다 | `BusinessException(코드)` | 전용 코드 |
> | **사용자가 이상한 값을 보냈다** | **`@Min`·`@Max`·`@AssertTrue`** | **400 + Swagger 문서화** |

### ⬜ 2. `LocalDate.now()` 가 또 서비스 안에 있다

**→ 반려([[PR 217 — 멤버 목록·상세·활동 3종과 대시보드 메트릭 REST|217]]-1 과 일관).** 대략치 지표라 `Clock` 주입은 과하고, 내가 걱정한 `days` 경계는 **1번 검증이 커버**한다.

> 같은 지적에 **두 번 같은 답**이 나왔다 — 이 레포에서 `Clock` 주입은 "정산·과금처럼 경계가 돈으로 직결될 때" 꺼낼 카드다.

### ✅ 3. `displayName` 스냅샷이 낡을 수 있다는 게 화면에 안 보인다

**→ 반영(FE).** 컬럼/툴팁에 **"요청 시점 이름"** 라벨을 단다. 서버는 무변경 — **판단 재료가 이미 응답에 있으면 FE 가 푼다**는, 이 시리즈에서 반복된 처방이다([[PR 209 — 어드민 벡터 아웃박스 조회·enqueue·재시도 REST|209]] 의 삭제 음식 retry 비활성 · [[PR 214 — 벡터 아웃박스 목록 검색 q 추가|214]] 의 검색창 힌트와 같은 계열).

### ✅ 4. `AdminLlmReportController` 를 따로 판 게 임시 분리

**→ 반영(방향 확정).** [[PR 217 — 멤버 목록·상세·활동 3종과 대시보드 메트릭 REST|217]] 머지 후 **대시보드 컨트롤러로 합류**한다는 계획을 **PR 본문에 명시** + 후속 이슈로 등록.

> **임시 조치에 만료일을 붙인 것.** 안 적으면 "왜 대시보드가 두 컨트롤러에 흩어져 있지?"가 6개월 뒤에 남는다.

### ✅ 5. 콘텐츠 아웃박스의 `attempts` 가 무엇을 세는지 안 보인다

**→ 반영(FE).** 화면에 **무엇을 세는 값인지 표기**한다. 벡터 쪽은 `attempts`+`lastError` 로 "5회 실패하면 멈춤"이 읽히는데, 콘텐츠 쪽은 FAILED 개념이 없어(§2-②) **같은 숫자가 다른 뜻**이다.

### ✅ 6. Thymeleaf 대조가 일회성이다

**→ 반영(문서).** **커버리지 매트릭스를 kickoff 문서에 상시화**해 재대조 기준으로 삼는다.

> 이 PR 자체가 "한 번 세어 본" 결과물인데, **그 세어 본 결과를 버리지 않고 남기는** 쪽으로 갔다. Thymeleaf 를 실제로 걷어낼 때 다시 쓸 목록이다.

---

## 6. 기존 노트와 이어지는 곳

| 이 PR 에서 본 것 | 원리 노트 · 앞선 PR |
|---|---|
| 아웃박스 두 종류의 책임 차이(재시도 주체) | [[27. 아웃박스와 SQS — 다른 시스템에 일을 넘기는 법]] · [[PR 209 — 어드민 벡터 아웃박스 조회·enqueue·재시도 REST]] |
| 스냅샷 컬럼 덕에 조인·N+1 회피가 통째로 불필요 | [[9. 연관관계를 쓰지 않는 설계 — N+1을 애초에 만들지 않기]] |
| 214 리뷰 반영분(네이밍·널 파라미터)을 처음부터 적용 | [[PR 214 — 벡터 아웃박스 목록 검색 q 추가]] |
| 스냅샷은 "조인 키가 아니라 표시용" | [[PR 218 — 표시 이름 편집·어드민 반려 유형·삭제 시 이름 반납]] |
| 넓게 긁어 메모리에서 자르기 · 빈 날 채우기 | [[PR 217 — 멤버 목록·상세·활동 3종과 대시보드 메트릭 REST]] |
| `LlmCallCost` 를 쌓아 둔 도메인 이벤트가 지표로 회수됨 | [[27. 아웃박스와 SQS — 다른 시스템에 일을 넘기는 법]] |
| `require` 의 분류(내부 버그 vs 사용자 입력) | [[PR 215 — 검수 비대상 거절에 전용 코드 FOOD-008 채번]] |

---

## 7. 다음에 이 PR 을 다시 볼 때

1. **"만들기 전에 세어 봤다"** — 4종 중 2종은 이미 있었다. 대조가 중복 엔드포인트를 막았다.
2. **같은 아웃박스라도 재시도 주체가 다르다** — 벡터는 사람(retry 버튼), 콘텐츠는 배치(자동 재발행) + recollect. **안 만든 것에 이유를 적으면 결정이 된다.**
3. **스냅샷 vs 조인** — "그때의 사실"(219, 항상 값 있지만 낡을 수 있음) vs "지금의 사실"(209, 최신이지만 삭제되면 널).
4. **리뷰가 다음 PR 의 출발점을 바꾼다** — 214 에서 고친 네이밍·널 파라미터가 여기선 처음부터 그 모양이다.
5. **리뷰 결과**(§5): **반영 5 · 반려 1.** 1번이 특히 값지다 — **215 가 정리한 `require` 분류에 "사용자 입력은 검증의 몫"이라는 세 번째 칸이 추가**됐다.
6. 2026-09-02 기준 **OPEN · Codex 리뷰 클린 종결** — 이후 변경은 [PR 219](https://github.com/team-skyjs/kbap-server/pull/219) 에서 확인.
