# 5단계 넷째 묶음(5d) 계획서 · 실코드 흐름 — 회원 랭킹(kb-123)

- **대상 독자**: FE(React Native/Expo)만 해온 개발자. 1~4단계 개념 노트 + 5a~5c(온보딩·스캔·JWT·홈) 이수. **5단계 규칙: 새 개념 설명 금지 — 기존 노트로 `[[번호. 링크]]`.**
- **담당**: `code-flow-writer` (1개 노트).
- **노트 골격**: ① 이 API가 하는 일(사용자 관점 한 줄) ② 흐름 표(계층 | 파일:줄 | 하는 일) ③ 계층 경계마다 "여기서 모듈이 바뀌는 이유" ④ 요청/응답 JSON ⑤ 관련 테스트가 보장하는 것 ⑥ spec·ADR 링크.
- **제약**: kbap-server 읽기 전용. 모든 단계에 `경로:줄` 근거. 상상 금지. 완성 노트는 iCloud 볼트 `kbap 백엔드/`에만.
- **번호 규칙**: 파일명·H1 앞 공부 순서 번호. 기존 17노트(`1.`~`17.`). **이번 새 노트는 `18.`.**
- **근거 파일 전부 실재 확인 완료**(2026-07-15). spec `specs/kb-123-member-ranking/` 실존 확인.

## 묶음 구성 근거 (한 줄)
**회원 랭킹(kb-123) 단독 1노트.** 랭킹은 "증가(write)"와 "조회(read)" 두 얼굴이지만, write는 **이미 note 14 스캔 흐름이 소유**하고 "랭킹에서 다룸"으로 forward-예고만 남긴 단일 호출(`increaseScanCount`)이고, read는 값 객체 하나 + readOnly 유스케이스 하나다. 둘을 쪼개면 원자적 UPDATE(증가)와 그 카운트로 점수를 계산하는 `Ranking` 값 객체(조회)가 갈라져 오히려 이해가 끊긴다 — 한 노트에서 write↔read를 맞물려 보이는 게 맞다.

---

> [!warning] 실측 발견 1 — 지금 실제로 오르는 점수는 스캔뿐 (리뷰·다양성은 항상 0)
> spec Acceptance 예시(US1 AS1: "리뷰 8개·음식 6종·스캔 9회 → 128점")는 **리뷰 기능이 있는 미래 상태**를 가정한다. 코드 실측: `increaseScanCount`만 존재하고(리뷰 카운트 증가 경로 없음) `reviewCount`·`uniqueReviewedFoodCount`는 엔티티 기본값 0에서 안 바뀐다(`MemberJpaEntity.kt:51,54`). → 점수는 현재 `scanCount × 2`만 실제로 오른다. **같은 레포 Swagger는 코드와 일치**("리뷰 기능 도입 전이라 reviews·diversity 는 현재 항상 0", `MemberApi.kt:125`) — 어긋난 건 spec의 AS 예시뿐. 노트는 **코드 기준**(스캔만 기여)으로 쓰고 `> [!warning] 스펙과 다름`으로 "AS 예시는 리뷰 도입 후 기준"이라 한 줄 짚는다. 점수식·등급표 자체는 코드(`Ranking.kt`)에 다 구현돼 있으니 그대로 소개.

> [!note] 실측 발견 2 — 이 노트의 뼈대: 원자적 UPDATE vs 변경 감지
> `increaseScanCount`는 엔티티를 로드해 필드 바꾸고 flush하는 5a식 **변경 감지(dirty checking)**가 아니라, `@Modifying @Query`로 `update … set scan_count = scan_count + 1 where id=…`를 **DB에 직접 쏘는 벌크 UPDATE**다(`MemberJpaRepository.kt:18-28`). 이게 동시성 **lost update 방지**의 핵심 — 두 스캔 요청이 동시에 와도 read-modify-write 경합 없이 DB가 원자적으로 +1 한다. `clearAutomatically=true`는 UPDATE 후 영속성 컨텍스트를 비워 낡은 엔티티 재사용을 막는다. → note 7(변경 감지)·note 2(동시성)와의 **대비가 노트의 척추**. 조회 쪽은 정반대로 `@Transactional(readOnly=true)`로 엔티티 로드→값 객체 계산.

---

## 노트 18 — 회원 랭킹 흐름 — 원자적 카운트 증가와 조회 시점 점수 계산

- **파일명(=제목)**: `18. 회원 랭킹 흐름 — 원자적 카운트 증가와 조회 시점 점수 계산.md`
- **H1**: 파일명과 동일
- **담당**: code-flow-writer
- **API**: `GET /api/v1/members/me/ranking` (조회) + 증가는 스캔 흐름에 얹힌 `increaseScanCount`(엔드포인트 아님)
- **한 줄**: 프로필/랭킹 화면에서 내 등급·점수·다음 등급까지 남은 점수·점수 내역(리뷰·다양성·스캔)을 한 번에 받는다. 점수는 저장해 둔 게 아니라 **조회 시점에** 누적 카운트로 계산되고, 카운트는 스캔할 때마다 DB가 원자적으로 +1 한다.

