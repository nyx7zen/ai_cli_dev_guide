# 프롬프트 전략

## Overview

- 이 문서는 Claude Code와 효과적으로 작업하기 위한 프롬프트 작성 전략을 다룬다.
- CLAUDE.md, rules, skills, agents 각 파일의 작성 방법과 예시를 포함한다.
- 환각(Hallucination) 방지 전략과 반복 개선 패턴을 함께 다룬다.

**사전 조건:**
- Claude Code가 설치되어 있다
- `.claude/` 폴더 구조가 생성되어 있다

---

## CLEAR 프레임워크

Claude Code에 요청할 때 응답 품질을 높이는 프롬프트 구성 원칙이다.

| 항목 | 의미 | 예시 |
|---|---|---|
| **C**ontext | 현재 상황과 배경 | "NumPy 배열을 처리하는 함수가 있고..." |
| **L**imit | 범위와 제약 조건 | "pytest만 사용, 외부 라이브러리 금지" |
| **E**xample | 입출력 예시 | "입력: [1,2,3] 출력: 6" |
| **A**ction | 요청할 구체적 행동 | "테스트 코드를 먼저 작성해라" |
| **R**ole | 역할 또는 관점 | ".claude/skills/tdd/SKILL.md 역할로" |

모든 항목을 항상 포함할 필요는 없다. 작업의 복잡도에 따라 필요한 항목만 사용한다.

---

## P-C-T-F 패턴

세션 시작 시 Claude Code에게 전달하는 초기 요청의 구조다.

```
[P] Persona  — 어떤 역할로 작업할지
[C] Context  — 현재 프로젝트 상태와 작업 배경
[T] Task     — 이번 세션에서 할 일
[F] Format   — 출력 형식과 제약 조건
```

**실제 예시 — Python TDD 세션 시작:**

```
[P] .claude/skills/tdd/SKILL.md의 TDD 절차를 따른다.

[C] Python 프로젝트 (WinPython / pytest).
    현재 src/calculator.py 파일이 비어 있다.
    sessions/01_calculator/task.md를 읽어라.

[T] add(a, b) 함수를 TDD로 구현한다.
    Red -> Green -> Refactor 순서를 반드시 지킨다.
    각 단계마다 테스트 실행 결과를 보여라.

[F] 테스트 파일: tests/test_calculator.py
    구현 파일: src/calculator.py
    커밋 메시지: conventional commits 형식
```

**실제 예시 — C++ TDD 세션 시작:**

```
[P] .claude/skills/tdd/SKILL.md의 TDD 절차를 따른다.

[C] C++ 프로젝트 (MinGW64 / CMake / Google Test).
    CMakeLists.txt가 설정되어 있다.
    sessions/02_sorting/task.md를 읽어라.

[T] bubble_sort 함수를 TDD로 구현한다.
    Given-When-Then 구조로 테스트를 작성한다.

[F] 테스트 파일: tests/test_sorting.cpp
    구현 파일: src/sorting.cpp
    빌드 명령: cmake --build build
```

---

## CLAUDE.md 작성 원칙

Claude Code가 매 세션 시작 시 자동으로 읽는 파일이다. 200줄 이하로 유지한다.

**포함할 내용:**
- 프로젝트 개요 (3줄 이내)
- 기술 스택
- 디렉터리 구조 (핵심만)
- 자주 사용하는 명령어
- 코딩 규칙 (핵심만)

**포함하지 않을 내용:**
- API 키, 비밀번호 (`.env`에 관리)
- 개인 로컬 경로 (`CLAUDE.local.md`에 관리)
- 상세한 절차 (`.claude/skills/`에 분산)
- 항상 적용할 규칙 (`.claude/rules/`에 분산)

**CLAUDE.md 템플릿:**

