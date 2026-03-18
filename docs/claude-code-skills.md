# Claude Code Skills 심층 조사

> 공식 문서 기반. 2026-03-18 기준.

---

## 1. Skills란?

YAML frontmatter + 마크다운으로 구성된 재사용 가능한 프롬프트 템플릿.
`/skill-name`으로 호출하거나 Claude가 자동으로 로드.

### 관련 기능 비교
| 기능 | 저장 | 호출 | 로딩 시점 | 용도 |
|------|------|------|-----------|------|
| **Skills** | `.claude/skills/` | `/name` 또는 자동 | 온디맨드/자동 | 재사용 워크플로우, 참조 지식 |
| **CLAUDE.md** | 프로젝트 루트 | 항상 | 매 세션 시작 | 영구 지시사항 |
| **서브에이전트** | `.claude/agents/` | 자동 위임 | 위임 시 | 격리된 복합 작업 |
| **Output Styles** | `.claude/output-styles/` | 전역 설정 | 세션 시작 | 톤/포맷 조정 |
| **Hooks** | `settings.json` | 이벤트 기반 | 생명주기 이벤트 | 확정적 자동화 |

---

## 2. 파일 위치

| 위치 | 스코프 | 우선순위 |
|------|--------|----------|
| `.claude/skills/<name>/SKILL.md` | 프로젝트 | 최고 |
| `~/.claude/skills/<name>/SKILL.md` | 모든 프로젝트 | 중간 |
| 플러그인/엔터프라이즈 | 조직 | 낮음 |

### 자동 탐색
- 하위 디렉토리의 `.claude/skills/`도 자동 발견 (모노레포 지원)
- `--add-dir` 외부 디렉토리의 스킬도 로드

---

## 3. Frontmatter 전체 필드

```yaml
---
# 기본
name: skill-identifier
description: 스킬 설명 (Claude가 자동 로드 판단 기준)

# 동작 제어
disable-model-invocation: false  # true면 수동 호출만 가능
user-invocable: true             # false면 / 메뉴에서 숨김
argument-hint: "[issue-number]"  # 자동완성 힌트

# 실행 제어
model: sonnet                    # sonnet, opus, haiku, inherit
context: fork                    # fork면 격리된 서브에이전트에서 실행
agent: Explore                   # 서브에이전트 타입

# 권한/도구
allowed-tools: Read, Grep        # 스킬 활성 시 허용 도구

# 자동화
hooks:
  PostToolUse:
    - matcher: "Edit|Write"
      hooks:
        - type: command
          command: "./scripts/format.sh"
---
```

### 필드별 사용 시점
- `disable-model-invocation: true` → 부작용 있는 스킬 (/deploy, /commit)
- `user-invocable: false` → 배경 지식용 (레거시 시스템 문서)
- `context: fork` + `agent: Explore` → 대량 리서치 격리
- `hooks` → 동적 검증/후처리

---

## 4. 스킬 콘텐츠 유형

### 참조 콘텐츠 (지식)
```yaml
---
name: api-conventions
description: API 설계 패턴
---
엔드포인트 작성 시:
- RESTful 네이밍: `/api/v1/resource`
- 응답 구조: `{ "data": {...}, "meta": {...}, "errors": [...] }`
- 타임스탬프: ISO 8601
```

### 태스크 콘텐츠 (워크플로우)
```yaml
---
name: deploy
description: 프로덕션 배포
context: fork
disable-model-invocation: true
---
$ARGUMENTS 배포:
1. 테스트 실행: `npm test`
2. 빌드: `npm run build`
3. 배포 스크립트 실행
4. 헬스 체크 확인
```

### 하이브리드 (지식 + 태스크)
가장 효과적. 참조 지식 + 단계별 작업 + 예제.

---

## 5. 문자열 치환

| 변수 | 설명 | 예시 |
|------|------|------|
| `$ARGUMENTS` | 전체 인자 | `/fix 123` → `123` |
| `$0`, `$1`, `$2` | 위치 인자 | `/migrate A B C` → `$0=A, $1=B, $2=C` |
| `${CLAUDE_SESSION_ID}` | 세션 ID | 로깅, 세션별 파일 생성 |
| `${CLAUDE_SKILL_DIR}` | 스킬 디렉토리 절대 경로 | 번들 스크립트 참조 |

### 인자 전달
```bash
/fix-issue 456                    # $ARGUMENTS = "456"
/migrate SearchBar React Vue      # $0=SearchBar, $1=React, $2=Vue
```

### 힌트 표시
```yaml
argument-hint: "[component] [from] [to]"
```

---

## 6. 동적 컨텍스트 주입 (`!` 문법)

셸 명령 결과를 스킬에 주입. Claude가 보기 전에 전처리됨.

```yaml
---
name: pr-summary
description: PR 요약
context: fork
agent: Explore
---
## PR 정보
- **PR 번호**: !`gh pr view --json number -q`
- **제목**: !`gh pr view --json title -q`
- **변경 파일**:
!`gh pr diff --name-only`

- **Diff** (5000자):
!`gh pr diff | head -c 5000`
```

**동작**: `!`command`` 즉시 실행 → 결과로 대체 → Claude가 실제 데이터와 함께 작업

### 활용
- GitHub PR 컨텍스트
- Git 상태 (`git log`, `git status`)
- 환경별 설정
- 프로젝트 타입 감지

---

## 7. 고급 패턴

