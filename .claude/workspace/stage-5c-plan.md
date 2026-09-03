# 5단계 셋째 묶음(5c) 계획서 · 실코드 흐름 — 홈 화면 조회(kb-111)

- **대상 독자**: FE(React Native/Expo)만 해온 개발자. 1~4단계 개념 노트 + 5a(온보딩·스캔)·5b(JWT) 이수. **5단계 규칙: 새 개념 설명 금지 — 기존 노트로 `[[번호. 링크]]`.**
- **담당**: `code-flow-writer` (1개 노트).
- **노트 골격**(code-flow-writer 정의): ① 이 API가 하는 일(사용자 관점 한 줄) ② 흐름 표(계층 | 파일:줄 | 하는 일) ③ 계층 경계마다 "여기서 모듈이 바뀌는 이유" ④ 요청/응답 JSON ⑤ 관련 테스트가 보장하는 것 ⑥ spec·ADR 링크.
- **제약**: kbap-server 읽기 전용. 모든 단계에 `경로:줄` 근거. 상상 금지. 완성 노트는 iCloud 볼트 `kbap 백엔드/`에만.
- **번호 규칙(신규)**: 파일명·H1 앞에 공부 순서 번호. 기존 16노트는 `1.`~`16.` 리네임 완료. **이번 새 노트는 `17.`.**
- **근거 파일 전부 실재 확인 완료**(2026-07-15). spec `specs/kb-111-home-screen/`(spec·plan·research·data-model·quickstart·contracts) 실존 확인.

## 묶음 구성 근거 (한 줄)
**홈 조회(kb-111) 단독 1노트.** 홈은 세 섹션을 모으지만 실체는 `HomeQueryUseCase.getHome` **단일 `@Transactional(readOnly=true)` 메서드의 fan-in read 하나**라(쓰기 경계 없음, 계층 경유 1회) 2노트로 쪼갤 접합선이 없다. 앞 흐름들과의 대비축(쓰기 vs 읽기, 단일 도메인 vs 다중 도메인 fan-in, `@AuthMemberId` vs `@AuthMemberIdOrNull`)이 한 노트 안에서 가장 선명하다.

---

> [!warning] 실측 발견 — 비회원 개인화 섹션은 `null`이 아니라 **빈 배열** (스펙·Swagger @Schema와 다름)
> spec US2 AS1("기피 성분과 최근 스캔은 **null** 로")과 `HomeResponse.kt:11-12`의 `@Schema` doc("false 면 개인화 섹션이 **null**")은 **코드와 어긋난다**. 실제 코드는 `HomeResponse`의 `avoidedSubstances`·`recentScans`가 **non-nullable `List<...>`**이고(`HomeResponse.kt:17,21`), UseCase가 `.orEmpty()`로 항상 리스트를 채운다(`HomeQueryUseCase.kt:40`) → 비회원은 **빈 배열**을 받는다. 같은 레포 안 `HomeApi.kt:26` Swagger 설명은 오히려 코드와 일치("모든 배열 필드는 null 을 내려주지 않는다 — 값이 없으면 빈 배열"). 즉 **문서 두 곳이 서로 모순**이고 코드는 빈 배열 편이다. → 노트는 **코드 기준**으로 "비회원 = 빈 배열 + `authenticated=false`로 구분"이라 쓰고 `> [!warning] 스펙과 다름`으로 이 불일치를 짚는다(클라이언트가 null 분기하면 안 됨).

> [!note] 앞 흐름과의 대비축 (노트의 뼈대로 삼을 것)
> - **읽기 전용 트랜잭션**: 5a·5b는 쓰기(`@Transactional`)였는데 홈은 `@Transactional(readOnly=true)`(`HomeQueryUseCase.kt:24`). "왜 읽기에도 트랜잭션 경계를 여나 / readOnly가 무엇을 바꾸나"를 `[[6. 설정과 프로필, 트랜잭션 첫걸음]]`으로 링크해 한 줄.
> - **선택 인증**: `@AuthMemberId`(필수, note 16)와 달리 `@AuthMemberIdOrNull`(`HomeController.kt:17`) — 헤더 없으면 `null`(비회원), 있는데 위조·만료면 여전히 401(`AuthMemberIdOrNullArgumentResolver.kt:22-26`). note 16의 형제 리졸버.
> - **다중 도메인 fan-in**: member·food·scan·avoidance 4개 port를 한 UseCase가 조합. 앞 흐름은 대체로 1~2 도메인이었다.

---

## 노트 17 — 홈 화면 조회 흐름 — 세 섹션을 한 응답으로 모으기

- **파일명(=제목)**: `17. 홈 화면 조회 흐름 — 세 섹션을 한 응답으로 모으기.md`
- **H1**: 파일명과 동일(`# 17. 홈 화면 조회 흐름 — 세 섹션을 한 응답으로 모으기`)
- **담당**: code-flow-writer
- **API**: `GET /api/v1/home` (`HomeApi.kt:48`, `HomeController.kt:12`)
- **한 줄**: 홈에 진입하면 한 번의 요청으로 세 섹션 — 내 기피 성분·인기 음식 5개·내 최근 스캔 10개 — 을 프로필 언어로 함께 받는다. 로그인 안 했으면 인기 음식만(영어), 개인화 섹션은 빈 배열.