```markdown
# Project Name

## Overview
한 줄 프로젝트 설명.

## Stack
- Language: Python 3.11 (WinPython)
- Test: pytest
- Build: (해당 없음)

## Directory
src/        # source code
tests/      # test code
sessions/   # per-session files

## Commands
# run tests
pytest tests/

# run single test
pytest tests/test_calculator.py::test_add

## Rules
- Test file naming: test_{module}.py
- Function naming: snake_case
- No print() in production code
```

**길이 관리 원칙:**
CLAUDE.md가 50줄을 넘기 시작하면 내용을 `.claude/rules/` 또는 `.claude/skills/`로 이동하는 것을 검토한다.

---

## .claude/rules/ 작성 방법

매 세션 자동으로 로드되는 규칙 파일이다. 항상 적용해야 하는 제약 조건만 넣는다.

**파일 구조:**
```
.claude/rules/
├── conventions.md   # 명명 규칙, 파일 구조 규칙
└── tdd.md           # TDD 적용 규칙
```

**conventions.md 예시:**

```markdown
# Code Conventions

## Naming
- Functions: snake_case
- Classes: PascalCase
- Constants: UPPER_SNAKE_CASE
- Test functions: test_{action}_{condition}

## Forbidden
- No print() in src/
- No hardcoded file paths
- No commented-out code in commits

## File Structure
- One class per file
- Test file mirrors source: src/foo.py -> tests/test_foo.py
```

**tdd.md 예시:**

```markdown
# TDD Rules

## Cycle
Always follow Red -> Green -> Refactor order.
Never write implementation before test.

## Commit Timing
Commit on every Green (test pass).
Commit message format: test: / feat: / refactor:

## Test Structure
Use AAA pattern: Arrange / Act / Assert.
One assertion per test function.
```

---

## .claude/skills/ 작성 방법

필요할 때만 참조하는 절차적 지식 파일이다. 각 스킬은 독립 폴더에 `SKILL.md`로 작성한다.

**파일 구조:**
```
.claude/skills/
├── tdd/
│   └── SKILL.md
├── refactoring/
│   └── SKILL.md
└── code-reviewer/
    └── SKILL.md
```

**skills/tdd/SKILL.md 예시:**

```markdown
# TDD Skill

## Role
Execute TDD cycle strictly. Do not skip steps.

## Red Phase
1. Read task.md to understand requirements
2. Write test function using AAA pattern
3. Run test and confirm FAIL
4. Show test output to user

## Green Phase
1. Write minimum implementation to pass the test
2. Run test and confirm PASS
3. Show test output to user
4. Commit: "feat: {description}"

## Refactor Phase
1. Identify duplication or unclear naming
2. Refactor without changing behavior
3. Run test and confirm PASS
4. Commit: "refactor: {description}"

## Rules
- Never write implementation before test
- Minimum implementation only in Green phase
- One cycle at a time, wait for user confirmation
```

**skills/refactoring/SKILL.md 예시:**

```markdown
# Refactoring Skill

## Role
Improve code structure without changing behavior.

## Steps
1. Run tests and confirm all PASS before starting
2. Identify target: duplication / long function / unclear name
3. Apply one refactoring at a time
4. Run tests after each change
5. Commit on each passing refactor

## Forbidden
- Do not change function signatures without explicit request
- Do not add new features during refactoring
```

---

## .claude/agents/ 작성 방법

독립된 컨텍스트에서 실행되는 전담 에이전트 파일이다. 메인 대화와 격리되어 실행된다.

**파일 구조:**
```
.claude/agents/
├── code_reviewer.md
└── tdd_agent.md
```

**agents/code_reviewer.md 예시:**

```markdown
# Code Reviewer Agent

## Role
Review code quality, test coverage, and convention compliance.
Operate independently from the main conversation.

## Review Checklist
- [ ] Test coverage: every public function has a test
- [ ] Naming: follows conventions.md rules
- [ ] No forbidden patterns (print, hardcoded paths)
- [ ] No commented-out code
- [ ] Functions under 20 lines

## Output Format
Report findings as:
PASS   {file}:{line} - {description}
WARN   {file}:{line} - {description}
FAIL   {file}:{line} - {description}

## Scope
Review files under src/ and tests/ only.
Do not modify files. Report only.
```