### context: fork (서브에이전트 실행)
```yaml
---
name: deep-research
context: fork
agent: Explore     # 읽기전용, Haiku(빠름)
---
$ARGUMENTS 깊이 조사:
1. Glob/Grep으로 관련 파일 찾기
2. 코드 분석
3. 패턴/아키텍처 결정사항 정리
```
- 메인 대화 오염 없이 대량 탐색
- 결과 요약만 메인으로 반환

### 에이전트 타입
| 에이전트 | 도구 | 모델 | 용도 |
|----------|------|------|------|
| `Explore` | 읽기전용 | Haiku | 코드 탐색 |
| `Plan` | 읽기전용 | 상속 | 리서치 중심 계획 |
| `general-purpose` | 전체 | 상속 | 복합 작업 |
| 커스텀 | 가변 | 가변 | `.claude/agents/`의 커스텀 |

### 스킬 접근 제한
```json
{
  "permissions": {
    "deny": ["Skill(deploy *)"],
    "allow": ["Skill(commit)", "Skill(review-pr *)"]
  }
}
```

---

## 8. 지원 파일

```
my-skill/
├── SKILL.md          # 필수: 메인 지시
├── api-reference.md  # 선택: 상세 문서
├── examples.md       # 선택: 사용 예제
├── template.md       # 선택: 템플릿
└── scripts/
    └── helper.sh     # 선택: 실행 스크립트
```

SKILL.md에서 `[examples.md](examples.md)` 형태로 참조.

---

## 9. 번들(내장) 스킬

| 스킬 | 명령 | 설명 |
|------|------|------|
| `/batch` | `batch <instruction>` | 5-30 worktree에서 병렬 대규모 리팩토링, 각각 PR 생성 |
| `/simplify` | `simplify [focus]` | 3개 에이전트로 코드 품질 리뷰 후 자동 수정 |
| `/debug` | `debug [description]` | 세션 디버그 로그 분석 |
| `/loop` | `loop [interval] <prompt>` | 반복 실행 (기본 10분, 예: `5m`, `30s`, `1h`) |
| `/claude-api` | `claude-api` | Claude API 레퍼런스 로드 (anthropic SDK import 시 자동) |

### /batch 상세
```bash
/batch src/에서 Solid.js를 React로 마이그레이션
```
1. 코드베이스 분석
2. 5-30개 독립 단위로 분해
3. 계획 승인 요청
4. 단위별 격리 worktree에서 에이전트 실행
5. 각 에이전트가 테스트 후 PR 생성

### /simplify 상세
```bash
/simplify                          # 전반적 리뷰
/simplify 메모리 효율에 집중      # 포커스 리뷰
```
3개 에이전트 (품질/성능/베스트 프랙티스) 병렬 리뷰 → 결과 종합 → 자동 수정

---

## 10. 트러블슈팅

### 스킬이 자동 트리거 안 됨
1. `What skills are available?`으로 목록 확인
2. description이 충분히 구체적인지 확인
3. `disable-model-invocation: true` 여부 확인
4. `/skill-name`으로 수동 호출 테스트

### 스킬이 너무 자주 트리거됨
1. description 더 구체적으로 수정
2. `disable-model-invocation: true` 설정 (수동 전용)
3. 일반적인 키워드 피하기

### 스킬이 목록에 안 보임
- 스킬 description에 **글자 수 예산** 있음 (컨텍스트 윈도우의 2%, 최소 16,000자)
- 해결: 스킬 수 줄이기, description 짧게, `SLASH_COMMAND_TOOL_CHAR_BUDGET=32000` 설정

---

## 11. 실전 예시

### 이슈 수정 스킬
```yaml
---
name: fix-issue
description: GitHub 이슈를 번호로 수정
disable-model-invocation: true
argument-hint: "[issue-number]"
---
GitHub issue #$ARGUMENTS 수정:
1. `gh issue view $ARGUMENTS`로 이슈 확인
2. 요구사항 파악
3. `git checkout -b fix/issue-$ARGUMENTS`
4. 수정 구현 + 테스트
5. `git commit -m "Fix: Resolve issue #$ARGUMENTS"`
```

### PR 리뷰 스킬 (동적 데이터)
```yaml
---
name: review-pr
description: 현재 PR 리뷰
context: fork
agent: Explore
allowed-tools: Bash(gh *)
---
## 변경 파일
!`gh pr diff --name-only`

## Diff
!`gh pr diff | head -c 10000`

## 리뷰 체크리스트
1. 코드 품질 (네이밍, 중복, 에러 처리)
2. 테스트 (커버리지, 엣지 케이스)
3. 보안 (하드코딩 시크릿, 인젝션)
4. 성능 (알고리즘, DB 쿼리)
```

### 테스트 작성 스킬 (기존 테스트 참조)
```yaml
---
name: write-test
description: 프로젝트 컨벤션에 맞는 테스트 작성
allowed-tools: Read, Write, Bash(npm test)
argument-hint: "[file-path]"
---
## 기존 테스트 컨벤션
!`find src/__tests__ -name "*.test.ts" -type f | head -1 | xargs head -30`

$0에 대한 테스트 작성:
1. 테스트 파일 생성
2. 모듈 import
3. describe/it 블록 (happy path + edge case)
4. `npm test` 실행
```

---

## 참고
- https://code.claude.com/docs/en/skills.md
- https://code.claude.com/docs/en/commands.md