**흐름 (경유 파일:줄, 실재 확인)**
| 계층 | 파일 | 하는 일 |
|---|---|---|
| Controller | `app/api/.../home/HomeController.kt:16-21` | `home(@AuthMemberIdOrNull memberId: Long?)` → `homeQueryUseCase.getHome(memberId)` → `HomeResponse.from(result, authenticated = memberId != null)` |
| 선택 인증 리졸버 | `app/api/.../common/auth/AuthMemberIdOrNull.kt`, `AuthMemberIdOrNullArgumentResolver.kt:14-27` | 헤더 없음→`null`(비회원), 있으면 `tokenParser.parseAccessToken`으로 memberId. 위조·만료는 예외(401). note 16 필터의 형제 |
| 응답 DTO | `app/api/.../home/HomeResponse.kt:9-42` | `HomeResult`→web. `authenticated` 플래그 + 세 리스트(모두 non-null, 값 없으면 빈 배열). `AvoidedSubstanceResponse`(code+지역화명), `popularFoods`·`recentScans`는 `FoodSummaryResponse`(메뉴 목록·검색과 동일 형태) |
| UseCase | `application/client/.../home/HomeQueryUseCase.kt:24-42` | `@Transactional(readOnly=true) getHome` — ①회원·언어 결정(`member?.profile?.appLanguage ?: EN`) ②기피코드 조회 ③기피 성분명 지역화 ④인기 음식 랜덤 5개 ⑤최근 스캔 10개(id 조회→음식 batch 조회→id 순서 복원). `POPULAR_SIZE=5`, `RECENT_SCAN_SIZE=10` |
| 결과 DTO | `application/client/.../home/dto/HomeResult.kt`, `AvoidedSubstanceView.kt` | 경계 DTO(도메인↔web 사이) |
| port ①회원 | `core/member/.../MemberRepository.kt`(`findById`) | 회원·프로필(언어·기피 성분 원천 = note 13 온보딩이 저장) |
| port ②기피 provider | `application/client/.../food/usecase/AvoidedSubstanceProvider.kt:6`(`avoidedCodes(memberId: Long?)`) | memberId→기피 성분 코드 집합. 비회원(null)이면 빈 집합 |
| port ③기피 성분 | `core/avoidance/.../AvoidanceSubstanceRepository.kt:4`(`findByCodes`) | 코드→성분(언어별 `displayName`) |
| port ④음식 | `core/food/.../FoodRepository.kt:16,18`(`findRandomReady(size)`, `findAllReadyByIds(ids)`) | 완성(READY) 카탈로그 랜덤 N / id 목록 batch 조회 |
| port ⑤스캔 이력 | `core/scan/.../ScanHistoryRepository.kt:6`(`findRecentReadyFoodIds(memberId, limit)`) | 최신순·중복 제거·READY만 foodId 목록(원천 write = note 14 스캔 흐름의 ScanHistory 저장) |
| 스키마 | `db/migration/`(food·scan_history·member — 이미 note 13·14에서 소개) | 새 테이블 없음(read 경로) |

**개념 노트 링크 맵** (각 구간 → 번호 노트)
- Controller가 UseCase 생성자 주입 / `BaseResponse` → `[[5. 의존성 주입(DI)과 계층 구조 — 코드를 어떻게 나누나]]`, `[[4. IoC 컨테이너와 Bean — 부품을 직접 만들지 않는다]]`
- `@Transactional(readOnly=true)` — 읽기에도 여는 경계, readOnly의 의미 → `[[6. 설정과 프로필, 트랜잭션 첫걸음]]`
- 최근 스캔: `findRecentReadyFoodIds`로 id만 뽑고 `findAllReadyByIds`로 **한 번에 batch 조회** 후 `associateBy`로 매칭(id별 개별 조회 안 함) — N+1 회피 패턴 → `[[8. LAZY 로딩과 N+1 문제]]`
- 도메인↔엔티티 변환·`displayName(lang)` 지역화 → `[[7. 영속성 컨텍스트와 엔티티 매핑]]`
- UseCase가 5개 **port에만** 의존, 구현은 adapter → `[[10. 클린 아키텍처와 ports & adapters — 도메인을 순수하게 지키기]]`
- member·food·scan·avoidance **네 바운디드 컨텍스트의 조합을 application 계층에서만** 수행(도메인끼리 직접 안 엮임) → `[[11. 모듈러 모놀리스와 DDD — 경계를 코드로 지키기]]`
- `@AuthMemberIdOrNull` = note 16 `@AuthMemberId`의 선택 버전 → `[[16. JWT 인증 필터와 세션 생명주기 — 매 요청 검증·재발급·로그아웃]]`
- 기피 성분·앱 언어의 저장 원천(프로필) → `[[13. 온보딩 프로필 저장 흐름 — 요청이 DB에 닿기까지]]`
- 최근 스캔 데이터의 write 원천(ScanHistory) → `[[14. 메뉴판 스캔 판정 흐름 — 메뉴명이 위험도가 되기까지]]`
- 테스트·spec → `[[12. SDD와 TDD — 스펙과 테스트로 개발하기]]`

