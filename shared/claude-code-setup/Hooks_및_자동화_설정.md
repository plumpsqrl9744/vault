# Claude Code Hooks 및 자동화 설정

#claude-code #hooks #automation

> Claude Code hooks를 활용한 자동화 설정 — 코드 품질 체크, 문서 동기화, 데스크톱 펫 연동

## Hooks 개요

Claude Code hooks는 특정 이벤트 발생 시 자동 실행되는 스크립트.
`~/.claude/settings.json`의 `hooks` 필드에서 설정한다.

## 커스텀 Hooks

### 1. ts-any-check (PostToolUse)
- **매처**: Write | Edit
- **동작**: TypeScript 파일 편집 시 `any` 타입 사용을 감지하여 경고
- **목적**: 타입 안전성 유지

### 2. vault-index-reminder (PostToolUse)
- **매처**: Write | Edit
- **동작**: vault 문서 파일 수정 시 `_index.md` 갱신 필요 여부 알림
- **목적**: 옵시디언 인덱스 최신화

### 3. doc-sync-reminder (PostToolUse)
- **매처**: Write | Edit
- **동작**: 코드 변경 시 관련 문서 동기화 필요 여부 알림
- **목적**: 코드-문서 간 일관성 유지

### 4. precompact-context (PreCompact)
- **동작**: 컨텍스트 압축 전 프로젝트 핵심 정보를 보존
- **목적**: 대화가 길어져도 중요 맥락 유지

## settings.json 구조

```jsonc
{
  "permissions": {
    "allow": ["mcp__pencil"],
    "deny": ["Read(**/.env*)", "Bash(*rm -rf*)"],
    "mode": "auto"
  },
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Write|Edit",
        "hooks": [
          { "type": "command", "command": "node hooks/ts-any-check.js" },
          { "type": "command", "command": "node hooks/vault-index-reminder.js" },
          { "type": "command", "command": "node hooks/doc-sync-reminder.js" }
        ]
      }
    ],
    "PreCompact": [
      {
        "hooks": [
          { "type": "command", "command": "node hooks/precompact-context.js" }
        ]
      }
    ]
  },
  "language": "korean",
  "effortLevel": "high"
}
```

## 이벤트 종류

| 이벤트 | 발생 시점 |
|-|-|
| SessionStart | 세션 시작 |
| SessionEnd | 세션 종료 |
| UserPromptSubmit | 사용자 입력 제출 |
| PreToolUse | 도구 실행 전 |
| PostToolUse | 도구 실행 후 |
| PostToolUseFailure | 도구 실행 실패 |
| PreCompact | 컨텍스트 압축 전 |
| PostCompact | 컨텍스트 압축 후 |
| Stop | 응답 완료 |
| SubagentStart | 서브에이전트 시작 |
| SubagentStop | 서브에이전트 종료 |
| Notification | 알림 발생 |
| PermissionRequest | 권한 요청 |

## 권한 설정

| 설정 | 의미 |
|-|-|
| `allow: ["mcp__pencil"]` | Pencil MCP 도구 자동 허용 |
| `deny: ["Read(**/.env*)"]` | .env 파일 읽기 차단 |
| `deny: ["Bash(*rm -rf*)"]` | rm -rf 명령 차단 |
| `mode: "auto"` | 나머지는 자동 판단 |
