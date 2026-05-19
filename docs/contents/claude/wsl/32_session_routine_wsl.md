# WSL 세션 루틴

## Overview

- **목적**: WSL2 (Ubuntu) 환경에서 프로젝트 최초 시작부터 세션 시작, 진행, 종료까지의 표준 절차
- **사전 조건**: `02_setup_wsl.md` 환경 설정 완료, Claude Code 인증 완료

---

## 프로젝트 최초 시작 (WSL)

프로젝트를 처음 만들 때 한 번만 실행한다.

### 폴더 생성

WSL 홈 디렉토리 아래에 프로젝트 폴더를 생성한다.
`/mnt/c/` 경로는 성능 저하가 발생하므로 사용하지 않는다.

```bash
cd ~
mkdir -p projects/my-project
cd projects/my-project
```

### GitHub 원격 저장소 생성

```bash
# Initialize local git
git init

# Create GitHub remote repository (private by default)
gh repo create my-project --private --source=. --remote=origin

# Verify remote
git remote -v
# Expected:
# origin  git@github.com:your-username/my-project.git (fetch)
# origin  git@github.com:your-username/my-project.git (push)
```

공개 저장소로 생성하려면 `--private` 대신 `--public`을 사용한다.

### .gitignore 생성

```bash
cat > .gitignore << 'EOF'
# Python
__pycache__/
*.pyc
*.pyo
*.pyd
.Python
*.egg-info/
dist/
build/
.eggs/

# Virtual environments
.venv/
venv/
env/

# Environment variables
.env
.env.local
.env.*.local

# IDE
.vscode/settings.json
.idea/

# Claude Code local settings
.claude/settings.local.json

# C++ build
build/
*.o
*.a
*.so
CMakeFiles/
cmake_install.cmake
Makefile
CTestTestfile.cmake

# OS
.DS_Store
Thumbs.db

# Jupyter
.ipynb_checkpoints/
*.ipynb

# Data / models
*.h5
*.pt
*.pth
data/
EOF
```

### .env 생성

```bash
cat > .env << 'EOF'
# Anthropic API
ANTHROPIC_API_KEY=your-api-key-here

# Ollama (Windows native Ollama 사용 시)
OLLAMA_HOST=http://localhost:11434

# Project settings
PROJECT_ENV=development
EOF
```

### .claude/ 구조 생성

```bash
mkdir -p .claude/rules
mkdir -p .claude/skills/tdd
mkdir -p .claude/skills/refactoring
mkdir -p .claude/skills/code-reviewer
mkdir -p .claude/agents

# Create settings
cat > .claude/settings.json << 'EOF'
{
  "model": "claude-opus-4-5",
  "permissions": {
    "allow": [
      "Bash(git:*)",
      "Bash(pytest:*)",
      "Bash(cmake:*)",
      "Bash(make:*)",
      "Bash(conda:*)",
      "Read(*)",
      "Write(src/*)",
      "Write(tests/*)"
    ],
    "deny": [
      "Bash(rm -rf /*)"
    ]
  }
}
EOF

cat > .claude/.mcp.json << 'EOF'
{
  "mcpServers": {}
}
EOF

touch .claude/rules/conventions.md
touch .claude/rules/tdd.md
touch .claude/skills/tdd/SKILL.md
touch .claude/skills/refactoring/SKILL.md
touch .claude/skills/code-reviewer/SKILL.md
touch .claude/agents/code_reviewer.md
touch .claude/agents/tdd_agent.md

echo ".claude/settings.local.json" >> .gitignore
```

### 기타 초기 파일 생성

```bash
mkdir -p src tests sessions docs

touch README.md
touch CLAUDE.md

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
- Conda environment name: project-env
- Activate: conda activate project-env

## Test Framework
- Python: pytest (pytest.ini in project root)
- C++: Google Test via CMake FetchContent
- Run Python tests: pytest tests/ -v
- Run C++ tests: cd build && ctest --output-on-failure

## Code Style
- Python: PEP 8, type hints required
- C++: Google Style Guide
- All code comments in English

## TDD Rules
- Write test first (Red)
- Minimum implementation to pass (Green)
- Refactor without breaking tests (Refactor)
- Commit on every test pass

## Git Commit Format
feat / test / refactor / fix / docs / chore
EOF
```

### 초기 커밋

```bash
git add -A
git commit -m "chore: initial project structure"
git push -u origin main
```

---

## 세션 시작 루틴 (WSL)

매 작업 세션 시작 시 실행한다.

### bash 명령 순서

```bash
# 1. Navigate to project
cd ~/projects/my-project

# 2. Sync latest code
git pull

# 3. Activate conda environment
conda activate project-env

# 4. Verify environment
python --version
pytest --version

# 5. Check current status
git status
git log --oneline -5
```

### context/ 파일 업데이트

세션을 시작하기 전에 진행 상태를 확인하고 오늘 작업 범위를 정리한다.

```bash
# Check previous session state
cat sessions/PROGRESS.md 2>/dev/null || echo "No previous progress"

# Check TODO
cat sessions/TODO.md 2>/dev/null || echo "No TODO file"
```

