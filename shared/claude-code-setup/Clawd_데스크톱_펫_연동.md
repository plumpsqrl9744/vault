# Clawd on Desk 데스크톱 펫 연동

#claude-code #clawd #desktop-pet #electron

> Claude Code 작업 상태를 실시간으로 시각화하는 데스크톱 펫 앱 설정

## 개요

Clawd on Desk는 AI 코딩 에이전트의 작업 상태를 데스크톱에서 실시간으로 보여주는 Electron 앱.
Claude Code hooks를 통해 상태를 전달받아 애니메이션으로 표현한다.

## 설치

```bash
git clone https://github.com/rullerzhou-afk/clawd-on-desk.git
cd clawd-on-desk
npm install
npm start  # hooks 자동 등록 + 앱 실행
```

## 지원 에이전트

| 에이전트 | 연동 방식 |
|-|-|
| Claude Code | command hooks + HTTP permission hooks |
| Gemini CLI | ~/.gemini/settings.json hooks |
| Cursor Agent | ~/.cursor/hooks.json |
| Codex CLI | JSONL 로그 폴링 |
| Copilot CLI | ~/.copilot/hooks/hooks.json |
| opencode | plugin integration |

## 동작 원리

1. `npm start` 시 Claude Code `settings.json`에 hooks 자동 등록 (15개 이벤트)
2. State server가 `127.0.0.1:23333`에서 리스닝
3. 각 hook 이벤트 발생 → `clawd-hook.js` 실행 → state server에 상태 전송
4. Clawd 앱이 상태에 따라 애니메이션 변경

## 등록되는 Hook 이벤트

| 이벤트 | Clawd 반응 |
|-|-|
| UserPromptSubmit | thinking 애니메이션 |
| PreToolUse / PostToolUse | typing 애니메이션 |
| SubagentStart | juggling (1개) / conducting (2개+) |
| Stop | happy 애니메이션 |
| PostToolUseFailure | error 애니메이션 |
| PermissionRequest | 팝업 버블 (Allow/Deny) |

## Permission Bubble

- Claude Code가 권한 요청 시 Clawd가 팝업 카드 표시
- `Ctrl+Shift+Y` — Allow, `Ctrl+Shift+N` — Deny
- 터미널에서 먼저 응답하면 버블 자동 소멸
- HTTP hook으로 동작: `http://127.0.0.1:23333/permission`

## 애니메이션 상태 (12종)

| 상태 | 설명 |
|-|-|
| idle | 대기 (커서 따라감, 60초 후 수면) |
| thinking | 프롬프트 처리 중 |
| typing | 도구 실행 중 |
| building | 빌드/컴파일 중 |
| juggling | 서브에이전트 1개 |
| conducting | 서브에이전트 2개+ |
| error | 에러 발생 |
| happy | 작업 완료 |
| notification | 알림 |
| sweeping | 정리 중 |
| sleeping | 수면 (60초 idle 후) |
| carrying | 파일 이동 중 |

## 기능

- **Mini Mode**: 우클릭 또는 화면 우측 끝 드래그 → 엣지에 숨김
- **Do Not Disturb**: 모든 hook 이벤트 무시, 수면 모드
- **사운드 이펙트**: 완료/권한요청 시 효과음 (10초 쿨다운)
- **위치 기억**: 재시작해도 마지막 위치 유지
- **싱글 인스턴스**: 중복 실행 방지
- **테마**: Clawd (픽셀 게), Calico (삼색 고양이), 커스텀 테마 지원

## 트레이 메뉴

- 크기 조절 (S/M/L)
- DND 모드
- 언어 전환 (EN/CN/KR)
- 자동 시작 설정
- 업데이트 확인
