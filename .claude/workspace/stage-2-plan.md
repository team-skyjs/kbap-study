# 2단계 계획서 · Spring 핵심 — 부품을 직접 만들지 않는 프레임워크

- **대상 독자**: FE(React Native/Expo)만 해온 개발자. 1단계(서버 프로세스·스레드·JVM/Kotlin)까지 이수.
- **단계 목표**: 1단계에서 "서버가 뜬다"를 알았다. 그 뜬 서버 안에서 **객체를 누가·어떻게 조립하고 연결하는가**(IoC/DI)를 잡고, 코드가 어떤 계층으로 나뉘는지, 설정·프로필·트랜잭션의 개념을 심는다. 3단계(JPA·영속성) 이전 필수.
- **노트 3개** (공부 지도의 예정 3개 유지 — 레포 실측 결과 조정 불필요. IoC/Bean, DI/계층, 설정·프로필·트랜잭션은 서로 독립 개념 덩어리라 그대로 3분할이 맞다). 담당 전부 `concept-writer`.
- **제약**: kbap-server 읽기 전용. dev 백업에 노트 쓰지 않음. 완성 노트는 iCloud 볼트 `study/kbap 백엔드/`에만.
- **근거 파일 전부 실재 확인 완료**(2026-07-14).

> [!warning] 트랜잭션 경계 — "첫걸음"만
> 3번 노트의 트랜잭션은 **"작업 하나를 전부 성공 or 전부 취소로 묶는 경계"라는 개념 + `@Transactional`이 붙는 위치(UseCase)** 까지만 다룬다. 영속성 컨텍스트·flush·롤백 시점·전파(propagation)·격리 수준은 **3단계(JPA·영속성)로 미룬다**. 노트에 "여기까지가 이번 범위, 나머지는 3단계"를 명시할 것.

> [!note] 기존 볼트 연결 — 실측 결과 정정
> team-lead가 후보로 준 `인증·보안/쿠키·세션 vs 토큰 인증`·`백엔드/페이징` 노트는 **볼트에 아직 없다**(2026-07-14 확인 — `인증·보안/`에는 `OAuth 2.0과 OIDC…`·`Google vs Apple…` 2개뿐, `백엔드/` 폴더는 비어 있음). 없는 파일로 `[[링크]]`를 걸면 깨진 링크가 된다.
> → **2단계의 실질 복리 접점은 1단계 노트 3개**다. 각 2단계 노트에서 관련 1단계 노트로 정확한 파일명으로 back-link를 건다. 인증/페이징 접점은 **5단계 실코드 흐름(JWT·커서 페이징)에서** 자연히 생기므로 그때 연결한다(1단계 계획서와 같은 방침).

**back-link 대상 1단계 노트 (파일명 정확 확인 완료)**
- `서버는 켜두는 프로그램 — 서버 프로세스와 두 개의 실행 파일`
- `요청 하나에 스레드 하나 — JVM 서버의 동시성과 Node 이벤트루프의 차이`
- `JVM 위의 Kotlin — 컴파일·타입·널 안전이 TypeScript와 다른 점`

---

## 노트 1 — IoC 컨테이너와 Bean — 부품을 직접 만들지 않는다

- **파일명(=제목)**: `IoC 컨테이너와 Bean — 부품을 직접 만들지 않는다.md`
- **담당**: concept-writer
- **한 줄 의도**: FE에선 필요한 객체를 그때그때 `new` 하거나 훅으로 만들었다. 스프링은 **컨테이너가 객체(Bean)들을 미리 만들어 보관·조립**하고, 코드는 만들지 않고 받아 쓴다 — 이 역전을 심는다.

**목차 초안**
1. 한 줄 요약
2. 개념표 (IoC / 컨테이너 / Bean / 컴포넌트 스캔)
3. 원리 — "제어의 역전(IoC)": 내가 부품을 만들어 조립하던 걸(FE) 프레임워크가 대신 한다. 왜 그게 이득인가(교체·테스트·수명 관리)
4. Bean이란 — 컨테이너가 관리하는 객체 한 개. `@Component`/`@Service`/`@RestController`/`@Configuration` 붙은 게 Bean이 된다
5. 컨테이너가 언제 Bean을 만드나 — 서버 뜰 때 컴포넌트 스캔으로 한 번(진입점 `com.meogo` 루트 스캔, 1단계 노트와 연결). FE의 매 렌더 vs 서버 부팅 1회
6. FE 비유 — React 루트가 Context Provider들을 조립해두는 것과 대비하되, "어디까지 같고 어디서 다른지"(Provider는 트리 스코프·리렌더, Bean은 앱 1개·싱글턴) 한 줄
7. 실코드 — `@Service class MemberProfileUseCase(...)`, `@Configuration`의 명시적 Bean(`CorsConfig`) 발췌
8. 치트시트 / 용어집 / 체크리스트

