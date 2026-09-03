---
tags: [백엔드, 페이징, pagination, 데이터베이스, 성능, api]
생성일: 2026-07-03
상태: 완료
---

> [!info] 관련 노트
> 🏠 [[🏠 홈]] · 📂 주제: 백엔드
> 🔗 kbap 실전: [[9. 연관관계를 쓰지 않는 설계 — N+1을 애초에 만들지 않기]] — 여기 이론상의 keyset 커서가 실제 백엔드에선 **"커서로 id만 페이징(`findFoodPageIds`) → 그 id로 본체 조회(`findByIdIn`)"** 2단계로 돈다(`common/src/main/kotlin/com/kbap/common/domain/food/FoodJpaRepository.kt:107-117`, kb-63). 조인이 아니라 **id 목록을 거치는** 2단계다.

# 페이징 — Offset vs No-Offset (Keyset · Cursor)

> 맥락: 페이징 API 만들면서 "no offset이 기존보다 빠르다"는 말을 듣고 정리.
> 목표: 두 방식의 **이름·원리·성능 차이**를 확실히 알고 언제 뭘 쓸지 판단하기.

---

## 0. 한 줄 요약
**기존 페이징 = "Offset 기반"(`LIMIT ? OFFSET ?`), no offset = "Keyset/Cursor 기반"(`WHERE id > ?`).**
- Offset은 **건너뛸 행을 전부 세다가 버려서** 뒤로 갈수록 느려짐 (페이지 깊어질수록 O(n)).
- Keyset은 **인덱스로 시작점을 바로 찾아서** 페이지가 아무리 깊어도 속도가 일정 (O(1)에 가까움).
- 실측: 1000번째 페이지가 8초 → 45ms, **177배 빨라진** 사례도 있음.

> [!important] 용어 정리 (헷갈리는 이름들)
> - **기존 페이징 = Offset pagination = Limit-Offset pagination** (당신이 "이거 뭐라 하지?" 한 그것)
> - **no offset = Keyset pagination ≈ Cursor pagination = Seek method**
> - "keyset"과 "cursor"는 거의 같은 뜻으로 쓰임 (cursor = keyset 값을 인코딩한 토큰인 경우가 많음).

---

## 1. 핵심 비교표

| | **Offset (기존)** | **Keyset/Cursor (no offset)** |
|---|---|---|
| 쿼리 | `LIMIT 20 OFFSET 99980` | `WHERE id > 1042 ORDER BY id LIMIT 20` |
| 원리 | 앞의 N개를 **세고 버림** | 인덱스로 **바로 그 지점 탐색(seek)** |
| 깊은 페이지 성능 | ❌ 뒤로 갈수록 느려짐 | ✅ 깊이 무관 일정 |
| 임의 페이지 점프 | ✅ "537페이지로" 가능 | ❌ 순차 이동만 (다음/이전) |
| 전체 페이지 수 | ✅ 알기 쉬움 | ❌ 알기 어려움 |
| 데이터 변경 시 정확성 | ❌ 행 밀림(중복·누락) | ✅ 안정적 |
| 구현 난이도 | 쉬움 | 조금 더 복잡 |

---

## 2. 왜 Offset이 느린가 (원리)

`LIMIT 20 OFFSET 99980` → DB는 **99,980개 행을 실제로 읽고 전부 버린 뒤** 그다음 20개만 반환한다. 버릴 행도 다 읽어야 하니, 페이지가 깊을수록 읽는 양이 선형으로 늘어남.

```sql
-- Offset: 100만 번째 근처 페이지
SELECT * FROM posts ORDER BY id LIMIT 20 OFFSET 99980;
-- → 99,980행 스캔 후 폐기 → 20행 반환 (PostgreSQL에서 약 8초 걸린 사례)
```

`★ 핵심 통찰 ─────────────────────────────`
OFFSET은 "몇 번째부터"를 **위치(순번)** 로 지정한다. DB엔 "99,981번째 행"으로 바로 가는 지름길이 없어서,
처음부터 세어 내려갈 수밖에 없다. 그래서 **깊이에 비례해 느려진다.**
`──────────────────────────────────────`

---

## 3. 왜 Keyset이 빠른가 (원리)

