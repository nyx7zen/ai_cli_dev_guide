# Windows 세션 루틴

## Overview

- 목적: Windows PowerShell 환경에서 Claude Code를 사용한 프로젝트 최초 시작부터 세션 시작/진행/종료까지의 루틴
- 사전 조건: `02_setup_win.md` 완료 (Claude Code, GitHub CLI, VSCode 설치 및 인증)

---

## 프로젝트 최초 시작

프로젝트당 한 번만 실행한다.

### 폴더 생성

```powershell
# Create project root
mkdir C:\projects\my-project
cd C:\projects\my-project

# Create standard directories
mkdir src, tests, docs, sessions, context
```

### GitHub 원격 저장소 생성

```powershell
# Initialize local Git
git init
git branch -M main

# Create remote repository (private by default)
gh repo create my-project --private --source=. --remote=origin

# Verify remote connection
git remote -v
```

예상 출력:

```
origin  https://github.com/<username>/my-project.git (fetch)
origin  https://github.com/<username>/my-project.git (push)
```

### .claude/ 구조 생성

```powershell
# Create .claude/ structure
New-Item -ItemType Directory -Path ".claude\rules"                  -Force
New-Item -ItemType Directory -Path ".claude\skills\tdd"             -Force
New-Item -ItemType Directory -Path ".claude\skills\refactoring"     -Force
New-Item -ItemType Directory -Path ".claude\skills\code-reviewer"   -Force
New-Item -ItemType Directory -Path ".claude\agents"                 -Force

New-Item -Path ".claude\settings.json"                -ItemType File -Force
New-Item -Path ".claude\settings.local.json"          -ItemType File -Force
New-Item -Path ".claude\.mcp.json"                    -ItemType File -Force
New-Item -Path ".claude\rules\conventions.md"         -ItemType File -Force
New-Item -Path ".claude\rules\tdd.md"                 -ItemType File -Force
New-Item -Path ".claude\skills\tdd\SKILL.md"          -ItemType File -Force
New-Item -Path ".claude\skills\refactoring\SKILL.md"  -ItemType File -Force
New-Item -Path ".claude\skills\code-reviewer\SKILL.md" -ItemType File -Force
New-Item -Path ".claude\agents\code_reviewer.md"      -ItemType File -Force
New-Item -Path ".claude\agents\tdd_agent.md"          -ItemType File -Force
```

### 베이스 파일 생성

```powershell
# Create project files
New-Item -Path "CLAUDE.md"       -ItemType File -Force
New-Item -Path "CLAUDE.local.md" -ItemType File -Force
New-Item -Path "README.md"       -ItemType File -Force
New-Item -Path ".gitignore"      -ItemType File -Force
New-Item -Path ".env"            -ItemType File -Force

# Create context tracking files
New-Item -Path "context\SESSION.md"   -ItemType File -Force
New-Item -Path "context\TODO.md"      -ItemType File -Force
New-Item -Path "context\PROGRESS.md"  -ItemType File -Force
New-Item -Path "context\DECISIONS.md" -ItemType File -Force

# Create session template
New-Item -Path "sessions\.gitkeep" -ItemType File -Force
```

### 초기 커밋

```powershell
git add .
git commit -m "chore: initial project structure"
git push -u origin main
```

---

## 세션 시작 루틴

매 작업 세션 시작 시 실행한다.

### PowerShell 명령 순서

```powershell
# 1. Navigate to project root
cd C:\projects\my-project

# 2. Sync latest code
git pull

# 3. Check current status
git status
git log --oneline -5
```

### context/ 파일 확인

```powershell
# Review previous session state and today's scope
code context\PROGRESS.md
code context\TODO.md
code context\SESSION.md
```

`SESSION.md`를 오늘의 작업 범위로 업데이트한다:

```markdown
# SESSION.md

## Date: 2025-01-15

## Scope
- Implement moving_average function in src/stats.py
- TDD cycle: Red -> Green -> Refactor

## Dependencies
- Requires: compute_zscore (completed in session 01)
- Blocks: visualization module (planned for session 03)

## Notes
- Continue from sessions/01_compute-zscore/report.md
```

### task.md 생성

```powershell
# Create session folder (format: NN_task-name)
$session = "02_moving-average"
mkdir "sessions\$session"

# Copy task template from previous session or create new
Copy-Item "sessions\01_compute-zscore\task.md" "sessions\$session\task.md"

# Open task.md for editing
code "sessions\$session\task.md"
```

`task.md` 내용 예시:

```markdown
# Task: moving_average

## Session: 02
## Date: 2025-01-15

## Goal
Implement moving_average(data, window) in src/stats.py
using TDD cycle.

## Acceptance Criteria
- [ ] Returns numpy array of same length as input
- [ ] First (window-1) values are NaN
- [ ] Handles window > len(data) with ValueError

## Constraints
- numpy only, no pandas
- Follow AAA pattern in tests

## Modification History
(none yet)
```

### VSCode Extension 실행 및 Claude Code 시작

```powershell
# Open VSCode in project root
code .
```

VSCode에서:
- 좌측 사이드바의 Claude Code 아이콘 클릭 (Extension 패널)
- 또는 통합 터미널(`Ctrl+\`\``)에서 `claude` 실행

```powershell
# Terminal-based launch
claude
```

---

## 세션 진행

### 초기 요청 명령 예시

세션 시작 시 Claude Code에 보내는 첫 번째 요청:

```
Read sessions/02_moving-average/task.md and
proceed with .claude/skills/tdd/ role.
Environment: Windows, WinPython, pytest.
Start with Red phase.
```

파일 직접 지정이 필요한 경우:

```
Read the following files for context:
- CLAUDE.md
- sessions/02_moving-average/task.md
- .claude/skills/tdd/SKILL.md

Then write a failing test for moving_average(data, window) in tests/test_stats.py.
Red phase only — do not implement yet.
```

### TDD 사이클 (Windows 환경)

#### Red

```powershell
pytest tests/test_stats.py -v
```

테스트 실패 확인. Claude Code에 요청:

```
Test is failing as expected (Red phase complete).
Error: <paste error output>
Now write minimal implementation for moving_average — Green phase.
```

#### Green

```powershell
pytest tests/test_stats.py -v
```

모든 테스트 통과 확인.

```powershell
git add .
git commit -m "feat: add moving_average to stats module"
```

#### Refactor

```powershell
# After refactoring, confirm tests still pass
pytest tests/test_stats.py -v

git add .
git commit -m "refactor: use numpy.convolve in moving_average"
```

### 추가 수정 요청 처리

같은 세션에서 추가 요청이 생긴 경우 task.md에 이력을 추가한다:

```
Add the following to task.md modification history:
  - Added edge case test for window=1
  - Modified moving_average to accept float arrays
Then proceed with Red phase for window=1 edge case.
```

task.md 수정 이력 형식:

```markdown
## Modification History

### 2025-01-15 14:30
- Added test for window=1 edge case
- Modified moving_average to accept float input
```

### git commit 명령

```powershell
# Standard commit formats
git add .
git commit -m "test: add edge case for window=1 in moving_average"
git commit -m "feat: support float arrays in moving_average"
git commit -m "fix: handle window larger than data length"
git commit -m "refactor: extract validation logic to _validate_window"
```

---

## 세션 종료

### report.md 생성 요청

```
Generate a session report and save it to sessions/02_moving-average/report.md.
Include:
- Summary of completed work
- Tests added (list with test names)
- Functions implemented
- Key decisions made
- Pending items for next session
```

### /export 명령 사용법

Claude Code 대화 로그를 파일로 내보낸다:

```
/export sessions/02_moving-average/prompt.md
```

내보내기 완료 확인:

```powershell
Test-Path "sessions\02_moving-average\prompt.md"
```

### context/ 파일 업데이트

```powershell
code context\PROGRESS.md
```

`PROGRESS.md` 업데이트 예시:

```markdown
## Session 02 — 2025-01-15

### Completed
- moving_average(data, window) implemented in src/stats.py
- 5 tests added in tests/test_stats.py
- All tests passing

### Decisions
- Use numpy.convolve instead of manual loop (performance)
- First (window-1) values set to NaN, not zero

### Next Session
- Session 03: visualization module
- Depends on: compute_zscore, moving_average (both done)
```

```powershell
code context\TODO.md
```

완료된 항목 체크, 새 항목 추가.

```powershell
code context\SESSION.md
```

다음 세션을 위한 핸드오프 메모로 내용 교체:

```markdown
# SESSION.md

## Handoff for Next Session

### Last completed: Session 02 (moving_average)
### Next: Session 03 (visualization module)

### State
- src/stats.py: compute_zscore, moving_average complete
- tests/test_stats.py: 7 tests, all passing
- branch: main, last commit: refactor: extract validation logic

### Start command for next session
Read sessions/03_visualization/task.md and proceed with .claude/skills/tdd/ role.
```

### git push

```powershell
# Final test run before push
pytest

# Push all commits from this session
git push origin main
```

---

## 세션 시작 / 종료 체크리스트

### 세션 시작

- [ ] `cd C:\projects\my-project` 이동
- [ ] `git pull` 최신 코드 동기화
- [ ] `context\PROGRESS.md` 이전 세션 상태 확인
- [ ] `context\TODO.md` 우선순위 확인
- [ ] `context\SESSION.md` 오늘 작업 범위 업데이트
- [ ] `sessions\NN_task-name\` 폴더 생성
- [ ] `task.md` 작성 (목표, 수락 조건, 제약)
- [ ] `code .` VSCode 실행
- [ ] Claude Code 시작 (Extension 패널 또는 터미널 `claude`)
- [ ] 초기 요청 전송 (task.md + SKILL.md 참조)

### 세션 종료

- [ ] 모든 테스트 통과 확인 (`pytest` 또는 `ctest`)
- [ ] `report.md` 생성 요청
- [ ] `/export sessions/NN_task-name/prompt.md` 실행
- [ ] `context\PROGRESS.md` 이번 세션 작업 기록
- [ ] `context\TODO.md` 완료 항목 체크 / 새 항목 추가
- [ ] `context\SESSION.md` 다음 세션 핸드오프 메모 작성
- [ ] `context\DECISIONS.md` 주요 결정 사항 기록 (해당 시)
- [ ] `git push origin main`

---

## 공통 오류 및 해결

| 오류 | 원인 | 해결 |
|---|---|---|
| `git pull` 충돌 | 로컬 변경사항이 있는 상태에서 pull | `git stash` 후 `git pull` 후 `git stash pop` |
| `claude` 명령 응답 없음 | 인증 만료 | `claude` 재실행 시 재인증 프롬프트 확인 |
| `/export` 파일 미생성 | sessions/ 폴더 미존재 | `mkdir sessions\NN_task-name` 후 재시도 |
| `pytest` 기존 통과 테스트 실패 | 새 코드가 기존 코드에 영향 | Claude Code에 `pytest output: <오류>` 붙여넣어 원인 분석 요청 |
| VSCode Claude Code Extension 패널 미표시 | Extension 미설치 또는 비활성화 | `Ctrl+Shift+X` -> "Claude Code" 검색 -> 설치 |
| `git push` rejected | 원격에 로컬에 없는 커밋 존재 | `git pull --rebase` 후 `git push` |
| task.md 경로 오류 (`/` vs `\`) | Windows 경로 혼용 | PowerShell에서는 `\` 사용, Claude Code 요청문 내에서는 `/` 사용 가능 |