**근거 kbap-server 파일**
- `application/client/src/main/kotlin/com/meogo/application/client/member/MemberProfileUseCase.kt:19-22` — `@Service class ...( private val memberRepository )` (Bean 선언 + 생성자 주입 실물)
- `app/api/src/main/kotlin/com/meogo/app/api/config/CorsConfig.kt` — `@Configuration`/`@Bean` 명시적 Bean 등록 예
- `app/api/src/main/kotlin/com/meogo/MeogoApiApplication.kt` — `@SpringBootApplication`(컴포넌트 스캔 시작점, 1단계 노트에서 이미 봄)
- `CLAUDE.md` 컨벤션 — "부트 진입점을 `com.meogo` 루트에 두어 컴포넌트 스캔이 전 계층 커버"

**연결**: back-link → `[[서버는 켜두는 프로그램 — 서버 프로세스와 두 개의 실행 파일]]`(스캔은 서버 부팅 때 1회). 이후 → `[[의존성 주입(DI)과 계층 구조 — 코드를 어떻게 나누나]]`.

---

## 노트 2 — 의존성 주입(DI)과 계층 구조 — 코드를 어떻게 나누나

- **파일명(=제목)**: `의존성 주입(DI)과 계층 구조 — 코드를 어떻게 나누나.md`
- **담당**: concept-writer
- **한 줄 의도**: 1번에서 "컨테이너가 Bean을 만든다"를 알았다. 그 Bean들을 **서로에게 꽂아 넣는 것(DI)** 과, kbap가 코드를 Controller→UseCase→도메인→영속의 **계층**으로 나누는 이유를 잡는다.

**목차 초안**
1. 한 줄 요약
2. 개념표 (DI / 생성자 주입 / 인터페이스(port) / 계층)
3. 원리 — DI란: 필요한 부품을 내가 만들지 않고 **생성자로 받는다**. `props`/`useContext`로 의존을 내려받던 FE와 대비("어디까지 같고 어디서 다른지" 한 줄)
4. 왜 인터페이스에 의존하나 — UseCase가 `MemberRepository`(인터페이스)만 알고, 실제 구현(`MemberRepositoryAdapter`)은 런타임에 컨테이너가 꽂는다. 구현을 바꿔도 UseCase는 무변경(테스트 시 가짜 주입도 이 덕분)
5. 계층 구조 — Controller(요청 받기)/UseCase(조율·트랜잭션 경계)/도메인(순수 규칙)/영속(DB). FE의 화면 컴포넌트/커스텀 훅/순수 유틸 분리와 대비
6. 한 요청이 계층을 타는 그림 — `MemberController` → `MemberProfileUseCase` → `MemberRepository`(port) → adapter. (실흐름 상세는 5단계, 여기선 "층이 나뉘어 있다"까지)
7. 실코드 — `MemberController(private val ...UseCase)` 생성자 주입, `MemberProfileUseCase(private val memberRepository: MemberRepository)`
8. 치트시트 / 용어집 / 체크리스트

**근거 kbap-server 파일**
- `app/api/src/main/kotlin/com/meogo/app/api/member/MemberController.kt:13-22` — `@RestController` + 생성자로 UseCase 2개 주입(실물)
- `application/client/src/main/kotlin/com/meogo/application/client/member/MemberProfileUseCase.kt:20-22` — UseCase가 `MemberRepository` **인터페이스**를 주입받음
- `infra/persistence/src/main/kotlin/com/meogo/infra/persistence/member/MemberRepositoryAdapter.kt` — 그 인터페이스의 구현체(런타임에 주입되는 Bean). *port/adapter 역전의 "왜"는 4단계, 여기선 "인터페이스에 의존하고 구현은 갈아끼운다"까지만*
- `CLAUDE.md` 모듈 구조 — 계층 의존 방향(application→도메인 port, 구현은 runtimeOnly 조립)

