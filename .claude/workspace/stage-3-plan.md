# 3단계 계획서 · JPA·영속성 — 객체를 DB에 저장하기

- **대상 독자**: FE(React Native/Expo)만 해온 개발자. 1단계(서버·스레드·JVM)·2단계(IoC/DI·계층·설정/트랜잭션 첫걸음) 이수.
- **단계 목표**: 2단계 맨 아래 계층(영속)이 DB와 어떻게 대화하는지. JPA가 객체↔테이블을 잇는 원리(영속성 컨텍스트), 로딩 전략(LAZY·N+1·fetch join), 스키마 버전 관리(Flyway)를 잡는다. 4단계(아키텍처) 이전 필수.
- **노트 3개** (공부 지도 예정 3개 유지 — 실측 결과 조정 불필요. 매핑/로딩/마이그레이션은 독립 덩어리라 3분할이 맞다). 담당 전부 `concept-writer`.
- **제약**: kbap-server 읽기 전용. dev 백업에 노트 쓰지 않음. 완성 노트는 iCloud 볼트 `study/kbap 백엔드/`에만.
- **근거 파일 전부 실재 확인 완료**(2026-07-14).

> [!warning] 낡은 문서 주의 — 테스트 DB
> CLAUDE.md는 "테스트=H2 create-drop, flyway off"라 하지만 **실측은 다르다**: MySQL Testcontainers(`@ServiceConnection`, KB-46) + `flyway.enabled: true` + `ddl-auto: validate`다. 노트가 테스트 DB를 언급하면 **실측 기준**으로 쓰고, CLAUDE.md가 낡았다는 건 노트 본문에 굳이 적지 말 것(독자 혼란). 근거: `app/api/src/test/resources/application.yml`(주석에 KB-46 명시), `infra/persistence/src/testFixtures/kotlin/com/meogo/infra/persistence/testsupport/MySqlContainerConfig.kt`.

**back-link 대상 (파일명 정확 확인 완료, 2026-07-14)**
- 2단계: `설정과 프로필, 트랜잭션 첫걸음`(트랜잭션·flush·전파·롤백을 3단계로 예고 → 노트 1이 이어받음)
- 2단계: `의존성 주입(DI)과 계층 구조 — 코드를 어떻게 나누나`(영속 계층·port/adapter)
- 1단계: `JVM 위의 Kotlin — 컴파일·타입·널 안전이 TypeScript와 다른 점`(엔티티가 Kotlin 클래스라는 점)
- 기존 볼트: `페이징 — Offset vs No-Offset(Keyset·Cursor)`(`백엔드/` 폴더, 실재 확인) — 노트 2와 **양방향** 연결

---

## 노트 1 — 영속성 컨텍스트와 엔티티 매핑

- **파일명(=제목)**: `영속성 컨텍스트와 엔티티 매핑.md`
- **담당**: concept-writer
- **한 줄 의도**: 2단계에서 "UseCase가 트랜잭션 경계"라 했다. 그 경계 안에서 JPA가 **객체를 DB 행으로, DB 행을 객체로** 바꾸고(엔티티 매핑), 그 사이를 **영속성 컨텍스트**(1차 캐시)가 중개하는 원리를 잡는다.
- **소관 고정 규약**: BaseEntity(`@MappedSuperclass`, id·status·createdAt·updatedAt 공통 + 소프트삭제 `@SQLRestriction`), 엔티티 안 `toDomain()`/`from()` 변환, MySQL 고정 컬럼 정의(`@Column(length=N)`).

