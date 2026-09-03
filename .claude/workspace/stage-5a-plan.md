# 5단계 첫 묶음(5a) 계획서 · 실코드 흐름 — 온보딩 · 스캔

- **대상 독자**: FE(React Native/Expo)만 해온 개발자. 1~4단계 개념 노트 이수. **5단계 규칙: 새 개념 설명 금지 — 1~4단계 노트로 `[[링크]]`한다.**
- **담당**: `code-flow-writer` (2개 노트 모두).
- **노트 골격**(code-flow-writer 정의): ① 이 API가 하는 일(사용자 관점 한 줄) ② 흐름 표(계층 | 파일 | 하는 일) ③ 계층 경계마다 "여기서 모듈이 바뀌는 이유"(4단계 클린아키텍처와 연결) ④ 요청/응답 JSON ⑤ 관련 테스트가 보장하는 것 ⑥ spec·ADR 링크.
- **제약**: kbap-server 읽기 전용. 모든 단계에 `경로:줄` 근거. 상상 금지. 완성 노트는 iCloud 볼트 `kbap 백엔드/`에만.
- **근거 파일 전부 실재 확인 완료**(2026-07-14).

> [!warning] 실측 발견 — ScanUseCase의 언어 하드코딩
> `ScanUseCase.assessMenuBoard`는 응답 언어를 `val lang = LanguageCode.KO`로 **하드코딩**하고 있고, 바로 위에 `// TODO: 회원 설정값(MemberProfile.appLanguage)에서 언어를 가져와…` 주석이 있다(주석 금지 레포인데 예외적으로 존재). → 스캔 노트는 "지금은 응답 메뉴명이 한국어 고정, 프로필 언어 반영은 미구현(TODO)"이라고 **실측대로** 쓴다. spec이 다국어를 말해도 코드 기준(`> [!warning] 스펙과 다름`).

---

## 노트 1 — 온보딩 프로필 저장 흐름 — 요청이 DB에 닿기까지

- **파일명(=제목)**: `온보딩 프로필 저장 흐름 — 요청이 DB에 닿기까지.md`
- **담당**: code-flow-writer
- **API**: `POST /api/v1/members/...`(온보딩 완료) — 정확한 서브경로는 `MemberApi` 인터페이스에서 확인해 적는다.
- **한 줄**: 온보딩 화면에서 닉네임·기피 성분·국가·앱 언어를 제출하면, 서버가 검증→도메인 조립→프로필 JSON으로 회원 행을 갱신하고 온보딩 완료로 표시한다.

**흐름 (경유 파일, 실재 확인)**
| 계층 | 파일 | 하는 일 |
|---|---|---|
| Controller | `app/api/src/main/kotlin/com/meogo/app/api/member/MemberController.kt:16-23` | `completeOnboarding(@AuthMemberId memberId, @RequestBody OnboardingRequest)` → `request.toInput(memberId)` → UseCase 호출, `BaseResponse.ok(Unit)` |
| 인증 리졸버 | `app/api/.../common/auth/AuthMemberIdArgumentResolver.kt`, `AuthMemberId.kt` | JWT에서 memberId 추출해 파라미터 주입(4단계 필터/횡단은 5b JWT 흐름 예고) |
| 요청 DTO | `app/api/.../member/OnboardingRequest.kt` | web 입력 → `MemberProfileInput`(경계 DTO)로 변환 |
| UseCase | `application/client/.../member/MemberProfileUseCase.kt:23-34` | `@Transactional completeOnboarding` — 필드별 검증(`validatedNickname/Codes/Country/Language`) → `MemberProfile.of(...)` → `member.updateProfile(profile).completeOnboarding()` → `memberRepository.update(...)` |
| 도메인 | `core/member/.../Member.kt`, `MemberProfile.kt`, `MemberErrorCode.kt` | 순수 Kotlin 불변 모델. `updateProfile`·`completeOnboarding`은 새 인스턴스 반환(4단계 불변) |
| port | `core/member/.../MemberRepository.kt` | UseCase가 의존하는 인터페이스(구현 모름) |
| Adapter | `infra/persistence/.../member/MemberRepositoryAdapter.kt` | port 구현. `MemberJpaEntity.from`/`toDomain` 호출 |
| 엔티티 | `infra/persistence/.../member/MemberJpaEntity.kt`, `MemberProfileJson.kt` | 프로필을 JSON 컬럼으로 매핑, `BaseEntity` 상속 |
| 스키마 | `app/api/src/main/resources/db/migration/`(member 테이블 생성 마이그레이션) | 회원/프로필 컬럼 |

