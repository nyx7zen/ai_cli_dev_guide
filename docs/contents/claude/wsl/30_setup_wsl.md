# WSL 개발 환경 설정

## Overview

- **목적**: WSL2 (Ubuntu) 환경에서 Claude Code CLI 워크플로를 사용하기 위한 환경 설정
- **사전 조건**: 아래 표의 도구가 이미 설치되어 있다고 가정한다

### 사전 설치 완료 가정 항목

| 도구 | 확인 명령 | 비고 |
|---|---|---|
| WSL2 (Ubuntu) | `wsl --version` (PowerShell) | Ubuntu 20.04 이상 |
| Anaconda | `conda --version` | base 환경 활성화 상태 |
| VSCode | `code --version` | Windows 측 설치 |
| VSCode Remote - WSL Extension | VSCode Extensions 확인 | ms-vscode-remote.remote-wsl |
| Git | `git --version` | Ubuntu 내부 |
| GitHub 계정 | github.com 로그인 확인 | - |

---

## nvm / Node.js / npm 설치

Claude Code는 Node.js 런타임이 필요하다. WSL에서는 nvm을 통해 Node.js를 설치한다.
시스템 패키지 관리자(apt)로 설치한 Node.js는 버전이 오래되어 Claude Code와 호환되지 않을 수 있으므로 nvm을 사용한다.

### nvm 설치

```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash
```

설치 스크립트가 `.bashrc` 또는 `.zshrc`에 자동으로 PATH 설정을 추가한다.
설치 후 셸을 재시작하거나 직접 반영한다.

```bash
source ~/.bashrc
```

### PATH 설정 확인

`.bashrc` 또는 `.zshrc` 파일에 아래 내용이 추가되었는지 확인한다.

```bash
export NVM_DIR="$HOME/.nvm"
[ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"
[ -s "$NVM_DIR/bash_completion" ] && \. "$NVM_DIR/bash_completion"
```

없으면 수동으로 추가한다.

```bash
echo 'export NVM_DIR="$HOME/.nvm"' >> ~/.bashrc
echo '[ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"' >> ~/.bashrc
source ~/.bashrc
```

### Node.js LTS 버전 설치

```bash
nvm install --lts
nvm use --lts
nvm alias default lts/*
```

### 설치 확인

```bash
node --version
# Expected: v20.x.x 이상

npm --version
# Expected: 10.x.x 이상

nvm --version
# Expected: 0.39.x 이상
```

---

## Claude Code 설치 및 인증

### 설치

```bash
npm install -g @anthropic-ai/claude-code
```

### 버전 확인

```bash
claude --version
```

### 인증

```bash
claude login
```

브라우저가 열리면서 Anthropic 계정 로그인 페이지가 표시된다.
Claude Pro 계정으로 로그인한다.
WSL에서 브라우저가 열리지 않으면 URL을 복사하여 Windows 브라우저에서 열고 인증 코드를 터미널에 붙여넣는다.

### 인증 상태 확인

```bash
claude auth status
# Expected: Logged in as: your-email@example.com
```

---

## VSCode Remote-WSL 연결 확인

VSCode는 Windows에 설치되어 있으나, Remote-WSL Extension을 통해 WSL 파일 시스템에서 직접 작업한다.

### 연결 방법

WSL 터미널에서 프로젝트 폴더 내에서 실행한다.

```bash
cd ~/projects/my-project
code .
```

Windows VSCode가 열리면서 좌하단에 `WSL: Ubuntu` 배지가 표시되면 정상이다.

### 연결 상태 확인 항목

- VSCode 좌하단: `WSL: Ubuntu` 표시
- 터미널 패널: `/bin/bash` 또는 `/bin/zsh` (Ubuntu 셸)
- 파일 탐색기: `/home/username/` 경로

---

## GitHub CLI 설치 및 인증

### 설치

```bash
sudo apt update
sudo apt install gh -y
```

설치 가능한 버전이 오래된 경우 GitHub 공식 저장소를 추가한다.

```bash
curl -fsSL https://cli.github.com/packages/githubcli-archive-keyring.gpg \
  | sudo dd of=/usr/share/keyrings/githubcli-archive-keyring.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/githubcli-archive-keyring.gpg] \
  https://cli.github.com/packages stable main" \
  | sudo tee /etc/apt/sources.list.d/github-cli.list > /dev/null
sudo apt update
sudo apt install gh -y
```

### 인증

```bash
gh auth login
```

