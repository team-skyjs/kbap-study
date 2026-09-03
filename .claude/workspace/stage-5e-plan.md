# 5단계 마지막 묶음(5e) 계획서 · 실코드 흐름 — 배치 LLM 스코어링(kb-53) · 커리큘럼 완주

- **대상 독자**: FE(React Native/Expo)만 해온 개발자. 1~4단계 개념 + 5a~5d(온보딩·스캔·JWT·홈·랭킹) 이수. **5단계 규칙: 새 개념 설명 금지 — 기존 노트로 `[[번호. 링크]]`.**
- **담당**: `code-flow-writer` (2개 노트).
- **노트 골격**: ① 이 잡/파이프라인이 하는 일(한 줄) ② 흐름 표(계층/모듈 | 파일:줄 | 하는 일) ③ 경계마다 "여기서 모듈이 바뀌는 이유" ④ 입출력(프롬프트·JSON·산출 DTO) ⑤ 관련 테스트가 보장하는 것 ⑥ spec·ADR 링크.
- **제약**: kbap-server 읽기 전용. 모든 단계 `경로:줄` 근거. 상상 금지. 완성 노트는 iCloud 볼트 `kbap 백엔드/`에만.
- **번호 규칙**: 기존 18노트(`1.`~`18.`). **이번 새 노트는 `19.`·`20.`.**
- **생성일**: 2026-07-16(오늘). **근거 파일 전부 실재 확인 완료**(2026-07-16). spec `specs/kb-53-llm-avoidance-scoring/` 실존 확인.

## 묶음 구성 근거 (한 줄)
**2노트로 분할.** 이 흐름은 개념 4개(두 번째 bootJar·조사 대기열 순회 / 가상 스레드 fan-out / Consensus 집계 / 프롬프트·파싱 계약)를 지나 한 노트에 "3개 이상 욱여넣기" 선을 넘는다. 접합선이 깨끗하다 — **"배치가 어떻게 돌고 무엇을 순회하나"(실행 골격, note 1 빚)** 와 **"한 청크가 3모델을 거쳐 신뢰도가 되기까지"(파이프라인, note 2 빚)** 로 갈리고, 각기 다른 forward-예고를 갚는다.

---

> [!note] 실측 발견 1 — 이 잡은 DB에 저장하지 않는다 (저장은 kb-54, 범위 밖)
> kb-53은 "저장 직전 산출물"(`FoodScoringResult`)까지만 책임진다(spec Clarifications). 코드도 그렇다 — `ScoringJobRunner`는 결과 개수를 로깅할 뿐(`ScoringJobRunner.kt:14-17`) score를 DB에 쓰지 않는다. **다른 흐름과 달리 persistence 쓰기로 끝나지 않는다.** 노트는 "여기서 끝, 저장·위험도 반영은 kb-54/kb-9의 몫"이라 명확히. (배치가 `infra:persistence`를 쓰는 건 음식 대기열을 **읽기** 위해서지 score 저장이 아님.)

> [!note] 실측 발견 2 — 위험도(SAFE/CAUTION/DANGER)는 kb-53이 안 만든다
> kb-53은 `inclusionConfidence`(정수 1~100)만 산출한다. note 14 스캔이 쓰는 위험도(`Food.overallRisk`)는 이 신뢰도를 (저장·kb-9 정책을 거쳐) 소비하는 별개다. → note 14 링크는 "이 배치가 스캔 위험도의 **먼 원천**"으로만, 단 **저장(kb-54) 미구현이라 현재는 직접 연결 안 됨**을 분명히(과장 금지).

> [!note] 실측 발견 3 — 부분 실패 = 전부 재조사 (코드=개정 스펙, 낡은 건 Jira DoD뿐)
> `confirmed = failures.isEmpty() && parsed.size==3 && 모든 음식 커버`(`AvoidanceScoringJob.kt:88-90`). 한 모델이라도 실패/파싱불가/커버리지 미달이면 청크 전체를 `FAILED`로 두고 확정 안 함(재조사로 남김, :91-101). spec은 이 정책으로 이미 개정됨("3개 모두 취합돼야 확정 — 기존 '≥1 성공이면 완결'을 대체, ⚠️ Jira DoD 갱신 필요"). 코드=개정 스펙 일치, 낡은 건 Jira DoD. `> [!note]`로 짚기(`스펙과 다름` 아님).

---

## 노트 19 — 배치 실행 골격 — 두 번째 실행 파일과 조사 대기열 순회

- **파일명(=제목)**: `19. 배치 실행 골격 — 두 번째 실행 파일과 조사 대기열 순회.md` · H1 동일
- **담당**: code-flow-writer
- **한 줄**: web 서버(계속 떠서 요청 대기)와 달리, 배치는 **한 번 돌고 끝나는 두 번째 실행 파일**이다. 켜지면 조사 대기열의 음식을 10개씩 청크로 끝까지 순회하며 스코어링 잡을 돌리고 종료한다.