**forward/보류 (본문 말미 한 줄씩)**
- 인기 음식은 지금 `findRandomReady`(무작위) — 인기도 지표가 쌓이기 전 임시. 응답 형태(`FoodSummaryResponse`)는 메뉴 목록·검색과 동일해 정렬만 바뀌면 클라이언트 무수정(spec 근거).
- ponytail: **페이징/커서 노트는 링크하지 않음** — 홈은 5개·10개 고정 크기 fan-in이라 커서 페이징이 없다(kb-63 메뉴 커서와 무관). 접점 없음.

**관련 테스트가 보장하는 것** (⑤ 섹션 근거)
- `application/client/.../home/HomeQueryUseCaseTest.kt`(세 섹션 조합·언어·기피 없음/스캔 없음 빈 배열·중복 제거·개수 제한), `app/api/.../home/HomeControllerTest.kt`(회원 200), `HomeGuestTest.kt`(**비회원 = 인기 음식만·개인화 빈 배열·`authenticated=false`, 위조 토큰 401** — 위 실측 발견의 코드 근거), `HomeTestSeed.kt`(픽스처).

**근거 spec/ADR**: `specs/kb-111-home-screen/`(spec US1·US2·US3, contracts, data-model). US3(스캔 이력 기록)은 note 14가 이미 다룬 write 쪽 — 홈 노트는 read만, US3는 note 14로 링크. 아키텍처(fan-in read·계층) → ADR-0001, `docs/architecture/use-case-flows.md`.

---

## 작성자 공통 지침 (code-flow-writer가 따를 것)
- `kbap-repo-map`으로 탐색 시작, `study-note-style` 규약 준수(**번호 접두 규칙 포함**). **모든 단계에 `경로:줄` 근거**, 상상 금지.
- **새 개념 설명 금지** — 위 링크 맵대로 번호 노트로 `[[링크]]`. 링크할 노트 없는 새 개념이 나오면 짧은 콜아웃 + 응답에 "개념 노트 후보" 보고(현재로선 없음 — 전부 기존 노트로 커버됨).
- 비회원 개인화 섹션은 **코드 기준 빈 배열**로 쓰고 `> [!warning] 스펙과 다름`(위 실측 발견). spec US2·HomeResponse @Schema의 "null"은 낡음, HomeApi 설명은 코드와 일치.
- 발췌는 15줄 이내, 주석 금지 레포이므로 발췌 아래 줄 단위 풀이(`HomeApi.kt`·`@Schema`의 문서화 주석은 인용 가능 — 실제 API 문서라).
- frontmatter: `tags: [kbap-backend, 흐름]`, `생성일: 2026-07-15`, `상태: 완료`.
- 파일명·H1 모두 `17. ` 접두.

## vault-keeper 지시 (노트 완성 후 vault-bookkeeping)
1. **복리 (양방향)** — 홈은 앞 세 흐름의 데이터를 소비하는 read라 접점이 많다. 노트 17 ↔ 각 노트 하단에 역링크 append:
   - `[[13. 온보딩 프로필 저장 흐름 …]]` ↔ 17 ("여기서 저장한 기피 성분·앱 언어가 홈에서 읽힌다").
   - `[[14. 메뉴판 스캔 판정 흐름 …]]` ↔ 17 ("여기서 저장한 ScanHistory가 홈 '최근 스캔'의 원천이다" — US3 write↔read 짝).
   - `[[16. JWT 인증 필터와 세션 생명주기 …]]` ↔ 17 ("`@AuthMemberId`의 선택 버전 `@AuthMemberIdOrNull`이 홈에서 쓰인다").
   기존 본문은 건드리지 말고 역링크 한 줄만 append.
2. **페이징 노트 복리는 하지 않음** — 홈은 커서 페이징이 없어 접점 없음(위 forward/보류). team-lead가 후보로 언급했으나 코드 실측상 부적합.
3. **지도 갱신**: `🗺️ kbap 백엔드 공부 지도.md` — ③ 홈 화면 체크박스 `[x]`, 링크를 `[[17. 홈 화면 조회 흐름 — 세 섹션을 한 응답으로 모으기]]`로, 진도 요약을 "완료 노트 17 / 다음 묶음: 회원 랭킹(kb-123)"으로. 지도는 새로 만들지 말고 체크·정정만.
4. 이전 개념 노트 역링크·홈(볼트 홈 노트) 등록·작업 로그·린트는 표준 절차대로. 새 노트 파일명 번호(`17.`)가 study-note-style 규칙과 맞는지 린트에서 확인.