**개념 노트 링크 맵** (각 구간 → 1~4단계 노트)
- Controller가 UseCase를 생성자 주입 / `BaseResponse` → `[[의존성 주입(DI)과 계층 구조 — 코드를 어떻게 나누나]]`, `[[IoC 컨테이너와 Bean — 부품을 직접 만들지 않는다]]`
- `@Transactional` 경계 → `[[설정과 프로필, 트랜잭션 첫걸음]]` + flush는 `[[영속성 컨텍스트와 엔티티 매핑]]`
- UseCase가 `MemberRepository` port에만 의존, 구현은 adapter → `[[클린 아키텍처와 ports & adapters — 도메인을 순수하게 지키기]]`
- 도메인 불변(`updateProfile`이 새 인스턴스) / member 바운디드 컨텍스트 → `[[클린 아키텍처와 ports & adapters — 도메인을 순수하게 지키기]]`, `[[모듈러 모놀리스와 DDD — 경계를 코드로 지키기]]`
- `MemberJpaEntity.from/toDomain`·프로필 JSON → `[[영속성 컨텍스트와 엔티티 매핑]]`
- 테스트·spec 근거 → `[[SDD와 TDD — 스펙과 테스트로 개발하기]]`

**kb-124(PATCH 부분 수정)는 콜아웃 한 줄** — 함께 다루지 않는다. 근거: 부분 수정 로직이 **같은 파일의 바로 옆 메서드** `MemberProfileUseCase.update()`(:36-49)라, 별도 흐름 노트 없이 온보딩 노트 안 `> [!note]`로 대비만 보이면 충분하다(POST=전 필드 필수 조립 vs PATCH=미전송 null은 유지·빈 배열은 해제, 병합 후 단일 `update`). 두 번째 흐름 노트로 승격할 가치 낮음(계층 경유 동일).

**근거 spec/ADR**: `specs/kb-104-onboarding-profile/`(spec·plan·tasks), 콜아웃용 `specs/kb-124-partial-profile-update/`. 도메인 불변·영속 변환 → ADR-0008 + `docs/architecture/meogo-conventions.md` "도메인 객체 불변성 & 영속 변환".

---

## 노트 2 — 메뉴판 스캔 판정 흐름 — 메뉴명이 위험도가 되기까지

- **파일명(=제목)**: `메뉴판 스캔 판정 흐름 — 메뉴명이 위험도가 되기까지.md`
- **담당**: code-flow-writer
- **API**: `POST /api/v1/scans`
- **한 줄**: 메뉴판 사진에서 뽑은 메뉴명 목록을 보내면, 서버가 각 메뉴를 한국 음식으로 매칭하고 회원의 기피 성분에 비춰 4단계 위험도(SAFE/CAUTION/DANGER/UNKNOWN)를 돌려준다.

**흐름 (경유 파일, 실재 확인)**
| 계층 | 파일 | 하는 일 |
|---|---|---|
| Controller | `app/api/.../scan/ScanController.kt:17-23` | `scan(@AuthMemberId, @RequestBody ScanRequest)` → `assessMenuBoard(request.toInput(...))` → `ScanResponse.from(result)` |
| 요청/응답 DTO | `app/api/.../scan/ScanRequest.kt`, `ScanResponse.kt` | rawMenuName·idx·바운딩박스 입력 / 항목별 위험도·매칭 결과 출력 |
| UseCase | `application/client/.../scan/usecase/ScanUseCase.kt:29-60` | `@Transactional assessMenuBoard` — ①정규화 `KoreanMenuNameNormalizer.matchKey` ②정제 `refineMenuNames`(폴백 degraded) ③매칭 `resolveFoods` ④기피코드 조회 ⑤`food.overallRisk(avoidedCodes)` ⑥이력·카운트 기록 |
| 정제 port | `core/kernel/.../scan/ScannedNameInterpreter.kt`, `InterpretedName.kt` | rawMenuName → StandardName(한국어 정규명) 또는 NotFood. 구현은 어댑터(LLM 계열, 5b 배치LLM 흐름 예고) |
| 음식 도메인 | `core/food/.../Food.kt`(`overallRisk`·`displayName`·`isReady`), `FoodRepository.kt`(port) | 음식별 위험도 판정 로직(도메인), 카탈로그 조회 port |
| 기피 provider | `application/client/.../food/usecase/AvoidedSubstanceProvider.kt` | memberId → 회원 기피 성분 코드 |
| 이력/카운트 | `core/scan/.../ScanHistory.kt`·`ScanHistoryRepository.kt`, `MemberRepository.increaseScanCount` | 매칭된 음식만 이력 저장(saveAll), 스캔 1회 원자적 증가(kb-123 랭킹) |
| Adapter/엔티티 | `infra/persistence/.../scan/ScanHistoryRepositoryAdapter.kt`·`ScanHistoryJpaEntity.kt`, `infra/persistence/.../food/Food*` | 도메인↔엔티티 변환, 카탈로그 fetch join |
| 스키마 | `db/migration/V2026.06.29.20.35.02__create_scan_tables.sql`, `…03__create_food_tables.sql` | scan_history·food 테이블 |

