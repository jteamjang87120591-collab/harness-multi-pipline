# code-review-checklist

변경된 코드를 받아 **버그·가독성·테스트·보안** 4관점의 코드 리뷰 체크리스트를 생성하는 Claude Code 스킬 플러그인. PR을 올리기 전 자가 점검용.

> harness 플러그인 구조를 학습하며 만든 단일 스킬 예제입니다.

## 핵심 기능

- 변경 범위(파일/줄/성격)를 한 줄로 요약
- 4관점에서 **변경에 실제 해당하는** 점검 항목만 도출 (무관한 일반론 배제)
- 리뷰어가 예/아니오로 답할 수 있는 구체적 체크리스트 출력

## 구조

```
code-review-checklist/
├── .claude-plugin/
│   └── plugin.json                    # 플러그인 매니페스트
├── skills/
│   └── code-review-checklist/
│       ├── SKILL.md                   # 스킬 정의 (트리거 + 작동 방식)
│       └── references/
│           └── examples.md            # 관점별 항목 예시 (필요 시 로드)
└── README.md
```

## 설치 (로컬 링크)

```shell
claude plugin link ./my-plugins/code-review-checklist
```

## 사용법

Claude Code에서 변경 내용과 함께 다음처럼 요청한다:

```
이 diff 코드 리뷰 체크리스트 만들어줘
머지 전에 이 변경 점검해줘 — 뭐 놓친 거 없나?
```

## 라이선스

MIT
