# TDD 워크플로

## Overview

- 이 문서는 Claude Code와 TDD 사이클을 통합하는 방법을 다룬다.
- C++ (Google Test), Python (pytest), JAX (pytest) 각 언어별 테스트 프레임워크 사용법을 포함한다.
- 커밋 타이밍 전략과 세션 분리 기준도 함께 다룬다.

**사전 조건:**
- Claude Code가 설치되어 있다
- `.claude/rules/tdd.md`와 `.claude/skills/tdd/SKILL.md`가 작성되어 있다
- 언어별 테스트 프레임워크가 설치되어 있다

---

## TDD 개념 (Red / Green / Refactor)

```
Red      테스트 먼저 작성 → 실행 → 실패 확인
Green    최소 구현 작성 → 실행 → 통과 확인
Refactor 코드 개선 → 실행 → 통과 유지 확인
```

**핵심 원칙:**
- 테스트 없이 구현 코드를 먼저 작성하지 않는다
- Green 단계에서는 테스트를 통과하는 최소한의 코드만 작성한다
- Refactor 단계에서는 동작을 변경하지 않는다
- 각 단계는 테스트 실행으로 완료를 확인한다

---

## AI CLI와 TDD 사이클 통합

Claude Code는 TDD 사이클의 각 단계를 직접 실행한다. 사람은 각 단계의 결과를 확인하고 다음 단계를 승인한다.

**세션 시작 요청:**
```
sessions/01_task-name/task.md를 읽고
.claude/skills/tdd/SKILL.md 역할로 TDD를 시작해라.
Red 단계부터 시작하고 각 단계마다 결과를 보여라.
```

**단계별 역할 분담:**

| 단계 | Claude Code | 사람 |
|---|---|---|
| Red | 테스트 코드 작성, 실패 확인 | 테스트 내용 검토, 다음 단계 승인 |
| Green | 최소 구현 작성, 통과 확인 | 구현 내용 검토, 커밋 또는 수정 지시 |
| Refactor | 코드 개선, 통과 유지 확인 | 개선 방향 검토, 커밋 승인 |

**단계 전환 명령 예시:**
```
# Red -> Green
"테스트 내용 확인했다. Green 단계를 진행해라."

# Green -> Refactor
"테스트 통과 확인했다. 커밋하고 Refactor 단계를 진행해라."

# 추가 기능
"이 함수의 TDD가 완료됐다. subtract 함수를 같은 방식으로 진행해라."
```

---

## 언어별 테스트 프레임워크

### C++ / Google Test

**프로젝트 구조:**
```
my-project/
├── CMakeLists.txt
├── src/
│   └── calculator.cpp
└── tests/
    └── test_calculator.cpp
```

**CMakeLists.txt 기본 설정:**
```cmake
cmake_minimum_required(VERSION 3.14)
project(my_project)

set(CMAKE_CXX_STANDARD 17)

# Google Test
include(FetchContent)
FetchContent_Declare(
  googletest
  URL https://github.com/google/googletest/archive/refs/tags/v1.14.0.zip
)
FetchContent_MakeAvailable(googletest)

# Test executable
add_executable(run_tests tests/test_calculator.cpp src/calculator.cpp)
target_link_libraries(run_tests gtest_main)
```

**테스트 파일 기본 구조 (Given-When-Then):**
```cpp
#include <gtest/gtest.h>
#include "../src/calculator.h"

// Given: initial state
// When: action
// Then: expected result
TEST(CalculatorTest, AddsTwoPositiveNumbers) {
    // Arrange
    Calculator calc;
    // Act
    int result = calc.add(2, 3);
    // Assert
    EXPECT_EQ(result, 5);
}

TEST(CalculatorTest, AddNegativeNumber) {
    Calculator calc;
    int result = calc.add(-1, 3);
    EXPECT_EQ(result, 2);
}
```