**에이전트 호출 방법:**

```
/agent code_reviewer -- src/ 전체를 리뷰해라
```

---

## 환각(Hallucination) 방지 전략

Claude Code가 존재하지 않는 함수, 라이브러리, 경로를 생성하는 것을 방지하는 방법이다.

**1. 파일 명시적 참조:**
```
# 나쁜 예
"calculator 모듈을 수정해라"

# 좋은 예
"src/calculator.py를 읽고 add 함수를 수정해라"
```

**2. 라이브러리 버전 고정:**
```
# CLAUDE.md 또는 rules/conventions.md에 명시
## Dependencies
- numpy==1.26.0
- pytest==7.4.0
```

**3. 단계적 확인 요청:**
```
"테스트를 실행하기 전에 작성한 코드를 먼저 보여라"
"구현 전에 테스트 코드만 먼저 작성하고 내용을 확인해라"
```

**4. Context7 MCP 활용:**
라이브러리 최신 API를 확인할 때 Context7 MCP를 사용하면 구버전 API 사용을 방지할 수 있다.
```
"Context7로 pytest 최신 fixture 사용법을 확인하고 테스트를 작성해라"
```

**5. 범위 제한:**
```
"src/ 폴더만 수정해라. tests/ 는 건드리지 마라"
"이 함수만 수정해라. 다른 파일은 변경 금지"
```

---

## 반복 개선 프롬프트 패턴

한 번의 요청으로 완성하려 하지 않고 단계적으로 개선하는 패턴이다.

**패턴 1 — 확인 후 진행:**
```
1. "task.md를 읽고 작업 계획을 먼저 보여라. 실행하지 마라."
2. (계획 확인)
3. "계획대로 Red 단계만 진행해라."
4. (결과 확인)
5. "Green 단계를 진행해라."
```

**패턴 2 — 점진적 범위 확장:**
```
1. "add 함수 테스트만 작성해라"
2. (확인)
3. "add 함수 구현을 작성해라"
4. (확인)
5. "subtract 함수도 같은 방식으로 진행해라"
```

**패턴 3 — 실패 시 재시도:**
```
# 테스트가 실패한 경우
"테스트 실패 메시지를 분석하고 원인만 설명해라. 수정하지 마라."
(원인 확인)
"원인에 따라 {파일명}만 수정해라"
```

**패턴 4 — 세션 인계:**
```
# 새 세션 시작 시
"sessions/01_calculator/report.md와 CLAUDE.md를 읽고
 현재 상태를 요약해라. 작업은 시작하지 마라."
```

---

## Verification Checklist

- CLAUDE.md가 200줄 이하인지 확인한다
- `.env` 파일이 CLAUDE.md에 포함되어 있지 않은지 확인한다
- rules/에 절차적 내용이 들어가 있지 않은지 확인한다 (규칙만 포함)
- skills/에 항상 적용할 내용이 들어가 있지 않은지 확인한다 (절차만 포함)
- 세션 시작 요청에 P-C-T-F 구조가 포함되어 있는지 확인한다

---

## Common Errors and Solutions

| 오류 | 원인 | 해결 |
|---|---|---|
| Claude Code가 존재하지 않는 함수를 사용한다 | 라이브러리 버전 미명시 | `rules/conventions.md`에 의존성 버전 고정 |
| 요청과 다른 파일을 수정한다 | 범위 미명시 | 수정할 파일 경로를 명시적으로 지정 |
| TDD 순서를 지키지 않는다 | rules/tdd.md 미작성 | `rules/tdd.md`에 순서 명시, 세션 시작 시 role 지정 |
| CLAUDE.md가 너무 길어진다 | 모든 내용을 한 파일에 집중 | rules/ skills/ 로 분산 |
| 이전 세션 내용을 모른다 | 세션 인계 누락 | 새 세션 시작 시 report.md 읽기 요청 |
