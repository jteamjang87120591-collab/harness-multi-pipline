# code-review-team

보안·성능·테스트 전문 리뷰어 **에이전트 팀**이 변경된 코드를 병렬로 감사하고, 교차 토론을 거쳐 하나의 통합 리뷰 리포트를 생성하는 Claude Code 멀티 에이전트 하네스.

> harness의 팬아웃/팬인 패턴을 학습하며 만든 팀 버전 예제입니다. 단일 스킬 버전은 `../code-review-checklist`를 참고하세요.

## 단일 스킬 vs 팀 버전

| | code-review-checklist (단일) | code-review-team (팀) |
|---|---|---|
| 구성 | 스킬 1개 | 에이전트 3 + 오케스트레이터 1 |
| 동작 | 한 에이전트가 4관점 순차 | 3명이 각 관점 병렬 + 교차 토론 |
| 산출물 | 체크리스트 | 통합 리뷰 리포트(머지 판정 포함) |
| 요구사항 | 없음 | `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` |

## 구조

```
code-review-team/
├── .claude-plugin/plugin.json
├── agents/                              # 에이전트 정의 (누가)
│   ├── security-reviewer.md
│   ├── performance-reviewer.md
│   └── test-reviewer.md
├── skills/
│   └── code-review-orchestrator/        # 오케스트레이터 (누가 언제 무엇을)
│       └── SKILL.md
└── README.md
```

## 설치 & 사용

```shell
export CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1
claude plugin link ./my-plugins/code-review-team
```

```
이 PR 종합 코드 리뷰해줘
이 변경 팀으로 리뷰해줘 — 보안·성능·테스트 다 봐줘
```

## 패턴

**팬아웃/팬인 + 토론** — 리뷰어들이 리더를 거치지 않고 직접 `SendMessage`로 교차 영역 이슈(예: "이 SQL 주입 가능 → 성능·테스트도 확인")를 공유한다.

## 라이선스

MIT
