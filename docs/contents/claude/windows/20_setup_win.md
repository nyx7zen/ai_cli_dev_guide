# Windows 개발 환경 설정

## Overview

- 목적: Windows 10/11 환경에서 Claude Code CLI 워크플로를 사용하기 위한 도구 설치 및 초기 설정
- 사전 조건: 아래 표의 도구가 이미 설치되어 있어야 한다

---

## 사전 조건 확인

### 기설치 도구 (설치 완료 가정)

| 도구 | 확인 명령 | 기대 출력 예시 |
|---|---|---|
| WinPython | `python --version` | `Python 3.11.x` |
| MinGW64 | `g++ --version` | `g++ (MinGW-W64) 13.x.x` |
| VSCode | `code --version` | `1.9x.x` |
| Git | `git --version` | `git version 2.x.x` |

```powershell
# Run all checks at once
python --version; g++ --version; code --version; git --version
```

WinPython을 사용하는 경우 `python.exe` 경로가 시스템 PATH에 등록되어 있어야 한다.
등록되어 있지 않으면 WinPython 설치 폴더의 `WinPython Command Prompt.exe`를 통해 실행한다.

---

## Node.js / npm 설치

Claude Code는 Node.js 런타임 위에서 실행된다. npm을 통해 전역 설치된다.

### 설치

