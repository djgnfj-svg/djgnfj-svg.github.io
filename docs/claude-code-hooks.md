# Claude Code Hooks 심층 조사

> 공식 문서 기반. 2026-03-18 기준.

---

## 1. Hook 생명주기

1. **이벤트 발생** (예: PreToolUse)
2. **Matcher 평가** (regex로 도구명/소스 매칭)
3. **병렬 실행** (매칭된 모든 훅 동시 실행)
4. **중복 제거** (동일 명령은 1회만)
5. **입력 전달** (stdin으로 JSON)
6. **훅 처리**
7. **출력 파싱** (exit code + stdout JSON)
8. **결정 적용** (allow/deny/block/modify)
9. **실행 계속**

### 타임아웃 기본값
| 타입 | 기본 타임아웃 |
|------|-------------|
| command | 600초 |
| http | 30초 |
| prompt | 30초 |
| agent | 60초 |

---

## 2. 전체 Hook 이벤트 (21개)

### A. SessionStart
- **시점**: 세션 시작/재개
- **Matcher**: `startup`, `resume`, `clear`, `compact`
- **차단 가능**: O
- **특별 기능**: `$CLAUDE_ENV_FILE`에 환경변수 영구 저장 가능

```json
{
  "session_id": "abc123",
  "hook_event_name": "SessionStart",
  "source": "startup|resume|clear|compact",
  "cwd": "/working/directory"
}
```

### B. InstructionsLoaded
- **시점**: CLAUDE.md 또는 rules 파일 로드 시
- **Matcher**: 없음
- **차단 가능**: O (파일 로딩 차단)
- **출력**: `updatedInstructions`로 내용 수정 가능

### C. UserPromptSubmit
- **시점**: 프롬프트 처리 전
- **Matcher**: 없음
- **차단 가능**: O
- **용도**: 입력 검증, 추가 컨텍스트 주입

### D. PreToolUse (가장 중요)
- **시점**: 도구 실행 전
- **Matcher**: 도구명 (`Bash`, `Edit|Write`, `mcp__.*`)
- **차단 가능**: O

**도구별 입력 스키마**:

**Bash**:
```json
{"tool_name": "Bash", "tool_input": {"command": "npm test"}}
```

**Write**:
```json
{"tool_name": "Write", "tool_input": {"file_path": "/path/file.js", "file_content": "..."}}
```

**Edit**:
```json
{"tool_name": "Edit", "tool_input": {"file_path": "/path/file.js", "edits": [{"type": "replace", "old_text": "...", "new_text": "..."}]}}
```

**Read/Glob/Grep/WebFetch/WebSearch/Agent**: 각각 해당 도구의 파라미터

**결정 제어**:
```json
{
  "hookSpecificOutput": {
    "hookEventName": "PreToolUse",
    "permissionDecision": "allow|deny|ask",
    "permissionDecisionReason": "거부 이유",
    "updatedInput": {"command": "수정된 명령"},
    "additionalContext": "Claude에게 추가 컨텍스트"
  }
}
```

### E. PermissionRequest
- **시점**: 권한 다이얼로그 표시 전
- **Matcher**: 도구명
- **차단 가능**: O
- **특별 기능**: `updatedPermissions`로 규칙 추가/모드 변경 가능

```json
{
  "hookSpecificOutput": {
    "hookEventName": "PermissionRequest",
    "decision": {
      "behavior": "allow|deny|ask",
      "updatedPermissions": [
        {"type": "addRule", "rule": "Bash(npm:*)", "destination": "session", "mode": "allow"},
        {"type": "setMode", "mode": "acceptEdits", "destination": "session"}
      ]
    }
  }
}
```

### F. PostToolUse
- **시점**: 도구 성공 후
- **Matcher**: 도구명
- **차단 가능**: O (결과 거부 → 재시도 유발)
- **특별**: `updatedMCPToolOutput`으로 MCP 도구 출력 수정 가능

### G. PostToolUseFailure
- **시점**: 도구 실패 후
- **Matcher**: 도구명
- **용도**: 에러 처리, 복구 안내

### H. Notification
- **시점**: 주의 필요 시
- **Matcher**: `permission_prompt`, `idle_prompt`, `auth_success`, `elicitation_dialog`
- **차단 불가** (정보성)
- **용도**: 데스크탑 알림 발송

### I. Stop
- **시점**: Claude 응답 완료 후
- **차단 가능**: O (계속 작업하게 할 수 있음)
- **주의**: `stop_hook_active` 확인으로 무한 루프 방지 필수

```bash
if [ "$(echo "$INPUT" | jq -r '.stop_hook_active')" = "true" ]; then
  exit 0  # 무한 루프 방지
fi
```

### J-K. SubagentStart / SubagentStop
- **Matcher**: 에이전트 타입 (`Explore`, `Plan`, 커스텀)
- **차단 가능**: O

### L. TeammateIdle
- **시점**: Agent Team 팀원이 idle 직전
- **차단 가능**: O (피드백 보내서 계속 작업시킴)

