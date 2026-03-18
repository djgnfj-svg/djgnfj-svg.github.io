---
title: Claude Code를 최고로 쓰는 워크플로우
date: 2026-03-18 00:00:00 +0900
categories: [AI]
tags: [Claude Code, workflow, productivity, Anthropic]
---

개발을 시작하기 전에 세팅해두려고 글을 남긴다. 남기는 김에 세상의 모든 정보를 정리해서 업데이트 해보고자 한다.

새 프로젝트 시작할 때 이 글 URL을 Claude Code에 주고 "이거 보고 세팅해줘"라고 할 생각이다.

Claude Code 창시자(Boris Cherny), Anthropic 내부 팀, 해커톤 우승자, 커뮤니티 파워유저들의 워크플로우를 전부 조사해서 하나로 합쳤다.


# 기본 흐름

다 아는 내용이니까 짧게. Plan Mode(탐색/계획) → Auto-Accept(구현+검증) → 커밋 → `/clear`. 검증은 반드시 같이 줘라("테스트 돌려서 실패하면 고쳐줘"). 실수하면 CLAUDE.md에 규칙 추가(Self-Correction Loop). 이게 기본이다.

> Boris Cherny: "Plan Mode에서 계획을 다듬고, Auto-Accept으로 전환하면 보통 한 번에 구현된다. **좋은 계획이 정말 중요하다!**"


# MCP 서버

Claude가 외부 서비스를 직접 다루게 한다. `.mcp.json`에 넣으면 팀 공유 가능. 안 쓰는 서버는 `/mcp`에서 끈다 — 도구 정의만으로 컨텍스트를 먹는다. **10개 이하로 유지**하는 게 좋다 (80개 넘으면 성능 저하).

## Boris Cherny (창시자)가 쓰는 MCP

Boris Cherny는 팀 `.mcp.json`에 체크인해서 공유한다:

- **Slack** — Claude가 직접 검색/포스팅
- **BigQuery** — 애널리틱스 질문에 쿼리 실행
- **Sentry** — 에러 로그 조회

> "Claude Code uses all of my tools for me — it often searches and posts to Slack, runs BigQuery queries, and grabs error logs from Sentry."

## 해커톤 우승자 (Affaan Mustafa)가 쓰는 MCP