**목차 초안**
1. 한 줄 요약
2. 개념표 (엔티티 / 영속성 컨텍스트 / 매핑 / 도메인↔엔티티 변환)
3. 엔티티 매핑이란 — Kotlin 클래스 `@Entity`가 테이블 한 개, 프로퍼티가 컬럼. FE의 "API 응답 타입"과 대비하되 이건 DB 스키마와 1:1
4. 영속성 컨텍스트 — 트랜잭션 안에서 조회한 엔티티를 담아두는 1차 캐시. **React Query 캐시와 대비**("어디까지 같고 어디서 다른지" 한 줄: RQ는 컴포넌트 전역·키 기반, 영속성 컨텍스트는 트랜잭션 1개 수명·변경 감지로 자동 flush). 2단계 트랜잭션 예고를 여기서 이어받는다
5. 변경 감지와 flush — 엔티티 필드만 바꾸면 트랜잭션 끝에 UPDATE가 자동 나감(save 호출 없이도). "왜 명시적 save가 안 보이나"
6. BaseEntity 규약 — 모든 엔티티가 상속, id·생성/수정시각·소프트삭제(status=DELETED, 조회는 `@SQLRestriction`으로 ACTIVE만 자동)를 공통 제공. "왜 엔티티에 id를 안 쓰나"
7. 도메인 ↔ 엔티티 변환 — kbap은 도메인 객체(순수 Kotlin)와 JPA 엔티티를 분리하고, 변환은 엔티티 안 `toDomain()`/`companion object.from()`에 둔다. 도메인은 JPA를 모른다(2단계 계층/4단계 ports&adapters와 연결)
8. 실코드 — `BaseEntity` 발췌, `FoodJpaEntity`의 `toDomain()`/`from()` 발췌
9. 치트시트 / 용어집 / 체크리스트

**근거 kbap-server 파일**
- `infra/persistence/src/main/kotlin/com/meogo/infra/persistence/BaseEntity.kt` — `@MappedSuperclass` + `@SQLRestriction("status = 'ACTIVE'")` + id/status/createdAt/updatedAt (실물, 발췌 15줄)
- `infra/persistence/src/main/kotlin/com/meogo/infra/persistence/food/FoodJpaEntity.kt:26,62,80-81` — `class FoodJpaEntity`, `fun toDomain()`, `companion object { fun from(food) }`
- `CLAUDE.md` 컨벤션 — "JPA 엔티티 작성"(BaseEntity 상속·MySQL 컬럼 길이 명시)·"도메인↔JPA 변환·불변"

**연결**: back-link → `[[설정과 프로필, 트랜잭션 첫걸음]]`(트랜잭션 경계 안에서 영속성 컨텍스트가 동작), `[[의존성 주입(DI)과 계층 구조 — 코드를 어떻게 나누나]]`(영속 계층). 이후 → `[[LAZY 로딩과 N+1 문제]]`. 4단계 예고 → "도메인이 JPA를 모르게 두는 이유(ports & adapters)".

---

## 노트 2 — LAZY 로딩과 N+1 문제

- **파일명(=제목)**: `LAZY 로딩과 N+1 문제.md`
- **담당**: concept-writer
- **한 줄 의도**: 연관된 데이터(음식 → 기피 성분들)를 언제 가져올지가 성능을 가른다. kbap이 **모든 연관을 LAZY로 고정**하고 필요할 때 **fetch join**으로 명시 로딩하는 이유 — 백엔드 성능 버그 1위인 N+1을 막는 규약을 잡는다.
- **소관 고정 규약**: 전 연관관계 `FetchType.LAZY` 고정 + fetch join 명시 조회.

**목차 초안**
1. 한 줄 요약
2. 개념표 (연관관계 / LAZY vs EAGER / N+1 / fetch join)
3. 연관관계 — 엔티티가 다른 엔티티를 참조(`@OneToMany` 등). 1번 노트의 매핑을 관계로 확장
4. LAZY vs EAGER — 참조를 실제 접근할 때 가져오나(LAZY) 조회 즉시 다 가져오나(EAGER). kbap은 전부 LAZY 고정
5. N+1 문제 — 목록 N건을 돌며 각각의 연관을 따로 조회하면 쿼리가 1+N번. **왜 LAZY만으론 안 되고 fetch join이 필요한가**(핵심)
6. fetch join — `@Query("... left join fetch ...")`로 한 방에 필요한 연관까지. EAGER로 해결 안 하는 이유(불필요 로딩·`LazyInitializationException`)
7. **fetch join과 페이징의 충돌** — 컬렉션을 join fetch하면서 LIMIT을 걸 수 없다. kbap은 **"커서로 ID만 페이징 → 그 ID들로 join fetch"** 2단계로 푼다. 이게 keyset/커서 페이지네이션과 만나는 지점 → `[[페이징 — Offset vs No-Offset(Keyset·Cursor)]]` 참고
8. 실코드 — `FoodJpaEntity`의 `@OneToMany(..., fetch = FetchType.LAZY)`, `FoodJpaRepository`의 `left join fetch`, `FoodRepositoryAdapter.findFoodPage`(ID 페이징 후 join fetch 2단계)
9. 주의점 / 치트시트 / 용어집 / 체크리스트

