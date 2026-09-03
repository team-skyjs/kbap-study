# 1단계 계획서 · 백엔드 기초 — 서버라는 프로그램

- **대상 독자**: FE(React Native/Expo)만 해온 개발자. Spring/JVM/스레드 용어 전무.
- **단계 목표**: "서버는 켜두고 요청을 기다리는 프로그램"이라는 세계관을 심고, JVM/Kotlin/Gradle이라는 실행 토대를 FE의 Node/TS와 대비해 잡는다. 2단계(Spring) 이전에 반드시 선행.
- **노트 3개**로 분해 (개념 3덩어리: ①서버 프로세스 ②동시성/스레드 ③JVM·Kotlin·빌드). 담당은 전부 `concept-writer`.
- **제약**: kbap-server(`/Users/yejinkim/dev/kfood/kbap-server`)는 읽기 전용. dev 백업(`/Users/yejinkim/dev/local-handoff/study`)에도 노트를 쓰지 않는다. 완성 노트는 iCloud 볼트 `study/kbap 백엔드/`에만.
- **근거 파일은 아래 경로 모두 실재 확인 완료**(2026-07-14).

> [!note] 기존 볼트 연결 후보
> 1단계는 순수 기초라 기존 볼트에 직접 이어질 노트가 **거의 없다**(백엔드 폴더는 비어 있음). FE OCR 노트(`Expo & React Native/`)와 인증 노트(`인증·보안/`)는 5단계 실코드 흐름에서 이어진다 — 1단계 노트에서는 억지로 링크하지 않는다. 대신 세 노트끼리 `[[ ]]`로 상호 연결하고, 각자 2단계 예정 노트를 forward-link로 예고한다.

---

## 노트 1 — 서버는 켜두는 프로그램 — 서버 프로세스와 두 개의 실행 파일

- **파일명(=제목)**: `서버는 켜두는 프로그램 — 서버 프로세스와 두 개의 실행 파일.md`
- **담당**: concept-writer
- **한 줄 의도**: FE는 빌드해 배포하면 끝나는 코드였다. 서버는 프로세스로 계속 떠서 포트를 열고 요청을 기다린다 — 이 차이를 진입점(main)과 kbap의 실행 파일 2개로 체감시킨다.

**목차 초안**
1. 한 줄 요약
2. 개념표 (프로세스 / 포트 / 진입점 main / bootJar)
3. 원리 — 앱을 "배포하고 끝"(FE) vs "띄워두고 대기"(서버). 요청이 오기 전에도 프로그램이 살아 있다
4. 진입점 `main`과 `@SpringBootApplication` — FE에 없는 "여기서 프로그램이 시작된다"는 단일 시작점
5. 포트와 프로필 — 서버는 포트(예: 8080)에서 듣는다, 환경별 프로필 4종(local/dev/staging/prod) 개념만 맛보기(상세는 2단계)
6. kbap는 왜 실행 파일이 2개인가 — web(`app:api`)과 배치(`app:batch`). 사용자 요청을 받는 앱과 정해진 작업을 도는 앱의 분리
7. 실행 명령어 (`./gradlew :app:api:bootRun` 등)
8. 치트시트 / 용어집 / 체크리스트

**근거 kbap-server 파일**
- `app/api/src/main/kotlin/com/meogo/MeogoApiApplication.kt` — 진입점 실물(9줄, 발췌에 딱 좋음: `@SpringBootApplication` + `runApplication`)
- `app/batch/src/main/kotlin/com/meogo/app/batch/MeogoBatchApplication.kt` — 두 번째 bootJar 진입점
- `app/api/src/main/resources/application.yml` — `server:` 블록(포트 등), 프로필 import
- `CLAUDE.md` "개요"(bootJar 2개 근거)·"명령어"(bootRun)·"실행 프로필 4종"
- ADR: `docs/adr/0001-multi-app-modular-layout.md`(앱 2개로 나눈 결정)

**연결**: 이후 → `[[요청 하나에 스레드 하나 — JVM 서버의 동시성과 Node 이벤트루프의 차이]]`. 2단계 예고 → "이 뜬 서버 안에서 객체를 조립하는 게 스프링".

---

## 노트 2 — 요청 하나에 스레드 하나 — JVM 서버의 동시성과 Node 이벤트루프의 차이

- **파일명(=제목)**: `요청 하나에 스레드 하나 — JVM 서버의 동시성과 Node 이벤트루프의 차이.md`
- **담당**: concept-writer
- **한 줄 의도**: Node는 싱글 스레드 이벤트루프였다. JVM 서버는 요청마다 스레드풀에서 스레드를 꺼내 쓴다 — 이 모델 차이와, 그래서 "블로킹 DB 호출"이 JVM에선 왜 문제가 아닌지를 잡는다.