프롬프트 응답 순서:

```
? What account do you want to log into? GitHub.com
? What is your preferred protocol for Git operations? SSH
? Generate a new SSH key to add to your GitHub account? Yes
? Enter a passphrase for your new SSH key (Optional): [Enter 또는 패스프레이즈 입력]
? Title for your SSH key: [Enter]
? How would you like to authenticate GitHub CLI? Login with a web browser
```

브라우저에서 one-time code를 입력하여 인증을 완료한다.

### 인증 상태 확인

```bash
gh auth status
# Expected:
# github.com
#   Logged in to github.com as your-username
#   Git operations protocol: ssh
#   Token: gho_xxxx
```

---

## SSH 키 생성 및 GitHub 등록

`gh auth login`에서 SSH 키를 자동 생성했다면 이 단계를 건너뛴다.
수동으로 SSH 키를 관리하려면 아래 절차를 따른다.

### SSH 키 생성

```bash
ssh-keygen -t ed25519 -C "your-email@example.com"
# 저장 경로: Enter (기본값 ~/.ssh/id_ed25519 사용)
# 패스프레이즈: Enter 또는 원하는 값 입력
```

### ssh-agent 설정

```bash
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_ed25519
```

매 세션마다 자동으로 실행되도록 `.bashrc`에 추가한다.

```bash
cat >> ~/.bashrc << 'EOF'

# Start ssh-agent automatically
if [ -z "$SSH_AUTH_SOCK" ]; then
  eval "$(ssh-agent -s)" > /dev/null
  ssh-add ~/.ssh/id_ed25519 2>/dev/null
fi
EOF
```

### GitHub 퍼블릭 키 등록

```bash
# gh CLI로 자동 등록
gh ssh-key add ~/.ssh/id_ed25519.pub --title "WSL-Ubuntu"

# 또는 수동으로 공개키를 복사하여 github.com/settings/keys 에서 등록
cat ~/.ssh/id_ed25519.pub
```

### 연결 확인

```bash
ssh -T git@github.com
# Expected: Hi your-username! You've successfully authenticated...
```

---

## 초기 파일 설정

### .gitconfig 설정

```bash
git config --global user.name "Your Name"
git config --global user.email "your-email@example.com"
git config --global init.defaultBranch main
git config --global core.editor "code --wait"
```

### .gitignore 글로벌 설정

```bash
cat > ~/.gitignore_global << 'EOF'
.env
.env.local
*.pyc
__pycache__/
.DS_Store
Thumbs.db
EOF

git config --global core.excludesfile ~/.gitignore_global
```

### .env 파일 구성 (프로젝트별)

프로젝트 루트에 `.env` 파일을 생성한다. 이 파일은 절대 Git에 커밋하지 않는다.

```bash
cat > .env << 'EOF'
# Anthropic
ANTHROPIC_API_KEY=your-api-key-here

# Ollama (Windows native Ollama 사용 시)
OLLAMA_HOST=http://localhost:11434

# Project
PROJECT_ENV=development
EOF
```

---

## .claude/ 폴더 구조 생성

프로젝트 루트에서 아래 명령을 실행하여 공식 .claude/ 구조를 일괄 생성한다.

```bash
# Create .claude/ directory structure
mkdir -p .claude/rules
mkdir -p .claude/skills/tdd
mkdir -p .claude/skills/refactoring
mkdir -p .claude/skills/code-reviewer
mkdir -p .claude/agents

# Create placeholder files
touch .claude/settings.json
touch .claude/settings.local.json
touch .claude/rules/conventions.md
touch .claude/rules/tdd.md
touch .claude/skills/tdd/SKILL.md
touch .claude/skills/refactoring/SKILL.md
touch .claude/skills/code-reviewer/SKILL.md
touch .claude/agents/code_reviewer.md
touch .claude/agents/tdd_agent.md
touch .claude/.mcp.json

echo ".claude/settings.local.json" >> .gitignore

echo "Directory structure created."
tree .claude/
```

### .claude/settings.json 초기 내용

```json
{
  "model": "claude-opus-4-5",
  "permissions": {
    "allow": [
      "Bash(git:*)",
      "Bash(pytest:*)",
      "Bash(cmake:*)",
      "Bash(make:*)",
      "Read(*)",
      "Write(src/*)",
      "Write(tests/*)"
    ],
    "deny": [
      "Bash(rm -rf /*)"
    ]
  }
}
```

### .claude/.mcp.json 초기 내용

