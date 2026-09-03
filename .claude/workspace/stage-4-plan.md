# 4단계 계획서 · 아키텍처·방법론 — 왜 이렇게 나눴나

- **대상 독자**: FE(React Native/Expo)만 해온 개발자. 1~3단계(서버·스레드·JVM / IoC·DI·계층·트랜잭션 / JPA·영속성) 이수.
- **단계 목표**: 1~3단계에서 만난 조각들(도메인 모듈이 Spring-free인 이유, ArchUnit 벽, `@Transactional` 경계, persistence 어댑터, BehaviorSpec 언급)을 **큰 그림으로 묶는다**. kbap이 왜 core/infra/app으로 쪼개고 도메인을 순수하게 두는지, 그리고 이 코드가 어떤 방법론(SDD·TDD)으로 만들어지는지를 잡는다. 5단계(실코드 흐름) 직전.
- **노트 3개** (공부 지도 예정 3개 유지 — 실측 결과 조정 불필요. 세 축이 독립: 계층 역전(ports&adapters) / 모듈 경계(모듈러 모놀리스·DDD) / 개발 방법론(SDD·TDD)). 담당 전부 `concept-writer`.
- **제약**: kbap-server 읽기 전용. dev 백업에 노트 쓰지 않음. 완성 노트는 iCloud 볼트 `study/kbap 백엔드/`에만.
- **근거 파일 전부 실재 확인 완료**(2026-07-14).

> [!warning] 낡은 ADR 주의 — 0005·0006
> team-lead가 준 근거 중 **ADR-0005·0006은 옛 패키지 이름 기준**이다(0005: `com.meogo.api.<모듈>`, 0006: `:meogo-api:persistence` + "batch 완전 디커플드"). 이 둘은 이후 **ADR-0008(모듈러 모놀리스 — 공유 도메인, batch 직접 의존)** 과 `005-module-restructure`로 **대체**됐다(현재 구조는 `com.meogo.<layer>`, `:infra:persistence`, batch가 도메인 직접 의존). → 노트에는 **현재 구조(ADR-0008 기준)** 를 쓰고, 0005·0006은 "이렇게 바뀌어 왔다"는 맥락이 필요할 때만 "과거 결정" 표시와 함께 인용. 옛 이름을 현재처럼 쓰지 말 것.

**이번 단계가 이어받는 조각 (기존 kbap 노트 → 4단계 노트)**
- 3단계 `영속성 컨텍스트와 엔티티 매핑`의 "도메인은 JPA를 모른다 / toDomain·from" → **노트 1**(왜 그렇게 뒤집나 = ports&adapters)
- 2단계 `의존성 주입(DI)과 계층 구조`의 "인터페이스에 의존, 구현은 런타임 주입" → **노트 1**(그 인터페이스=port, 구현=adapter)
- 1단계 `JVM 위의 Kotlin…`의 "Gradle 멀티모듈=모노레포" → **노트 2**(그 모듈들이 바운디드 컨텍스트 경계)
- 3단계 `LAZY 로딩과 N+1`·`Flyway`가 스치듯 언급한 Kotest/테스트 DB → **노트 3**(BehaviorSpec·TDD 사이클로 정식화)

---

## 노트 1 — 클린 아키텍처와 ports & adapters — 도메인을 순수하게 지키기

- **파일명(=제목)**: `클린 아키텍처와 ports & adapters — 도메인을 순수하게 지키기.md`
- **담당**: concept-writer
- **한 줄 의도**: 2단계에서 "인터페이스에 의존한다", 3단계에서 "도메인은 JPA를 모른다"를 봤다. 그 둘은 같은 원리 하나의 조각이었다 — **도메인(핵심 규칙)을 프레임워크·DB로부터 격리**하고, 바깥이 도메인의 port를 구현(adapter)한다. 이 역전을 큰 그림으로 묶는다.

