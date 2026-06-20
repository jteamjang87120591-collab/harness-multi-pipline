# 팀 간 계약(Artifact Contract) 가이드

한 스테이지의 출력 양식과 다음 스테이지의 입력 양식을 잇는 명시적 계약의 포맷·버전·검증 방법. SKILL.md Phase 2·5의 보충 레퍼런스.

## 목차
1. [왜 계약인가](#1-왜-계약인가)
2. [계약 파일 포맷](#2-계약-파일-포맷)
3. [의미 계약 — 타입 너머](#3-의미-계약)
4. [버전 관리 (semver)](#4-버전-관리)
5. [검증 — 경계면 교차 비교](#5-검증)

---

## 1. 왜 계약인가

파이프라인에서 가장 흔한 고장은 **두 팀이 각자 "올바르게" 일했는데 연결 지점에서 양식이 어긋나는 것**이다. 조사팀이 `{ "findings": [...] }`로 냈는데 설계팀이 `[...]`를 기대하면, 둘 다 버그가 없어도 파이프라인은 깨진다.

계약은 이 연결 지점을 **팀을 만들기 전에 미리 못박는** 장치다. 계약이 먼저 있으면 producer는 "무엇을 내야 하는지", consumer는 "무엇을 받는지"를 처음부터 알고 작업한다. 사후에 양식을 맞추는 재작업이 사라진다.

## 2. 계약 파일 포맷

- 위치: `_workspace/contracts/{from-stage}__to__{to-stage}.md` (예: `01-research__to__02-design.md`)
- 구조: **frontmatter(기계 검증용 매니페스트) + 본문(사람·에이전트가 읽는 의미 계약)**

이중 구조인 이유: frontmatter는 게이트가 "어떤 파일을 어떤 포맷으로 검증할지" 기계적으로 읽는 부분이고, 본문은 에이전트가 "무엇을 어떤 의미로 담아야 하는지" 이해하는 부분이다. 분리해야 자동 검증과 가독성을 동시에 만족한다.

```markdown
---
contract_id: 01-research__to__02-design
version: 1.0.0
producer_stage: 01-research
consumer_stage: 02-design
status: active          # draft | active | deprecated
produces:
  - path: _workspace/stage01_research/findings.json
    format: json
    schema_ref: "#출력-스키마"
consumes:
  - path: _workspace/stage01_research/findings.json
    required: true
---

# 계약: 01-research → 02-design (v1.0.0)

## 출력 스키마 (producer가 보장한다)

| 필드 | 타입 | 필수 | 설명 |
|------|------|:---:|------|
| topic | string | ✅ | 조사 주제 |
| pain_points | array<string> | ✅ | 사용자 페인포인트 (우선순위 내림차순) |
| references | array<{title,url}> | ✅ | 근거 출처 |

예시:
```json
{ "topic": "결제 UX", "pain_points": ["...", "..."], "references": [{"title":"...","url":"..."}] }
```

## 입력 스키마 (consumer가 기대한다)

출력 스키마의 **부분집합**이어야 한다. 설계팀은 `topic`, `pain_points`만 필요:

| 필드 | 타입 | 필수 |
|------|------|:---:|
| topic | string | ✅ |
| pain_points | array<string> | ✅ |

## 의미 계약
- `pain_points`는 우선순위 내림차순, 빈 배열 불가(최소 1개).
- `references`의 url은 실제 접근 가능한 절대 URL.

## 검증 체크리스트 (핸드오프 게이트에서 실행)
- [ ] produces.path 파일이 존재하고 선언된 format(json)으로 파싱된다
- [ ] 필수 필드가 모두 존재하고 타입이 일치한다
- [ ] 의미 계약 위반이 없다 (내림차순/빈 배열 등)
- [ ] consumer 입력이 producer 출력의 부분집합이다 (경계면 교차 비교)
- [ ] 래핑 불일치가 없다 ({ data: [...] } vs [...])

## 변경 이력
| 날짜 | 버전 | 변경 | 사유 |
|------|------|------|------|
| 2026-06-20 | 1.0.0 | 초기 계약 | - |
```

## 3. 의미 계약

스키마(타입)만으로는 부족하다. 타입은 맞지만 의미가 어긋나는 경우가 많기 때문이다. 예: `pain_points`가 `array<string>` 타입은 맞아도, 순서가 뒤죽박죽이면 "우선순위 상위 3개만 설계에 반영"하는 다음 팀이 엉뚱한 걸 고른다.

그래서 **타입 너머의 약속**을 명시한다:
- 정렬/순서 ("내림차순", "시간순")
- 카디널리티 ("빈 배열 불가", "최대 10개")
- 값 제약 ("url은 절대 경로", "금액은 양수")
- 단위 ("ms 단위", "KRW")

의미 계약은 자동 검증이 어렵지만, 에이전트가 읽고 따르며 게이트에서 사람이 점검할 수 있다.

## 4. 버전 관리

계약은 semver(`MAJOR.MINOR.PATCH`)로 관리한다:
- **MAJOR**: 호환이 깨지는 변경 (필드 삭제/타입 변경/의미 반전) → 양쪽 팀 모두 수정 필요
- **MINOR**: 하위 호환 추가 (선택 필드 추가) → consumer는 그대로 동작
- **PATCH**: 의미 명확화/오타 (스키마 불변)

계약을 바꿀 때는 `version`과 `변경 이력`을 함께 갱신한다. 옛 계약은 삭제하지 않고 `status: deprecated`로 표시해 추적성을 남긴다.

## 5. 검증 — 경계면 교차 비교

검증의 핵심은 **"파일이 존재하는가"가 아니라 "producer가 실제로 쓴 파일의 shape이 consumer가 기대하는 입력과 맞는가"** 이다. 이 방법론은 harness `references/qa-agent-guide.md`의 "경계면 교차 비교(Boundary Mismatch)"를 그대로 차용한다.

점검 순서:
1. **존재·파싱**: produces.path 파일이 있고 선언 format으로 파싱되는가.
2. **shape 대조**: producer가 실제로 쓴 JSON 구조 ↔ consumer 입력 스키마를 나란히 놓고 비교한다(한쪽만 보지 않는다).
3. **흔한 불일치 패턴 점검:**
   - 래핑: `{ "data": [...] }` vs `[...]`
   - 키 표기: `snake_case` vs `camelCase`
   - 단일/배열: `{...}` vs `[{...}]`
   - 페이지네이션: 전체 배열 vs `{ items, nextCursor }`
4. **의미 계약 점검**: 정렬·카디널리티 등.

위반이 발견되면 게이트를 차단한다. 깨진 입력을 다음 팀에 넘기면 그 팀이 "올바르게" 일해도 결과가 무너지기 때문이다.
