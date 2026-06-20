---
name: code-review-orchestrator
description: "보안·성능·테스트 리뷰어로 구성된 코드 리뷰 에이전트 팀을 조율하여 변경된 코드의 통합 리뷰 리포트를 생성한다. (1) '코드 리뷰 팀 돌려줘', '종합 코드 리뷰해줘' 요청 시, (2) PR/머지 전 보안·성능·테스트를 한꺼번에 점검하고 싶을 때, (3) '이 변경 팀으로 리뷰해줘'처럼 캐주얼하게 말해도 사용한다. 후속 작업: 리뷰 다시 실행, 특정 관점만 다시(보안만/성능만/테스트만), 이전 리뷰 기반 업데이트·보완·수정 요청 시에도 반드시 이 스킬을 사용한다. 단일 관점의 가벼운 점검이면 code-review-checklist 스킬을 쓰고, 코드 설명/실행 요청에는 사용하지 않는다."
---

# Code Review Orchestrator

보안·성능·테스트 전문 리뷰어 팀을 조율하여, 변경된 코드를 병렬로 감사하고 교차 토론을 거친 **통합 리뷰 리포트**를 생성하는 스킬.

## 실행 모드: 에이전트 팀

팬아웃/팬인 패턴. 3명의 리뷰어가 같은 변경을 서로 다른 관점에서 동시에 감사하고, 리더를 거치지 않고 직접 발견을 공유한다. 한 리뷰어의 발견(예: SQL 주입)이 다른 리뷰어의 점검(성능·테스트)을 실시간으로 유도하기 때문에 단독 리뷰보다 깊다.

> **요구사항:** 에이전트 팀 기능 활성화 — `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1`

## 에이전트 구성

| 팀원 | 타입 | 역할 | 출력 |
|------|------|------|------|
| security-reviewer | 커스텀 | 보안 취약점 | `_workspace/01_security_review.md` |
| performance-reviewer | 커스텀 | 성능 영향 | `_workspace/01_performance_review.md` |
| test-reviewer | 커스텀 | 테스트 커버리지 | `_workspace/01_test_review.md` |
| (리더 = 오케스트레이터) | — | 통합 리포트 | `code-review-report.md` |

## 워크플로우

### Phase 0: 컨텍스트 확인
1. `_workspace/` 존재 여부를 확인한다.
2. 실행 모드를 결정한다:
   - **미존재** → 초기 실행. Phase 1로 진행.
   - **존재 + 특정 관점만 재요청** → 부분 재실행. 해당 리뷰어만 재호출하고 그 산출물만 덮어쓴다.
   - **존재 + 새 변경 제공** → 새 실행. 기존 `_workspace/`를 `_workspace_prev/`로 이동 후 Phase 1.

### Phase 1: 준비
1. 리뷰 대상(diff/PR/파일)과 변경 성격(신규/수정/리팩터링)을 한 줄로 요약한다.
2. `_workspace/`를 만들고 변경 코드를 `_workspace/00_input/`에 저장한다.

### Phase 2: 팀 구성
1. 팀 생성 — 모든 팀원에 `model: "opus"` 명시:
   ```
   TeamCreate(team_name: "review-team", members: [
     { name: "security-reviewer",   agent_type: "security-reviewer",   model: "opus" },
     { name: "performance-reviewer", agent_type: "performance-reviewer", model: "opus" },
     { name: "test-reviewer",        agent_type: "test-reviewer",        model: "opus" }
   ])
   ```
2. 작업 등록:
   ```
   TaskCreate(tasks: [
     { title: "보안 리뷰",   assignee: "security-reviewer" },
     { title: "성능 리뷰",   assignee: "performance-reviewer" },
     { title: "테스트 리뷰", assignee: "test-reviewer" }
   ])
   ```

### Phase 3: 병렬 리뷰 + 교차 토론
**실행 방식:** 팀원들이 자체 조율.

- 3명이 `_workspace/00_input/`를 Read하여 각자 관점으로 감사한다.
- 교차 영역 이슈는 `SendMessage`로 직접 공유한다:
  - security → performance: 보안 수정의 성능 영향 확인 요청
  - performance → test: 발견한 핫패스/N+1의 테스트 유무 질의
  - test → security: 미검증 경로의 보안 우선순위 의견 요청
- 각 팀원은 완료 시 자기 파일을 저장하고 리더에게 알린다.

### Phase 4: 통합
1. 세 산출물을 Read로 수집한다.
2. 심각도/영향도/우선순위 순으로 정렬해 통합한다. 상충하는 의견은 삭제하지 않고 출처를 병기한다.
3. 최종 리포트 `code-review-report.md`를 생성한다:
   ```markdown
   # 코드 리뷰 리포트 — {변경 요약}

   ## 종합 의견 (머지 권장: ✅ / ⚠️ 조건부 / ❌ 보류)
   ## 🔒 보안   (security-reviewer)
   ## ⚡ 성능   (performance-reviewer)
   ## 🧪 테스트 (test-reviewer)
   ## 교차 발견 (리뷰어 간 토론에서 나온 항목)
   ```

### Phase 5: 정리
1. 팀원 종료 요청 후 `TeamDelete`로 팀을 정리한다.
2. `_workspace/`는 보존한다(사후 추적용).
3. 사용자에게 결과를 요약 보고한다.

## 에러 핸들링
| 상황 | 전략 |
|------|------|
| 리뷰어 1명 실패 | 1회 재시도. 재실패 시 리포트에 "{관점} 일부 미점검" 명시하고 진행 |
| 과반 실패 | 사용자에게 알리고 진행 여부 확인 |
| 리뷰어 간 의견 충돌 | 삭제하지 않고 양쪽 출처 병기 |
| 변경 입력 없음 | 어떤 diff/파일을 리뷰할지 먼저 되묻는다 (빈 입력으로 팀을 만들지 않음) |

## 테스트 시나리오
- **정상 흐름**: "이 PR 종합 코드 리뷰해줘" + diff → 팀 구성(3명) → 병렬 리뷰 + 교차 토론 → `code-review-report.md` 생성(머지 권장 판정 포함).
- **에러 흐름**: Phase 3에서 performance-reviewer 중지 → 리더가 유휴 알림 수신 → 재시도 → 실패 시 보안·테스트 결과만으로 Phase 4 진행, 리포트에 "성능 관점 미점검" 명시.