### M. TaskCompleted
- **시점**: 태스크 완료 표시 시
- **차단 가능**: O (완료 방지 + 피드백)

### N. ConfigChange
- **시점**: 설정 파일 변경 시
- **Matcher**: `user_settings`, `project_settings`, `local_settings`, `skills`
- **용도**: 설정 변경 감사

### O-P. WorktreeCreate / WorktreeRemove
- **WorktreeCreate**: stdout에 경로 출력으로 기본 git worktree 대체 가능 (커스텀 VCS 지원)

### Q-R. PreCompact / PostCompact
- **Matcher**: `manual`, `auto`
- **PostCompact**: 컨텍스트 재주입에 핵심

### S. SessionEnd
- **Matcher**: `clear`, `logout`, `prompt_input_exit`, `other`
- **차단 불가** (정리 전용)

### T-U. Elicitation / ElicitationResult
- **시점**: MCP 서버가 사용자 입력 요청 시
- **용도**: 자동 응답 또는 응답 수정

---

## 3. Hook 핸들러 타입 (4가지)

### A. Command (가장 일반적)
```json
{
  "type": "command",
  "command": ".claude/hooks/validate.sh",
  "timeout": 30,
  "async": false
}
```
- stdin으로 JSON 입력
- exit 0 = 허용, exit 2 = 차단
- stdout으로 JSON 결정 반환 가능

### B. HTTP
```json
{
  "type": "http",
  "url": "http://localhost:8080/hooks/validate",
  "headers": {"Authorization": "Bearer $MY_TOKEN"},
  "allowedEnvVars": ["MY_TOKEN"],
  "timeout": 30
}
```
- POST로 JSON 전송
- 응답 body로 결정
- 팀 전체 감사 로깅, 외부 승인 시스템에 적합

### C. Prompt (LLM 판단)
```json
{
  "type": "prompt",
  "prompt": "이 명령이 안전한가요? $ARGUMENTS. {\"ok\": true/false, \"reason\": \"...\"} 반환",
  "model": "claude-haiku-4",
  "timeout": 30
}
```
- Haiku 기본, 단일 턴
- 도구 접근 없음
- `{"ok": true/false, "reason": "..."}` 스키마
- 판단 기반 결정에 적합

### D. Agent (멀티턴 검증)
```json
{
  "type": "agent",
  "prompt": "모든 테스트가 통과하는지 확인. {\"ok\": true/false, \"reason\": \"...\"} 반환",
  "timeout": 120
}
```
- 최대 50번 도구 호출 가능
- 파일 읽기, bash 실행 등 가능
- 복잡한 조건 검증에 적합

---

## 4. Matcher 패턴

| 이벤트 | 매칭 대상 | 예시 |
|--------|-----------|------|
| PreToolUse, PostToolUse 등 | 도구명 | `Bash`, `Edit\|Write`, `mcp__github__.*` |
| SessionStart | 소스 | `startup`, `resume`, `compact` |
| SessionEnd | 종료 이유 | `clear`, `logout` |
| Notification | 알림 타입 | `idle_prompt`, `permission_prompt` |
| SubagentStart/Stop | 에이전트 타입 | `Explore`, `Plan` |
| PreCompact/PostCompact | 트리거 | `manual`, `auto` |
| ConfigChange | 설정 소스 | `user_settings`, `skills` |

### MCP 도구 패턴
```json
{"matcher": "mcp__github__.*"}      // GitHub 모든 도구
{"matcher": "mcp__.*__write.*"}     // 모든 서버의 write 계열
```

### 생략/와일드카드
```json
{"matcher": ""}     // 전체 매칭
{"matcher": ".*"}   // 전체 매칭
// matcher 생략     // 전체 매칭
```

---

## 5. Exit Code 동작

| 코드 | 동작 | stdout | 용도 |
|------|------|--------|------|
| **0** | 성공 | JSON 파싱 | 허용/결정 반환 |
| **2** | 차단 | 무시, stderr → 피드백 | 거부/방지 |
| **기타** | 비차단 에러 | 무시 | 로깅, 정리 |

---

## 6. 비동기(Async) Hooks

```json
{
  "type": "command",
  "command": "long-task.sh",
  "async": true,
  "timeout": 300
}
```

- 백그라운드 실행 (차단하지 않음)
- 결정 제어 불가 (로깅/감사 전용)
- 중복 제거 적용

**활용**: 파일 편집 후 백그라운드 테스트, 감사 로깅

---

## 7. Hooks 설정 위치

| 위치 | 스코프 |
|------|--------|
| `~/.claude/settings.json` | 모든 프로젝트 |
| `.claude/settings.json` | 프로젝트 (git 공유) |
| `.claude/settings.local.json` | 로컬 전용 |
| 스킬 frontmatter | 스킬 활성 시만 |
| 서브에이전트 frontmatter | 에이전트 실행 시만 |

---

## 8. 실전 예시 모음

### 1. 파일 편집 후 자동 포맷팅
```json
{
  "hooks": {
    "PostToolUse": [{
      "matcher": "Edit|Write",
      "hooks": [{
        "type": "command",
        "command": ".claude/hooks/format.sh",
        "async": true
      }]
    }]
  }
}
```

