# 확장 묶음 계획서 · 개념 노트 — 캐싱 & Redis (지도 "이후 확장" ①)

- **대상 독자**: FE(React Native/Expo)만 해온 개발자. 커리큘럼 1~20 완주. **본편 뒤 확장 노트** — 5단계 "새 개념 금지" 규칙 밖(개념 노트라 새 개념을 처음부터 설명해도 됨).
- **담당**: `concept-writer` (kbap 실코드 인용 포함).
- **제약**: kbap-server 읽기 전용. 실코드 인용은 `경로:줄`. 상상 금지. 완성 노트는 iCloud 볼트 `kbap 백엔드/`에만. **생성일 2026-07-16.**
- **번호**: 기존 20노트(`1.`~`20.`). **이번 새 노트 `21.`**(파일명=H1).
- **근거 파일 전부 실재 확인 완료**(2026-07-16).

## 묶음 구성 근거 (한 줄)
**1노트.** "왜 캐시하나(개념) → 키-값 저장소 → Redis 성질 → kbap 실사용"은 **끊기지 않는 하나의 사다리**다. 쪼개면 개념 노트가 kbap 앵커 없이 뜨고 React Query 다리·저장소 vs 캐시 대비가 두 곳에 중복된다. kbap의 실사용도 어댑터 하나뿐이라 2노트는 과설계.

---

> [!important] 실측 전수 확인 — kbap은 Redis를 "캐시"가 아니라 refresh 세션 저장소로만 쓴다
> 레포 전수(`grep redis|@Cacheable|@EnableCaching|Caffeine|cache`) 결과: Redis 사용처는 **`RefreshTokenRedisAdapter` 단 하나**(refresh 토큰 세션). `@Cacheable`·`@EnableCaching`·캐시 추상화·Caffeine **전혀 없음**. → 노트는 **정직하게** "kbap은 아직 캐시 용도로 Redis를 쓰지 않는다. 지금 쓰임은 TTL 세션 저장소다"라고 쓴다. 이게 오히려 좋은 교보재: **캐시와 세션 저장은 같은 도구(인메모리·TTL·키-값)를 다른 목적에 쓴 사촌**임을 보여준다. "캐시 노트인데 kbap 예시가 캐시가 아님"을 약점이 아니라 **핵심 통찰**로 서술.

---

## 노트 21 — 캐싱과 키-값 저장소 — 왜 캐시하나부터 kbap의 Redis 세션까지

- **파일명(=제목)**: `21. 캐싱과 키-값 저장소 — 왜 캐시하나부터 kbap의 Redis 세션까지.md` · H1 동일
- **담당**: concept-writer
- **한 줄 취지**: "느리거나 비싼 걸 두 번 안 하려고 결과를 가까이·임시로 둔다"는 캐싱 개념을 FE에서 이미 쓰던 것(React Query 캐시)에서 출발해, 키-값 저장소 → Redis의 성질 → kbap이 그 도구를 (캐시가 아니라) refresh 세션에 쓰는 실사용까지 차근차근 잇는다.

**목차 초안**
1. **FE에서 이미 캐시를 쓰고 있었다** — React Query 캐시(같은 쿼리 재요청 시 서버 안 감), 브라우저 HTTP 캐시. *왜* 캐시하나: 느리거나(네트워크·계산) 비싼(LLM·DB) 걸 반복하지 않으려고. → 캐시의 한 줄 정의.
2. **캐싱의 세 질문** — ① 왜 캐시하나(비용 회피) ② 어디에 두나(프로세스 메모리 / 로컬 디스크 / 별도 분산 서버) ③ **언제 버리나**(무효화·TTL — 캐시의 진짜 어려움은 "낡은 값"). React Query의 `staleTime`/무효화가 좋은 다리.
3. **키-값 저장소란** — `Map`처럼 key→value. 값의 구조·의미는 앱이 정함(문자열·JSON). RDB(테이블·조인·스키마)와 다른 결.
4. **Redis의 성질** — 인메모리(빠름·기본 휘발), **TTL 내장**(키에 수명), 단일 스레드 원자 연산, **프로세스 밖 별도 서버**(여러 앱 인스턴스가 공유 = 분산). 왜 이런 걸 캐시/세션에 쓰나 vs MySQL.
5. **kbap 실사용 (정직하게)** — kbap은 Redis를 **캐시가 아니라 refresh 세션 저장소**로 쓴다.
   - `RefreshTokenRedisAdapter`: `opsForValue().set(key, memberId, ttl)`(TTL로 토큰 수명=저장 수명), `getAndDelete`(회전 원자성), `delete`(로그아웃 폐기).
   - 설정: `application-*.yml`의 `spring.data.redis`, `docker-compose.yml`의 `redis:8` 서비스, 의존성 `infra/persistence/build.gradle.kts`.
   - 테스트: `RedisContainerConfig`(Testcontainers `redis:8`).
   - **`@Cacheable`·캐시 추상화는 없음** — 실측 명시.
6. **그럼 왜 "캐싱" 노트에 Redis 세션이?** — 캐시와 세션 저장은 **같은 도구(인메모리·TTL·키-값)를 다른 목적**에 쓴 사촌. refresh 세션은 "임시(TTL)·빠른 조회·프로세스 밖 공유"라 캐시와 같은 성질을 요구한다. + **이미 배운 캐시**: JPA 영속성 컨텍스트 = 트랜잭션 범위 **1차 캐시**(note 7) — 캐시는 낯선 개념이 아니라 이미 지나온 것.
7. **kbap이 언제 진짜 캐시로 확장할까** (forward, 강제 아님) — 예: 홈 인기 음식·카탈로그 같은 "자주 읽고 잘 안 변하는" read. 지금은 매번 DB. 캐시를 넣는다면 무효화 전략이 숙제.

