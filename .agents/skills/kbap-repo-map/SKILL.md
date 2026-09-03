---
name: kbap-repo-map
description: kbap-server 레포 탐색 지도. kbap-server 코드를 읽거나 인용하거나 요청 흐름을 추적하기 전에 읽는다. 모듈 구조·의존 방향·핵심 파일 위치·문서 읽기 순서를 담는다.
---

# kbap-server 레포 지도

레포 루트: `/Users/yejinkim/dev/kfood/kbap-server` (**읽기 전용** — 절대 수정 금지). 내부 이름은 `meogo`(패키지 `com.meogo.*`)다.

## 먼저 읽을 문서 (순서대로)

1. `AGENTS.md` — 모듈 구조·컨벤션·고정 규약의 단일 요약. 가장 신뢰.
2. `docs/architecture/meogo-api-module-structure.md`, `docs/architecture/meogo-conventions.md`
3. `docs/adr/` — 결정 근거 11건 (특히 0008 모듈러 모놀리스, 0006 persistence 어댑터, 0010 infra:llm)
4. 기능별 상세는 `specs/{번호-슬러그}/`의 spec.md → plan.md → tasks.md

## 모듈 지도 (의존 방향은 단방향)

```
:common  ← 모두가 의존 가능, 아무도 의존 안 함 (Spring-free)
core:kernel ← core:{food,member,scan,avoidance,research,review} ← application:client
   ▲    ▲                                                            ▲
   │  infra:persistence (JPA, 도메인 port 구현)                        │
   └──────┴──── app:api (web bootJar, 조립+Flyway owner) · app:batch (배치 bootJar)
```

- **도메인(core:\*)은 ORM-free·Spring-free** — 순수 Kotlin model + port 인터페이스 + 정책.
- **JPA 전부**(엔티티·리포지토리·어댑터·BaseEntity)는 `infra/persistence`, 패키지 `com.meogo.infra.persistence.<도메인>`.
- LLM 어댑터는 `infra/llm` (Spring AI, 소비자는 app:batch뿐).
- 파일 수 감: application/client 64 · app/api 64 · infra/persistence 33 · infra/llm 23 · core 합계 ~74 · app/batch 9.

## 요청 흐름 추적 시작점

- 컨트롤러: `app/api/src/main/kotlin/com/meogo/app/api/<도메인>/` — 경로 상수 `ApiPaths.V1`(`/api/v1`), 응답은 전부 `ResponseEntity<BaseResponse<T>>`.
- 유스케이스: `application/client/src/main/kotlin/com/meogo/application/client/`
- 도메인: `core/<도메인>/src/main/kotlin/com/meogo/core/<도메인>/`
- 영속: `infra/persistence/src/main/kotlin/com/meogo/infra/persistence/<도메인>/` — `*JpaEntity`(`toDomain()`/`from()`), `*RepositoryAdapter`
- 스키마: `app/api/src/main/resources/db/migration/` (Flyway, timestamp 버전)
- 배치: `app/batch/src/main/kotlin/com/meogo/app/batch/`

## 인용 시 유의할 고정 규약 (노트에서 설명거리가 되는 것들)

- Kotlin 소스 주석 전면 금지(self-documenting) — 발췌에 주석이 없는 이유.
- 모든 연관관계 `FetchType.LAZY` + fetch join 명시 조회.
- `BaseEntity`: id·status(소프트 삭제 `@SQLRestriction`)·createdAt/updatedAt 공통 제공.
- 테스트는 전부 Kotest `BehaviorSpec`, given/when/then 한국어.
- 아키텍처 경계는 ArchUnit 테스트(`app/api/.../architecture/ModuleBoundaryTest.kt`)로 강제.
- SpecKit(SDD): 기능마다 specs/에 spec→plan→tasks, TDD Red→Green→Refactor. 레포 자체 하네스는 `AGENTS.md`의 "하네스: SpecKit·TDD 개발 팀" 참고(공부 하네스와 별개).