**근거 kbap-server 파일**
- `infra/persistence/src/main/kotlin/com/meogo/infra/persistence/food/FoodJpaEntity.kt:58` — `@OneToMany(cascade=..., fetch = FetchType.LAZY, orphanRemoval=true)`
- `infra/persistence/src/main/kotlin/com/meogo/infra/persistence/food/FoodJpaRepository.kt:15,25,44` — `left join fetch f.foodAvoidanceSubstances`; `:39` `findFoodPageIds(cursor, pageable)`; `:34,67` `where (:cursor is null or f.id < :cursor)`
- `infra/persistence/src/main/kotlin/com/meogo/infra/persistence/food/FoodRepositoryAdapter.kt:38-45` — `findFoodPage(cursor, size)` = ID 커서 페이징 → ID로 재조회(fetch join+페이징 2단계 실물)
- `specs/kb-63-menu-list-cursor/spec.md` — 커서 페이지네이션 근거(무한 스크롤 메뉴 목록)
- `CLAUDE.md` 컨벤션 — "JPA 연관관계 로딩"(전부 LAZY + fetch join)

**연결**: back-link → `[[영속성 컨텍스트와 엔티티 매핑]]`. **양방향** → `[[페이징 — Offset vs No-Offset(Keyset·Cursor)]]`(그 노트에도 이 노트로 역링크 추가: "kbap 실코드에서 keyset 커서가 fetch join과 어떻게 결합되는지"). 이후 → `[[Flyway 마이그레이션 — 스키마를 코드로 관리하기]]`.

> [!note] 페이징 노트 역링크 이행 (vault-bookkeeping 시)
> 2단계 린트에서 예약한 복리다. `백엔드/페이징 — Offset vs No-Offset(Keyset·Cursor).md` 하단(예: "관련 노트"/"이어보기")에 `[[LAZY 로딩과 N+1 문제]]`로 한 줄 역링크를 추가한다 — "이론상 keyset 페이징이 kbap에선 fetch join과 2단계로 결합된다(`FoodRepositoryAdapter.findFoodPage`)". 기존 본문은 건드리지 말고 링크 한 줄만 append.

---

## 노트 3 — Flyway 마이그레이션 — 스키마를 코드로 관리하기

- **파일명(=제목)**: `Flyway 마이그레이션 — 스키마를 코드로 관리하기.md`
- **담당**: concept-writer
- **한 줄 의도**: 테이블을 손으로 만들지 않고 **버전 매긴 SQL 파일**로 관리한다. FE에 없던 개념 — DB 스키마도 코드처럼 버전 관리하고, 팀·환경이 같은 스키마를 재현하는 원리를 잡는다.
- **소관 고정 규약**: Flyway timestamp 버전(`Vyyyy.MM.dd.HH.mm.ss__desc.sql`) + out-of-order, MySQL 기준 컬럼(엔티티와 일치), 테스트도 같은 마이그레이션 적용(Testcontainers).