**흐름 (모듈/계층, 파일:줄, 실재 확인)**
| 계층/모듈 | 파일 | 하는 일 |
|---|---|---|
| batch 진입점 | `app/batch/.../MeogoBatchApplication.kt:7-13` | 두 번째 `@SpringBootApplication`(별도 bootJar). `scanBasePackages="com.meogo"`로 필요한 모듈 조립 |
| 실행 트리거 | `app/batch/.../scoring/ScoringJobRunner.kt:8-18` | `ApplicationRunner` — 부팅 완료 후 `job.run()` 1회 실행, total/scored/failed 로깅. **결과 DB 저장 없음**(실측1) |
| 게이팅 | `app/batch/.../scoring/ScoringJobConfig.kt:46-48` | `@ConditionalOnProperty(meogo.scoring.runner.enabled=true)` — 기본 off, 부팅만 하고 잡은 안 돎(안전). `@Bean` 조립(:17-44) |
| 설정 | `app/batch/.../resources/application.yml:27-40` | 실행법 주석(`--meogo.scoring.runner.enabled=true`), `chunk-size:10`(`@Value`, :34), LLM enabled 기본 off. 프로필은 런타임 `SPRING_PROFILES_ACTIVE`(:78-80) |
| 잡 순회 | `app/batch/.../scoring/AvoidanceScoringJob.kt:33-52` | 후보 성분 로드(비면 fail-fast) → `foodScoringSource.nextChunk(page, chunkSize)` 페이지 순회, 빈 청크·중복이면 종료, 청크마다 `scoreChunk`(→note 20) |
| 대기열 port | `core/food/.../FoodScoringSource.kt:3-5` | `nextChunk(page, size): List<Food>` — 조사 대상 음식 큐(구현은 infra adapter, batch가 runtimeOnly 조립) |
| 모듈 조립 | `app/batch/build.gradle.kts:10-17` | `:infra:llm`·`:core:research`·`:core:food`·`:core:avoidance` 직접 의존, `:infra:persistence` **runtimeOnly**(대기열 읽기용). **flyway 미의존**(실측 — 아래) |

**개념 노트 링크 맵** (각 구간 → 번호 노트)
- **왜 두 번째 실행 파일인가** — web(상주 서버)과 batch(1회 실행 후 종료)의 수명 차이 → `[[1. 서버는 켜두는 프로그램 — 서버 프로세스와 두 개의 실행 파일]]` **(예고 갚기, 핵심)**
- Gradle 멀티모듈을 batch가 골라 조립 → `[[3. JVM 위의 Kotlin — 컴파일·타입·널 안전이 TypeScript와 다른 점]]`
- `@Bean` 조립·`ApplicationRunner`·생성자 주입 → `[[4. IoC 컨테이너와 Bean — 부품을 직접 만들지 않는다]]`, `[[5. 의존성 주입(DI)과 계층 구조 — 코드를 어떻게 나누나]]`
- `@ConditionalOnProperty` 게이팅·`@Value` chunk-size·프로필 → `[[6. 설정과 프로필, 트랜잭션 첫걸음]]`
- **batch는 flyway를 의존하지 않는다** — 스키마 마이그레이션은 web(api) 한 곳이 소유(`app/api/build.gradle.kts:22-25`가 flyway 의존, batch build엔 없음). batch는 같은 DB를 **읽기만**(runtimeOnly persistence) → `[[9. Flyway 마이그레이션 — 스키마를 코드로 관리하기]]` **(핵심)**
- batch가 core/infra 모듈을 조립하되 도메인은 순수 → `[[10. 클린 아키텍처와 ports & adapters — 도메인을 순수하게 지키기]]`, `[[11. 모듈러 모놀리스와 DDD — 경계를 코드로 지키기]]`
- 잡·게이팅 테스트 → `[[12. SDD와 TDD — 스펙과 테스트로 개발하기]]`

**forward (본문 말미)**: 청크 1개가 실제로 어떻게 스코어링되나 → **노트 20**.

**테스트가 보장하는 것**: `app/batch/.../scoring/ScoringJobConfigGatingTest.kt`(enabled=false면 runner 빈 없음), `ScoringJobRunnerTest.kt`(1회 실행·집계 로깅), `MeogoBatchApplicationTests.kt`(부팅), `AvoidanceScoringSmokeTest.kt`.

**근거 spec/ADR**: `specs/kb-53-llm-avoidance-scoring/`(개요·Clarifications·quickstart 실행법). 모듈 조립 → ADR-0001(멀티앱), ADR-0008/0010(배치 모듈 직접 의존, build 주석 근거).

---

