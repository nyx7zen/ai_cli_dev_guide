# VSCode 설정

## Overview

- 이 문서는 Claude Code VSCode Extension 설치와 설정을 다룬다.
- Extension 주요 기능, 권장 설정, 단축키, 사용 패턴을 포함한다.
- Windows와 WSL 환경 모두에 적용된다.

**사전 조건:**
- VSCode가 설치되어 있다
- Claude Code CLI가 설치되어 있다 (`npm install -g @anthropic-ai/claude-code`)
- Claude Code 인증이 완료되어 있다

---

## Claude Code Extension 설치

**설치 방법:**

1. VSCode에서 `Ctrl+Shift+X` (Extensions 패널 열기)
2. 검색창에 `Claude Code` 입력
3. 게시자가 **Anthropic**인 항목 선택
4. Install 클릭

**또는 명령 팔레트로 설치:**
```
Ctrl+Shift+P -> "Extensions: Install Extensions" -> "Claude Code" 검색
```

**설치 확인:**
- 왼쪽 사이드바에 Claude Code 아이콘이 나타난다
- 하단 상태 표시줄에 Claude Code 상태가 표시된다

**WSL 환경 주의사항:**
VSCode Remote-WSL 세션에서는 Extension이 WSL 측에 별도로 설치되어야 한다. Remote-WSL로 연결된 상태에서 위 설치 과정을 반복한다.

---

## Extension 주요 기능

### 사이드바 패널

왼쪽 사이드바의 Claude Code 아이콘을 클릭하면 대화 패널이 열린다.

- 파일을 열지 않고도 Claude Code와 대화할 수 있다
- `@` 기호로 파일, 폴더, 심볼을 컨텍스트로 추가한다
- 대화 이력이 패널에 유지된다

### 인라인 diff

Claude Code가 파일을 수정할 때 변경 전/후를 나란히 표시한다.

- 수락(Accept): 변경 내용을 파일에 적용한다
- 거절(Reject): 변경 내용을 취소하고 원본을 유지한다
- 부분 수락: 변경 내용 중 일부만 선택해 적용한다

### @mention

대화 중 `@` 기호로 특정 파일이나 폴더를 컨텍스트로 추가한다.

```
@src/calculator.py 파일을 읽고 add 함수를 설명해라
@tests/ 폴더의 모든 테스트를 확인해라
```

### Rewind

이전 대화 상태로 되돌리는 기능이다. Claude Code가 잘못된 방향으로 진행했을 때 사용한다.

- 패널 상단의 Rewind 버튼 클릭
- 되돌릴 대화 시점 선택
- 파일 변경 사항도 함께 복원된다

### Checkpoint

현재 작업 상태를 저장하는 기능이다. 대규모 변경 전에 체크포인트를 만들어두면 이전 상태로 복구할 수 있다.

```
# 체크포인트 생성
/checkpoint save "before refactoring"

# 체크포인트 복구
/checkpoint restore "before refactoring"
```

---

## .vscode/settings.json 권장 설정

프로젝트 루트에 `.vscode/settings.json`을 생성하고 아래 설정을 적용한다.

```json
{
  "files.autoSave": "afterDelay",
  "files.autoSaveDelay": 1000,
  "editor.formatOnSave": true,
  "editor.rulers": [88],
  "editor.tabSize": 4,
  "python.defaultInterpreterPath": "${workspaceFolder}/.venv/bin/python",
  "python.testing.pytestEnabled": true,
  "python.testing.pytestArgs": ["tests"],
  "cmake.buildDirectory": "${workspaceFolder}/build",
  "[python]": {
    "editor.defaultFormatter": "ms-python.black-formatter"
  },
  "[cpp]": {
    "editor.defaultFormatter": "ms-vscode.cpptools"
  }
}
```

**autoSave 설정은 필수다.** Claude Code가 파일을 수정한 후 VSCode가 저장하지 않은 상태로 남아있으면 다음 작업에서 충돌이 발생할 수 있다. `afterDelay: 1000` (1초 후 자동 저장)을 반드시 설정한다.

**WSL 환경 추가 설정:**
```json
{
  "remote.WSL.fileWatcher.polling": true,
  "files.watcherExclude": {
    "**/node_modules/**": true,
    "**/.git/**": true
  }
}
```

---

## VSCode에서 .claude/ 폴더 구조 생성

Extension 설치 후 VSCode 터미널에서 `.claude/` 폴더 구조를 생성한다.