**빌드 및 테스트 실행 명령 (Windows / MinGW64):**
```powershell
# Build
cmake -B build -G "MinGW Makefiles"
cmake --build build

# Run tests
.\build\run_tests.exe
```

**빌드 및 테스트 실행 명령 (WSL / gcc):**
```bash
# Build
cmake -B build
cmake --build build

# Run tests
./build/run_tests
```

**Red 단계 확인 — 헤더만 있고 구현 없는 상태:**
```
[ RUN      ] CalculatorTest.AddsTwoPositiveNumbers
[  FAILED  ] CalculatorTest.AddsTwoPositiveNumbers
```

**Green 단계 확인 — 최소 구현 후:**
```
[ RUN      ] CalculatorTest.AddsTwoPositiveNumbers
[       OK ] CalculatorTest.AddsTwoPositiveNumbers
[  PASSED  ] 1 test.
```

---

### Python / pytest

**프로젝트 구조:**
```
my-project/
├── pytest.ini
├── src/
│   └── calculator.py
└── tests/
    └── test_calculator.py
```

**pytest.ini 기본 설정:**
```ini
[pytest]
testpaths = tests
python_files = test_*.py
python_functions = test_*
```

**테스트 파일 기본 구조 (AAA 패턴):**
```python
import pytest
from src.calculator import add, subtract


def test_add_two_positive_numbers():
    # Arrange
    a, b = 2, 3
    # Act
    result = add(a, b)
    # Assert
    assert result == 5


def test_add_negative_number():
    # Arrange
    a, b = -1, 3
    # Act
    result = add(a, b)
    # Assert
    assert result == 2


def test_add_raises_on_non_integer():
    with pytest.raises(TypeError):
        add("a", 1)
```

**테스트 실행 명령:**
```bash
# Run all tests
pytest

# Run single file
pytest tests/test_calculator.py

# Run single test
pytest tests/test_calculator.py::test_add_two_positive_numbers

# Verbose output
pytest -v
```

**Red 단계 확인 — 구현 없는 상태:**
```
FAILED tests/test_calculator.py::test_add_two_positive_numbers
ImportError: cannot import name 'add' from 'src.calculator'
```

**Green 단계 확인 — 최소 구현 후:**
```
tests/test_calculator.py::test_add_two_positive_numbers PASSED
tests/test_calculator.py::test_add_negative_number PASSED
2 passed in 0.12s
```

---

### JAX / pytest

**프로젝트 구조:**
```
my-project/
├── pytest.ini
├── src/
│   └── jax_ops.py
└── tests/
    └── test_jax_ops.py
```

**테스트 파일 기본 구조:**
```python
import pytest
import jax.numpy as jnp
from src.jax_ops import normalize


def test_normalize_unit_vector():
    # Arrange
    x = jnp.array([3.0, 4.0])
    # Act
    result = normalize(x)
    # Assert
    expected_norm = jnp.float32(1.0)
    assert jnp.allclose(jnp.linalg.norm(result), expected_norm)


def test_normalize_zero_vector_raises():
    with pytest.raises(ValueError):
        normalize(jnp.zeros(3))
```

**JAX 특이사항:**
```python
# JAX array 비교는 == 대신 jnp.allclose 사용
assert jnp.allclose(result, expected, atol=1e-6)

# GPU 사용 확인
import jax
print(jax.devices())  # [GpuDevice(id=0, process_index=0)]
```

**테스트 실행 명령 (WSL / Anaconda):**
```bash
# Activate conda environment
conda activate jax-env

# Run tests
pytest tests/test_jax_ops.py -v
```

---

## AAA 패턴 (Arrange / Act / Assert)

모든 테스트 함수는 AAA 구조로 작성한다.

```
Arrange  테스트에 필요한 객체와 데이터를 준비한다
Act      테스트 대상 함수를 실행한다
Assert   결과가 기대값과 일치하는지 확인한다
```

