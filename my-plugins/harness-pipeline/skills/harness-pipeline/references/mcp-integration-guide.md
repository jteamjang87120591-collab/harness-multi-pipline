# MCP 외부 툴 연동 가이드

harness가 생성한 에이전트가 외부 MCP 툴(예: Figma)을 사용하도록 설계하는 방법. (a) MCP 서버 등록과 (b) 에이전트 의존성 선언 두 가지를 다룬다. SKILL.md Phase 3의 보충 레퍼런스.

## 목차
1. [MCP란 — 왜 계약과 묶는가](#1-mcp란)
2. [(a) MCP 서버 등록](#2-mcp-서버-등록)
3. [(b) 에이전트 의존성 선언](#3-에이전트-의존성-선언)
4. [실패 처리 — 외부 의존의 비결정성](#4-실패-처리)
5. [연결 테스트 (preflight)](#5-연결-테스트-preflight)
6. [다른 MCP 서버로 교체 (GitHub·DB)](#6-다른-mcp-서버로-교체)

---

## 1. MCP란

MCP(Model Context Protocol) 서버는 Claude Code에 **외부 툴**을 붙이는 표준 방식이다. Figma, GitHub, 데이터베이스 등 외부 시스템의 기능을 에이전트가 도구처럼 호출하게 해준다. 도구 이름은 `mcp__{server}__{tool}` 규칙을 따른다(예: `mcp__figma__get_file`).

파이프라인에서 MCP가 중요한 이유: 외부 데이터를 가져오는 스테이지(예: Figma 디자인 → 스펙 추출)는 그 산출물이 다음 스테이지의 **계약 입력**이 된다. 그래서 MCP는 단독으로 두지 않고 **계약과 묶어 설계한다** — MCP 호출 실패가 계약 필수 필드를 비우면 게이트에서 막아야 하기 때문이다.

## 2. MCP 서버 등록

MCP 서버는 `.claude/settings.json`의 `mcpServers` 블록에 등록한다.

**stdio 방식 (로컬 프로세스로 실행):**
```json
{
  "mcpServers": {
    "figma": {
      "command": "npx",
      "args": ["-y", "figma-mcp"],
      "env": { "FIGMA_TOKEN": "${FIGMA_TOKEN}" }
    }
  }
}
```

**SSE/HTTP 방식 (원격 서버에 연결):**
```json
{
  "mcpServers": {
    "figma": {
      "url": "https://mcp.example.com/figma/sse",
      "headers": { "Authorization": "Bearer ${FIGMA_TOKEN}" }
    }
  }
}
```

**규칙:**
- **토큰은 절대 하드코딩하지 않는다.** `${FIGMA_TOKEN}` 환경변수 참조를 쓰고, 실제 값은 셸 환경이나 시크릿에 둔다. (보안: 비밀값이 settings.json·git에 노출되면 안 된다)
- 권한 프롬프트를 줄이려면 `permissions.allow`에 `mcp__figma__*`를 추가한다:
  ```json
  { "permissions": { "allow": ["mcp__figma__*"] } }
  ```
- Figma는 예시일 뿐이다. 다른 MCP 서버도 같은 구조로 등록한다(서버 이름만 바뀜).

> 자동 설정이 필요하면 Claude Code의 `update-config` 스킬 흐름과 정합한다(settings.json 편집).

### 사용자 셋업 안내 (필수 출력)

**MCP를 쓰는 파이프라인은 이 등록을 사용자에게 반드시 안내해야 한다.** 토큰은 비밀값이라 에이전트가 파일에 넣을 수 없다(넣으면 유출). 그래서 생성된 파이프라인은 **settings.json의 `mcpServers` 블록을 `${TOKEN}` 자리표시자까지만 스캐폴드**하고, 실제 토큰 발급·환경변수 설정·권한 허용·연결 확인은 **사용자에게 체크리스트로 넘긴다.** 이 안내를 빠뜨리면 사용자는 "왜 MCP가 안 되는지" 모른 채 실행에서 실패한다(이 문서가 다루는 가장 흔한 실수).

완료 보고에 아래 형식으로 출력한다:

```markdown
## ⚙️ MCP 셋업이 필요합니다 (실행 전 1회)

이 파이프라인의 {스테이지명} 스테이지는 {서버명} MCP를 사용합니다.
실행 전에 아래를 직접 설정해 주세요(토큰은 비밀값이라 자동 설정 불가):

1. settings.json 확인 — `.claude/settings.json`에 mcpServers.{서버명} 블록이 스캐폴드돼 있습니다.
2. 토큰 발급 — {서버}에서 API 토큰을 발급받습니다.
3. 환경변수 설정 — `export {SERVER}_TOKEN="..."` (PowerShell: `$env:{SERVER}_TOKEN="..."`). 절대 파일에 직접 넣지 마세요.
4. 권한 허용 — settings.json `permissions.allow`에 `mcp__{서버}__*` (이미 스캐폴드됨).
5. 연결 확인 — 첫 실행 시 preflight가 자동으로 접근을 점검합니다(실패 시 안내).
```

## 3. 에이전트 의존성 선언

MCP 툴을 쓰는 에이전트의 정의 파일(`.claude/agents/{name}.md`)에 **`## MCP 도구 의존성`** 섹션을 추가한다. harness의 기존 6섹션(핵심역할/작업원칙/입출력/팀 통신/에러/협업)에 더해지는 신규 섹션이다.

```markdown
## MCP 도구 의존성
- 필요 서버: figma (`mcp__figma__*` 도구)
- 사용 도구: `mcp__figma__get_file`, `mcp__figma__get_node`
- 등록 위치: `.claude/settings.json` 의 `mcpServers.figma`
- 권한: `permissions.allow` 에 `"mcp__figma__*"` 등록 권장
- 폴백: 1회 재시도 후 실패 시 결과 없이 진행 + "Figma 데이터 미수집" 명시.
  단, 계약상 필수 입력이면 게이트를 차단한다(더미값 생성 금지).
```

이 섹션을 명시하는 이유: MCP 의존성이 에이전트 정의에 적혀 있어야 (1) 어떤 서버 등록이 선행돼야 하는지 한눈에 보이고, (2) 다음 세션·다른 팀에서 같은 에이전트를 재사용할 때 빠진 설정을 빨리 찾는다.

## 4. 실패 처리

MCP는 **외부 의존**이라 인증 만료·네트워크 오류·레이트 리밋으로 비결정적으로 실패한다. 내부 도구(Read/Grep)와 달리 "항상 성공"을 가정하면 안 된다. 그래서 두 가지를 설계에 박는다:

1. **의존성을 명시 선언**(위 §3) — 실패 원인을 빠르게 좁힌다.
2. **계약과 연결** — MCP 산출물이 계약 **필수** 필드면 실패 시 게이트 차단(더미값으로 채우지 않는다). **선택** 필드면 누락을 명시하고 진행한다.

**보수적 안내:** 에이전트 팀의 팀원은 독립 Claude 인스턴스다. 팀원이 리더의 `settings.json`(MCP 등록·권한)을 그대로 상속하는지는 환경/버전에 따라 다를 수 있으므로, 파이프라인 설계 시 "MCP 사용 팀원이 실제로 해당 툴에 접근 가능한지"를 첫 스테이지에서 한 번 확인(연결 테스트)하도록 권한다. 불확실하면 MCP 호출을 단일 서브 에이전트로 격리하는 방안도 고려한다. 구체 방법은 §5 참조.

---

## 5. 연결 테스트 (preflight)

팀원은 독립 Claude 인스턴스라 리더의 `settings.json`을 상속하지 못할 수 있다(§4). 이를 본 작업 전에 싸게 확인하는 것이 연결 테스트다. 소프트웨어의 헬스 체크와 같은 개념 — 비싼 작업을 시키기 전에 "콘센트에 전기가 들어오는지"만 먼저 본다.

**왜 먼저 하는가:** MCP 접근 불가를 본 작업 끝에서 발견하면, 이미 소모한 토큰·시간이 버려진다. 실패는 일찍 터뜨릴수록 비용이 작다.

### 프로브 단계 삽입

오케스트레이터 스테이지 루프에서 `TeamCreate`와 본 작업 `TaskCreate` 사이에 프로브를 끼운다:

```
2. TeamCreate("design-team", [figma-extractor, spec-writer])

2.5 (preflight) MCP 사용 팀원에게 프로브 작업만 먼저 할당:
      "mcp__figma__get_me 를 1회 호출하라.
       성공하면 'MCP_OK', 실패하면 'MCP_FAIL: <에러요약>' 을
       리더에게 SendMessage로 보고하라. 본 작업은 시작하지 마라."

3. (MCP_OK일 때만) 본 작업 TaskCreate 진행
```

**프로브 호출 선택 기준:** 부작용 없고 빠른 호출을 쓴다. Figma `get_me`(계정 정보), DB `SELECT 1`, GitHub 현재 사용자 조회 등. 디자인 전체 추출처럼 비싼 호출을 프로브로 쓰지 않는다.

### 실패 시 에스컬레이션 사다리

```
MCP_FAIL 발생
 ├─ ① 재시도: 일시적 네트워크일 수 있으니 1회 더 프로브 → 성공 시 진행
 ├─ ② 격리(isolation): 팀원이 상속을 못 받는 것이라면, MCP 호출을
 │       팀원이 아니라 리더(또는 단일 서브 에이전트)가 직접 수행한다.
 │       리더는 settings.json을 확실히 가진 메인 세션이므로 접근된다.
 │       → 리더가 데이터를 _workspace/stageNN_*/source_raw.json 에 저장
 │       → 팀원은 그 파일을 Read만 한다 (MCP 직접 호출 안 함)
 └─ ③ 폴백/차단: 격리도 불가하면 계약의 필수/선택 분기(§4)
         필수 필드 → 게이트 차단, 사용자에게 접근 불가 보고
         선택 필드 → 누락 명시 후 진행
```

**②번 격리가 실무 핵심이다.** "팀원이 MCP를 못 쓰면, 확실히 되는 리더가 대신 가져와 파일로 깔아주고 팀원은 읽기만 한다"는 우회로다. 팀의 협업 이점은 유지하면서 MCP 접근 문제만 리더로 몰아넣는다.

**한 번만 하면 된다:** 첫 MCP 스테이지에서 상속 여부가 판명되면, 같은 세션의 이후 스테이지는 그 결과를 재사용한다(매 스테이지 반복 불필요).

---

## 6. 다른 MCP 서버로 교체

Figma는 예시일 뿐이다. GitHub·DB 등으로 바꿔도 **3층 구조(서버 등록 → 에이전트 선언 → 계약 게이트)는 그대로**이며, 각 층의 내용물만 서버 특성에 맞춰 교체한다.

### 그대로 유지되는 뼈대
- `mcpServers`에 등록 (서버 이름만 변경)
- 에이전트에 `## MCP 도구 의존성` 섹션 작성
- 실패를 계약 필수/선택과 연결
- 툴 이름 규칙 `mcp__{server}__{tool}`

### 등록 블록 — 전송·인증이 다름 (예시)

```json
// GitHub (stdio + PAT)
{ "mcpServers": { "github": {
  "command": "npx", "args": ["-y", "@modelcontextprotocol/server-github"],
  "env": { "GITHUB_PERSONAL_ACCESS_TOKEN": "${GITHUB_TOKEN}" }
}}}

// Postgres DB (stdio + 연결 문자열, 비밀번호는 env로 분리 권장)
{ "mcpServers": { "postgres": {
  "command": "npx",
  "args": ["-y", "@modelcontextprotocol/server-postgres",
           "postgresql://readonly@localhost:5432/mydb"]
}}}
```

> 인증 방식이 서버마다 다르다: Figma=API 토큰(env), GitHub=PAT(env), DB=연결 문자열(args). 구체 패키지명·인자는 각 서버 문서를 따른다(위는 예시).

### 비교표

| | Figma | GitHub | DB(Postgres) |
|---|---|---|---|
| 대표 툴 | `mcp__figma__get_file` | `mcp__github__get_file`·`search_code`·`create_issue` | `mcp__postgres__query` |
| 산출물 성격 | 디자인 토큰/노드 | 코드/이슈/PR 메타 | 행(rows) 데이터 |
| 계약 스키마 예 | `layout_tokens`,`components` | `files[]`,`issues[]` | `rows[]`,`columns[]` |
| 주요 실패 모드 | 토큰 만료, 레이트 리밋 | 레이트 리밋, 권한 스코프 부족 | 쿼리 오류, 타임아웃, 결과 과대 |
| **가장 큰 위험** | 읽기 전용이라 비교적 안전 | **쓰기 가능**(이슈/PR 생성) | **파괴적 쿼리**(DELETE/DROP) |

### 가장 중요한 차이: 읽기 vs 쓰기 (안전성)

Figma는 보통 읽기 전용이라 안전하다. 그러나 GitHub·DB는 **쓰기/파괴가 가능**하므로 권한 설계가 완전히 달라진다. **기본적으로 외부 쓰기 가능 툴은 의심하고 최소 권한만 부여한다.**

```json
// GitHub — 조사용 파이프라인이면 읽기만 허용, 쓰기 차단
{ "permissions": {
  "allow": ["mcp__github__get_*", "mcp__github__search_*"],
  "deny":  ["mcp__github__create_*", "mcp__github__delete_*"]
}}
```

DB는 **이중 방어**가 정석이다: (1) MCP 권한에서 변경 툴 차단 + (2) DB 쪽 **readonly 계정**(연결 문자열의 `readonly@`). `DELETE`/`DROP`은 한 줄로 돌이킬 수 없으므로, 에이전트 정의의 작업 원칙에 "SELECT 전용, 변경 쿼리 금지"를 명시한다.

### 에이전트 `## MCP 도구 의존성` 변형 예

```markdown
# GitHub (읽기 전용 조사)
## MCP 도구 의존성
- 필요 서버: github (mcp__github__* 도구)
- 사용 도구: mcp__github__search_code, mcp__github__get_file
- 권한: allow는 get_*/search_* 만, create_*/delete_*는 deny
- 폴백: 레이트 리밋 시 백오프 후 1회 재시도. 실패 시 "GitHub 데이터 부분 수집" 명시. 필수 입력이면 게이트 차단.

# Postgres (읽기 전용 분석)
## MCP 도구 의존성
- 필요 서버: postgres (mcp__postgres__query)
- 작업 원칙: SELECT 전용. INSERT/UPDATE/DELETE/DDL 금지 (readonly 계정 사용)
- 폴백: 타임아웃 시 LIMIT 줄여 재시도. 결과 과대 시 페이지네이션. 필수 데이터 미수집 시 게이트 차단.
```

### 정리

| 층 | 교체 시 바뀌는 것 |
|----|------------------|
| 서버 등록 | 서버명·전송·인증만 교체 (구조 동일) |
| 툴 이름 | `mcp__figma__*` → `mcp__github__*` / `mcp__postgres__*` |
| 계약 스키마 | 산출물 성격이 달라 필드가 달라짐 |
| **권한·안전** | **가장 크게 달라짐** — 읽기전용 vs 쓰기/파괴 → deny + 최소권한 + (DB)readonly 계정 |
| 폴백 | 실패 모드에 맞춰 재시도 전략만 조정 |

**핵심:** 뼈대(3층)는 재사용하고, 인증 방식과 읽기/쓰기 안전성만 서버 특성에 맞춰 갈아끼운다.