**개념 노트 링크 맵**
- Controller 주입/`BaseResponse` → `[[의존성 주입(DI)과 계층 구조 — 코드를 어떻게 나누나]]`
- `@Transactional`(이력·카운트·조회를 한 트랜잭션) → `[[설정과 프로필, 트랜잭션 첫걸음]]`
- UseCase가 `FoodRepository`·`ScannedNameInterpreter`·`ScanHistoryRepository`·`MemberRepository` **port들**에만 의존 → `[[클린 아키텍처와 ports & adapters — 도메인을 순수하게 지키기]]`
- 위험도 판정이 `Food` 도메인 안(`overallRisk`) / food·scan·member 세 컨텍스트 조합은 application에서만 → `[[모듈러 모놀리스와 DDD — 경계를 코드로 지키기]]`
- 카탈로그 조회 fetch join / N+1 → `[[LAZY 로딩과 N+1 문제]]`
- `ScanHistoryJpaEntity` 변환·저장 → `[[영속성 컨텍스트와 엔티티 매핑]]`
- scan/food 테이블 마이그레이션 → `[[Flyway 마이그레이션 — 스키마를 코드로 관리하기]]`
- 테스트(매칭·위험도·메뉴판 1장=1회)·spec → `[[SDD와 TDD — 스펙과 테스트로 개발하기]]`

**FE 복리 연결** (역링크 append는 vault-keeper 몫)
스캔은 **FE recall → BE precision 2단 판단**의 뒷단이다. FE는 OCR로 메뉴판에서 후보 문자열을 뽑고(recall: 박스 그룹핑·라인 분류), 그 rawMenuName 목록이 `POST /api/v1/scans` 입력이 된다. BE는 그걸 정규화→정제(StandardName/NotFood)→카탈로그 매칭으로 **정밀 확정**(precision). 노트의 "① 입력이 어디서 오나" 지점에서 아래 두 노트로 링크:
- `[[OCR 메뉴 스캔 — 박스 그룹핑]]` (`Expo & React Native/`, 실재 확인) — 사진→메뉴명 후보 묶기
- `[[OCR 라인 분류 — 휴리스틱의 한계와 대안]]` (`Expo & React Native/`, 실재 확인) — 후보 정밀화의 FE 한계 → BE 정제가 이어받는 지점

> [!note] FE 노트 역링크 이행 (vault-bookkeeping 시)
> 위 두 FE 노트 하단에 `[[메뉴판 스캔 판정 흐름 — 메뉴명이 위험도가 되기까지]]`로 역링크 한 줄 append("이 recall 결과가 백엔드에서 어떻게 precision 매칭되는지"). 기존 본문은 건드리지 말 것.

**forward 예고** (본문 말미 한 줄씩)
- `ScannedNameInterpreter`(정제) 구현이 LLM 계열 → **5b 배치 LLM 파이프라인**에서 다룸.
- `increaseScanCount` → **회원 랭킹 흐름**(kb-123)에서 다룸.

**근거 spec/ADR**: `specs/001-menu-scan-mock/`(주 spec), 위험도 정책 `specs/kb-9-avoidance-risk-policy/`, 메뉴명 정제 `specs/kb-90-menu-name-refinement/`·`kb-99-always-korean-menu-name/`, foodId 정합 `specs/kb-98-food-detail-by-food-id/`. 아키텍처 → ADR-0008, `docs/architecture/api-request-flows.md`.

---

## 작성자 공통 지침 (code-flow-writer가 따를 것)
- `kbap-repo-map`으로 탐색 시작, `study-note-style` 규약 준수. **모든 단계에 `경로:줄` 근거**, 상상 금지.
- **새 개념 설명 금지** — 위 링크 맵대로 1~4단계 노트로 `[[링크]]`. 링크할 노트가 없는 새 개념이 나오면 짧은 콜아웃 + 응답에 "개념 노트 후보" 보고.
- 실측이 spec과 다르면 코드 기준 + `> [!warning] 스펙과 다름`(특히 스캔 언어 하드코딩).
- 발췌는 15줄 이내, 주석 금지 레포이므로 발췌 아래 줄 단위 풀이(스캔 파일의 예외 TODO 주석은 "미구현 표시"로만 언급).
- frontmatter: `tags: [kbap-backend, 흐름]`, `생성일: 2026-07-14`, `상태: 완료`.
- 노트 완성 후 `vault-bookkeeping` — 스캔 노트는 FE 노트 2개 역링크 이행(위 [!note])·이전 개념 노트 역링크·홈 등록·작업 로그·린트.
- 두 흐름이 `MemberController`/`MemberRepository`를 함께 스치지만 본문 설명은 각자 자기 흐름 구간만. 겹치면 온보딩 노트가 member 도메인 설명 주인, 스캔 노트는 링크.
</content>