**흐름 A — 증가 (write, note 14 스캔에 얹힘)**
| 계층 | 파일 | 하는 일 |
|---|---|---|
| 호출 지점 | `application/client/.../scan/usecase/ScanUseCase.kt:62` | 스캔 판정 끝에 `memberRepository.increaseScanCount(memberId)`(메뉴판 1장 = 1회) |
| port | `core/member/.../MemberRepository.kt:12` | `increaseScanCount(memberId)` 인터페이스 |
| adapter | `infra/persistence/.../member/MemberRepositoryAdapter.kt:37-43` | `@Transactional`, JPA 벌크 UPDATE 호출, 영향 행 `0`이면 `MEMBER_NOT_FOUND` |
| 벌크 UPDATE | `infra/persistence/.../member/MemberJpaRepository.kt:18-28` | `@Modifying(clearAutomatically=true, flushAutomatically=true) @Query` — `set scan_count = scan_count + 1 where id=:memberId and memberStatus=ACTIVE and status=ACTIVE`. **원자적·변경 감지 우회** |
| 스키마 | `infra/persistence/.../member/MemberJpaEntity.kt:48-56` | `scan_count`·`review_count`·`unique_reviewed_food_count` 컬럼(전부 not-null 기본 0) |

**흐름 B — 조회 (read)**
| 계층 | 파일 | 하는 일 |
|---|---|---|
| Controller | `app/api/.../member/MemberController.kt:34-39` | `getMyRanking(@AuthMemberId memberId)` → `memberRankingUseCase.getRanking(memberId)` → `MemberRankingResponse.from(result)` |
| UseCase | `application/client/.../member/MemberRankingUseCase.kt:14-20` | `@Transactional(readOnly=true)` — `findById` 없으면 `MEMBER_NOT_FOUND`, `MemberRankingResult.from(member.ranking)` |
| 도메인 값 객체 | `core/member/.../Ranking.kt:3-33` | private 생성자 + `of`/`initial` 팩토리, `require`로 음수 방어. **계산 프로퍼티**: `reviewPoints`·`diversityPoints`·`scanPoints`(각 ×10/×5/×2) → `score` → `tier` → `nextTier` → `pointsToNext`. 저장값이 아니라 **조회 시점 계산** |
| 등급 enum | `core/member/.../RankingTier.kt:3-21` | 7단계(newcomer 0 ~ korean_at_heart 1000) minScore 임계, `of(score)=마지막으로 넘긴 구간`, `next`, 최고등급이면 `next=null` |
| 결과/응답 DTO | `application/client/.../member/dto/MemberRankingResult.kt`, `app/api/.../member/MemberRankingResponse.kt` | 값 객체 → 평평한 result → web `{tier, level, score, nextTier, pointsToNext, breakdown{reviews, diversity, scans}{count, points}}` |
| 엔티티→도메인 | `MemberJpaEntity.kt:57-66`(`toDomain` → `Ranking.of(...)`), 초기값 | 저장은 카운트 3개만, `Ranking`은 로드 시 재조립 |

**개념 노트 링크 맵** (각 구간 → 번호 노트)
- Controller/UseCase 주입 → `[[5. 의존성 주입(DI)과 계층 구조 — 코드를 어떻게 나누나]]`, `[[4. IoC 컨테이너와 Bean — 부품을 직접 만들지 않는다]]`
- 조회 `@Transactional(readOnly=true)` / adapter의 write `@Transactional` → `[[6. 설정과 프로필, 트랜잭션 첫걸음]]`
- **핵심 대비**: 벌크 UPDATE(원자적, 변경 감지·flush 우회, `clearAutomatically`로 영속성 컨텍스트 비움) ↔ 5a식 변경 감지 → `[[7. 영속성 컨텍스트와 엔티티 매핑]]`
- **동시성**: 두 스캔 동시 요청의 lost update를 DB 원자적 +1로 방지 → `[[2. 요청 하나에 스레드 하나 — JVM 서버의 동시성과 Node 이벤트루프의 차이]]`
- `MemberRepository` port에만 의존, 구현은 adapter / `Ranking`·`RankingTier` 순수 도메인 → `[[10. 클린 아키텍처와 ports & adapters — 도메인을 순수하게 지키기]]`
- `Ranking`이 계산 로직을 품은 **불변 값 객체**, member 바운디드 컨텍스트 → `[[11. 모듈러 모놀리스와 DDD — 경계를 코드로 지키기]]`
- `@AuthMemberId`로 memberId 주입 → `[[16. JWT 인증 필터와 세션 생명주기 — 매 요청 검증·재발급·로그아웃]]`
- 증가의 write 원천(스캔) → `[[14. 메뉴판 스캔 판정 흐름 — 메뉴명이 위험도가 되기까지]]`
- 회원 도메인·`Ranking.initial()`(가입 시) → `[[13. 온보딩 프로필 저장 흐름 — 요청이 DB에 닿기까지]]`
- 테스트·spec → `[[12. SDD와 TDD — 스펙과 테스트로 개발하기]]`