"몇 번째"가 아니라 **"마지막으로 본 값 다음부터"** 로 지정한다. 정렬 컬럼에 인덱스가 있으면, DB가 그 값을 **바로 찾아(index seek)** 거기서부터 20개만 읽는다. 건너뛸 행을 안 읽으니 깊이와 무관하게 빠름.

```sql
-- Keyset: 직전 페이지 마지막 id가 99980이었다면
SELECT * FROM posts WHERE id > 99980 ORDER BY id LIMIT 20;
-- → 인덱스로 99980 위치 바로 탐색 → 20행만 읽음 (약 45ms)
```

> 비유: **책에서 특정 페이지 찾기.**
> - Offset = 1쪽부터 손가락으로 넘기며 세다가 537쪽에서 멈춤 (뒤 페이지일수록 오래 걸림).
> - Keyset = 책갈피(마지막 읽은 위치)를 끼워뒀다가 **바로 그 자리를 펼침** (몇 쪽이든 한 번에).

> [!note] 인덱스가 필수
> Keyset의 속도는 **정렬 컬럼에 인덱스**가 있어야 나옴. `WHERE id > ?`의 `id`가 PK(자동 인덱스)면 OK.
> 정렬 기준이 여러 개면(예: 생성일 + id) **복합 인덱스**가 필요.

---

## 4. 정확성 문제 — Offset의 숨은 함정

Offset은 성능뿐 아니라 **정확성**도 문제. 페이지를 넘기는 사이 데이터가 바뀌면 행이 밀린다:

- 1페이지(1~20번) 본 뒤, 누가 3번 행을 **삭제** → 21번이 20번 자리로 당겨짐
- 2페이지(21~40 위치)를 요청 → **원래 21번이던 행을 건너뜀(누락)**
- 반대로 행이 추가되면 **같은 행이 두 번 보임(중복)**

Keyset은 "id > 마지막값"으로 잡으니, 중간에 뭐가 지워지든 **다음 값부터** 정확히 이어짐.

---

## 5. 언제 뭘 쓰나 (판단 기준)

> [!tip] Offset을 쓸 때
> - 데이터가 적고 거의 안 변함 (관리자 테이블, CMS 목록)
> - **"537페이지로 점프", "총 1,203페이지"** 같은 임의 접근·전체 페이지 수가 꼭 필요할 때
> - 대략 **10만 행 미만**

> [!tip] Keyset/Cursor를 쓸 때
> - 무한 스크롤, 피드, "더 보기" (순차 이동만 필요)
> - 데이터 내보내기, 이벤트 스트리밍, 로그 수백만 건 페이징
> - 자주 바뀌는 동적 데이터 (정확성 중요)
> - **10만 행 넘으면 keyset으로 전환** 권장

---

## 6. API 설계 관점

### Offset 방식 API
```
GET /posts?page=3&size=20
→ 서버: LIMIT 20 OFFSET 40
응답: { items: [...], page: 3, totalPages: 1203, totalItems: 24051 }
```

### Cursor 방식 API
```
GET /posts?limit=20&cursor=eyJpZCI6OTk5ODB9   ← 마지막 항목 정보를 인코딩한 토큰
→ 서버: WHERE id > 99980 ORDER BY id LIMIT 20
응답: { items: [...], nextCursor: "eyJpZCI6MTAwMDAwfQ" }  ← 없으면 마지막 페이지
```
- **cursor**: 보통 "마지막 항목의 정렬값(id 등)"을 base64로 인코딩한 불투명 토큰. 클라이언트는 내용 몰라도 그대로 다음 요청에 넣기만.
- 정렬이 복합키면 cursor에 여러 값을 담음 (예: `{created_at, id}`).

> [!warning] 복합 정렬 keyset 주의
> `created_at`으로 정렬 시 같은 시각 행이 여럿이면 누락 위험. **동점 처리용 tie-breaker(보통 id)** 를 정렬·조건에 같이 넣어야 함:
> ```sql
> WHERE (created_at, id) > (:last_created_at, :last_id)
> ORDER BY created_at, id LIMIT 20
> ```

---

## 6-1. 실제 구현 코드 예제 (MySQL · Postgres · Spring JPA)

