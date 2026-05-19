# 전체 통합 워크플로

## Overview

- 이 문서는 PRD 작성부터 코딩, TDD, 리팩터링, GitHub 푸시까지 전체 흐름을 다룬다.
- Claude Code, .claude/skills/, .claude/agents/, Ollama, MCP를 조합하는 방법을 포함한다.
- 프로젝트 시작부터 세션 종료까지의 완전한 예제를 포함한다.

**사전 조건:**
- Claude Code가 설치되어 있다
- GitHub CLI (gh)가 설치되어 있다
- `.claude/` 폴더 구조가 생성되어 있다
- MCP 서버가 설정되어 있다 (선택)

---

## 전체 흐름

```
PRD 작성
  └── task.md 작성 (이번 세션 작업 정의)
        └── TDD 사이클
              └── Red (테스트 작성 → 실패 확인)
              └── Green (최소 구현 → 통과 확인)
              └── Refactor (코드 개선 → 통과 유지)
                    └── git commit (통과마다)
                          └── 코드 리뷰 (agents/code_reviewer)
                                └── report.md 생성
                                      └── git push
```

---

## Claude Code + .claude/skills/ + .claude/agents/ 조합

**skills/ 사용 패턴:**

세션 시작 시 필요한 skill을 명시적으로 지정한다.

```
# TDD + 리팩터링 순서로 진행
"sessions/01_calculator/task.md를 읽어라.
 먼저 .claude/skills/tdd/SKILL.md 역할로 TDD를 완료해라.
 TDD 완료 후 .claude/skills/refactoring/SKILL.md 역할로
 리팩터링을 진행해라."
```

**agents/ 사용 패턴:**

독립 작업이 필요할 때 에이전트를 명시적으로 호출한다.

```
# TDD 완료 후 코드 리뷰
/agent code_reviewer -- src/ 전체를 리뷰하고 결과를 보고해라
```

**skills + agents 순서:**
```
1. skills/tdd       TDD 사이클 실행
2. skills/refactoring  리팩터링 실행
3. agents/code_reviewer  독립 컨텍스트에서 코드 리뷰
4. 리뷰 결과 반영 후 최종 커밋
```

---

## Claude Code + Ollama 조합

**역할 분담:**

| 작업 | 도구 |
|---|---|
| TDD 사이클, 파일 수정, 리팩터링 | Claude Code |
| 초안 작성, 아이디어 탐색 | Ollama |
| 민감 데이터 처리 | Ollama |
| 반복적 단순 생성 | Ollama |

**Python에서 도구 전환 예시:**

```python
import os
import requests


def ask_ai(prompt: str, use_local: bool = False) -> str:
    if use_local:
        return ask_ollama(prompt)
    return ask_claude(prompt)


def ask_ollama(prompt: str) -> str:
    url = os.environ.get("OLLAMA_URL", "http://localhost:11434")
    response = requests.post(
        f"{url}/api/generate",
        json={"model": "qwen2.5-coder:14b", "prompt": prompt, "stream": False}
    )
    return response.json()["response"]


def ask_claude(prompt: str) -> str:
    # Claude Code agent handles this directly
    # This is for programmatic access via API
    import anthropic
    client = anthropic.Anthropic(api_key=os.environ["ANTHROPIC_API_KEY"])
    message = client.messages.create(
        model="claude-sonnet-4-20250514",
        max_tokens=1024,
        messages=[{"role": "user", "content": prompt}]
    )
    return message.content[0].text
```

**.env 설정:**
```bash
ANTHROPIC_API_KEY=sk-ant-xxxxxxxxxxxx
OLLAMA_URL=http://localhost:11434
```

**Ollama 전환 기준:**
```
# Ollama로 전환하는 경우
"이 텍스트는 내부 데이터를 포함하므로 Ollama로 처리해라.
 OLLAMA_URL은 .env를 참조해라."
```

---

## Claude Code + MCP 조합

**MCP 조합 패턴:**

```
# Context7 + TDD
"Context7로 pytest 최신 fixture 문법을 확인하고
 .claude/skills/tdd/SKILL.md 역할로 TDD를 시작해라."

# Sequential Thinking + 버그 분석
"Sequential Thinking으로 이 버그를 단계적으로 분석해라.
 분석 완료 후 수정 계획을 보여라. 수정은 하지 마라."

# GitHub MCP + 세션 종료
"GitHub MCP로 현재 변경사항을 기반으로 PR을 생성해라.
 PR 제목과 설명은 sessions/01_calculator/report.md를 참조해라."
```

**MCP + skills + agents 전체 조합:**
```
1. Context7로 라이브러리 API 확인
2. skills/tdd로 TDD 사이클 실행
3. Sequential Thinking으로 복잡한 로직 설계
4. agents/code_reviewer로 코드 리뷰
5. GitHub MCP로 PR 생성
```

---

## 완전한 예제: 프로젝트 시작부터 세션 종료까지

**목표:** Python으로 간단한 숫자 통계 모듈(`stats.py`) TDD 구현

### 프로젝트 최초 시작

**1. 폴더 생성 및 Git 초기화 (Windows PowerShell):**
```powershell
mkdir stats-project
cd stats-project
git init
```

**2. GitHub 원격 저장소 생성:**
```powershell
gh repo create stats-project --private --source=. --remote=origin
```

**3. 기본 파일 생성:**
```powershell
# .gitignore
@"
.env
CLAUDE.local.md
__pycache__/
.pytest_cache/
*.pyc
"@ | Out-File -Encoding utf8 .gitignore

# .env
@"
ANTHROPIC_API_KEY=sk-ant-xxxxxxxxxxxx
GITHUB_TOKEN=ghp_xxxxxxxxxxxx
"@ | Out-File -Encoding utf8 .env
```