**목차 초안**
1. 한 줄 요약
2. 개념표 (port / adapter / 의존성 역전 / 도메인 순수성)
3. 문제 — 도메인 규칙이 Spring·JPA에 묶이면? 프레임워크 갈아끼우기·테스트가 지옥. FE에서 비즈니스 로직이 특정 UI 라이브러리에 엮인 상황과 대비
4. 해법: 의존성 역전 — 도메인이 필요로 하는 걸 **인터페이스(port)** 로 선언하고, 바깥(infra)이 구현(adapter). 화살표가 안쪽(도메인)을 향하게 뒤집음
5. kbap에서 — 도메인(`core:*`)은 완전 Spring-free·ORM-free(model + port + 정책). `MemberRepository`(port, 도메인)를 `MemberRepositoryAdapter`(adapter, infra:persistence)가 구현. 부트앱이 `runtimeOnly`로 조립(2단계 DI 조각을 여기서 완성)
6. 왜 이득인가 — 도메인 단위 테스트에 DB 불필요(가짜 port 주입), 영속 기술 교체가 도메인 무변경. 3단계 `toDomain()/from()`이 "도메인↔JPA 경계 지킴이"였던 이유
7. LLM도 같은 패턴 — `infra:llm`이 어댑터 모듈(ADR-0010). 외부 호출도 도메인 밖으로
8. 실코드 — `MemberProfileUseCase`(port에 의존) / `MemberRepositoryAdapter`(구현) 대비, ADR-0006이 말하는 "중앙 영속 어댑터"의 현재형(ADR-0008)
9. 치트시트 / 용어집 / 체크리스트

**근거 kbap-server 파일**
- `docs/adr/0006-central-persistence-adapter-and-decoupled-batch.md` — 영속 어댑터 결정(★옛 이름, ADR-0008이 현재형)
- `docs/adr/0010-llm-adapter-module-named-infra-llm.md` — LLM 어댑터 모듈(포트/어댑터의 또 다른 예)
- `docs/architecture/meogo-conventions.md` "도메인 객체 불변성 & 영속 변환"(81줄~), "도메인 모듈 빌딩블록"(40줄~)
- `application/client/.../member/MemberProfileUseCase.kt`(port 의존) ↔ `infra/persistence/.../member/MemberRepositoryAdapter.kt`(구현)
- `CLAUDE.md` 모듈 구조 — "도메인은 ORM-free·완전 Spring-free", "port 구현체는 부트앱이 runtimeOnly로 주입"

**연결**: back-link → `[[의존성 주입(DI)과 계층 구조 — 코드를 어떻게 나누나]]`, `[[영속성 컨텍스트와 엔티티 매핑]]`. 이후 → `[[모듈러 모놀리스와 DDD — 경계를 코드로 지키기]]`. 5단계 예고 → "실제 요청이 Controller→UseCase→port→adapter를 지나는 걸 온보딩 흐름에서 눈으로 확인".

---

## 노트 2 — 모듈러 모놀리스와 DDD — 경계를 코드로 지키기

- **파일명(=제목)**: `모듈러 모놀리스와 DDD — 경계를 코드로 지키기.md`
- **담당**: concept-writer
- **한 줄 의도**: 1단계에서 본 "Gradle 멀티모듈"이 사실 **바운디드 컨텍스트 경계**였다. 하나의 배포 단위(모놀리스)를 모듈로 쪼개 의존 방향을 단방향으로 고정하고, 그 경계를 **ArchUnit 테스트로 강제**하는 설계를 잡는다.

**목차 초안**
1. 한 줄 요약
2. 개념표 (모듈러 모놀리스 / 바운디드 컨텍스트 / 의존 방향 / ArchUnit)
3. 모놀리스 vs 마이크로서비스, 그 사이 — 하나로 배포하되 안은 모듈로 쪼갬. "쪼개되 나누지 않은" 선택(왜 MSA 아닌지 한 줄)
4. 바운디드 컨텍스트(DDD) — food·member·scan·avoidance… 각 도메인이 자기 언어·모델을 소유. kernel은 공유 vocabulary만. FE 모노레포 패키지 경계와 대비하되 "여긴 도메인 언어 경계"
5. 단방향 의존 — `common ← core:kernel ← core:도메인 ← application ← app`. 왜 방향이 하나여야 하나(순환·계층 역전 방지)
6. 경계를 코드로 강제 — ArchUnit `ModuleBoundaryTest`: "도메인은 spring/jpa 패키지에 의존하면 실패", "app:api가 도메인/persistence 직접 의존하면 실패". 문서가 아니라 테스트가 벽. 3단계 규약들이 왜 안 무너지는지의 답
7. 두 부트앱이 공유 — api·batch가 같은 도메인/영속 재사용(ADR-0008). 1단계 "실행 파일 2개" 큰 그림 완성
8. 실코드 — `ModuleBoundaryTest.kt`의 `given("도메인 모듈 경계")` 발췌(noClasses…should…dependOn spring/jpa)
9. 치트시트 / 용어집 / 체크리스트