> 공통 가정: `posts(id BIGINT PK, created_at, title)` 테이블. 최신순 아님, **오래된 순(오름차순)** 으로 20개씩.
> 정렬 = `created_at` 기준 + 동점 tie-breaker `id`. → **인덱스 `(created_at, id)` 꼭 생성.**

```sql
-- 세 DB 공통: keyset 성능의 전제 조건
CREATE INDEX idx_posts_created_id ON posts (created_at, id);
```

### 🐬 MySQL

```sql
-- ❌ Offset (기존): 깊은 페이지일수록 느림
SELECT id, created_at, title
FROM posts
ORDER BY created_at, id
LIMIT 20 OFFSET 99980;

-- ✅ Keyset (no offset): 직전 페이지 마지막 (created_at, id) = (:ca, :id)
--    MySQL도 8.0+ 에서 row-value 비교 (a,b) > (c,d) 지원 (인덱스 잘 탐)
SELECT id, created_at, title
FROM posts
WHERE (created_at, id) > (:ca, :id)
ORDER BY created_at, id
LIMIT 20;

-- (호환용) row-value가 부담이면 OR 풀어쓰기 — 결과 동일, 인덱스는 살짝 덜 최적
SELECT id, created_at, title
FROM posts
WHERE created_at > :ca
   OR (created_at = :ca AND id > :id)
ORDER BY created_at, id
LIMIT 20;
```

### 🐘 PostgreSQL

```sql
-- ❌ Offset (기존)
SELECT id, created_at, title
FROM posts
ORDER BY created_at, id
LIMIT 20 OFFSET 99980;

-- ✅ Keyset (no offset): Postgres는 튜플 비교가 가장 깔끔하고 인덱스도 잘 탐
SELECT id, created_at, title
FROM posts
WHERE (created_at, id) > (:ca, :id)
ORDER BY created_at, id
LIMIT 20;
```
> [!note] Postgres 팁
> `(created_at, id) > (:ca, :id)` 튜플(row-value) 비교가 `(created_at, id)` 복합 인덱스를 그대로 사용해서 index seek 한 번으로 끝남. Postgres에서 keyset의 정석.

### ☕ Spring Data JPA — 3가지 방법

**① Offset (가장 익숙, Pageable)**
```java
public interface PostRepository extends JpaRepository<Post, Long> {
    Page<Post> findAllByOrderByCreatedAtAscIdAsc(Pageable pageable);
}

// 호출부 — page 번호로 접근 (내부적으로 LIMIT/OFFSET)
var pageable = PageRequest.of(page, 20);          // page=0,1,2...
Page<Post> result = repo.findAllByOrderByCreatedAtAscIdAsc(pageable);
result.getTotalPages();   // 총 페이지 수 알 수 있음(장점) — 대신 깊으면 느림
```

**② Keyset 수동 (@Query) — 원리가 그대로 보임**
```java
@Query("""
    SELECT p FROM Post p
    WHERE p.createdAt > :ca
       OR (p.createdAt = :ca AND p.id > :id)
    ORDER BY p.createdAt ASC, p.id ASC
""")
List<Post> findNextPage(@Param("ca") Instant ca,
                        @Param("id") Long id,
                        Pageable pageable);   // PageRequest.of(0, 20) 로 LIMIT만 사용

// 첫 페이지는 커서 없이 (맨 앞부터)
@Query("SELECT p FROM Post p ORDER BY p.createdAt ASC, p.id ASC")
List<Post> findFirstPage(Pageable pageable);
```
> JPA는 튜플 비교 `(a,b) > (c,d)` 를 지원 안 해서 **OR로 풀어써야** 함(위 MySQL 호환 버전과 동일한 형태).

**③ Keyset 자동 (Scroll API, Spring Data 3.1+) — 커서 관리를 프레임워크가 대신**
```java
public interface PostRepository extends JpaRepository<Post, Long> {
    // 정렬은 메서드 이름에, 시작점은 ScrollPosition으로
    Window<Post> findFirst20ByOrderByCreatedAtAscIdAsc(ScrollPosition position);
}

// 첫 페이지: keyset 커서를 "맨 처음"으로
Window<Post> first = repo.findFirst20ByOrderByCreatedAtAscIdAsc(ScrollPosition.keyset());

// 다음 페이지: 직전 결과의 마지막 위치를 그대로 넘김 (커서 인코딩/디코딩 불필요)
if (!first.isLast()) {
    ScrollPosition next = first.positionAt(first.size() - 1);
    Window<Post> second = repo.findFirst20ByOrderByCreatedAtAscIdAsc(next);
}
```
> [!tip] 실무 추천
> - 관리자 화면·"N페이지로 점프" 필요 → **① Offset**
> - 무한스크롤·대용량 피드 → **③ Scroll API**(가장 편함). 세밀한 제어가 필요하면 **② @Query**.