1. [https://nodejs.org](https://nodejs.org) 에서 LTS 버전 Windows Installer (.msi) 다운로드
2. 설치 시 "Add to PATH" 옵션 활성화 확인
3. 설치 완료 후 PowerShell 재시작

### 설치 확인

```powershell
node --version
npm --version
```

예상 출력:

```
v22.x.x
10.x.x
```

### npm 전역 설치 경로 확인

```powershell
npm config get prefix
```

출력 경로가 시스템 PATH에 포함되어 있어야 한다. 포함되지 않은 경우:

```powershell
# Add to user PATH (current session)
$env:PATH += ";$(npm config get prefix)"

# Persist across sessions via system settings
[System.Environment]::SetEnvironmentVariable(
  "PATH",
  $env:PATH + ";$(npm config get prefix)",
  "User"
)
```

---

## Claude Code 설치 및 인증

### 설치

```powershell
npm install -g @anthropic-ai/claude-code
```

### 설치 확인

```powershell
claude --version
```

예상 출력:

```
1.x.x
```

### 인증

Claude Code는 Anthropic API key 또는 Claude.ai 계정으로 인증한다.

```powershell
claude
```

최초 실행 시 인증 방식 선택 프롬프트가 표시된다:

```
? How would you like to authenticate?
  > Login with Claude.ai (recommended)
    Enter API Key
```

Claude Pro 구독 계정은 "Login with Claude.ai" 선택 후 브라우저 인증을 완료한다.

인증 완료 확인:

```powershell
claude --print "hello"
```

응답이 출력되면 인증 성공이다.

---

## GitHub CLI 설치 및 인증

### 설치

```powershell
winget install --id GitHub.cli
```

설치 완료 후 PowerShell 재시작.

### 설치 확인

```powershell
gh --version
```

예상 출력:

```
gh version 2.x.x
```

### 인증

```powershell
gh auth login
```

프롬프트 선택 순서:

```
? Where do you use GitHub?  GitHub.com
? What is your preferred protocol for Git operations?  HTTPS
? How would you like to authenticate GitHub CLI?  Login with a web browser
```

브라우저에서 표시되는 one-time code를 입력하여 인증 완료.

### 인증 확인

```powershell
gh auth status
```

예상 출력:

```
github.com
  Logged in to github.com as <username>
  Git operations for github.com configured to use https protocol.
```

---

## .gitignore / .env 초기 설정

### .gitignore 기본 템플릿

프로젝트 루트에 생성:

```powershell
New-Item -Path ".gitignore" -ItemType File
```

내용:

```
# Environment variables
.env
.env.local

# Claude Code personal settings
CLAUDE.local.md
.claude/settings.local.json

# Python
__pycache__/
*.py[cod]
*.egg-info/
.pytest_cache/
dist/
build/

# C++
*.o
*.obj
*.exe
*.out
build/
cmake-build-*/

# WinPython
Scripts/
Lib/

# VSCode
.vscode/launch.json

# OS
Thumbs.db
desktop.ini
```

### .env 작성 규칙

```powershell
New-Item -Path ".env" -ItemType File
```

내용 예시:

```
# Anthropic API Key (never commit this file)
ANTHROPIC_API_KEY=sk-ant-...

# Ollama endpoint (Windows native)
OLLAMA_HOST=http://localhost:11434

# LLM provider selection: claude | ollama
LLM_PROVIDER=claude
```

`.env` 파일은 반드시 `.gitignore`에 포함되어야 한다. 커밋하지 않는다.

---

## .claude/ 폴더 구조 생성

### PowerShell 명령

```powershell
# Navigate to project root first
cd C:\path\to\my-project

# Create .claude/ directory structure
New-Item -ItemType Directory -Path ".claude\rules"    -Force
New-Item -ItemType Directory -Path ".claude\skills\tdd"         -Force
New-Item -ItemType Directory -Path ".claude\skills\refactoring" -Force
New-Item -ItemType Directory -Path ".claude\skills\code-reviewer" -Force
New-Item -ItemType Directory -Path ".claude\agents"   -Force

# Create configuration files
New-Item -Path ".claude\settings.json"       -ItemType File -Force
New-Item -Path ".claude\settings.local.json" -ItemType File -Force
New-Item -Path ".claude\.mcp.json"           -ItemType File -Force

# Create rule files
New-Item -Path ".claude\rules\conventions.md" -ItemType File -Force
New-Item -Path ".claude\rules\tdd.md"         -ItemType File -Force

# Create skill files
New-Item -Path ".claude\skills\tdd\SKILL.md"            -ItemType File -Force
New-Item -Path ".claude\skills\refactoring\SKILL.md"    -ItemType File -Force
New-Item -Path ".claude\skills\code-reviewer\SKILL.md"  -ItemType File -Force

# Create agent files
New-Item -Path ".claude\agents\code_reviewer.md" -ItemType File -Force
New-Item -Path ".claude\agents\tdd_agent.md"     -ItemType File -Force
```

### 구조 확인

```powershell
tree .claude /F
```

예상 출력:

```
.claude
|-- .mcp.json
|-- settings.json
|-- settings.local.json
|-- agents
|   |-- code_reviewer.md
|   |-- tdd_agent.md
|-- rules
|   |-- conventions.md
|   |-- tdd.md
|-- skills
    |-- code-reviewer
    |   |-- SKILL.md
    |-- refactoring
    |   |-- SKILL.md
    |-- tdd
        |-- SKILL.md
```

---

## CLAUDE.md 초기 템플릿

프로젝트 루트에 `CLAUDE.md`를 생성한다. 이 파일은 Claude Code가 모든 세션에서 자동으로 로드하는 프로젝트 컨텍스트 파일이다.

```powershell
New-Item -Path "CLAUDE.md" -ItemType File
```

템플릿 내용 (Windows 환경 기준):

```markdown
# Project Context

## Environment
- OS: Windows 10/11
- Shell: PowerShell
- Python: WinPython 3.11 (portable)
- C++: MinGW64, CMake
- Editor: VSCode (native Windows)
- GPU: CPU only

## Path Rules
- Use os.path style (backslash on Windows: C:\path\to\file)
- Project root: C:\Users\<username>\projects\<project-name>

## Language Rules
- Code comments: English only
- Variable names: snake_case (Python), camelCase (C++)
- No print() debugging — use logging module

## Test Rules
- Python: pytest
- C++: Google Test
- All new functions require a corresponding test

## Commit Rules
- Commit every time a test passes
- Format: feat|test|refactor|fix|docs|chore: <description>

## Forbidden Patterns
- No hardcoded paths
- No API keys in source code
- No commented-out code in commits
```

200줄을 초과하지 않도록 유지한다. 프로젝트별로 필요한 항목만 포함한다.

---

## 설치 확인 체크리스트

- [ ] `node --version` 출력 확인
- [ ] `npm --version` 출력 확인
- [ ] `claude --version` 출력 확인
- [ ] `claude --print "hello"` 응답 확인 (인증 완료)
- [ ] `gh --version` 출력 확인
- [ ] `gh auth status` 로그인 상태 확인
- [ ] `.gitignore` 파일 존재 확인 (`.env` 포함 여부)
- [ ] `.env` 파일 존재 확인 (`.gitignore`에 의해 추적 제외)
- [ ] `.claude/` 폴더 구조 존재 확인 (`tree .claude /F`)
- [ ] `CLAUDE.md` 파일 존재 확인

---

## 공통 오류 및 해결

| 오류 | 원인 | 해결 |
|---|---|---|
| `claude : The term 'claude' is not recognized` | npm 전역 경로가 PATH에 없음 | `$env:PATH += ";$(npm config get prefix)"` 실행 후 PowerShell 재시작 |
| `npm install -g` 권한 오류 | PowerShell을 관리자 권한으로 실행하지 않음 | PowerShell을 "관리자 권한으로 실행"하거나 `npm config set prefix` 로 사용자 경로 변경 |
| `gh : command not found` | winget 설치 후 PATH 미반영 | PowerShell 재시작 또는 `$env:PATH` 수동 추가 |
| Claude Code 인증 브라우저 미열림 | 기본 브라우저 설정 문제 | 출력된 URL을 수동으로 브라우저에 붙여넣어 인증 |
| `python --version` 오류 | WinPython PATH 미등록 | WinPython 설치 폴더의 `WinPython Command Prompt.exe` 사용 또는 수동 PATH 등록 |
| `g++ --version` 오류 | MinGW64 PATH 미등록 | MinGW64 `bin` 폴더를 시스템 PATH에 수동 추가 |
| `.env` 파일이 git status에 표시됨 | `.gitignore`에 `.env` 미포함 | `.gitignore`에 `.env` 라인 추가 후 `git rm --cached .env` 실행 |