**근거 kbap-server 파일**
- `docs/adr/0008-modular-monolith-shared-domain.md` — 현재 구조의 근거(★1차 소스)
- `docs/adr/0001-multi-app-modular-layout.md` — 멀티앱 레이아웃 출발점
- `docs/architecture/meogo-api-module-structure.md`, `docs/architecture/meogo-conventions.md`("DDD 정의" 11줄~, "모듈 구성" 20줄~, "도메인 간 의존 규칙" 99줄~)
- `app/api/src/test/kotlin/com/meogo/app/api/architecture/ModuleBoundaryTest.kt:53-66` — 도메인 경계 규칙 실물(spring/jpa 의존 금지)
- `CLAUDE.md` 모듈 구조·"ArchUnit 테스트로 강제"

**연결**: back-link → `[[JVM 위의 Kotlin — 컴파일·타입·널 안전이 TypeScript와 다른 점]]`(Gradle 멀티모듈), `[[클린 아키텍처와 ports & adapters — 도메인을 순수하게 지키기]]`(경계 안에서의 역전). 이후 → `[[SDD와 TDD — 스펙과 테스트로 개발하기]]`. 5단계 예고 → "여러 컨텍스트를 조합하는 홈 화면 흐름에서 경계 넘나듦을 관찰".

---

## 노트 3 — SDD와 TDD — 스펙과 테스트로 개발하기

- **파일명(=제목)**: `SDD와 TDD — 스펙과 테스트로 개발하기.md`
- **담당**: concept-writer
- **한 줄 의도**: 3단계에서 스친 Kotest·테스트 DB를 방법론으로 정식화한다. kbap은 기능마다 **스펙 먼저(SDD: spec→plan→tasks)** 쓰고, **테스트 먼저(TDD: Red→Green→Refactor)** 로 구현한다 — 헌법이 Test-First를 강제하고, 에이전트 팀이 이 사이클을 돌린다.

**목차 초안**
1. 한 줄 요약
2. 개념표 (SDD / SpecKit / TDD / BehaviorSpec / 헌법)
3. SDD(명세 주도) — 코드 전에 `specs/<기능>/`에 spec(무엇/왜)→plan(어떻게)→tasks(할 일)를 쌓는다. FE에서 "일단 만들고 보기"와 대비
4. TDD Red→Green→Refactor — 실패하는 테스트부터(Red), 통과시키고(Green), 정리(Refactor). 헌법 원칙 I "Test-First(NON-NEGOTIABLE)"
5. Kotest BehaviorSpec — `given("대상") > when("상황") > then("기대")`, 한국어. 3단계에서 본 통합 테스트도 이 스타일. FE의 Jest describe/it과 대비
6. 테스트가 진짜 DB를 쓴다 — MySQL Testcontainers로 운영-동등 검증(3단계 Flyway 노트와 연결)
7. 이 레포는 에이전트 팀이 돌린다 — test-writer→implementer→(code-reviewer∥database-expert) 하네스가 Red→Green→리뷰를 자동화(kbap CLAUDE.md "하네스"). 내가 배우는 공부 하네스와 같은 발상의 다른 적용
8. 실코드 — `specs/kb-104-onboarding-profile/`(spec/plan/tasks/data-model 실물 구조), `.specify/memory/constitution.md`(원칙 I)
9. 치트시트 / 용어집 / 체크리스트

**근거 kbap-server 파일**
- `.specify/memory/constitution.md:32` — "I. Test-First Development (NON-NEGOTIABLE)"
- `specs/kb-104-onboarding-profile/`(spec.md·plan.md·tasks.md·data-model.md·contracts·checklists — SDD 사이클 실물)
- `CLAUDE.md` "하네스: SpecKit·TDD 개발 팀"(test-writer→implementer→리뷰 에이전트, 원칙 I 강제), "테스트 스타일(고정)"(전부 BehaviorSpec 한국어)
- `app/api/src/test/resources/application.yml`(Testcontainers·validate — 3단계와 공유)