**kbap 실코드 인용 (concept-writer가 `경로:줄`로)**
- `infra/persistence/.../auth/RefreshTokenRedisAdapter.kt:12-23` (set+TTL / getAndDelete / delete / key prefix)
- `app/api/src/main/resources/application-local.yml:15-17` (`spring.data.redis.host/port`), `application.yml:49` (TTL 주석)
- `infra/persistence/build.gradle.kts:21-22` (`spring-boot-starter-data-redis`, "TTL로 토큰 수명=저장 수명" 주석)
- `docker-compose.yml:69-77` (`redis:8` 서비스·healthcheck), `docker-compose.prod.yml:53-61`
- `infra/persistence/src/testFixtures/.../RedisContainerConfig.kt:10-20` (Testcontainers `redis:8`, `@ServiceConnection`)

**개념 노트 링크 맵** (전부 실재 확인)
- Redis가 처음 콜아웃으로 등장한 본체 → `[[15. 소셜 로그인 흐름 — Firebase 토큰이 자체 JWT가 되기까지]]`, `[[16. JWT 인증 필터와 세션 생명주기 — 매 요청 검증·재발급·로그아웃]]` (양방향 — 아래 vault-keeper)
- **이미 배운 캐시**: 영속성 컨텍스트 = 1차 캐시 → `[[7. 영속성 컨텍스트와 엔티티 매핑]]` (핵심 대비)
- `spring.data.redis` 설정·프로필 → `[[6. 설정과 프로필, 트랜잭션 첫걸음]]`
- Redis는 **프로세스 밖 별도 서버**(여러 인스턴스 공유) → `[[1. 서버는 켜두는 프로그램 — 서버 프로세스와 두 개의 실행 파일]]`
- adapter가 `infra:persistence`에, port(`RefreshTokenStore`)는 core에 → `[[11. 모듈러 모놀리스와 DDD — 경계를 코드로 지키기]]`
- **저장소 vs 캐시 대비**(FE): access/refresh 토큰을 FE가 SecureStore에 보관 ↔ BE가 refresh 세션을 Redis에 → `[[클라이언트 저장소 — AsyncStorage·SecureStore·메모리 (K-Bap 실전)]]` (`Expo & React Native/`, 실재 확인)

**연결하지 않는 것 (근거)**
- **React Query "노트"로 링크 안 함** — 볼트에 React Query 전용 노트 없음(실측). RQ는 본문 안 **다리(비유)**로만 서술.
- **페이징 노트(`백엔드/페이징 — Offset vs No-Offset`) 링크 안 함** — 캐싱/Redis와 실질 접점 없음(ponytail: 억지 링크는 노이즈). team-lead 후보였으나 제외.

**근거 spec/문서**: Redis 도입 맥락은 `specs/kb-118-firebase-jwt-auth/`(refresh 세션 저장소 결정, research R5·데이터모델). Testcontainers 동등성 `specs/kb-46-mysql-testcontainers/` 계열.

---

## 작성자 지침 (concept-writer)
- `study-note-style` 준수(**번호 접두 규칙 포함**), `kbap-repo-map`으로 인용 경로 확인. FE 눈높이(캐시=이미 쓰던 것에서 출발).
- **정직 원칙**: kbap은 Redis를 캐시로 안 쓴다(실측). "캐시 아님"을 통찰로 서술(실측 콜아웃대로).
- 인용 발췌 15줄 이내, 주석 금지 레포지만 `build.gradle`·`yml`·docker-compose 주석은 실제 문서라 인용 가능.
- frontmatter: `tags: [kbap-backend, 개념]`(또는 style 규약의 개념 태그), `생성일: 2026-07-16`, `상태: 완료`. 파일명·H1 `21.` 접두.
- 새 외부 개념(캐시·TTL·인메모리)은 공식 문서 리서치로 뒷받침하되 FE 비유 우선.

## vault-keeper 지시 (노트 완성 후 vault-bookkeeping)
1. **복리 (양방향)** — 노트 21 ↔ 각 노트 하단 역링크 append(기존 본문 불변, 한 줄):
   - `[[7. 영속성 컨텍스트와 엔티티 매핑]]` ↔ 21 — "영속성 컨텍스트가 1차 캐시"의 일반 개념이 21.
   - `[[클라이언트 저장소 — AsyncStorage·SecureStore·메모리 (K-Bap 실전)]]`(`Expo & React Native/`) ↔ 21 — 저장소 vs 캐시 대비.
   - **15·16 콜아웃 갱신**: 두 노트의 Redis 콜아웃(TTL·getAndDelete) 줄 옆/아래에 `[[21. 캐싱과 키-값 저장소 — 왜 캐시하나부터 kbap의 Redis 세션까지]]` 링크 **병기**("Redis가 뭔지·왜 이걸 쓰는지는 21"). 콜아웃 본문은 유지, 링크만 추가.
2. **지도 반영** — `🗺️ kbap 백엔드 공부 지도.md`의 "이후 확장" 섹션 **후보 ①**을 체크박스+wikilink로 전환: `- [x] ① [[21. 캐싱과 키-값 저장소 …]]`. 본편(1~20)은 완료 유지, 21은 **본편 뒤 확장**임이 드러나게(섹션 그대로 두고 ①만 노트로 승격). 진도요약에 "확장 노트 1(캐싱·Redis) 추가" 한 줄. 지도 상태(`완료`)는 유지.
3. 볼트 홈 등록·작업 로그(확장 노트 기록)·린트 표준 절차. 파일명 `21.` 번호 규칙 린트 확인.
