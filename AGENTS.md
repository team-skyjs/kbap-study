# AGENTS.md

## 이 폴더의 성격

옛 옵시디언 볼트의 **읽기 전용 백업** + 시드/브리프 수신함(`_briefs/`)이다. 정식 볼트는 iCloud(`~/Library/Mobile Documents/iCloud~md~obsidian/Documents/study`)이며, 완성 노트는 전부 그쪽에 쓴다. 이 폴더의 기존 노트는 사용자가 명시적으로 허락하기 전까지 삭제·수정 금지.

## 하네스: kbap 백엔드 공부

**목표:** FE 개발자인 사용자가 kbap-server(`~/dev/kfood/kbap-server`, 읽기 전용) 합류를 위해 알아야 할 백엔드 지식을 기초→실코드 단계별 공부 노트로 iCloud 볼트 `kbap 백엔드/`에 생성한다.

**트리거:** kbap 공부 노트의 생성·다음 단계 진행·재실행·부분 보완 요청 시(예: "다음 단계 해줘", "kbap 공부 진행", "스캔 흐름 노트 만들어줘", "JPA 노트 보완해줘") `kbap-study-orchestrator` 스킬을 사용하라. 개념 단순 질문은 직접 응답 가능. (에이전트: `.Codex/agents/`, 스킬: `.Codex/skills/`.)

**변경 이력:**
| 날짜 | 변경 내용 | 대상 | 사유 |
|------|----------|------|------|
| 2026-07-14 | 초기 구성 — curriculum-architect·concept-writer·code-flow-writer·vault-keeper 4에이전트 + 4스킬(스타일·레포지도·장부정리·오케스트레이터) | 전체 | kbap-server 합류 대비 단계별 공부 노트 하네스 요청 |
| 2026-07-15 | 파일명 번호 접두 규칙 추가 (`N. 제목`, 기존 16노트 일괄 리네임) | study-note-style 스킬 | 사이드바 가나다순에서 공부 순서가 안 보인다는 사용자 요청 |