**연결**: back-link → `[[Flyway 마이그레이션 — 스키마를 코드로 관리하기]]`(테스트가 같은 마이그레이션 적용), `[[모듈러 모놀리스와 DDD — 경계를 코드로 지키기]]`(경계도 ArchUnit 테스트로 강제 = 테스트가 설계 지킴이). **복리(볼트)** → `[[5. 여러 에이전트의 협업]]`, `[[2. 행동을 다듬는 장치 — 훅·투두·서브에이전트·스킬]]`(내 Claude Code 지식과 kbap 에이전트 하네스의 실접점 — 양방향). 5단계 예고 → "각 실코드 흐름 노트에서 그 기능의 spec·테스트를 근거로 인용".

> [!note] Claude Code 작동원리 노트 복리 (vault-bookkeeping 시)
> `Claude Code 작동원리/5. 여러 에이전트의 협업.md`·`2. 행동을 다듬는 장치 — 훅·투두·서브에이전트·스킬.md`는 실재 확인됨(2026-07-14, 6개 노트 세트). 노트 3에서 이 둘로 링크하고, 두 노트 하단에 `[[SDD와 TDD — 스펙과 테스트로 개발하기]]`로 역링크 한 줄 append("이 개념이 실제 백엔드 레포의 개발 하네스에 어떻게 쓰이나"). 기존 본문은 건드리지 말 것.

---

## 5단계(실코드 흐름) 후보 — 지도 기준 재확인 + 4단계 forward 예고
지도의 후보 6개 그대로 유효(우선순위=사용자 가치×계층 관통):
1. **온보딩 프로필**(kb-104/124) — 노트 1이 "ports&adapters를 실물로 볼 곳"으로 예고
2. **메뉴 스캔 판정**(001, scan) — FE OCR 노트 입력이 들어오는 핵심 흐름
3. **홈 화면 조회**(kb-111) — 노트 2가 "여러 컨텍스트 조합"으로 예고
4. **Firebase JWT 인증**(kb-118) — 기존 `OAuth 2.0과 OIDC…` 노트와 이어짐
5. **회원 랭킹**(kb-123) — 도메인 값 객체·동시성
6. **배치 LLM 파이프라인**(kb-53) — 두 번째 bootJar + `infra:llm`, 노트 1의 "LLM도 어댑터"·노트 2의 "두 부트앱 공유"가 여기서 합쳐짐

**각 4단계 노트의 forward 한 줄** (본문 말미에 배치):
- 노트 1 → "다음 단계부터 이 port→adapter가 실제 요청에서 어떻게 이어지는지 흐름 노트로 따라간다(첫 흐름: 온보딩 프로필)."
- 노트 2 → "다음 단계에서 이 컨텍스트 경계를 실제로 넘나드는 흐름(홈 화면 조합·스캔 판정)을 관찰한다."
- 노트 3 → "다음 단계 흐름 노트마다 그 기능의 spec·테스트를 근거로 인용해 SDD·TDD가 실제로 어떻게 남는지 본다."

---

## 작성자 공통 지침 (concept-writer가 따를 것)
- `study-note-style` 규약: FE 비유마다 "어디까지 같고 어디서 다른지" 한 줄, 새 용어 첫 등장 시 정의(이전 단계에 있으면 `[[링크]]`), 실코드/문서 인용은 `경로:줄` + 발췌 15줄 이내.
- **1차 소스는 ADR·컨벤션 문서** — 코드보다 docs/adr·docs/architecture를 먼저 인용. 단 **ADR-0005·0006은 옛 이름(과거 결정)**, 현재는 ADR-0008 기준으로 쓴다.
- kbap-server는 **Kotlin 주석 금지 레포** — 발췌 아래 줄 단위 풀이.
- **종합 단계**: 새 개념을 늘어놓기보다 1~3단계 조각을 "이게 그거였다"로 묶는 데 집중. 위 "이어받는 조각" 매핑을 본문에 녹일 것.
- **깨진 링크 금지** — `[[ ]]`는 위 확인된 파일명과 정확히 일치할 때만.
- frontmatter: `tags: [kbap-backend, 아키텍처]`, `생성일: 2026-07-14`, `상태: 완료`.
- 노트 완성 후 `vault-bookkeeping` — 특히 **Claude Code 작동원리 2개 노트 역링크 이행**(노트 3 [!note] 참조)·이전 단계 노트 역링크·홈 등록·작업 로그·린트.
</content>