**Windows (PowerShell 터미널):**
```powershell
# Create .claude folder structure
$dirs = @(
    ".claude\rules",
    ".claude\skills\tdd",
    ".claude\skills\refactoring",
    ".claude\skills\code-reviewer",
    ".claude\agents"
)
foreach ($dir in $dirs) { New-Item -ItemType Directory -Force -Path $dir }

# Create placeholder files
New-Item -ItemType File -Force -Path ".claude\rules\conventions.md"
New-Item -ItemType File -Force -Path ".claude\rules\tdd.md"
New-Item -ItemType File -Force -Path ".claude\skills\tdd\SKILL.md"
New-Item -ItemType File -Force -Path ".claude\agents\code_reviewer.md"
New-Item -ItemType File -Force -Path ".claude\.mcp.json"
```

**WSL (bash 터미널):**
```bash
# Create .claude folder structure
mkdir -p .claude/rules \
         .claude/skills/tdd \
         .claude/skills/refactoring \
         .claude/skills/code-reviewer \
         .claude/agents

# Create placeholder files
touch .claude/rules/conventions.md
touch .claude/rules/tdd.md
touch .claude/skills/tdd/SKILL.md
touch .claude/agents/code_reviewer.md
touch .claude/.mcp.json
```

생성 후 VSCode 탐색기(Explorer)에서 `.claude/` 폴더가 보이는지 확인한다.

---

## 단축키 표

| 기능 | Windows | WSL (Remote) |
|---|---|---|
| Claude Code 패널 열기/닫기 | `Ctrl+Shift+C` | `Ctrl+Shift+C` |
| 인라인 편집 요청 | 코드 선택 후 `Ctrl+K` | 코드 선택 후 `Ctrl+K` |
| 변경 수락 | `Ctrl+Enter` | `Ctrl+Enter` |
| 변경 거절 | `Escape` | `Escape` |
| Extensions 패널 | `Ctrl+Shift+X` | `Ctrl+Shift+X` |
| 통합 터미널 열기 | `Ctrl+` ` | `Ctrl+` ` |
| 명령 팔레트 | `Ctrl+Shift+P` | `Ctrl+Shift+P` |
| 파일 빠른 열기 | `Ctrl+P` | `Ctrl+P` |

---

## Extension 패널 vs 통합 터미널 선택 기준

| 상황 | 권장 방식 |
|---|---|
| 코드 설명, 단순 질문 | Extension 패널 |
| 특정 파일 수정 요청 | Extension 패널 (@mention 활용) |
| TDD 사이클 전체 진행 | 통합 터미널 (`claude` 명령) |
| 세션 시작 / 종료 루틴 | 통합 터미널 |
| `/export`, `/mcp` 등 슬래시 명령 | 통합 터미널 |
| agents 호출 | 통합 터미널 |

**원칙:**
단발성 질문이나 파일 수정은 Extension 패널이 편리하다. TDD 사이클처럼 여러 단계가 연속되고 슬래시 명령을 사용하는 작업은 통합 터미널에서 `claude` 명령으로 실행한다.

---

## Verification Checklist

- VSCode 사이드바에 Claude Code 아이콘이 보인다
- `.vscode/settings.json`에 `autoSave: afterDelay, 1000`이 설정되어 있다
- WSL Remote 환경에서도 Extension이 설치되어 있다
- `.claude/` 폴더가 VSCode 탐색기에서 보인다
- 통합 터미널에서 `claude --version` 명령이 실행된다

---

## Common Errors and Solutions

| 오류 | 원인 | 해결 |
|---|---|---|
| Extension이 설치됐지만 아이콘이 보이지 않는다 | VSCode 재시작 필요 | `Ctrl+Shift+P` -> "Reload Window" |
| WSL에서 Extension이 동작하지 않는다 | Remote-WSL 측 미설치 | Remote-WSL 세션에서 Extension 재설치 |
| 인라인 diff가 표시되지 않는다 | Claude Code CLI 미인증 | 터미널에서 `claude` 실행 후 인증 완료 |
| autoSave 후 Claude Code가 충돌한다 | 동시 저장 충돌 | `files.autoSaveDelay`를 2000으로 증가 |
| `.claude/` 폴더가 탐색기에서 보이지 않는다 | 숨김 파일 표시 비활성화 | VSCode 탐색기에서 숨김 파일 표시 설정 활성화 |