**4. CLAUDE.md 작성:**
```markdown
# stats-project

## Overview
Python statistics module with TDD.

## Stack
- Language: Python 3.11 (WinPython)
- Test: pytest

## Directory
src/      # source code
tests/    # test code
sessions/ # per-session files

## Commands
pytest tests/
pytest tests/test_stats.py -v

## Rules
- Test file: tests/test_{module}.py
- Function naming: snake_case
- No print() in src/
```

**5. .claude/ 구조 생성:**
```powershell
mkdir .claude, .claude\rules, .claude\skills, .claude\skills\tdd, .claude\agents

# rules/tdd.md
@"
# TDD Rules
Always follow Red -> Green -> Refactor.
Never write implementation before test.
Commit on every Green and Refactor pass.
"@ | Out-File -Encoding utf8 .claude\rules\tdd.md
```

**6. 초기 커밋:**
```powershell
git add .gitignore CLAUDE.md .claude/
git commit -m "chore: initial project setup"
git push -u origin main
```

---

### 세션 시작

**1. 최신 코드 동기화:**
```powershell
cd stats-project
git pull
```

**2. task.md 작성:**
```powershell
mkdir sessions\01_stats-mean
@"
# Task: Implement stats.mean

## Goal
Implement mean(numbers) function with TDD.

## Requirements
- Input: list of numbers
- Output: float (arithmetic mean)
- Raise ValueError on empty list
- Raise TypeError on non-numeric input

## Files
- src/stats.py
- tests/test_stats.py
"@ | Out-File -Encoding utf8 sessions\01_stats-mean\task.md
```

**3. VSCode 및 Claude Code 실행:**
```powershell
code .
claude
```

---

### 세션 진행

**초기 요청:**
```
sessions/01_stats-mean/task.md를 읽어라.
.claude/skills/tdd/SKILL.md 역할로 TDD를 시작해라.
Red 단계부터 시작하고 각 단계마다 결과를 보여라.
```

**Red 단계 — Claude Code 작업:**
```python
# tests/test_stats.py (Claude Code 작성)
import pytest
from src.stats import mean


def test_mean_of_positive_numbers():
    # Arrange
    numbers = [1, 2, 3, 4, 5]
    # Act
    result = mean(numbers)
    # Assert
    assert result == 3.0


def test_mean_raises_on_empty_list():
    with pytest.raises(ValueError):
        mean([])


def test_mean_raises_on_non_numeric():
    with pytest.raises(TypeError):
        mean([1, "a", 3])
```

```
# Red 단계 확인
pytest tests/test_stats.py
-> FAILED (ImportError: cannot import name 'mean')
```

**커밋:**
```powershell
git add tests/test_stats.py
git commit -m "test: add tests for stats.mean"
```

**Green 단계 — Claude Code 작업:**
```python
# src/stats.py (Claude Code 작성)
def mean(numbers: list) -> float:
    if len(numbers) == 0:
        raise ValueError("Cannot compute mean of empty list")
    for n in numbers:
        if not isinstance(n, (int, float)):
            raise TypeError(f"Expected numeric, got {type(n)}")
    return sum(numbers) / len(numbers)
```

```
# Green 단계 확인
pytest tests/test_stats.py
-> 3 passed in 0.08s
```

**커밋:**
```powershell
git add src/stats.py
git commit -m "feat: implement stats.mean"
```

**Refactor 단계 — Claude Code 작업:**
```
"코드를 리뷰하고 개선할 부분을 제안해라. 수정하지 마라."
(제안 확인)
"제안대로 리팩터링해라."
```

**코드 리뷰 — agents 호출:**
```
/agent code_reviewer -- src/stats.py를 리뷰해라
```

---

### 세션 종료

**1. 전체 테스트 통과 확인:**
```powershell
pytest tests/ -v
```

**2. report.md 생성 요청:**
```
"sessions/01_stats-mean/report.md를 작성해라.
 완료 항목, 테스트 결과, 다음 세션 작업을 포함해라."
```

**3. 대화 로그 내보내기:**
```
/export sessions/01_stats-mean/prompt.md
```

**4. 최종 커밋 및 푸시:**
```powershell
git add sessions/01_stats-mean/
git commit -m "docs: add session 01 report"
git push
```

---

## Verification Checklist

- 세션 시작 전 task.md가 작성되어 있다
- Red 단계에서 테스트가 실제로 실패했다
- Green 단계 후 커밋이 완료됐다
- 세션 종료 전 report.md가 생성됐다
- git push가 완료됐다
- `.env`가 커밋에 포함되지 않았다

---

## Common Errors and Solutions

| 오류 | 원인 | 해결 |
|---|---|---|
| Claude Code가 task.md 내용을 무시한다 | 초기 요청에 파일 읽기 미포함 | "task.md를 읽어라"를 요청 첫 줄에 명시 |
| 세션 중간에 컨텍스트가 사라진다 | 컨텍스트 창 초과 | 새 세션 시작, report.md로 인계 |
| Ollama가 응답하지 않는다 | Ollama 서비스 미실행 | Windows에서 Ollama 실행 후 재시도 |
| PR 생성이 실패한다 | GitHub MCP 인증 오류 | `.env`의 GITHUB_TOKEN 확인 |
| git push가 거부된다 | 원격 저장소와 이력 충돌 | `git pull --rebase` 후 재시도 |