`sessions/TODO.md`와 `sessions/PROGRESS.md`가 없으면 생성한다.

```bash
mkdir -p sessions

cat > sessions/TODO.md << 'EOF'
# TODO

## In Progress
- [ ] task item 1

## Pending
- [ ] task item 2

## Done
EOF

cat > sessions/PROGRESS.md << 'EOF'
# Progress

## Last Session
- Date: YYYY-MM-DD
- Completed: 
- Remaining:

## Current Session
- Date: (today)
- Goal:
EOF
```

### task.md 생성

세션 번호와 태스크 이름으로 폴더를 만들고 task.md를 생성한다.

```bash
# Create session folder
SESSION_NUM="01"
TASK_NAME="implement-stat-calculator"
mkdir -p sessions/${SESSION_NUM}_${TASK_NAME}

# Create task.md from template (if template exists)
cp sessions/task_template.md sessions/${SESSION_NUM}_${TASK_NAME}/task.md 2>/dev/null || \
cat > sessions/${SESSION_NUM}_${TASK_NAME}/task.md << 'EOF'
# Task: Implement StatCalculator

## Goal
Implement statistical calculator class with NumPy.

## Requirements
- mean(data): return mean value
- std(data): return standard deviation
- normalize(data): min-max normalization to [0, 1]

## Acceptance Criteria
- All pytest tests pass
- Edge case handling: empty array, identical values
- Type hints on all methods

## Modification History
| Date | Change | Reason |
|---|---|---|
EOF
```

### VSCode Remote-WSL 실행

```bash
code .
```

VSCode 좌하단에 `WSL: Ubuntu` 배지가 표시되면 정상이다.

### Claude Code 실행

VSCode 통합 터미널에서 실행한다.

```bash
claude
```

또는 VSCode Extension을 사용한다: `Ctrl+Shift+P` > `Claude Code: Open`

---

## 세션 진행 (WSL)

### 초기 요청 명령 예시

세션 시작 시 Claude Code에게 컨텍스트를 전달한다.

```
Read sessions/01_implement-stat-calculator/task.md and
.claude/skills/tdd/SKILL.md, then proceed with the TDD cycle.
Environment: WSL2, Python 3.11, conda activate project-env, pytest.
Start with Red: write failing tests first.
```

특정 스킬과 에이전트를 함께 사용할 경우:

```
Read the following files:
- sessions/01_implement-stat-calculator/task.md (task definition)
- .claude/skills/tdd/SKILL.md (TDD procedure)
- .claude/rules/conventions.md (coding conventions)
Then implement using TDD cycle. Commit after each Green step.
```

### TDD 사이클 (WSL 환경)

**Red 단계**

```
Write failing tests for StatCalculator in tests/test_calculator.py.
Only write tests, not implementation.
Run: pytest tests/test_calculator.py
Confirm: tests fail (FAILED or ERROR)
```

테스트 실행 확인:

```bash
pytest tests/test_calculator.py
# Expected: FAILED or ImportError
```

**Green 단계**

```
Implement minimum code in src/calculator.py to make all tests pass.
Run: pytest tests/test_calculator.py
Confirm: all tests pass
Then commit: git add -A && git commit -m "feat: implement StatCalculator"
```

테스트 통과 확인:

```bash
pytest tests/test_calculator.py -v
# Expected: all PASSED
git add -A && git commit -m "feat: implement StatCalculator mean/std/normalize"
```

**Refactor 단계**

```
Refactor src/calculator.py:
- Add input validation (empty array, type check)
- Improve error messages
- Add docstrings
Run: pytest tests/test_calculator.py
Confirm: all tests still pass
Commit: git add -A && git commit -m "refactor: add validation to StatCalculator"
```

### 추가 수정 요청 처리

같은 세션 내에서 요구사항이 추가되면 task.md의 수정 이력에 기록한다.

```
New requirement: add median(data) and percentile(data, q) methods.
Update sessions/01_implement-stat-calculator/task.md modification history.
Then continue TDD cycle for these new methods.
```

task.md 수정 이력 예시:

```markdown
## Modification History
| Date | Change | Reason |
|---|---|---|
| 2025-05-19 | Added median, percentile methods | New requirement |
```

### git commit 명령

```bash
# After Red (optional - when writing tests only)
git add tests/
git commit -m "test: add tests for StatCalculator"

# After Green
git add -A
git commit -m "feat: implement StatCalculator mean/std/normalize"

# After Refactor
git add -A
git commit -m "refactor: add input validation to StatCalculator"

# Additional feature
git add -A
git commit -m "feat: add median and percentile to StatCalculator"
```

---

## 세션 종료 (WSL)

### 모든 테스트 통과 확인

```bash
pytest tests/ -v
# Expected: all PASSED

# C++ tests (if applicable)
cd build && ctest --output-on-failure && cd ..
```

### report.md 생성 요청

Claude Code에게 요청한다.

```
Generate a session report and save it to
sessions/01_implement-stat-calculator/report.md.
Include:
- Summary of what was implemented
- TDD cycle count (Red/Green/Refactor)
- Files created or modified
- Test coverage summary
- Remaining items (if any)
```

