# Claude Code 최고로 활용하기 - 사전 자료 조사

> 공식 문서 (https://code.claude.com/docs/) 기반 조사. 2026-03-18 기준.

---

## 목차

1. [Claude Code 개요](#1-claude-code-개요)
2. [CLAUDE.md - 프로젝트 맞춤 설정의 핵심](#2-claudemd---프로젝트-맞춤-설정의-핵심)
3. [슬래시 커맨드 & 키보드 단축키](#3-슬래시-커맨드--키보드-단축키)
4. [Permission 시스템](#4-permission-시스템)
5. [Hooks - 워크플로우 자동화](#5-hooks---워크플로우-자동화)
6. [MCP 서버 - 외부 도구 연동](#6-mcp-서버---외부-도구-연동)
7. [서브에이전트 & 멀티에이전트](#7-서브에이전트--멀티에이전트)
8. [Plan Mode - 안전한 코드 분석](#8-plan-mode---안전한-코드-분석)
9. [Skills - 재사용 가능한 프롬프트](#9-skills---재사용-가능한-프롬프트)
10. [Git 통합 & Worktree](#10-git-통합--worktree)
11. [컨텍스트 윈도우 관리](#11-컨텍스트-윈도우-관리)
12. [메모리 시스템](#12-메모리-시스템)
13. [비용 최적화](#13-비용-최적화)
14. [Checkpointing & Rewind](#14-checkpointing--rewind)
15. [프롬프팅 베스트 프랙티스](#15-프롬프팅-베스트-프랙티스)
16. [알려진 제한사항 & 해결책](#16-알려진-제한사항--해결책)
17. [블로그 글 구성 제안](#17-블로그-글-구성-제안)

---

## 1. Claude Code 개요

터미널 기반 AI 코딩 어시스턴트. 코드베이스를 읽고, 파일을 편집하고, 명령어를 실행하고, 외부 도구와 연동할 수 있다.

**사용 가능한 환경:**
- Terminal CLI (가장 풀기능)
- VS Code 확장
- JetBrains IDE 플러그인
- Desktop App (시각적 diff 리뷰, 스케줄링)
- Web (claude.ai/code) - 로컬 설치 없이 사용
- Chrome 통합 - 브라우저 자동화
- Remote Control - 원격 세션 접근
- Slack 통합

**핵심 동작 원리 (Agentic Loop):**
1. 사용자 프롬프트 수신
2. 컨텍스트 분석 (파일 읽기, 검색 등)
3. 도구 사용 결정 (Read, Edit, Bash, Grep 등)
4. 권한 확인 후 실행
5. 결과 확인 및 반복

---

## 2. CLAUDE.md - 프로젝트 맞춤 설정의 핵심

### 파일 위치 & 로딩 순서
| 위치 | 용도 | 공유 |
|------|------|------|
| `~/.claude/CLAUDE.md` | 개인 설정 (모든 프로젝트) | X |
| `.claude/CLAUDE.md` 또는 `./CLAUDE.md` | 프로젝트 설정 | O (git) |
| 하위 디렉토리 CLAUDE.md | 디렉토리별 컨텍스트 | O |
| `.claude/rules/` | 모듈화된 규칙 파일 | O |

### 포함해야 할 것
- 빌드/테스트/배포 명령어 (예: `npm run test`, `make build`)
- 코딩 컨벤션 (들여쓰기, 네이밍 등)
- 아키텍처 의사결정 사항
- 자주 발생하는 실수나 주의사항
- 브랜치 네이밍, PR 컨벤션

### 제외해야 할 것
- Claude가 이미 아는 언어 표준 컨벤션
- 자주 바뀌는 정보
- 긴 튜토리얼 (대신 파일 링크)

### 베스트 프랙티스
- **200줄 이하** 유지 (넘으면 준수율 저하)
- 구체적이고 검증 가능한 규칙 작성
- `@path/to/file` 문법으로 다른 파일 import 가능
- `/init` 커맨드로 자동 생성 가능
- 모순되는 규칙 없도록 주기적 점검

### 고급 기능: Rules 시스템
```
.claude/rules/
  typescript.md    # TypeScript 관련 규칙
  testing.md       # 테스트 관련 규칙
  security.md      # 보안 규칙
```
- YAML frontmatter에 `paths:` 필드로 특정 파일 패턴에만 적용 가능
- 심볼릭 링크로 프로젝트 간 공유 가능

---

## 3. 슬래시 커맨드 & 키보드 단축키

### 주요 슬래시 커맨드
| 커맨드 | 설명 |
|--------|------|
| `/init` | CLAUDE.md 자동 생성 |
| `/compact` | 대화 컨텍스트 압축 |
| `/clear` | 새 세션 시작 |
| `/rewind` | 변경사항 되돌리기 |
| `/memory` | CLAUDE.md 및 메모리 조회/편집 |
| `/resume` | 이전 세션 재개 |
| `/rename` | 세션 이름 지정 |
| `/permissions` | 권한 관리 |
| `/mcp` | MCP 서버 설정 |
| `/hooks` | 훅 설정 조회 |
| `/agents` | 서브에이전트 관리 |
| `/config` | 설정 변경 |
| `/effort` | 추론 깊이 조절 (low/medium/high/max) |
| `/vim` | Vim 모드 활성화 |
| `/model` | 모델 변경 |
| `/cost` | 토큰 사용량 확인 |
| `/btw` | 사이드 질문 (컨텍스트 오염 없이) |
| `/add-dir` | 추가 디렉토리 접근 |
| `/batch` | 여러 파일에 병렬 리팩토링 (5-30 worktree) |
| `/loop` | 반복 작업 실행 |

### 핵심 키보드 단축키
| 단축키 | 기능 |
|--------|------|
| `Ctrl+C` | 입력/생성 취소 |
| `Ctrl+R` | 명령어 히스토리 역방향 검색 |
| `Ctrl+G` | 텍스트 에디터에서 프롬프트 편집 |
| `Ctrl+O` | verbose 출력 토글 |
| `Esc+Esc` | Rewind 메뉴 열기 |
| `Shift+Tab` | Permission 모드 전환 |
| `Alt+T` | Extended thinking 토글 |
| `Alt+P` | 모델 전환 |
| `!` 접두사 | Bash 모드 (직접 명령 실행) |
| `@` 접두사 | 파일 경로 자동완성 |
| `Tab` | 프롬프트 제안 수락 |

---

## 4. Permission 시스템

### Permission 모드
| 모드 | 설명 | 사용 시나리오 |
|------|------|---------------|
| `default` | 첫 사용 시 확인 | 일반 개발 |
| `acceptEdits` | 파일 편집 자동 승인 | 빠른 코딩 |
| `plan` | 읽기 전용 (수정/실행 불가) | 코드 분석, 계획 수립 |
| `dontAsk` | 미승인 도구 자동 거부 | 엄격한 보안 환경 |
| `bypassPermissions` | 대부분 프롬프트 스킵 | 컨테이너/VM 전용 |

### Permission 규칙 문법
```json
{
  "permissions": {
    "allow": [
      "Bash(npm run *)",
      "Bash(git commit *)",
      "Read",
      "WebFetch(domain:github.com)"
    ],
    "deny": [
      "Bash(git push *)",
      "Edit(.env)",
      "Bash(rm -rf *)"
    ]
  }
}
```

### 규칙 평가 순서
**deny -> ask -> allow** (deny가 항상 우선)

### 설정 우선순위
1. Managed settings (조직 관리자)
2. CLI 인자
3. `.claude/settings.local.json` (로컬)
4. `.claude/settings.json` (프로젝트)
5. `~/.claude/settings.json` (사용자)

---

## 5. Hooks - 워크플로우 자동화

CLAUDE.md가 "조언"이라면, Hooks는 "강제" - 확정적으로 실행되는 자동화.

### Hook 이벤트 목록
| 이벤트 | 시점 | 주요 용도 |
|--------|------|-----------|
| `SessionStart` | 세션 시작/재개 | 환경 변수 설정, 컨텍스트 주입 |
| `UserPromptSubmit` | 프롬프트 처리 전 | 입력 검증, 변환 |
| `PreToolUse` | 도구 실행 전 | 보호 파일 차단, 검증 |
| `PostToolUse` | 도구 성공 후 | 자동 포맷, 린트 |
| `PostToolUseFailure` | 도구 실패 후 | 에러 처리 |
| `PermissionRequest` | 권한 다이얼로그 전 | 자동 승인/거부 |
| `Notification` | 주의 필요 시 | 데스크탑 알림 |
| `PreCompact/PostCompact` | 컨텍스트 압축 전후 | 컨텍스트 재주입 |
| `Stop` | 응답 완료 후 | 후처리 |
| `SubagentStart/Stop` | 서브에이전트 생명주기 | 모니터링 |

### 실전 예시

**1. 파일 편집 후 자동 포맷팅 (Prettier)**
```json
{
  "hooks": {
    "PostToolUse": [{
      "matcher": "Edit|Write",
      "hooks": [{
        "type": "command",
        "command": "jq -r '.tool_input.file_path' | xargs npx prettier --write"
      }]
    }]
  }
}
```

**2. .env 파일 편집 차단**
```bash
#!/bin/bash
INPUT=$(cat)
FILE=$(echo "$INPUT" | jq -r '.tool_input.file_path')
if [[ "$FILE" == *".env"* ]]; then
  echo "Blocked: .env files" >&2
  exit 2  # exit 2 = 차단
fi
exit 0  # exit 0 = 허용
```

**3. Compaction 후 컨텍스트 재주입**
```json
{
  "hooks": {
    "PostCompact": [{
      "hooks": [{
        "type": "command",
        "command": "echo 'Reminder: Bun을 사용하세요. 커밋 전에 테스트를 실행하세요.'"
      }]
    }]
  }
}
```

**4. 입력 대기 시 데스크탑 알림**
```json
{
  "hooks": {
    "Notification": [{
      "hooks": [{
        "type": "command",
        "command": "notify-send 'Claude Code' '입력이 필요합니다'"
      }]
    }]
  }
}
```

### Hook 타입
- `command` - 셸 명령 (가장 일반적)
- `http` - HTTP 엔드포인트로 POST
- `prompt` - LLM(Haiku) 기반 판단
- `agent` - 서브에이전트 기반 검증

---

## 6. MCP 서버 - 외부 도구 연동

### 인기 MCP 서버
- GitHub, Slack, Notion, Figma, Gmail
- Sentry, Datadog (모니터링)
- PostgreSQL, MongoDB, Airtable (데이터베이스)
- Playwright (브라우저 자동화)

### 설치 방법
```bash
# HTTP (권장)
claude mcp add --transport http notion https://mcp.notion.com/mcp

# Stdio (로컬 프로세스)
claude mcp add --transport stdio --env API_KEY=xxx airtable \
  -- npx -y airtable-mcp-server
```

### MCP 스코프
| 스코프 | 저장 위치 | 용도 |
|--------|-----------|------|
| local (기본) | `~/.claude.json` | 개인 |
| project | `.mcp.json` | 팀 공유 (git) |
| user | `~/.claude.json` | 모든 프로젝트 |

### 팁
- CLI 도구(gh, aws, gcloud)가 MCP보다 컨텍스트 효율적
- 안 쓰는 서버는 `/mcp`에서 비활성화
- 서브에이전트에 스코프를 제한하면 메인 컨텍스트 절약
- `ENABLE_TOOL_SEARCH=auto:5` 설정으로 자동 도구 검색
- `MAX_MCP_OUTPUT_TOKENS` 설정으로 큰 응답 제한

---

## 7. 서브에이전트 & 멀티에이전트

### 빌트인 서브에이전트
| 에이전트 | 용도 | 모델 |
|----------|------|------|
| Explore | 빠른 코드베이스 검색 (읽기전용) | Haiku |
| Plan | Plan Mode 중 리서치 | - |
| general-purpose | 복합 탐색 및 작업 | 상속 |

### 커스텀 서브에이전트 만들기
`.claude/agents/code-reviewer.md`:
```markdown
---
name: code-reviewer
description: 코드 변경 후 자동으로 리뷰. 보안, 성능, 가독성 확인.
tools: Read, Grep, Glob, Bash
model: sonnet
---

시니어 코드 리뷰어로서 다음을 점검하세요:
- 보안 취약점
- 테스트 커버리지
- 성능 이슈
- 코드 가독성
```

### 주요 설정 옵션
- `description` - Claude가 언제 위임할지 결정하는 기준
- `tools` - 사용 가능한 도구 화이트리스트
- `model` - 모델 오버라이드 (Haiku로 비용 절감)
- `permissionMode` - 권한 동작 오버라이드
- `memory: user` - 대화 간 지속적 학습
- `isolation: worktree` - Git worktree에서 격리 실행
- `skills` - 사전 로드할 스킬

### 서브에이전트 사용 시점
- 탐색 작업 → 메인 컨텍스트 오염 방지
- 도메인별 전문 작업 → 전용 에이전트
- 병렬 리서치 → 여러 에이전트 동시 실행
- 대량 출력 작업 → 테스트/로그 처리 격리
- 비용 절감 → Haiku로 간단한 작업 위임

### Agent Teams (멀티에이전트)
- 여러 에이전트가 병렬로 독립 작업
- Lead 에이전트가 조율
- 각 팀원은 독립 컨텍스트
- 주의: 토큰 사용량 약 7배 증가

---

## 8. Plan Mode - 안전한 코드 분석

### 활성화 방법
```bash
# 시작 시
claude --permission-mode plan

# 세션 중
Shift+Tab  # 모드 순환
```

### 워크플로우
1. Claude가 파일을 읽고 분석 (수정 불가)
2. 명확화 질문을 함
3. 상세 구현 계획 제안
4. 사용자가 검토/승인
5. Normal Mode로 전환하여 구현

### 사용 시점
- 멀티 파일 변경
- 익숙하지 않은 코드베이스 탐색
- 복잡한 리팩토링 전 계획
- 코드 리뷰/분석

### 주의
- 작은 변경(오타, 한 줄 수정)에는 오버헤드 → 그냥 바로 수정
- 계획 단계도 토큰을 소비함

---

## 9. Skills - 재사용 가능한 프롬프트

### 스킬이란?
마크다운 파일로 저장하는 재사용 가능한 프롬프트 템플릿. `/skill-name`으로 호출.

### 파일 위치
- `.claude/skills/` (프로젝트)
- `~/.claude/skills/` (개인)

### 번들 스킬
- `/batch` - 여러 파일 병렬 리팩토링 (5-30 worktree 사용)
- `/simplify` - 코드 품질 리뷰 및 개선
- `/debug` - 세션 트러블슈팅
- `/loop` - 반복 작업 실행
- `/claude-api` - Claude API 레퍼런스 로드

### 스킬 예시
`.claude/skills/deploy-check.md`:
```markdown
---
name: deploy-check
description: 배포 전 점검 실행
user-invocable: true
allowed-tools: Bash, Read, Grep
---

배포 전 다음을 확인하세요:
1. 모든 테스트 통과 (`npm test`)
2. 린트 에러 없음 (`npm run lint`)
3. 빌드 성공 (`npm run build`)
4. 환경 변수 확인
```

### 스킬 vs CLAUDE.md
- CLAUDE.md: 매 세션 자동 로드 (항상 적용)
- Skills: 필요할 때만 로드 (컨텍스트 효율적)
- 전문화된 지시는 CLAUDE.md에서 skills로 이동하면 비용 절감

---

## 10. Git 통합 & Worktree

### Git 기본 기능
```bash
claude "내 변경사항 커밋해줘"
claude "PR 만들어줘"
```

### Git Worktree - 병렬 개발
```bash
# 자동 이름 생성
claude --worktree

# 특정 이름 지정 (브랜치명이 됨)
claude --worktree feature-auth
claude --worktree bugfix-123
```

**동작 방식:**
- `.claude/worktrees/<name>/` 디렉토리 생성
- `worktree-<name>` 브랜치 자동 생성
- 메인 레포와 git 히스토리 공유
- 격리된 파일시스템

**정리:**
- 변경 없으면 종료 시 자동 삭제
- 변경 있으면 유지 여부 확인

### CI/CD 통합
- GitHub Actions 지원 (`@claude` 멘션으로 코드 리뷰)
- GitLab CI/CD 지원
- PR 상태 표시 (초록: 승인, 노란: 대기, 빨강: 변경 요청)

---

## 11. 컨텍스트 윈도우 관리

### 컨텍스트 소비 요소
- CLAUDE.md 파일 (매 세션 로드)
- 대화 히스토리
- 파일 읽기 결과
- MCP 도구 정의
- Extended thinking 토큰

### 관리 전략
1. **`/compact`** - 수동 압축 (포커스 지시 가능)
   ```
   /compact API 변경사항과 테스트 명령에 집중
   ```
2. **`/clear`** - 관련 없는 작업 간 전환 시
3. **`/btw`** - 사이드 질문 (히스토리 오염 방지)
4. **서브에이전트** - 대량 출력 작업 격리
5. **`/rewind` → Summarize** - 특정 지점부터 압축
6. **`@file`** - 파일 참조 (직접 내용 붙여넣기 대신)

### 자동 압축
- 약 95% 도달 시 자동 실행
- `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE`로 임계값 조정 가능
- CLAUDE.md 내용은 압축 후에도 완전히 보존

### 상태 모니터링
- `/statusline` 설정으로 컨텍스트 사용량 실시간 확인
- `/cost` 명령으로 토큰 사용량 체크

---

## 12. 메모리 시스템

### 두 가지 메모리
| | CLAUDE.md | Auto Memory |
|---|-----------|-------------|
| 작성자 | 사용자 | Claude |
| 로딩 | 매 세션 자동 | 첫 200줄 자동 + 온디맨드 |
| 용도 | 코딩 표준, 워크플로우 | 빌드 명령, 디버깅 패턴 |
| 위치 | 프로젝트 루트 | `~/.claude/projects/<project>/memory/` |

### Auto Memory 특징
- Claude가 교정을 받으면 자동 저장
- `MEMORY.md`가 인덱스 역할 (200줄 제한)
- 토픽별 파일은 필요할 때 로드
- `/memory` 명령으로 조회/편집 가능
- "이거 기억해줘"로 명시적 저장 가능
- `autoMemoryEnabled` 설정으로 on/off

---

## 13. 비용 최적화

### 토큰 비용 주요 요인
1. CLAUDE.md 길이 (매 세션 로드)
2. 컨텍스트 윈도우 크기
3. 모델 선택 (Opus > Sonnet > Haiku)
4. Extended thinking 토큰
5. MCP 도구 정의 수
6. 대용량 파일 읽기

### 비용 절감 전략

**가장 효과적:**
1. **컨텍스트 관리** - `/clear`로 작업 간 분리, `/compact` 활용
2. **모델 선택** - Sonnet이 대부분 충분, 서브에이전트는 Haiku
3. **MCP 정리** - 안 쓰는 서버 비활성화, CLI 도구 선호
4. **CLAUDE.md → Skills 이동** - 전문 지시를 스킬로 옮겨 온디맨드 로드

**추가 전략:**
5. `/effort low` - 간단한 작업에 추론 깊이 줄이기
6. `MAX_THINKING_TOKENS=8000` - thinking 토큰 예산 제한
7. 서브에이전트로 대량 작업 위임 (요약만 반환)
8. 구체적인 프롬프트 작성 (모호한 요청은 스캐닝 유발)

### 비용 모니터링
- `/cost` - 현재 세션 사용량
- `--max-budget-usd` - 비용 한도 설정
- 상태 라인에 컨텍스트 사용량 표시

---

## 14. Checkpointing & Rewind

### 동작 방식
- 매 프롬프트마다 자동 체크포인트 생성
- 파일 편집 도구로 만든 변경만 추적
- Bash 명령 변경은 추적 안 됨
- 세션 간 지속 (30일 자동 정리)

### Rewind 방법
`Esc+Esc` 또는 `/rewind` → 복원 지점 선택

**복원 옵션:**
| 옵션 | 코드 | 대화 |
|------|------|------|
| Restore code and conversation | 되돌림 | 되돌림 |
| Restore conversation | 유지 | 되돌림 |
| Restore code | 되돌림 | 유지 |
| Summarize from here | 유지 | 압축 |

### 활용 시나리오
- 다른 구현 방식 실험
- 실수 복구
- 기능 이터레이션
- 컨텍스트 공간 확보 (Summarize)

---

## 15. 프롬프팅 베스트 프랙티스

### 핵심 원칙: "검증 수단을 함께 제공하라"
가장 효과적인 단일 습관. 테스트, 스크린샷, 기대 결과를 포함.

```
# Bad
"이메일 유효성 검사 함수 만들어줘"

# Good
"validateEmail 함수 작성. user@example.com은 true, invalid는 false,
user@.com은 false. 구현 후 테스트 실행해줘."
```

### 패턴 1: Explore → Plan → Code → Commit
1. Plan Mode로 코드 읽기
2. 구현 계획 수립
3. Normal Mode로 전환, 구현
4. 커밋 & PR

### 패턴 2: 구체적 컨텍스트 제공
```
# Bad
"버그 고쳐줘"

# Good
"로그인 후 세션 타임아웃되면 실패함. src/auth/의 토큰 리프레시 확인해줘"
```

### 패턴 3: 풍부한 입력
- `@file` 문법으로 파일 참조
- 이미지/스크린샷 직접 붙여넣기
- 파이프: `cat error.log | claude "이 에러 분석해줘"`

### 패턴 4: 대화형 협업
- 모호한 요구사항은 Claude에게 인터뷰 요청
- 초기에 방향 교정 (잘못된 방향 2회 후 `/clear` 후 재시작)

### 안티패턴
- 주제 전환 없이 긴 세션 → `/clear` 사용
- 모호한 "코드베이스 개선" → 광범위 스캐닝 유발
- CLAUDE.md에 너무 많은 규칙 → 200줄 이하 유지
- 검증 없는 신뢰 → 항상 테스트 포함

---

## 16. 알려진 제한사항 & 해결책

| 제한사항 | 해결책 |
|----------|--------|
| 컨텍스트 윈도우 빠르게 참 | `/clear`, 서브에이전트, CLAUDE.md 200줄 제한 |
| 체크포인트가 외부 변경 추적 안 함 | Git을 진실의 원천으로 사용 |
| Bash 권한 패턴이 취약함 | deny 리스트, WebFetch, hooks 조합 사용 |
| Compaction 후 대화 내 지시 유실 | CLAUDE.md에 넣거나 hooks로 재주입 |
| MCP 도구가 컨텍스트 소비 | Tool Search, CLI 도구 선호, 서브에이전트 스코프 |
| Plan Mode 오버헤드 | 작은 변경에는 사용 안 함 |
| CLAUDE.md 비준수 | 200줄 이하, 구체적 규칙, 모순 제거 |
| 백그라운드 작업이 질문 불가 | 인터랙티브 작업은 포그라운드에서 |

---

## 17. 블로그 글 구성 제안

### 제안 1: "실전 중심" 구성
1. Claude Code란? (간단 소개)
2. 설치 후 첫 번째로 할 일 - `/init`과 CLAUDE.md
3. 프롬프트 작성법 - 검증 수단 포함하기
4. Permission 설정으로 워크플로우 가속
5. Hooks로 자동화하기 (포맷, 알림, 보호)
6. 서브에이전트 활용 - 컨텍스트 효율의 핵심
7. 컨텍스트 관리 - `/compact`, `/clear`, `/btw`
8. 비용 최적화 팁

### 제안 2: "레벨별" 구성
1. **초급**: 기본 사용법, 슬래시 커맨드, 단축키
2. **중급**: CLAUDE.md, Permission, Plan Mode
3. **고급**: Hooks, MCP, 서브에이전트, Skills
4. **마스터**: Agent Teams, Worktree, 비용 최적화, 자동화

### 제안 3: "최고 효과 Top 10" 구성
1. 검증 수단 제공 (테스트, 스크린샷)
2. 컨텍스트 관리 (clear, compact, subagent)
3. CLAUDE.md 200줄 이하 유지
4. Plan Mode로 복잡한 변경 계획
5. 서브에이전트로 컨텍스트 격리
6. Permission 사전 설정으로 마찰 제거
7. Hooks로 확정적 자동화
8. Skills로 반복 프롬프트 제거
9. Git Worktree로 병렬 개발
10. 코드 인텔리전스 플러그인 설치

---

## 참고 자료

- 공식 문서: https://code.claude.com/docs/
- 베스트 프랙티스: https://code.claude.com/docs/en/best-practices.md
- Hooks 가이드: https://code.claude.com/docs/en/hooks-guide.md
- MCP 가이드: https://code.claude.com/docs/en/mcp.md
- 서브에이전트: https://code.claude.com/docs/en/sub-agents.md
- Skills: https://code.claude.com/docs/en/skills.md
- 메모리: https://code.claude.com/docs/en/memory.md
- 비용 관리: https://code.claude.com/docs/en/costs.md
- 설정: https://code.claude.com/docs/en/settings.md
- 권한: https://code.claude.com/docs/en/permissions.md
- GitHub: https://github.com/anthropics/claude-code
