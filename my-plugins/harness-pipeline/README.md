# harness-pipeline

여러 에이전트 팀을 **순차로** 잇는 팀 레벨 파이프라인을 설계하는 Claude Code 메타 스킬 플러그인. 한 팀이 끝내면 산출물을 내고, 다음 팀이 **계약(artifact contract)** 으로 그것을 받아 작업을 이어간다. Figma 같은 **MCP 외부 툴** 연동도 지원한다.

> harness 플러그인을 학습하며 한 단계 확장해 만든 메타 스킬입니다. 동작 예제가 아니라 "이런 파이프라인 하네스를 생성하는 법"을 가르치는 가이드(메타 지침)입니다.

## harness vs harness-pipeline

| | harness | harness-pipeline |
|---|---|---|
| 정체 | 팀 **빌더** (에이전트 레벨) | 팀 **시퀀서** (팀 레벨) |
| 만드는 것 | 한 팀 (에이전트들 + 오케스트레이터) | 여러 팀 + 팀 간 계약 + 핸드오프 |
| 데이터 전달 | 팀 내부 (SendMessage) | 팀 간 (파일 + 계약) |
| 단위 | Phase = 작업 단계 | 스테이지 = 팀 1개 |
| 관계 | — | **각 스테이지 = harness 1회 실행** |

> 단일 팀으로 충분한 작업은 harness를 쓰세요. 전문성·산출물이 확연히 다른 여러 팀을 순차로 이어야 할 때 harness-pipeline을 씁니다.

## 3가지 핵심 능력

1. **다중 팀 순차 파이프라인** — "세션당 1팀" 제약 하에서 `팀A → TeamDelete → 팀B` 순차 핸드오프
2. **팀 간 계약** — 출력 양식 ↔ 입력 양식을 명시 계약으로 못박아 안정적으로 연결 (frontmatter + 의미 계약)
3. **MCP 외부 툴 연동** — settings.json 서버 등록 + 에이전트 `## MCP 도구 의존성` 선언 (Figma 등)

## 구조

```
harness-pipeline/
├── .claude-plugin/plugin.json
├── README.md
└── skills/harness-pipeline/
    ├── SKILL.md                       # 메타-오케스트레이션 워크플로우 (7-Phase 확장)
    └── references/
        ├── stage-handoff-guide.md     # 다중 팀 순차 핸드오프 모델
        ├── artifact-contract-guide.md # 팀 간 계약 포맷·버전·검증
        ├── mcp-integration-guide.md   # MCP 서버 등록 + 에이전트 의존성 선언
        └── pipeline-examples.md       # 3-스테이지 파이프라인 예시
```

## ⚠️ 선행 요구사항 (설치 전 반드시 확인)

1. **harness 플러그인 먼저 설치.** harness-pipeline은 각 스테이지를 **harness로 빌드**한다("각 스테이지 = harness 1회 실행"). harness가 없으면 동작하지 않는다.
2. **에이전트 팀 기능 활성화** — 각 팀원이 셸에서 직접 설정해야 한다:
   ```shell
   export CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1
   ```
   영구 적용은 `~/.zshrc` / `~/.bashrc`에 추가.
3. **MCP 연동 시** 해당 MCP 서버를 `.claude/settings.json`에 등록하고 토큰을 환경변수로 둔다(커밋 금지).

> **실험적 의존성 경고:** 이 플러그인은 실험적 기능(Agent Teams) + 다중 팀 + MCP에 의존한다. Claude Code 버전·환경에 따라 동작이 달라질 수 있다. 팀 도입 전 한 명이 먼저 검증할 것을 권한다.

## 설치 (깃 저장소에서)

```shell
# 1. 마켓플레이스 추가 (저장소 URL은 실제 값으로 교체)
claude plugin marketplace add jteamjang/harness-pipeline
# 2. 플러그인 설치
claude plugin install harness-pipeline@harness-pipeline-marketplace
# 3. 에이전트 팀 활성화
export CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1
```

로컬에서 먼저 시험하려면: `claude plugin link ./harness-pipeline`

## 사용

```
다중 팀 파이프라인 설계해줘 — 리서치 → 디자인 → 빌드를 팀마다 넘기는 흐름
Figma 디자인을 받아 스펙 추출하고 코드까지 만드는 파이프라인 하네스 구성해줘
```

## 라이선스

MIT