[everything-claude-code](https://github.com/affaan-m/everything-claude-code/blob/main/mcp-configs/mcp-servers.json)에서 가져온 전체 목록. 20-30개를 `~/.claude.json`에 넣되 **프로젝트당 5-6개만 활성화**한다:

| 서버 | 용도 |
|------|------|
| **github** | PR, 이슈, 레포 관리 |
| **playwright** | 브라우저 자동화, E2E 테스트 |
| **supabase** | DB 작업 |
| **memory** | 세션 간 지식 유지 |
| **sequential-thinking** | 복잡한 추론 체인 |
| **context7** | 라이브 문서 검색 (최신 API 문서) |
| **exa-web-search** | 웹 검색, 리서치 |
| **firecrawl** | 웹 스크래핑/크롤링 |
| **vercel** | 배포 관리 |
| **cloudflare-docs** | Cloudflare 문서 검색 |
| **clickhouse** | 애널리틱스 쿼리 |
| **confluence** | 사내 문서 검색 |
| **fal-ai** | 이미지/비디오/오디오 생성 |
| **token-optimizer** | 95%+ 컨텍스트 압축 |
| **insaits** | AI 보안 모니터링 (로컬) |

핵심만 뽑으면:

```json
{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": { "GITHUB_PERSONAL_ACCESS_TOKEN": "YOUR_PAT" }
    },
    "playwright": {
      "command": "npx",
      "args": ["-y", "@playwright/mcp", "--browser", "chrome"]
    },
    "memory": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-memory"]
    },
    "context7": {
      "command": "npx",
      "args": ["-y", "@upstash/context7-mcp@latest"]
    },
    "exa-web-search": {
      "command": "npx",
      "args": ["-y", "exa-mcp-server"],
      "env": { "EXA_API_KEY": "YOUR_KEY" }
    }
  }
}
```

전체 설정은 [mcp-servers.json 원본](https://github.com/affaan-m/everything-claude-code/blob/main/mcp-configs/mcp-servers.json)에서 필요한 것만 복사하면 된다. 1000개+ MCP 서버가 [MCP Registry](https://modelcontextprotocol.io)에 있다.


# 자동화 파이프라인 설정

Hooks, Skills, Agents를 조합하면 하나의 파이프라인이 된다. 코드를 수정하면 자동으로 포맷팅 → 보안 리뷰 → 테스트가 돌고, `/fix-issue 456` 한 줄이면 이슈 확인부터 PR까지 끝난다.

## 전체 구조

```
.claude/
├── settings.json          # Hooks + Permissions
├── hooks/
│   ├── protect-files.sh   # .env 등 보호 파일 차단
│   └── test-before-commit.sh
├── agents/
│   ├── security-reviewer.md
│   ├── code-reviewer.md
│   ├── debugger.md
│   └── scout.md
└── skills/
    ├── fix-issue/SKILL.md
    ├── review-pr/SKILL.md
    └── deploy-check/SKILL.md
```

## settings.json

```json
{
  "permissions": {
    "allow": [
      "Bash(npm run *)",
      "Bash(git commit *)",
      "Bash(git status)",
      "Bash(git diff *)"
    ],
    "deny": [
      "Bash(git push *)",
      "Bash(rm -rf *)"
    ]
  },
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [{
          "type": "command",
          "command": ".claude/hooks/protect-files.sh"
        }]
      },
      {
        "matcher": "Bash",
        "hooks": [{
          "type": "command",
          "command": ".claude/hooks/test-before-commit.sh"
        }]
      }
    ],
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [{
          "type": "command",
          "command": "npx prettier --write $(cat | jq -r '.tool_input.file_path') 2>/dev/null || true"
        }]
      }
    ],
    "PostCompact": [
      {
        "hooks": [{
          "type": "command",
          "command": "echo '{\"hookSpecificOutput\":{\"hookEventName\":\"PostCompact\",\"additionalContext\":\"커밋 전 테스트 필수. feature-branch에서만 작업.\"}}'"
        }]
      }
    ],
    "Notification": [
      {
        "hooks": [{
          "type": "command",
          "command": "notify-send 'Claude Code' '입력이 필요합니다'"
        }]
      }
    ]
  },
  "env": {
    "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1"
  }
}
```

이게 돌아가면:
- **파일 수정 전** → `.env`, 시크릿 파일이면 차단
- **파일 수정 후** → Prettier 자동 포맷팅
- **git commit 시도 전** → 테스트 실패하면 커밋 차단
- **컨텍스트 압축 후** → 핵심 규칙 재주입
- **입력 대기 시** → 데스크탑 알림

그것도 귀찮으면 YOLO 모드: `claude --dangerously-skip-permissions`. 허락 없이 전부 실행한다. git으로 되돌릴 수 있는 상태에서만 쓰자.

## Hook 스크립트

```bash
#!/bin/bash
# .claude/hooks/protect-files.sh
FILE=$(cat | jq -r '.tool_input.file_path')
if [[ "$FILE" == *".env"* ]] || [[ "$FILE" == *"secret"* ]] || [[ "$FILE" == *"credential"* ]]; then
  echo "차단: 보호 파일 ($FILE)" >&2
  exit 2
fi
exit 0
```

```bash
#!/bin/bash
# .claude/hooks/test-before-commit.sh
COMMAND=$(cat | jq -r '.tool_input.command // empty')
if echo "$COMMAND" | grep -q "git commit"; then
  if ! npm test > /dev/null 2>&1; then
    echo '{"hookSpecificOutput":{"hookEventName":"PreToolUse","permissionDecision":"deny","permissionDecisionReason":"테스트 실패. npm test로 확인."}}'
    exit 0
  fi
fi
exit 0
```

## Agents

에이전트는 **description이 핵심**이다. Claude가 이걸 보고 언제 위임할지 판단한다.

참고로 [Anthropic 해커톤 우승자 Affaan Mustafa](https://github.com/affaan-m/everything-claude-code)는 에이전트 12개, 스킬 60+개, 커맨드 40+개를 조합해서 쓴다. 10개월간 매일 Claude Code를 쓰면서 다듬은 셋업이다. 전체 목록:

| 에이전트 | 역할 |
|----------|------|
| planner | 기능 구현 계획 |
| architect | 시스템 설계 결정 |
| tdd-guide | TDD 가이드 |
| code-reviewer | 품질/보안 리뷰 |
| security-reviewer | 취약점 분석 |
| build-error-resolver | 빌드 에러 해결 |
| e2e-runner | Playwright E2E 테스트 |
| refactor-cleaner | 데드 코드 정리 |
| doc-updater | 문서 동기화 |
| database-reviewer | DB/Supabase 리뷰 |
| go-reviewer | Go 코드 리뷰 |
| python-reviewer | Python 코드 리뷰 |

아래는 [해커톤 우승자 레포](https://github.com/affaan-m/everything-claude-code)에서 가져온 원본이다. 그대로 `.claude/agents/`에 넣으면 된다.

### planner.md

```markdown
---
name: planner
description: Expert planning specialist for complex features and refactoring. Use PROACTIVELY when users request feature implementation, architectural changes, or complex refactoring.
tools: ["Read", "Grep", "Glob"]
model: opus
---

You are an expert planning specialist focused on creating comprehensive, actionable implementation plans.

## Planning Process

### 1. Requirements Analysis
- Understand the feature request completely
- Identify success criteria
- List assumptions and constraints

### 2. Architecture Review
- Analyze existing codebase structure
- Identify affected components
- Review similar implementations
- Consider reusable patterns

### 3. Step Breakdown
Create detailed steps with:
- Clear, specific actions
- File paths and locations
- Dependencies between steps
- Estimated complexity
- Potential risks

### 4. Implementation Order
- Prioritize by dependencies
- Group related changes
- Enable incremental testing

## Red Flags to Check
- Large functions (>50 lines)
- Deep nesting (>4 levels)
- Duplicated code
- Missing error handling
- Missing tests
- Plans with no testing strategy
- Steps without clear file paths
```

### code-reviewer.md

```markdown
---
name: code-reviewer
description: Senior code review specialist. Use PROACTIVELY to assess code quality, security, and maintainability.
tools: ["Read", "Grep", "Glob", "Bash"]
model: sonnet
---

## Review Approach

1. Gather context via git diff --staged, git diff, or recent commits
2. Understand scope by identifying changed files and relationships
3. Read surrounding code to avoid isolated reviews
4. Apply review checklist across CRITICAL → LOW severity tiers
5. Report findings only when >80% confident they represent real issues

## Core Review Areas

- **Security (CRITICAL)**: Hardcoded credentials, SQL injection, XSS, path traversal, CSRF, auth bypasses, vulnerable dependencies
- **Code Quality (HIGH)**: Large functions/files, deep nesting, missing error handling, mutation patterns, debug logging, untested code
- **React/Next.js (HIGH)**: Dependency arrays, render-time state updates, list keys, prop drilling, re-renders, client/server boundaries
- **Node.js/Backend (HIGH)**: Input validation, rate limiting, unbounded queries, N+1 patterns, timeouts, error leakage, CORS
- **Performance (MEDIUM)**: Algorithm efficiency, unnecessary re-renders, bundle size, caching
- **Best Practices (LOW)**: TODO tracking, naming clarity, magic numbers, formatting

Consolidate similar issues, skip stylistic noise.
Every review concludes with severity summary table and verdict (Approve/Warning/Block).
```

### security-reviewer.md

```markdown
---
name: security-reviewer
description: Security vulnerability detection and remediation specialist. Use PROACTIVELY when code handles user input, authentication, API endpoints, or sensitive data.
tools: ["Read", "Grep", "Glob", "Bash"]
model: sonnet
---

## Key Responsibilities

- Identify OWASP Top 10 and common security issues
- Detect hardcoded secrets (API keys, passwords, tokens)
- Validate all user inputs properly
- Verify authentication and authorization controls
- Check dependency vulnerabilities

## Workflow

1. Initial scan: npm audit, eslint-plugin-security
2. Systematic OWASP Top 10 assessment
3. Pattern-based code review

## Critical Patterns (Immediate Flag)

- Hardcoded secrets
- Shell commands with user input
- SQL concatenation
- Unsafe deserialization
- Missing authentication checks on routes

## Activation

- ALWAYS: new endpoints, auth changes, user input handling, database modifications
- IMMEDIATELY: production incidents, CVE discoveries

Security is not optional. Defense-in-depth. Least privilege. All user input is untrusted.
```

### tdd-guide.md

```markdown
---
name: tdd-guide
description: Test-Driven Development specialist enforcing write-tests-first methodology. Use PROACTIVELY when writing new features, fixing bugs, or refactoring. Ensures 80%+ test coverage.
tools: ["Read", "Write", "Edit", "Bash", "Grep"]
model: sonnet
---

## TDD Workflow

### 1. Write Test First (RED)
### 2. Run Test — Verify it FAILS
### 3. Write Minimal Implementation (GREEN)
### 4. Run Test — Verify it PASSES
### 5. Refactor (IMPROVE)
### 6. Verify Coverage (80%+)

## Edge Cases You MUST Test

1. Null/Undefined input
2. Empty arrays/strings
3. Invalid types
4. Boundary values (min/max)
5. Error paths (network failures, DB errors)
6. Race conditions
7. Large data (10k+ items)
8. Special characters (Unicode, emojis, SQL chars)

## Anti-Patterns to Avoid

- Testing implementation details instead of behavior
- Tests depending on each other (shared state)
- Asserting too little
- Not mocking external dependencies
```

### build-error-resolver.md

```markdown
---
name: build-error-resolver
description: Build and TypeScript error resolution specialist. Use PROACTIVELY when build fails or type errors occur. Fixes build/type errors only with minimal diffs, no architectural edits.
tools: ["Read", "Write", "Edit", "Bash", "Grep", "Glob"]
model: sonnet
---

## Core Principle: Fix the error, verify the build passes, move on.

## Diagnostic Commands
npx tsc --noEmit --pretty
npm run build
npx eslint . --ext .ts,.tsx,.js,.jsx

## Common Fixes

| Error | Fix |
|-------|-----|
| implicitly has 'any' type | Add type annotation |
| Object is possibly 'undefined' | Optional chaining ?. or null check |
| Property does not exist | Add to interface or use optional ? |
| Cannot find module | Check tsconfig paths, install package |
| Type 'X' not assignable to 'Y' | Parse/convert type |
| Hook called conditionally | Move hooks to top level |

## DO: Add type annotations, null checks, fix imports, update type definitions
## DON'T: Refactor unrelated code, change architecture, rename variables, add features

Speed and precision over perfection.
```

프로젝트에 맞게 추가하면 된다. Go 프로젝트면 `go-reviewer`, DB 많이 쓰면 `database-reviewer`, E2E 테스트 있으면 `e2e-runner`. 전체 12개는 [원본 레포](https://github.com/affaan-m/everything-claude-code/tree/main/agents)에서 볼 수 있다.

## 병렬 실행 에이전트: `isolation: worktree`

에이전트 frontmatter에 `isolation: worktree`를 넣으면 **호출될 때마다 자동으로 독립 worktree에서 실행**된다. 수동으로 `--worktree` 플래그 칠 필요 없다.

```markdown
<!-- .claude/agents/feature-worker.md -->
---
name: feature-worker
description: 기능 구현을 격리된 worktree에서 수행. 병렬 작업 시 사용.
isolation: worktree
tools: Read, Edit, Grep, Glob, Bash
model: sonnet
---
격리된 worktree에서 기능 구현:
1. 요구사항 파악
2. 구현 + 테스트 작성
3. 테스트 실행, 실패하면 수정
4. 커밋 + PR 생성 (gh pr create)
완료 후 PR URL 보고.
```

이걸 여러 개 동시에 스폰하면 각각 독립 브랜치에서 병렬로 돈다:

```
"feature-worker 에이전트 3개로 병렬 작업해줘:
 1. OAuth 구현
 2. 결제 모듈 구현
 3. 알림 시스템 구현"
```

각 에이전트가 자기 worktree에서 구현 → 테스트 → 커밋 → PR. 변경 없으면 worktree 자동 정리.

`/batch`가 하는 걸 에이전트 설정만으로 가능하게 한 것이다. `/batch`는 "같은 종류 반복 작업"에 강하고, 이건 "다른 종류 작업을 병렬로" 돌릴 때 쓴다.

**merge는 자동으로 안 된다** — PR이 생성되니까 사람이 리뷰 후 머지한다. 자동 merge가 필요하면 `SubagentStop` 훅으로 가능하다:

```bash
#!/bin/bash
# .claude/hooks/auto-merge.sh
# 서브에이전트 완료 시 테스트 통과한 PR 자동 merge
INPUT=$(cat)
RESULT=$(echo "$INPUT" | jq -r '.result // empty')
PR_URL=$(echo "$RESULT" | grep -oP 'PR: \K(https://\S+)')

if [ -n "$PR_URL" ]; then
  # CI 통과 확인 후 merge
  gh pr checks "$PR_URL" --watch && gh pr merge "$PR_URL" --squash --auto
fi
exit 0
```

```json
{
  "hooks": {
    "SubagentStop": [{
      "hooks": [{
        "type": "command",
        "command": ".claude/hooks/auto-merge.sh",
        "async": true
      }]
    }]
  }
}
```

서브에이전트가 끝날 때마다 `SubagentStop` 훅이 발동 → PR URL 추출 → CI 통과 확인 → 자동 squash merge. `async: true`라서 메인 세션을 안 막는다. 위험하다 싶으면 `gh pr merge --auto`만 쓰면 CI 통과 후에만 merge된다.

## Skills

Skills는 Agents + Hooks를 묶어서 한 줄로 실행한다.

```markdown
<!-- .claude/skills/fix-issue/SKILL.md -->
---
name: fix-issue
description: GitHub 이슈 수정
disable-model-invocation: true
argument-hint: "[issue-number]"
hooks:
  PostToolUse:
    - matcher: "Edit|Write"
      hooks:
        - type: command
          command: "npx prettier --write $(cat | jq -r '.tool_input.file_path') 2>/dev/null || true"
---
GitHub issue #$ARGUMENTS 수정:
1. `gh issue view $ARGUMENTS`로 확인
2. `git checkout -b fix/issue-$ARGUMENTS`
3. 수정 + 테스트 작성 + 실행
4. 린트/타입체크 통과
5. security-reviewer 에이전트로 보안 리뷰
6. 커밋 + PR 생성
```

```markdown
<!-- .claude/skills/review-pr/SKILL.md -->
---
name: review-pr
description: 현재 PR을 다각도로 리뷰
context: fork
agent: Explore
---
## PR 정보
!`gh pr view --json title,number -q '"\(.number): \(.title)"'`

## 변경 파일
!`gh pr diff --name-only`

## Diff
!`gh pr diff | head -c 10000`

위 변경에 대해:
1. security-reviewer 에이전트로 보안 점검
2. code-reviewer 에이전트로 코드 품질 점검
3. 결과를 Critical/Warning/Suggestion으로 종합
```

`context: fork` + `` !`command` ``로 실행 시점에 PR 데이터를 가져오고, 서브에이전트에서 돌아가니까 메인 컨텍스트는 깨끗하다.

```markdown
<!-- .claude/skills/deploy-check/SKILL.md -->
---
name: deploy-check
description: 배포 전 점검
disable-model-invocation: true
allowed-tools: Bash, Read, Grep
---
배포 전 확인:
1. `npm test` 전체 통과
2. `npm run lint` 에러 없음
3. `npm run build` 성공
4. `.env.example`과 실제 환경변수 비교
5. security-reviewer 에이전트로 최근 변경 보안 점검
6. 전체 결과 요약 — 하나라도 실패하면 배포 중단 권고
```


# 이렇게 쓴다

위 설정을 깔아놓으면 실제 사용은 이렇게 된다.

## 버그 수정

```
나: "로그인 후 세션이 유지 안 되는 버그가 있어"

→ Claude가 debugger 에이전트에 위임
→ debugger가 가설 3개 세우고 검증 → 근본 원인 리포트

나: "고쳐줘"

→ Claude가 수정
→ protect-files.sh가 .env 건드리는지 확인
→ Prettier 자동 포맷
→ security-reviewer가 변경 자동 리뷰
→ 커밋 시도 → test-before-commit가 테스트 실행 → 통과 → 커밋
```

사람이 하는 건 "버그가 있어"와 "고쳐줘" 두 마디.

## 이슈 처리

```
/fix-issue 456

→ gh issue view 456으로 이슈 확인
→ git checkout -b fix/issue-456
→ 코드 수정 + 테스트 작성
→ Prettier 포맷 (스킬 자체 훅)
→ security-reviewer 보안 리뷰
→ npm test 통과 확인
→ 커밋 + PR 생성
```

한 줄.

## PR 리뷰

```
/review-pr

→ gh pr diff로 현재 PR 데이터 가져옴 (동적 주입)
→ 서브에이전트(Explore)에서 실행 — 메인 컨텍스트 안 먹음
→ security-reviewer로 보안 점검
→ code-reviewer로 품질 점검
→ Critical/Warning/Suggestion 종합 리포트
```

## 배포

```
/deploy-check

→ npm test → npm run lint → npm run build
→ 환경변수 비교
→ security-reviewer로 최근 변경 점검
→ 전체 Pass/Fail 요약
```

## 병렬 세션

한 세션에서 구현, 다른 세션에서 리뷰:

| 세션 A (구현) | 세션 B (리뷰) |
|---------------|---------------|
| "레이트 리미터 구현해" | |
| | "/review-pr" |
| "리뷰 피드백 반영해: [결과]" | |

Boris Cherny는 **10-15개 세션을 동시에** 돌린다. Git Worktree로 충돌을 피한다:

```bash
claude --worktree feature-auth
claude --worktree bugfix-123
```

## Agent Teams: 대규모 리팩토링

```
나: "인증을 JWT에서 세션 기반으로 바꿔야 해. 에이전트 팀 만들어줘:
     - 백엔드: 세션 저장소, 토큰 생성, 검증
     - 프론트: 인증 처리, 스토리지, 요청 헤더
     - 테스트: 새 플로우 기반 테스트 전체 재작성
     백엔드가 먼저 인터페이스 확정하고, 나머지가 따라가게"
```

팀원별로 다른 디렉토리 소유. `Shift+Down`으로 순환, 방향 교정. 토큰 7배니까 이 규모에서만 쓴다.

## Fan Out: 대규모 마이그레이션

```bash
claude -p "마이그레이션 대상 파일 목록" > files.txt

for file in $(cat files.txt); do
  claude -p "Migrate $file from React to Vue." \
    --allowedTools "Edit,Bash(git commit *)" &
done
```

내장 스킬 `/batch`로도 가능 — 5-30개 worktree에서 병렬 처리 후 각각 PR 생성.


# Anthropic은 실제로 어떻게 쓰나

- **보안팀**: "설계→대충 코드→테스트 포기"가 "Claude에 의사코드→TDD→주기적 체크인"으로 바뀜. 인시던트 분석 **3배 빠름**.
- **디자인팀**: Figma → Claude → 코드/테스트/수정 **자율 반복** → 사람은 최종 리뷰만.
- **추론팀**: 비 ML 배경 인원의 리서치 시간 **80% 감소**.
- **마케팅팀**: 2개 서브에이전트로 수백 개 광고 변형. **몇 시간 → 몇 분**.

공통점: Claude를 코드 생성기가 아니라 **사고 파트너**로 쓴다.


# 수치

| 요소 | 효과 | 출처 |
|------|------|------|
| 검증 루프 | 품질 2-3배 | Boris Cherny |
| Plan → Auto-Accept | 원샷 구현 | Boris Cherny |
| 서브에이전트 탐색 | 리서치 80% 감소 | Anthropic |
| Self-Correction Loop | 50세션 후 교정 최소화 | Pro Workflow |
| 병렬 세션 | 속도 40% 향상 | 커뮤니티 |

GitHub 전체 커밋의 4%가 Claude Code에서 나온다.


# 참고 자료

- [Claude Code 공식 베스트 프랙티스](https://code.claude.com/docs/en/best-practices)
- [Anthropic 팀의 Claude Code 활용법](https://claude.com/blog/how-anthropic-teams-use-claude-code)
- [Boris Cherny 워크플로우 (InfoQ)](https://www.infoq.com/news/2026/01/claude-code-creator-workflow/)
- [Boris Cherny 병렬 워크플로우 분석](https://glenrhodes.com/boris-chernys-parallel-claude-code-workflow-and-the-gap-between-power-users-and-average-developers/)
- [Pro Workflow](https://github.com/rohitg00/pro-workflow)
- [Everything Claude Code — 해커톤 우승자 Affaan Mustafa의 전체 셋업](https://github.com/affaan-m/everything-claude-code)
- [Awesome Claude Code Subagents — 100+ 서브에이전트 모음](https://github.com/VoltAgent/awesome-claude-code-subagents)