---

## 7. 용어집
- **Offset pagination**: `LIMIT/OFFSET`으로 "몇 번째부터 N개". 기존 방식.
- **Keyset pagination (= Seek method)**: `WHERE 정렬컬럼 > 마지막값`으로 다음 페이지. no offset.
- **Cursor pagination**: keyset 값을 토큰(cursor)으로 감싼 API 형태. 실무에선 keyset과 거의 동의어.
- **cursor(커서)**: 마지막 항목 위치를 인코딩한 불투명 토큰. 다음 요청의 시작점.
- **index seek**: 인덱스로 특정 값 위치를 바로 찾아가는 것 (스캔의 반대).
- **tie-breaker**: 정렬값이 같을 때 순서를 확정하는 보조 컬럼(보통 PK id).

---

## 8. 다음에 빠른 재현 체크리스트
1. 기존=Offset(`LIMIT/OFFSET`), no offset=Keyset/Cursor(`WHERE id > ?`).
2. Offset은 건너뛸 행을 다 읽어 버림 → 깊은 페이지 느림 + 데이터 변하면 밀림.
3. Keyset은 인덱스로 바로 seek → 깊이 무관 빠름 + 정확. **단 정렬 컬럼 인덱스 필수.**
4. 임의 페이지 점프·총 페이지 수 필요 → Offset. 무한스크롤·대용량·동적 → Keyset.
5. 복합 정렬이면 cursor에 tie-breaker(id) 포함, `(created_at, id) > (?, ?)`.
6. 대략 10만 행 넘어가면 Keyset 전환 고려.

---