**Python 예시:**
```python
def test_matrix_multiply():
    # Arrange
    a = jnp.array([[1, 2], [3, 4]])
    b = jnp.array([[5, 6], [7, 8]])
    expected = jnp.array([[19, 22], [43, 50]])

    # Act
    result = matrix_multiply(a, b)

    # Assert
    assert jnp.array_equal(result, expected)
```

**규칙:**
- 테스트 함수 하나에 Assert는 하나를 원칙으로 한다
- Arrange 블록이 길어지면 fixture로 분리한다
- 테스트 함수 이름은 `test_{행동}_{조건}` 형식으로 작성한다

---

## 커밋 타이밍 전략

```
커밋 시점: 테스트 통과마다 (Green 또는 Refactor 완료 시)
푸시 시점: 세션 종료 전 1회
```

**커밋 메시지 형식:**
```
test:     테스트 코드 추가
feat:     새 기능 구현 (Green 완료)
refactor: 리팩터링 (Refactor 완료)
fix:      버그 수정
docs:     문서 업데이트
chore:    설정 변경
```

**커밋 명령 예시:**
```bash
# Red 단계 후 (테스트만 작성)
git add tests/test_calculator.py
git commit -m "test: add test for calculator.add"

# Green 단계 후 (구현 완료)
git add src/calculator.py
git commit -m "feat: implement calculator.add"

# Refactor 단계 후
git add src/calculator.py
git commit -m "refactor: extract validation logic in calculator.add"
```

**Claude Code에 커밋 요청:**
```
"테스트가 통과했다. 변경된 파일을 확인하고 커밋해라.
 메시지 형식: feat: {설명}"
```

---

## 세션 계속 vs 새 세션 분리 기준

**같은 세션을 계속하는 경우:**
- 동일한 기능의 TDD 사이클이 완료되지 않았다
- 컨텍스트 창이 충분하다 (응답 품질 저하 없음)
- 연속된 소규모 수정 작업이다

**새 세션으로 분리하는 경우:**
- 작업 목표가 바뀐다 (예: 구현 완료 후 리팩터링 집중)
- 응답 품질이 저하되거나 이전 내용을 반복해서 묻는다
- 하나의 기능 구현이 완료됐다
- 다른 모듈 작업으로 전환한다

**세션 종료 전 인계 작업:**
```
"현재까지 작업한 내용을 sessions/01_task-name/report.md에 작성해라.
 완료 항목, 미완료 항목, 다음 세션에서 할 일을 포함해라."
```

**새 세션 시작 시 인계 요청:**
```
"sessions/01_task-name/report.md와 CLAUDE.md를 읽고
 현재 상태를 요약해라. 작업은 시작하지 마라."
```

---

## Verification Checklist

- Red 단계에서 테스트가 실제로 실패하는지 확인한다
- Green 단계에서 최소 구현만 작성했는지 확인한다
- 각 테스트 함수가 AAA 구조를 따르는지 확인한다
- 테스트 통과마다 커밋했는지 확인한다
- 세션 종료 전 report.md가 생성됐는지 확인한다

---

## Common Errors and Solutions

| 오류 | 원인 | 해결 |
|---|---|---|
| Red 단계를 건너뛰고 구현부터 작성한다 | rules/tdd.md 미적용 | 세션 시작 시 `[P]` 항목에 tdd SKILL 명시 |
| pytest가 테스트 파일을 찾지 못한다 | pytest.ini 설정 누락 | `testpaths = tests` 설정 확인 |
| C++ 빌드가 실패한다 | CMakeLists.txt 경로 오류 | `cmake -B build` 실행 디렉터리 확인 |
| JAX에서 GPU를 인식하지 못한다 | CUDA 환경 미설정 | `jax.devices()` 출력 확인, WSL CUDA 설정 점검 |
| 테스트 통과 후 커밋을 잊는다 | 수동 커밋 누락 | Claude Code에 "테스트 통과 후 커밋해라" 규칙 추가 |