**목차 초안**
1. 한 줄 요약
2. 개념표 (스레드 / 스레드풀 / 블로킹 vs 논블로킹 / 이벤트루프)
3. 원리 — 요청 하나의 여정: 소켓 수신 → 스레드 배정 → 컨트롤러 실행 → 응답 → 스레드 반납
4. 스레드풀 vs 이벤트루프 표 (Node 싱글스레드 논블로킹 ↔ JVM 요청당 스레드). "어디까지 같고 어디서 다른지" 한 줄 필수
5. 블로킹 호출이 JVM에선 왜 괜찮은가 — DB 조회로 스레드가 멈춰도 다른 스레드가 다른 요청을 처리. Node라면 이벤트루프가 막힌다
6. 가상 스레드 살짝 — kbap의 LLM fan-out이 JDK21 가상 스레드를 쓰는 이유(스레드가 싸져서 많이 띄움). 상세는 5단계 배치 흐름
7. 주의점 — 여러 스레드가 같은 객체를 건드리면? 공유 상태·스레드 안전 감만(깊이는 뒤로)
8. 치트시트 / 용어집 / 체크리스트

**근거 kbap-server 파일**
- `app/api/src/main/resources/application.yml` — `server:` 블록(내장 톰캣이 스레드풀로 요청 처리)
- `app/api/src/main/kotlin/com/meogo/app/api/scan/ScanController.kt` — 요청이 들어오는 진입 지점 실물(한 요청 = 한 핸들러 호출)
- `CLAUDE.md` 기술 스택 — "fan-out은 JDK21 가상스레드 + `CompletableFuture`"(가상 스레드 근거)
- (개념 대비용 FE 지식: Node 이벤트루프 — 독자 기존 지식, 코드 인용 불요)

**연결**: 이전 → `[[서버는 켜두는 프로그램 — 서버 프로세스와 두 개의 실행 파일]]`, 이후 → `[[JVM 위의 Kotlin — 컴파일·타입·널 안전이 TypeScript와 다른 점]]`.

---

## 노트 3 — JVM 위의 Kotlin — 컴파일·타입·널 안전이 TypeScript와 다른 점

- **파일명(=제목)**: `JVM 위의 Kotlin — 컴파일·타입·널 안전이 TypeScript와 다른 점.md`
- **담당**: concept-writer
- **한 줄 의도**: JS는 브라우저/Node 런타임에서 그대로 돌았다. Kotlin은 JVM용 바이트코드로 컴파일돼 돈다. 컴파일 단계가 잡아주는 안전망(특히 널 안전)과 Gradle 멀티모듈을 FE의 TS·모노레포와 대비한다.

**목차 초안**
1. 한 줄 요약
2. 개념표 (JVM / 바이트코드 / 컴파일 / Kotlin↔Java / Gradle 멀티모듈)
3. 원리 — JVM이란: "한 번 컴파일하면 어디서든 도는" 실행 엔진. 브라우저 JS 엔진과 대비
4. Kotlin과 Java 생태계 — Kotlin은 Java 라이브러리(Spring 등) 위에서 돈다. TS가 JS 생태계를 쓰는 것과 닮은 관계
5. 컴파일 언어의 안전망 — 실행 전에 타입 오류를 잡음. TS도 그렇지만 JVM은 런타임 타입이 진짜로 강제됨(한 줄 차이 설명)
6. 널 안전 — Kotlin `String?` vs `String`. TS `strictNullChecks`와 대비하되, kbap는 `-Xjsr305=strict`로 Java API 널까지 강제
7. Gradle 멀티모듈 — `core/`·`infra/`·`app/`로 쪼갠 구조는 pnpm workspace/모노레포 패키지와 같은 발상. `libs.versions.toml`이 버전 단일 출처(모노레포의 루트 의존성 관리와 대비)
8. 치트시트 / 용어집 / 체크리스트

**근거 kbap-server 파일**
- `gradle/libs.versions.toml` — 버전 카탈로그(버전 단일 출처)
- `settings.gradle.kts` — 멀티모듈 등록(어떤 모듈들이 한 빌드인지)
- `buildSrc/src/main/kotlin/meogo.kotlin-common.gradle.kts` — Java 21 toolchain·Kotlin 엄격성 플래그(`-Xjsr305=strict` 등) 적용 지점
- `docs/guides/gradle-made-easy.md` — Gradle 입문 설명서(멀티모듈 비유 보강용)
- `CLAUDE.md` "기술 스택"(Kotlin 2.3 / JVM Java 21)·"컴파일러 엄격성 플래그"·"빌드 구성"

**연결**: 이전 → `[[요청 하나에 스레드 하나 — JVM 서버의 동시성과 Node 이벤트루프의 차이]]`. 2단계 예고 → "이 모듈들 안의 객체를 스프링이 어떻게 조립하나".

---

## 작성자 공통 지침 (concept-writer가 따를 것)
- `study-note-style` 스킬 규약 준수: FE 비유마다 "어디까지 같고 어디서 다른지" 한 줄, 새 용어는 첫 등장 시 정의, 실코드 인용은 `경로:줄` + 발췌 15줄 이내.
- kbap-server는 **Kotlin 주석 금지 레포** — 발췌에 주석 달지 말고 발췌 아래에 줄 단위 풀이.
- frontmatter: `tags: [kbap-backend, 기초]`, `생성일: 2026-07-14`, `상태: 완료`.
- 노트 완성 후 `vault-bookkeeping` 절차(복리 링크·홈 등록·작업 로그·린트) 수행.
- 완성되면 공부 지도의 1단계 체크박스 3개를 체크한다.
</content>