**연결**: back-link → `[[IoC 컨테이너와 Bean — 부품을 직접 만들지 않는다]]`, `[[JVM 위의 Kotlin — 컴파일·타입·널 안전이 TypeScript와 다른 점]]`(멀티모듈 경계가 계층 경계와 겹침). 이후 → `[[설정과 프로필, 트랜잭션 첫걸음]]`. 4단계 예고 → "왜 도메인을 Spring-free로 두고 port/adapter로 뒤집나".

---

## 노트 3 — 설정과 프로필, 트랜잭션 첫걸음

- **파일명(=제목)**: `설정과 프로필, 트랜잭션 첫걸음.md`
- **담당**: concept-writer
- **한 줄 의도**: 코드 바깥의 값(DB 주소·키)을 어떻게 주입하고 환경(local/dev/prod)별로 바꾸는지(`.yml`·프로필), 그리고 UseCase에 붙는 `@Transactional`이 무엇을 묶는지 **개념만** 잡는다.

**목차 초안**
1. 한 줄 요약
2. 개념표 (외부 설정 / 프로필 / 트랜잭션)
3. 외부 설정이란 — 코드에 값을 박지 않고 `application.yml`에서 주입. FE의 `.env`/`app.config`와 대비
4. 프로필 4종 — `application-{local,dev,staging,prod}.yml`, `SPRING_PROFILES_ACTIVE`로 선택. 1단계 노트에서 "포트/프로필 맛보기" 했던 걸 여기서 확장
5. 트랜잭션 첫걸음 — "작업 하나를 전부 성공 아니면 전부 취소로 묶는 경계". `@Transactional`이 UseCase 메서드에 붙는 이유(여러 DB 변경이 중간에 깨지면 되돌린다). **여기까지가 이번 범위** — 영속성 컨텍스트·flush·롤백 규칙·전파는 3단계
6. 왜 UseCase가 경계인가 — Controller나 도메인이 아니라 조율 계층(UseCase)이 "한 작업" 단위라서. 2번 노트의 계층과 연결
7. 주의점 — 트랜잭션 안에서 외부 호출(LLM·네트워크) 하지 말 것 같은 감만(상세는 뒤)
8. 치트시트 / 용어집 / 체크리스트

**근거 kbap-server 파일**
- `app/api/src/main/resources/application.yml` — 베이스 설정(`spring:`, `flyway:`, 프로필 강제 안 함)
- `app/api/src/main/resources/application-local.yml`(+ `-dev`/`-staging`/`-prod`.yml) — 프로필별 오버라이드 실물
- `application/client/src/main/kotlin/com/meogo/application/client/member/MemberProfileUseCase.kt:23-25` — `@Transactional fun completeOnboarding(...)` (트랜잭션 경계가 UseCase에 붙은 실물)
- `CLAUDE.md` 명령어/컨벤션 — 프로필 4종·`.yml` 확장자 통일·프로필 미지정 시 H2 동작

**연결**: back-link → `[[요청 하나에 스레드 하나 — JVM 서버의 동시성과 Node 이벤트루프의 차이]]`(프로필로 뜨는 서버 프로세스와 이어짐), `[[의존성 주입(DI)과 계층 구조 — 코드를 어떻게 나누나]]`(트랜잭션 경계=UseCase 계층). 3단계 예고 → "트랜잭션 안에서 영속성 컨텍스트가 실제로 뭘 하나".

---

## 작성자 공통 지침 (concept-writer가 따를 것)
- `study-note-style` 스킬 규약 준수: FE 비유마다 "어디까지 같고 어디서 다른지" 한 줄, 새 용어 첫 등장 시 정의(이전 단계에 정의 있으면 `[[링크]]`로 대체), 실코드 인용은 `경로:줄` + 발췌 15줄 이내.
- kbap-server는 **Kotlin 주석 금지 레포** — 발췌에 주석 달지 말고 발췌 아래 줄 단위 풀이.
- **트랜잭션은 3번 노트에서 "첫걸음"만** — 영속성 상세로 넘어가지 말 것(경계 문구 명시).
- **깨진 링크 금지** — `[[ ]]`는 위에 확인된 1단계 파일명과 정확히 일치할 때만. 없는 노트(인증·페이징)로 링크 걸지 말 것.
- frontmatter: `tags: [kbap-backend, spring]`, `생성일: 2026-07-14`, `상태: 완료`.
- 노트 완성 후 `vault-bookkeeping` 절차(1단계 노트에 역링크·홈 등록·작업 로그·린트) 수행.
</content>