```bash
#!/bin/bash
# .claude/hooks/format.sh
INPUT=$(cat)
FILE=$(echo "$INPUT" | jq -r '.tool_input.file_path')
EXT="${FILE##*.}"
case "$EXT" in
  js|ts|jsx|tsx) npx prettier --write "$FILE" 2>/dev/null ;;
  py) black "$FILE" 2>/dev/null ;;
  json) jq . "$FILE" > "${FILE}.tmp" && mv "${FILE}.tmp" "$FILE" ;;
esac
exit 0
```

### 2. .env 파일 편집 차단
```bash
#!/bin/bash
# .claude/hooks/protect-env.sh
INPUT=$(cat)
FILE=$(echo "$INPUT" | jq -r '.tool_input.file_path')
if [[ "$FILE" == *".env"* ]]; then
  echo ".env 파일 편집 차단됨" >&2
  exit 2
fi
exit 0
```

### 3. 파괴적 DB 명령 차단
```bash
#!/bin/bash
COMMAND=$(cat | jq -r '.tool_input.command // empty')
if echo "$COMMAND" | grep -iE '(DROP|TRUNCATE|DELETE)\s+(TABLE|DATABASE)'; then
  jq -n '{hookSpecificOutput: {hookEventName: "PreToolUse", permissionDecision: "deny", permissionDecisionReason: "파괴적 DB 명령은 명시적 승인 필요"}}'
  exit 0
fi
exit 0
```

### 4. 커밋 전 테스트 강제
```bash
#!/bin/bash
COMMAND=$(cat | jq -r '.tool_input.command // empty')
if echo "$COMMAND" | grep -q "git commit\|git push"; then
  if ! npm test > /dev/null 2>&1; then
    jq -n '{hookSpecificOutput: {hookEventName: "PreToolUse", permissionDecision: "deny", permissionDecisionReason: "테스트 실패. npm test 확인 필요."}}'
    exit 0
  fi
fi
exit 0
```

### 5. Compaction 후 컨텍스트 재주입
```json
{
  "hooks": {
    "PostCompact": [{
      "hooks": [{
        "type": "command",
        "command": "echo '{\"hookSpecificOutput\":{\"hookEventName\":\"PostCompact\",\"additionalContext\":\"중요: Bun 사용, 커밋 전 테스트, feature-branch에서만 작업\"}}'"
      }]
    }]
  }
}
```

### 6. 감사 로깅 (비동기)
```bash
#!/bin/bash
# .claude/hooks/audit.sh
INPUT=$(cat)
TOOL=$(echo "$INPUT" | jq -r '.tool_name')
COMMAND=$(echo "$INPUT" | jq -r '.tool_input.command // "N/A"')
TIMESTAMP=$(date -u "+%Y-%m-%dT%H:%M:%SZ")
jq -n --arg ts "$TIMESTAMP" --arg tool "$TOOL" --arg cmd "$COMMAND" \
  '{timestamp: $ts, tool: $tool, command: $cmd}' >> audit.jsonl
exit 0
```

### 7. 멀티 기준 Stop 검증 (Agent 훅)
```json
{
  "hooks": {
    "Stop": [{
      "hooks": [{
        "type": "agent",
        "prompt": "확인: 1) 모든 테스트 통과 2) 코드 문서화 3) console.log 없음. {\"ok\": true/false, \"reason\": \"...\"}"
      }]
    }]
  }
}
```

### 8. 데스크탑 알림
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

---

## 9. 보안 고려사항

1. **훅 스크립트는 사용자 권한으로 실행** - 모든 파일, 환경변수, SSH 키 접근 가능
2. **입력 검증 필수** - 사용자 입력이 포함된 데이터, shell injection 방지
3. **HTTP 훅**: `allowedEnvVars`로 환경변수 노출 제한
4. **Matcher 좁게**: `^Bash$` (not `.*`)
5. **훅도 코드 리뷰 대상** - 짧고 집중된 스크립트 유지

---

## 10. 디버그

```bash
# 디버그 모드
claude --debug

# verbose 토글 (세션 중)
Ctrl+O

# 수동 테스트
echo '{"tool_name":"Bash","tool_input":{"command":"rm -rf /"}}' | ./.claude/hooks/block-rm.sh
echo "Exit: $?"

# 훅 설정 확인
/hooks
```

### 흔한 문제
| 문제 | 원인 | 해결 |
|------|------|------|
| JSON 검증 실패 | 셸 프로파일의 무조건 출력 | `if [[ $- == *i* ]]`로 감싸기 |
| command not found | 상대 경로 | 절대 경로 사용 |
| jq 없음 | 미설치 | `brew install jq` |
| 훅 안 실행 | 실행 권한 없음 | `chmod +x script.sh` |
| 훅 안 뜸 | matcher 오류 | 대소문자 확인, `/hooks`에서 설정 확인 |

---

## 참고
- https://code.claude.com/docs/en/hooks.md
- https://code.claude.com/docs/en/hooks-guide.md
