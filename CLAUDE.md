# CLAUDE.md

## 이 폴더의 성격

옛 옵시디언 볼트의 **읽기 전용 백업** + 시드/브리프 수신함(`_briefs/`)이다. 정식 볼트는 iCloud(`~/Library/Mobile Documents/iCloud~md~obsidian/Documents/study`)이며, 완성 노트는 전부 그쪽에 쓴다. 이 폴더의 기존 노트는 사용자가 명시적으로 허락하기 전까지 삭제·수정 금지.

## 하네스: kbap 백엔드 공부

**목표:** FE 개발자인 사용자가 kbap-server(`~/dev/kfood/kbap-server`, 읽기 전용) 합류를 위해 알아야 할 백엔드 지식을 기초→실코드 단계별 공부 노트로 iCloud 볼트 `kbap 백엔드/`에 생성한다.

**트리거:** kbap 공부 노트의 생성·다음 단계 진행·재실행·부분 보완 요청 시(예: "다음 단계 해줘", "kbap 공부 진행", "스캔 흐름 노트 만들어줘", "JPA 노트 보완해줘") `kbap-study-orchestrator` 스킬을 사용하라. 개념 단순 질문은 직접 응답 가능. (에이전트: `.claude/agents/`, 스킬: `.claude/skills/`.)

**변경 이력:**
| 날짜 | 변경 내용 | 대상 | 사유 |
|------|----------|------|------|
| 2026-07-14 | 초기 구성 — curriculum-architect·concept-writer·code-flow-writer·vault-keeper 4에이전트 + 4스킬(스타일·레포지도·장부정리·오케스트레이터) | 전체 | kbap-server 합류 대비 단계별 공부 노트 하네스 요청 |
| 2026-07-15 | 파일명 번호 접두 규칙 추가 (`N. 제목`, 기존 16노트 일괄 리네임) | study-note-style 스킬 | 사이드바 가나다순에서 공부 순서가 안 보인다는 사용자 요청 |
| 2026-08-26 | 머리말 두 줄 고정 서식 규칙 추가 (🏠홈·📂주제=지도링크 / ⬅️앞·➡️다음 / 미집필은 `(집필 예정)` 텍스트) | study-note-style 스킬 | kbap 재구축 R1·R2에서 노트마다 머리말 서식이 갈려 QA에 반복 지적 |
| 2026-08-26 | 머리 블록 고정 규칙 추가 (H1 아래 `작성·맥락/목표` + `## 0. 한 줄 요약`, 절 번호 0부터) | study-note-style 스킬 | R3에서 노트 14만 인라인 한 줄 요약·절 번호 1부터로 갈림 |
| 2026-08-27 | 응답 JSON 실측 대조 규칙 추가 (봉투는 `success·payload·message·code` 4필드, `data`·`error` 없음) | study-note-style 스킬 | 재구축 중 `"data"` 오기가 노트22·27에서 두 번 발생 — FE가 베끼면 undefined |
| 2026-09-03 | **독자 상 정정 + TS/JS 비교 금지** — 자바 CRUD 조금 해봄·내부 동작 전혀 모름 전제, FE 대비표 폐기, "다음 노트가 정본" 회피 금지 | study-note-style 스킬 | 사용자가 노트2 §7·노트3에서 막힘 — "ts 전문가 아니다, ts js 비교는 빼라" |