## 노트 20 — LLM 스코어링 파이프라인 — 한 청크가 3모델 병렬을 거쳐 신뢰도가 되기까지

- **파일명(=제목)**: `20. LLM 스코어링 파이프라인 — 한 청크가 3모델 병렬을 거쳐 신뢰도가 되기까지.md` · H1 동일
- **담당**: code-flow-writer
- **한 줄**: 음식 10개 청크와 기피성분 81종 후보를 한 프롬프트로 묶어 **3개 LLM 모델에 가상 스레드로 동시 호출**하고, 세 응답을 성분별 하나의 신뢰도(1~100)로 **Consensus Ensemble** 집계한다. 셋 다 성공해야 확정.

**흐름 (모듈/계층, 파일:줄, 실재 확인)**
| 계층/모듈 | 파일 | 하는 일 |
|---|---|---|
| 청크 처리 | `app/batch/.../scoring/AvoidanceScoringJob.kt:59-105` | 프롬프트 빌드 → fan-out → 파싱 → 커버리지 검사 → confirmed면 집계, 아니면 전부 FAILED |
| 프롬프트 | `core/research/.../prompt/ScoringPromptFactory.kt` | 음식 목록 + 81종 후보 + 응답 형식(포함된 것만·score 0/1/2·probability 1~100 강제)을 프롬프트로 |
| fan-out 클라이언트 | `infra/llm/.../client/LlmFanoutClient.kt:20-43` | 모델별 `CompletableFuture.supplyAsync({caller.call(req)}, executor)` + `orTimeout(30s)` → `join`, 성공/실패 격리(`LlmFanoutResult`) |
| **가상 스레드 executor** | `infra/llm/.../config/LlmConfiguration.kt:72` | `Executors.newVirtualThreadPerTaskExecutor()` — 3개 blocking LLM 호출을 값싼 가상 스레드로 병렬 |
| 모델 caller port/adapter | `infra/llm/.../client/LlmModelCaller.kt`, `provider/SpringAiModelCaller.kt` | `call(request): String` 원시 문자열 in/out. 구현은 Spring AI(모델별 caller 리스트) |
| 파싱 | `core/research/.../parse/ScoringResponseParser.kt`, `ModelScoring.kt` | 모델 JSON → (음식→성분→{score,probability}) + 이름번역·설명. 범위이탈·미지코드 규칙 처리, 없는 성분=미포함 |
| **집계(도메인)** | `core/research/.../ensemble/ConsensusEnsembleAggregator.kt:13-77` | `base = 0.6·(avgScore/2) + 0.4·(avgProb/100)`, `agreementFactor`(3동일 1.0/2동일 0.9/상이 0.75), `inclusionConfidence = round(base·factor·100).coerceIn(1,100)`. 순수 Kotlin 계산 |
| 산출 DTO | `core/research/.../ensemble/FoodScoringResult.kt`, `FoodInclusionScore.kt`, `FoodScoringStatus.kt` | `SCORED`/`FAILED` + 성분별 신뢰도 + 이름번역·설명. **저장 안 함(kb-54)** |
| 후보 입력 | `core/avoidance/.../AvoidanceSubstanceRepository.kt`(`findByCodes`), `core/research/.../input/CandidateSubstance.kt` | 81종 후보(코드+한국어 라벨) |

**개념 노트 링크 맵**
- **3개 blocking LLM 호출을 가상 스레드로 병렬** — note 2의 스레드풀 vs 이벤트루프에 이어 **세 번째 모델(가상 스레드)** → `[[2. 요청 하나에 스레드 하나 — JVM 서버의 동시성과 Node 이벤트루프의 차이]]` **(예고 갚기, 핵심)**
- `@Bean` executor·fanout client 조립 → `[[4. IoC 컨테이너와 Bean — 부품을 직접 만들지 않는다]]`, `[[5. 의존성 주입(DI)과 계층 구조 — 코드를 어떻게 나누나]]`
- `LlmModelCaller` port(infra), 집계는 `core/research`의 순수 도메인 — 네트워크(infra)와 계산(core)의 분리 → `[[10. 클린 아키텍처와 ports & adapters — 도메인을 순수하게 지키기]]`
- **`core/research`는 새 도메인 모듈** — 앙상블 공식·파싱이 Spring/네트워크 없이 core에 → `[[11. 모듈러 모놀리스와 DDD — 경계를 코드로 지키기]]`
- 산출 신뢰도가 (저장 kb-54·위험도 kb-9을 거쳐) 스캔 위험도의 **먼 원천**(현재 저장 미구현이라 직접 연결 안 됨) → `[[14. 메뉴판 스캔 판정 흐름 — 메뉴명이 위험도가 되기까지]]`
- consensus 공식·파서·프롬프트 단위 테스트 → `[[12. SDD와 TDD — 스펙과 테스트로 개발하기]]`