## 9. 참고 링크
- [Keyset Cursors, Not Offsets, for Postgres Pagination — Sequin](https://blog.sequinstream.com/keyset-cursors-not-offsets-for-postgres-pagination/)
- [PostgreSQL Keyset vs Offset — Stacksync](https://www.stacksync.com/blog/keyset-cursors-postgres-pagination-fast-accurate-scalable)
- [Pagination: Offset vs Cursor with Benchmarks — 0x.run](https://0x.run/pagination-offset-vs-cursor)
- [API Pagination Guide: Cursor vs Offset vs Keyset — Design Gurus](https://designgurus.substack.com/p/api-pagination-guide-cursor-vs-offset)

---

> [!note] 2026-08-26 수정 — 맨 위 "kbap 실전" 줄을 실측으로 교체했다
> 원래는 **"여기 이론상의 keyset 커서가 실제 백엔드에선 fetch join과 충돌해, '커서로 ID만 페이징 → 그 ID로 join fetch' 2단계로 결합된다(`FoodRepositoryAdapter.findFoodPage`)"** 라고 적혀 있었다. 2026-08-26 레포 실측 결과 **그 문장의 두 근거가 지금은 둘 다 없다** — `join fetch` 는 kbap-server 전체에 0건이고, `FoodRepositoryAdapter` 클래스는 모듈·계층 개편으로 사라졌다. 애초에 kbap 은 JPA 연관관계를 하나도 매핑하지 않아(ArchUnit 이 금지한다) fetch join 을 쓸 일 자체가 없다.
> **잘못 배운 건 아니다.** "페이징과 조인은 같이 쓰면 사고가 나므로 2단계로 나눈다"는 뼈대는 그대로 맞고, 실제 코드도 여전히 2단계다. 틀렸던 건 **2단계째의 정체**(fetch join 이 아니라 id 목록 `IN` 조회)와 **클래스 이름** 둘뿐이다. 이 노트 2~6절의 keyset 원리는 손댈 곳이 없다.
> 실코드 추적 → [[9. 연관관계를 쓰지 않는 설계 — N+1을 애초에 만들지 않기]] §4. (옛 링크 대상이던 `8. LAZY 로딩과 N+1 문제` 는 `kbap 백엔드 (old)/` 로 보관됐다.)

---

> [!note] 2026-08-27 역링크 — 8절 5번(복합 정렬 tie-breaker)의 실물
> 이 노트가 *"복합 정렬이면 cursor 에 tie-breaker(id) 를 포함하라"* 고만 적어둔 자리에, kbap 리뷰 목록 API 의 구현이 있다 → [[22. 리뷰 작성 흐름 — 쓰기 경로와 랭킹 원장]] §7-2. 정렬 5종마다 지표 SQL 조각이 갈리고, 커서 조건이 `(지표 비교) or (지표 동점 and r.id <)` 로 조립된다. 커서 인코딩도 `latest` 만 숫자 하나, 나머지는 `"지표_id"` 를 base64url 로 감싼 불투명 토큰이라 **정렬을 바꾸면 커서를 버려야 한다** — 이 노트에 없던 운영 제약이다.

---

> [!note] 2026-08-27 역링크 — tie-breaker 가 **필요 없는** 쪽의 실물, 그리고 부등호가 뒤집히는 자리
> 위 역링크가 8절 5번의 "복합 정렬 → tie-breaker" 실물이라면, 이번엔 그 반대편이다. kbap 의 주문·커뮤니티 목록은 정렬이 `id` **하나뿐**이라 동점이 생기지 않고, 그래서 커서가 **id 숫자 하나**로 끝난다 → [[24. 주문 흐름 — 스캔 결과가 주문 이력이 되기까지]] §7-1, [[25. 커뮤니티 흐름 — 포스팅·댓글·커서 페이징]] §4-1. "언제 복합 커서가 필요한가" 의 답은 **정렬 키에 동점이 생기느냐** 하나다.
> 이 노트에 없던 운영 디테일 셋 — ① **정렬 방향이 반대면 커서 부등호도 뒤집힌다**(최신순 `id < :cursor` vs 등록순 `id > :cursor`, 25 §4-1). ② `size` 는 400 으로 거절하지 않고 `coerceIn(1, 30)` 으로 **조용히 클램프**한다(24 §7-1). ③ 커서 페이징이라 `totalCount` 가 응답에 없다는 성질이 FE 계약으로 그대로 새어 나간다(24 §9-2).

---

> [!note] 2026-08-27 역링크 — 커서를 쓰는 이유가 **성능이 아니라 정합성**인 자리
> 앞의 두 역링크는 전부 목록 API(사람이 보는 화면) 이야기였다. 커서 페이징은 화면이 없는 곳에서도 쓰인다 → [[27. 아웃박스와 SQS — 다른 시스템에 일을 넘기는 법]] §7, [[28. 임베딩과 벡터 검색 — 뜻으로 음식 찾기]] §6-2. 두 배치 잡의 Reader 가 `WHERE id > :afterId AND outbox_status = 'PENDING' ORDER BY id ASC LIMIT :limit` 로 아웃박스 행을 100건씩 읽는다.
> **여기선 OFFSET 을 피하는 이유가 다르다.** 이 노트 3절이 든 이유는 "OFFSET N 은 앞의 N 행을 세어 버리느라 뒤로 갈수록 느리다" 였다. 배치에서는 그보다 **결과 집합이 도는 중에 움직인다**는 게 문제다 — 처리하면서 상태가 `PENDING → COMPLETE` 로 바뀌어 집합이 줄고, 동시에 새 `PENDING` 이 INSERT 되어 늘어난다. OFFSET 이면 그때마다 항목이 건너뛰어지거나 중복된다. **정지된 목록의 성능 문제가 아니라 움직이는 목록의 정확성 문제**다.
> 파생 성질 하나 — 커서라서 **실패한 행은 그 실행에서 다시 안 본다.** 커서가 이미 지나갔기 때문이다. 그래서 재시도는 "이번 실행 안" 이 아니라 **다음 주기**(한 시간 뒤 커서 0부터)에 일어난다([[27. 아웃박스와 SQS — 다른 시스템에 일을 넘기는 법]] §7 의 5번).
> 인덱스도 같은 모양이다 — `(outbox_status, id)`. **조건 컬럼이 앞, 정렬·커서 컬럼이 뒤**. 이 노트 6절의 keyset 인덱스 규칙 그대로다.
