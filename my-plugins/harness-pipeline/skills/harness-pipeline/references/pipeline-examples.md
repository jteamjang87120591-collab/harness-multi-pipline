# 파이프라인 예시

3-스테이지 다중 팀 파이프라인의 텍스트 예시. 스테이지 분해 → 계약 → 핸드오프 → MCP 연동이 어떻게 맞물리는지 보여준다. (동작 코드가 아니라 설계 산출물 예시)

## 목차
1. [예시: 디자인→코드 파이프라인](#1-예시-디자인코드-파이프라인)
2. [계약 파일 전문 2개](#2-계약-파일-전문)
3. [handoff_log 예시](#3-handoff_log-예시)

---

## 1. 예시: 디자인→코드 파이프라인

**목표:** "Figma 디자인을 받아 리서치 → 디자인 스펙 추출 → 프론트엔드 코드까지 만드는 파이프라인"

### 스테이지 분해 (Phase 1)

| 스테이지 | 팀 | 아키텍처 패턴 | 산출물 | MCP |
|---------|-----|--------------|--------|:---:|
| 01-research | research-team (조사 3명) | 팬아웃/팬인 | `findings.json` | — |
| 02-design | design-team (Figma추출 + 스펙작성 2명) | 생성-검증 | `design_spec.json` | ✅ figma |
| 03-build | build-team (프론트 + QA 2명) | 생성-검증 | `src/` 코드 | — |

선형 의존: `01-research → 02-design → 03-build`. 계약 2개 필요(인접 쌍마다 1개).

### 핸드오프 흐름 (Phase 4)

```
[01-research 팀]  팬아웃/팬인으로 페인포인트 조사
   → _workspace/stage01_research/findings.json 저장
   → (게이트-OUT) 계약 01→02 출력 검증
   → TeamDelete
                ↓  contracts/01-research__to__02-design.md
[02-design 팀]   (게이트-IN 검증) findings.json Read
   → figma-extractor 가 mcp__figma__get_file 로 디자인 노드 추출
   → spec-writer 가 findings + figma 데이터로 design_spec.json 작성
   → reviewer 가 검수 (생성-검증)
   → _workspace/stage02_design/design_spec.json 저장
   → (게이트-OUT) 계약 02→03 출력 검증
   → TeamDelete
                ↓  contracts/02-design__to__03-build.md
[03-build 팀]    (게이트-IN 검증) design_spec.json Read
   → frontend 가 스펙대로 코드 생성, qa 가 검증
   → src/ 출력
   → TeamDelete
최종: src/ 를 사용자 경로로 출력, _workspace/ 보존
```

핵심: 02-design 팀의 `figma-extractor` 에이전트만 MCP에 의존한다. Figma 호출이 실패하면 `design_spec`의 필수 필드(레이아웃 토큰)가 비므로, 계약 02→03 게이트에서 차단된다(더미 스펙으로 진행하지 않는다).

## 2. 계약 파일 전문

### `_workspace/contracts/01-research__to__02-design.md`
```markdown
---
contract_id: 01-research__to__02-design
version: 1.0.0
producer_stage: 01-research
consumer_stage: 02-design
status: active
produces:
  - path: _workspace/stage01_research/findings.json
    format: json
    schema_ref: "#출력-스키마"
consumes:
  - path: _workspace/stage01_research/findings.json
    required: true
---
# 계약: 01-research → 02-design (v1.0.0)

## 출력 스키마
| 필드 | 타입 | 필수 | 설명 |
|------|------|:---:|------|
| topic | string | ✅ | 조사 주제 |
| pain_points | array<string> | ✅ | 페인포인트(우선순위 내림차순) |
| competitor_ui | array<{name,note}> | ⬜ | 경쟁사 UI 관찰 |

## 입력 스키마 (consumer가 기대 — 부분집합)
| 필드 | 타입 | 필수 |
|------|------|:---:|
| topic | string | ✅ |
| pain_points | array<string> | ✅ |

## 의미 계약
- pain_points 빈 배열 불가, 우선순위 내림차순.

## 검증 체크리스트
- [ ] findings.json 존재·파싱됨
- [ ] topic, pain_points 존재·타입 일치
- [ ] pain_points 비어있지 않음
- [ ] 래핑 불일치 없음

## 변경 이력
| 날짜 | 버전 | 변경 | 사유 |
|------|------|------|------|
| 2026-06-20 | 1.0.0 | 초기 계약 | - |
```

### `_workspace/contracts/02-design__to__03-build.md`
```markdown
---
contract_id: 02-design__to__03-build
version: 1.0.0
producer_stage: 02-design
consumer_stage: 03-build
status: active
produces:
  - path: _workspace/stage02_design/design_spec.json
    format: json
    schema_ref: "#출력-스키마"
consumes:
  - path: _workspace/stage02_design/design_spec.json
    required: true
---
# 계약: 02-design → 03-build (v1.0.0)

## 출력 스키마
| 필드 | 타입 | 필수 | 설명 |
|------|------|:---:|------|
| components | array<{name,props}> | ✅ | 만들 컴포넌트 목록 |
| layout_tokens | object | ✅ | 색/간격/타이포 토큰 (Figma에서 추출) |
| screens | array<{name,components}> | ✅ | 화면 구성 |

## 입력 스키마 (consumer가 기대 — 전체 사용)
출력 스키마와 동일.

## 의미 계약
- layout_tokens 는 Figma MCP 추출값이며 비어있으면 안 된다(필수). 비면 게이트 차단.
- components[].name 은 화면(screens)에서 참조되는 이름과 일치.

## 검증 체크리스트
- [ ] design_spec.json 존재·파싱됨
- [ ] components, layout_tokens, screens 존재·타입 일치
- [ ] layout_tokens 비어있지 않음 (Figma 추출 성공 확인)
- [ ] screens[].components 가 components[].name 의 부분집합

## 변경 이력
| 날짜 | 버전 | 변경 | 사유 |
|------|------|------|------|
| 2026-06-20 | 1.0.0 | 초기 계약 | - |
```

## 3. handoff_log 예시

`_workspace/handoff_log.md` — Phase 0 재개의 기준이 된다.

```markdown
# 핸드오프 로그

| 날짜 | 스테이지 | 결과 | 산출물 |
|------|---------|------|--------|
| 2026-06-20 | 01-research 완료 | 계약 01-research__to__02-design v1.0.0 충족 | _workspace/stage01_research/findings.json |
| 2026-06-20 | 02-design 완료 | 계약 02-design__to__03-build v1.0.0 충족 | _workspace/stage02_design/design_spec.json |
```

이 로그가 "02-design까지 완료"를 알려주므로, 재실행 시 Phase 0이 **03-build부터 재개**한다.