**back (본문 말미)**: 이 파이프라인을 돌리는 배치 실행·순회 골격 → **노트 19**.

**테스트가 보장하는 것**: `core/research/.../ensemble/ConsensusEnsembleAggregatorTest.kt`(공식·agreement·clamp), `FoodContentSelectorTest.kt`(번역·설명 한 모델 채택), `parse/ScoringResponseParserTest.kt`(포함만·미지코드·범위), `prompt/ScoringPromptFactoryTest.kt`, `infra/llm/.../client/LlmFanoutClientTest.kt`(병렬·타임아웃·실패 격리), `app/batch/.../scoring/AvoidanceScoringJobTest.kt`(부분 실패=전부 FAILED).

**근거 spec/ADR**: `specs/kb-53-llm-avoidance-scoring/`(spec US1~·Clarifications=공식 α=0.6·agreement·1~100 clamp·부분실패 정책, `llm_consensus_ensemble_review.md` §4). 선행 스캐폴딩 → `specs/kb-49-llm-client-foundation/`. LLM 계약 → `docs/architecture/` 관련.

---

## 작성자 공통 지침 (code-flow-writer가 따를 것)
- `kbap-repo-map` 탐색 시작, `study-note-style` 준수(**번호 접두 포함**). **모든 단계 `경로:줄`**, 상상 금지.
- **새 개념 설명 금지** — 링크 맵대로. 새 개념(가상 스레드는 note 2로 흡수, core/research 도메인은 note 11로) 나오면 짧은 콜아웃 + "개념 노트 후보" 보고.
- 부분 실패 정책·저장 범위 밖은 **코드 기준** `> [!note]`(위 실측 1·3). 위험도 미산출은 실측 2대로 과장 없이.
- 발췌 15줄 이내, 발췌 아래 줄 단위 풀이(yml·build.gradle 주석은 인용 가능).
- frontmatter: `tags: [kbap-backend, 흐름]`, `생성일: 2026-07-16`, `상태: 완료`. 파일명·H1 `19.`/`20.` 접두.
- 노트 19가 실행 골격·순회의 주인, 노트 20이 청크 처리·집계의 주인. `AvoidanceScoringJob`을 둘이 공유하니 19는 `run()`(순회) 설명, 20은 `scoreChunk()`(처리) 설명으로 나눈다.

## vault-keeper 지시 (노트 완성 후 vault-bookkeeping)
1. **복리 (양방향)** — 각 노트 하단 역링크 append(기존 본문 불변, 한 줄):
   - `[[1. 서버는 켜두는 프로그램 …]]` ↔ 19 — note 1의 "두 번째 bootJar(batch)에서 다룸" **예고 갚기**.
   - `[[2. 요청 하나에 스레드 하나 …]]` ↔ 20 — 스레드풀/이벤트루프(2)에 가상 스레드(20) 추가.
   - `[[9. Flyway 마이그레이션 …]]` ↔ 19 — batch는 flyway 미의존, 스키마 소유는 web 한 곳.
   - `[[14. 메뉴판 스캔 판정 흐름 …]]` ↔ 20 — **간접·먼 원천**으로만(저장 kb-54 미구현). note 14 하단엔 "이 위험도가 소비하는 신뢰도의 생산 잡" 한 줄만, 과장 금지.
   - 19 ↔ 20 서로 (같은 흐름 두 노트) 양방향 필수.
2. **커리큘럼 완주 처리** — 이번이 지도 마지막 항목:
   - `🗺️ kbap 백엔드 공부 지도.md`: ⑥ 배치 LLM 체크박스 `[x]`, 링크를 `[[19. …]]`·`[[20. …]]` 두 줄로. 5단계 이후 6개 흐름 전부 완료.
   - 지도 frontmatter `상태: 진행중` → `완료`.
   - 진도 요약을 완주 문구로: "완료 노트 20 (1~4단계 개념 12 + 5단계 흐름 8[온보딩·스캔·로그인·JWT필터·홈·랭킹·배치골격·배치파이프라인]). 커리큘럼 완주 — kbap-server 합류 준비 완료."
   - **다음 후보 제안**(지도 하단 "이후 확장" 섹션에 추가, 강제 아님): ① Redis 개념 노트 승격(5b서 후보로 남긴 키-값 저장소·TTL), ② kb-54 스코어링 결과 저장·kb-9 위험도 정책 흐름(이 배치의 직접 후속), ③ 리뷰 기능 도입 시 랭킹 후속(reviewCount 살아나는 경로, 5d 콜아웃).
3. 이전 개념 노트 역링크·볼트 홈 등록·작업 로그(완주 기록)·린트 표준 절차. 새 파일명 `19.`·`20.` 번호 규칙 린트 확인.