### /export 명령 사용

대화 로그를 파일로 내보낸다.

```
/export sessions/01_implement-stat-calculator/prompt.md
```

### context/ 파일 업데이트

```bash
# Update PROGRESS.md
cat >> sessions/PROGRESS.md << 'EOF'

## Session 2025-05-19
- Completed: StatCalculator (mean, std, normalize, median, percentile)
- Tests: 7 passed
- Commits: 3
EOF

# Update TODO.md (mark completed, add new items)
# Edit manually or ask Claude Code:
# "Update sessions/TODO.md: mark task-1 as done, add task-2 from report"

# Reset SESSION.md for next session
cat > sessions/SESSION.md << 'EOF'
# Current Session Handoff

## Last Session Completed
- Implemented StatCalculator
- All tests passing

## Next Session Goal
- Implement DataLoader class
- Target: sessions/02_implement-data-loader/

## Environment State
- conda env: project-env
- Last commit: refactor: add input validation to StatCalculator
EOF
```

### git push

```bash
git push origin main
```

---

## WSL 경로 주의사항

### 프로젝트 폴더 위치 권장

WSL 홈 디렉토리 아래에 프로젝트를 두는 것을 원칙으로 한다.

```
권장:   ~/projects/my-project/
비권장: /mnt/c/Users/username/projects/my-project/
```

| 위치 | 파일 I/O 속도 | Git 속도 | 비고 |
|---|---|---|---|
| `~/` (WSL 홈) | 빠름 | 빠름 | 권장 |
| `/mnt/c/` (Windows 드라이브) | 느림 (5-10배 차이) | 매우 느림 | 비권장 |

### /mnt/c/ 경로 사용 시 주의사항

불가피하게 `/mnt/c/` 경로를 사용해야 하는 경우:

```bash
# Git 성능 개선 설정
git config --global core.fsmonitor false
git config --global core.untrackedCache false

# pytest 실행 시 명시적 경로 지정
cd /mnt/c/Users/username/projects/my-project
PYTHONPATH=/mnt/c/Users/username/projects/my-project pytest tests/
```

CLAUDE.md에 경로 규칙을 명시한다.

```markdown
## Path Rules
- Project root: /mnt/c/Users/username/projects/my-project
- Use absolute paths when referencing files
- Do NOT use ~ shorthand for /mnt/c/ paths
```

### Claude Code에서 경로 지정

```
# WSL 홈 경로 사용 시 (권장)
Read ~/projects/my-project/sessions/01_task/task.md

# /mnt/c/ 사용 시 (비권장, 필요한 경우에만)
Read /mnt/c/Users/username/projects/my-project/sessions/01_task/task.md
```

---

## 세션 시작 / 종료 체크리스트

### 세션 시작 체크리스트

- [ ] `cd ~/projects/my-project` 프로젝트 폴더 이동
- [ ] `git pull` 최신 코드 동기화
- [ ] `conda activate project-env` 환경 활성화
- [ ] `git status` 미커밋 변경사항 없음 확인
- [ ] `cat sessions/PROGRESS.md` 이전 세션 상태 확인
- [ ] `sessions/{NUM}_{TASK}/task.md` 생성
- [ ] `code .` VSCode Remote-WSL 연결 확인 (WSL: Ubuntu 배지)
- [ ] `claude` Claude Code 실행 확인

### 세션 종료 체크리스트

- [ ] `pytest tests/ -v` 모든 테스트 통과 확인
- [ ] Claude Code에 `report.md` 생성 요청 및 내용 확인
- [ ] `/export sessions/{NUM}_{TASK}/prompt.md` 대화 로그 내보내기
- [ ] `sessions/PROGRESS.md` 이번 세션 내용 기록
- [ ] `sessions/TODO.md` 완료 항목 체크, 신규 항목 추가
- [ ] `sessions/SESSION.md` 다음 세션 인계 내용 작성
- [ ] `git add -A && git status` 누락된 파일 없는지 확인
- [ ] `git push origin main` 원격 저장소 동기화
- [ ] VSCode / Claude Code 종료

---

## 공통 오류 및 해결 방법

| 오류 | 원인 | 해결 방법 |
|---|---|---|
| `conda: command not found` | Anaconda PATH 미설정 | `conda init bash && source ~/.bashrc` |
| `claude: command not found` | npm PATH 미설정 | `source ~/.bashrc` 또는 `nvm use --lts` |
| `code .` 실행 시 VSCode 미열림 | Remote-WSL Extension 미설치 | VSCode에서 `ms-vscode-remote.remote-wsl` 설치 |
| `git pull` 충돌 | 원격과 로컬 분기 | `git fetch && git rebase origin/main` |
| `pytest` ImportError | PYTHONPATH 미설정 | `pip install -e .` 또는 `PYTHONPATH=$PWD pytest` |
| `/export` 명령 실패 | 경로 미존재 | `mkdir -p sessions/01_task/` 후 재시도 |
| 세션 종료 후 변경사항 유실 | `git push` 누락 | 체크리스트 항목 확인 |