**forward/보류 (본문 말미 한 줄씩)**
- 리뷰 기능 도입 시 `reviewCount`·`uniqueReviewedFoodCount` 증가 경로가 생겨 점수 3항목이 다 살아난다(현재 스캔만).
- ponytail: **홈(note 17) 링크는 걸지 않음** — 홈은 랭킹을 조회하지 않는다(인기 음식은 랜덤). 접점 없음.

**관련 테스트가 보장하는 것** (⑤ 섹션 근거)
- `core/member/.../RankingTest.kt`(점수식·등급 경계·최고등급 null·음수 방어), `application/client/.../member/MemberRankingUseCaseTest.kt`(조회·회원 없음 예외). 원자적 증가의 동시성 보장은 스캔 쪽 테스트가 커버(note 14) — 필요 시 `MemberRepositoryAdapter`/`increaseScanCount` 통합 테스트 확인.

**근거 spec/ADR**: `specs/kb-123-member-ranking/`(spec US1~US3, data-model=랭킹 카운트 컬럼·점수 정책, research). 점수·등급 정책은 spec FR-025 계열. 값 객체·불변·영속 변환 → ADR-0008 계열 + `docs/architecture/meogo-conventions.md`.

---

## 작성자 공통 지침 (code-flow-writer가 따를 것)
- `kbap-repo-map`으로 탐색 시작, `study-note-style` 규약 준수(**번호 접두 규칙 포함**). **모든 단계에 `경로:줄` 근거**, 상상 금지.
- **새 개념 설명 금지** — 위 링크 맵대로 번호 노트로 `[[링크]]`. 새 개념 나오면 짧은 콜아웃 + "개념 노트 후보" 보고(현재로선 없음 — 전부 기존 노트로 커버).
- 리뷰·다양성 항상 0은 **코드 기준**으로 쓰고 `> [!warning] 스펙과 다름`(spec AS 예시는 리뷰 도입 후). 점수식·등급표는 코드에 구현돼 있으니 그대로.
- 발췌 15줄 이내, 주석 금지 레포이므로 발췌 아래 줄 단위 풀이(`MemberApi.kt`·`@Query` JPQL은 인용 가능).
- frontmatter: `tags: [kbap-backend, 흐름]`, `생성일: 2026-07-15`, `상태: 완료`. 파일명·H1 모두 `18. ` 접두.

## vault-keeper 지시 (노트 완성 후 vault-bookkeeping)
1. **복리 (양방향)** — 노트 18 ↔ 각 노트 하단 역링크 append(기존 본문 불변, 한 줄만):
   - `[[14. 메뉴판 스캔 판정 흐름 …]]` ↔ 18 — note 14가 남긴 "`increaseScanCount` → 회원 랭킹 흐름(kb-123)에서 다룸" **forward-예고 갚기**. write(증가)↔read(조회) 짝.
   - `[[7. 영속성 컨텍스트와 엔티티 매핑]]` ↔ 18 — 변경 감지(7) vs 원자적 벌크 UPDATE(18) 개념 심화 대비.
   - `[[2. 요청 하나에 스레드 하나 …]]` ↔ 18 — 동시성 lost update(2) ↔ DB 원자적 증가 해법(18).
2. **홈(note 17) 복리는 하지 않음** — 홈은 랭킹을 조회하지 않아 접점 없음(코드 실측, 위 forward/보류). team-lead 후보였으나 부적합.
3. **지도 갱신**: `🗺️ kbap 백엔드 공부 지도.md` — ⑤ 회원 랭킹 체크박스 `[x]`, 링크를 `[[18. 회원 랭킹 흐름 — 원자적 카운트 증가와 조회 시점 점수 계산]]`로, 진도 요약을 "완료 노트 18 / 다음 묶음: 배치 LLM 스코어링(kb-53) — 마지막 흐름"으로. 지도는 새로 만들지 말고 체크·정정만.
4. 이전 개념 노트 역링크·볼트 홈 등록·작업 로그·린트는 표준 절차대로. 새 파일명 번호(`18.`)가 study-note-style 규칙과 맞는지 린트 확인.