```json
{
  "mcpServers": {}
}
```

---

## CLAUDE.md 초기 템플릿 (WSL 환경)

프로젝트 루트에 `CLAUDE.md`를 생성한다. 200줄 이내로 유지한다.

```bash
cat > CLAUDE.md << 'EOF'
# Project Context

## Environment
- OS: WSL2 (Ubuntu)
- Python: Anaconda (conda environments)
- GPU: NVIDIA GTX 1080 Ti (VRAM 10GB)
- Shell: bash

## Path Rules
- All paths use Linux format: ~/projects/my-project/
- Do NOT use Windows paths (/mnt/c/...)
- Project root is always WSL home directory

## Python Environment
- Use conda for environment management
- Activate environment before running: conda activate project-env
- Package manager: conda first, pip second

## Test Framework
- Python: pytest
- C++: Google Test via CMake
- Run tests: pytest tests/ or cd build && ctest

## Code Style
- Python: PEP 8, type hints required
- C++: Google Style Guide
- All code comments in English

## TDD Rules
- Write test first (Red)
- Implement minimum code to pass (Green)
- Refactor without breaking tests (Refactor)
- Commit on every test pass

## Git Rules
- Commit message format: feat/test/refactor/fix/docs/chore
- Never commit: .env, __pycache__, *.pyc, build/
- Branch naming: feature/task-name

## Skills Available
- .claude/skills/tdd/SKILL.md
- .claude/skills/refactoring/SKILL.md
- .claude/skills/code-reviewer/SKILL.md
EOF
```

---

## .wslconfig 권장 설정

Windows 측 사용자 홈 디렉토리(`C:\Users\username\`)에 `.wslconfig` 파일을 생성한다.

PowerShell에서 실행한다.

```powershell
notepad $env:USERPROFILE\.wslconfig
```

아래 내용을 입력한다.

```ini
[wsl2]
# Memory allocation (adjust based on system RAM)
memory=16GB

# CPU cores
processors=8

# Swap space
swap=8GB

# Enable localhost forwarding (for Ollama access from WSL)
localhostForwarding=true

# Optional: GPU support (requires WSL2 + CUDA support)
# nestedVirtualization=true
```

설정 적용을 위해 WSL을 재시작한다.

```powershell
wsl --shutdown
wsl
```

---

## 설치 확인 체크리스트

- [ ] `nvm --version` 출력 확인
- [ ] `node --version` v20 이상 확인
- [ ] `npm --version` 확인
- [ ] `claude --version` 출력 확인
- [ ] `claude auth status` 로그인 상태 확인
- [ ] WSL 터미널에서 `code .` 실행 시 VSCode Remote-WSL 연결 확인
- [ ] `gh auth status` 로그인 상태 확인
- [ ] `ssh -T git@github.com` 인증 성공 확인
- [ ] `.claude/` 폴더 구조 생성 확인 (`tree .claude/`)
- [ ] `CLAUDE.md` 파일 존재 확인
- [ ] `.env` 파일 존재 및 `.gitignore` 등록 확인

---

## 공통 오류 및 해결 방법

| 오류 | 원인 | 해결 방법 |
|---|---|---|
| `nvm: command not found` | PATH 미적용 | `source ~/.bashrc` 후 재시도 |
| `claude: command not found` | npm global PATH 미설정 | `export PATH="$HOME/.nvm/versions/node/$(node -v)/bin:$PATH"` |
| `claude login` 시 브라우저 미열림 | WSL 브라우저 미설정 | URL 복사 후 Windows 브라우저에서 열기 |
| `gh auth login` 실패 | GitHub CLI 버전 구버전 | 공식 저장소 추가 후 재설치 |
| `ssh -T git@github.com` Permission denied | SSH 키 미등록 | `gh ssh-key add` 또는 github.com/settings/keys 수동 등록 |
| `code .` 실행 시 VSCode 미열림 | Remote-WSL Extension 미설치 | VSCode에서 `ms-vscode-remote.remote-wsl` 설치 |
| conda 명령 없음 | Anaconda PATH 미설정 | `conda init bash` 후 `source ~/.bashrc` |
| WSL에서 GPU 인식 안됨 | NVIDIA 드라이버 미설정 | Windows 측 NVIDIA 드라이버 최신 버전으로 업데이트 |
| `.wslconfig` 설정 미적용 | WSL 재시작 필요 | PowerShell에서 `wsl --shutdown` 후 `wsl` 재실행 |