**목차 초안**
1. 한 줄 요약
2. 개념표 (마이그레이션 / 버전 / 스키마 owner / out-of-order)
3. 마이그레이션이란 — 스키마 변경을 순서 있는 SQL 파일로 쌓기. 한 번 적용된 파일은 다시 안 돌린다(적용 이력 관리). 앱 코드 배포와 DB 스키마를 함께 굴리는 법
4. kbap 버전 규칙 — 정수(V1,V2) 대신 **timestamp 버전**을 쓰는 이유: 병렬 브랜치의 버전번호 머지 충돌 제거. `out-of-order=true` 전제로 "다른 마이그레이션 순서에 의존하지 않게" 독립 작성
5. 스키마 owner — `app:api`만 Flyway로 스키마를 만들고(단일 소스), 배치는 flyway off. 1단계 "실행 파일 2개"와 연결
6. 엔티티와의 일치 — 컬럼 길이·타입은 MySQL 기준, 엔티티 `@Column`과 마이그레이션 SQL이 일치해야 함(1번 노트 매핑과 연결). 테스트가 `ddl-auto: validate`로 이 정합을 검증
7. 테스트에서도 같은 마이그레이션 — 테스트는 MySQL Testcontainers에 **운영과 동일한 마이그레이션**을 돌려 SQL 자체를 검증(실측 기준). "테스트가 진짜 DB를 쓴다"
8. 주의점 — 이미 적용된 마이그레이션 파일은 수정·리네임 금지(checksum 파손), 신규는 정수 버전 금지
9. 치트시트 / 용어집 / 체크리스트

**근거 kbap-server 파일**
- `app/api/src/main/resources/db/migration/` — 실제 마이그레이션 목록(`V2026.06.29.20.35.02__create_scan_tables.sql` 등 timestamp 버전 실물)
- `app/api/src/main/resources/application.yml` — `spring.flyway`(+ `out-of-order` 근거는 CLAUDE.md)
- `app/api/src/test/resources/application.yml` — 테스트도 `flyway.enabled: true` + `ddl-auto: validate`(Testcontainers, KB-46) *실측 기준*
- `infra/persistence/src/testFixtures/kotlin/com/meogo/infra/persistence/testsupport/MySqlContainerConfig.kt` — Testcontainers 설정 실물
- `CLAUDE.md` 컨벤션 — "Flyway 마이그레이션 버전 규칙"(timestamp·out-of-order·금지 사례)
- `specs/kb-44-flyway-timestamp-versioning/spec.md`, `specs/kb-46-mysql-testcontainers/spec.md`

**연결**: back-link → `[[영속성 컨텍스트와 엔티티 매핑]]`(엔티티↔스키마 일치), `[[서버는 켜두는 프로그램 — 서버 프로세스와 두 개의 실행 파일]]`(스키마 owner=api, batch flyway off). 4단계 예고 → "SDD/TDD에서 마이그레이션도 스펙·테스트로 검증".

---

## 작성자 공통 지침 (concept-writer가 따를 것)
- `study-note-style` 스킬 규약 준수: FE 비유마다 "어디까지 같고 어디서 다른지" 한 줄, 새 용어 첫 등장 시 정의(이전 단계에 있으면 `[[링크]]`), 실코드 인용은 `경로:줄` + 발췌 15줄 이내.
- kbap-server는 **Kotlin 주석 금지 레포** — 발췌에 주석 달지 말고 발췌 아래 줄 단위 풀이.
- **테스트 DB는 실측(MySQL Testcontainers) 기준** — CLAUDE.md의 "H2" 서술을 따르지 말 것. 낡았다는 메타설명은 노트에 쓰지 않는다(그냥 맞는 내용만).
- **트랜잭션 심화는 이 단계 소관** — 노트 1이 2단계 "첫걸음"을 이어받아 영속성 컨텍스트·flush까지. (전파/격리수준은 필요 없으면 굳이 넣지 말 것 — 독자 눈높이 우선.)
- **깨진 링크 금지** — `[[ ]]`는 위 확인된 파일명과 정확히 일치할 때만.
- frontmatter: `tags: [kbap-backend, jpa]`, `생성일: 2026-07-14`, `상태: 완료`.
- 노트 완성 후 `vault-bookkeeping` 절차 수행 — 특히 **페이징 노트 역링크 이행**(노트 2 아래 [!note] 참조)·이전 단계 노트 역링크·홈 등록·작업 로그·린트.
</content>
