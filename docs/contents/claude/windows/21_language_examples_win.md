# Windows 언어별 TDD 예제

## Overview

- 목적: Windows 환경(MinGW64, WinPython)에서 C++과 Python TDD 사이클을 Claude Code와 함께 실행하는 방법
- 사전 조건: `02_setup_win.md` 완료 (Node.js, Claude Code, VSCode 설치)

---

## C++ TDD 예제 (MinGW64 / CMake / Google Test)

### Google Test 설치

vcpkg를 사용하는 방법과 소스 직접 빌드 방법 두 가지를 제공한다.

#### 방법 A: vcpkg 사용 (권장)

```powershell
# Clone vcpkg to a stable location
cd C:\tools
git clone https://github.com/microsoft/vcpkg.git
cd vcpkg
.\bootstrap-vcpkg.bat

# Install Google Test (x64)
.\vcpkg.exe install gtest:x64-windows

# Integrate with MSBuild / CMake globally
.\vcpkg.exe integrate install
```

CMake에서 vcpkg toolchain을 사용하려면 cmake 호출 시 `-DCMAKE_TOOLCHAIN_FILE` 옵션을 추가한다 (아래 CMakeLists.txt 섹션 참조).

#### 방법 B: FetchContent (인터넷 빌드)

외부 도구 없이 CMakeLists.txt 내에서 Google Test를 자동 다운로드한다. 아래 CMakeLists.txt 예시에 포함되어 있다.

---

### CMakeLists.txt 설정

프로젝트 루트 기준 예시 (`calculator` 프로젝트):

```
my-project/
|-- CMakeLists.txt
|-- src/
|   |-- calculator.h
|   |-- calculator.cpp
|-- tests/
    |-- test_calculator.cpp
```

```cmake
# CMakeLists.txt
cmake_minimum_required(VERSION 3.20)
project(calculator CXX)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# Download Google Test via FetchContent
include(FetchContent)
FetchContent_Declare(
  googletest
  URL https://github.com/google/googletest/archive/refs/tags/v1.14.0.zip
)
# Prevent overriding the parent project's compiler/linker settings on Windows
set(gtest_force_shared_crt ON CACHE BOOL "" FORCE)
FetchContent_MakeAvailable(googletest)

# Source library
add_library(calculator_lib src/calculator.cpp)
target_include_directories(calculator_lib PUBLIC src)

# Test executable
add_executable(calculator_test tests/test_calculator.cpp)
target_link_libraries(calculator_test
  PRIVATE calculator_lib
  PRIVATE GTest::gtest_main
)

include(GoogleTest)
gtest_discover_tests(calculator_test)
```

---

### Given-When-Then 테스트 구조

Google Test에서 Given-When-Then을 주석으로 명시하는 방식:

```cpp
// tests/test_calculator.cpp

#include <gtest/gtest.h>
#include "calculator.h"

// Test fixture for shared setup
class CalculatorTest : public ::testing::Test {
protected:
    Calculator calc;  // shared across tests in this fixture
};

TEST_F(CalculatorTest, AddTwoPositiveNumbers) {
    // Given: two positive integers
    int a = 3;
    int b = 4;

    // When: add is called
    int result = calc.add(a, b);

    // Then: result equals their sum
    EXPECT_EQ(result, 7);
}

TEST_F(CalculatorTest, DivideByZeroThrows) {
    // Given: a valid numerator and zero denominator
    int a = 10;
    int b = 0;

    // When / Then: division throws std::invalid_argument
    EXPECT_THROW(calc.divide(a, b), std::invalid_argument);
}
```

---

### Red / Green / Refactor 전체 사이클 예제

#### Red 단계 — 실패하는 테스트 작성

```cpp
// tests/test_calculator.cpp (Red)
TEST_F(CalculatorTest, MultiplyTwoNumbers) {
    // Given
    int a = 5;
    int b = 6;

    // When
    int result = calc.multiply(a, b);

    // Then
    EXPECT_EQ(result, 30);
}
```

`multiply` 함수가 아직 없으므로 빌드 실패.

```powershell
# Build and run tests
cmake -S . -B build -G "MinGW Makefiles"
cmake --build build
cd build && ctest --output-on-failure
```

예상 출력 (Red):

```
error: 'class Calculator' has no member named 'multiply'
```

#### Green 단계 — 최소 구현

```cpp
// src/calculator.h
#pragma once
#include <stdexcept>

class Calculator {
public:
    int add(int a, int b);
    int divide(int a, int b);
    int multiply(int a, int b);  // Add declaration
};
```

```cpp
// src/calculator.cpp
#include "calculator.h"

int Calculator::add(int a, int b) { return a + b; }

int Calculator::divide(int a, int b) {
    if (b == 0) throw std::invalid_argument("Division by zero");
    return a / b;
}

int Calculator::multiply(int a, int b) {
    return a * b;  // Minimal implementation to pass the test
}
```

```powershell
cmake --build build
cd build && ctest --output-on-failure
```

예상 출력 (Green):

```
[==========] 3 tests from 1 test suite ran.
[  PASSED  ] 3 tests.
```

```powershell
# Commit on green
git add .
git commit -m "feat: add multiply to Calculator"
```

#### Refactor 단계

테스트가 통과된 상태에서 코드 개선. 구현이 단순한 경우 이 단계는 생략 가능.

---

### Claude Code 요청 명령 예시

```powershell
# Start Claude Code in project root
claude

# Example requests inside Claude Code session
```

```
Read .claude/skills/tdd/SKILL.md and proceed with TDD role.
Current file: tests/test_calculator.cpp
Task: Write a failing test for subtract(a, b) — Red phase only.
```

```
Build failed. Error output:
  error: 'class Calculator' has no member named 'subtract'
Write minimal implementation for subtract in src/calculator.cpp — Green phase.
```

---

## Python TDD 예제 (WinPython / pytest)

### pytest 설치 및 설정

```powershell
pip install pytest pytest-cov
```

설치 확인:

```powershell
pytest --version
```

프로젝트 루트에 `pytest.ini` 생성:

```ini
[pytest]
testpaths = tests
python_files = test_*.py
python_classes = Test*
python_functions = test_*
addopts = -v --tb=short
```

---

### 프로젝트 구조

```
my-project/
|-- src/
|   |-- stats.py
|-- tests/
|   |-- test_stats.py
|-- pytest.ini
```

---

### NumPy 코드 TDD 예제

#### Red 단계 — 실패하는 테스트 작성

```python
# tests/test_stats.py

import numpy as np
import pytest
from src.stats import compute_zscore


class TestComputeZscore:

    def test_standard_normal_values(self):
        # Arrange
        data = np.array([1.0, 2.0, 3.0, 4.0, 5.0])

        # Act
        result = compute_zscore(data)

        # Assert
        expected_mean = 0.0
        expected_std = 1.0
        assert abs(result.mean()) < 1e-10
        assert abs(result.std() - expected_std) < 1e-10

    def test_constant_array_raises(self):
        # Arrange
        data = np.array([3.0, 3.0, 3.0])

        # Act / Assert
        with pytest.raises(ValueError, match="zero standard deviation"):
            compute_zscore(data)
```

```powershell
pytest tests/test_stats.py
```

예상 출력 (Red):

```
ERROR tests/test_stats.py - ModuleNotFoundError: No module named 'src.stats'
```

#### Green 단계 — 최소 구현

```python
# src/__init__.py  (empty)
```

```python
# src/stats.py

import numpy as np


def compute_zscore(data: np.ndarray) -> np.ndarray:
    """Compute z-score normalization of a 1D array."""
    std = data.std()
    if std == 0:
        raise ValueError("zero standard deviation: cannot compute z-score")
    return (data - data.mean()) / std
```

```powershell
pytest tests/test_stats.py
```

예상 출력 (Green):

```
tests/test_stats.py::TestComputeZscore::test_standard_normal_values PASSED
tests/test_stats.py::TestComputeZscore::test_constant_array_raises PASSED
2 passed in 0.12s
```

```powershell
git add .
git commit -m "feat: add compute_zscore with zero-std guard"
```

#### Refactor 단계

```python
# src/stats.py (refactored)

import numpy as np


def compute_zscore(data: np.ndarray) -> np.ndarray:
    """
    Compute z-score normalization of a 1D array.

    Raises:
        ValueError: if standard deviation is zero.
    """
    mean = data.mean()
    std = data.std()

    if std == 0:
        raise ValueError("zero standard deviation: cannot compute z-score")

    return (data - mean) / std
```

```powershell
pytest tests/test_stats.py
```

통과 확인 후:

```powershell
git add .
git commit -m "refactor: clarify variable names in compute_zscore"
```

---

### Claude Code 요청 명령 예시

```
Read .claude/skills/tdd/SKILL.md and proceed with TDD role.
Current file: tests/test_stats.py
Task: Write a failing test for moving_average(data, window) — Red phase only.
Constraint: Use numpy only, no pandas.
```

```
pytest output:
  FAILED tests/test_stats.py::TestMovingAverage::test_basic_window
  AssertionError: ...
Write minimal implementation for moving_average in src/stats.py — Green phase.
```

---

## VSCode 빌드 설정 (Windows)

### .vscode/tasks.json

C++ 빌드 태스크와 Python pytest 실행 태스크를 포함한다.

```json
{
  "version": "2.0.0",
  "tasks": [
    {
      "label": "C++: CMake Configure",
      "type": "shell",
      "command": "cmake -S . -B build -G \"MinGW Makefiles\"",
      "group": "build",
      "presentation": { "reveal": "always", "panel": "shared" }
    },
    {
      "label": "C++: CMake Build",
      "type": "shell",
      "command": "cmake --build build",
      "group": { "kind": "build", "isDefault": true },
      "dependsOn": "C++: CMake Configure",
      "presentation": { "reveal": "always", "panel": "shared" }
    },
    {
      "label": "C++: Run Tests (ctest)",
      "type": "shell",
      "command": "cd build && ctest --output-on-failure",
      "group": { "kind": "test", "isDefault": false },
      "dependsOn": "C++: CMake Build",
      "presentation": { "reveal": "always", "panel": "shared" }
    },
    {
      "label": "Python: pytest",
      "type": "shell",
      "command": "pytest",
      "group": { "kind": "test", "isDefault": true },
      "presentation": { "reveal": "always", "panel": "shared" }
    }
  ]
}
```

태스크 실행: `Ctrl+Shift+B` (기본 빌드 태스크) 또는 `Ctrl+Shift+P` -> "Tasks: Run Task".

### .vscode/launch.json

```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "C++: Debug Test",
      "type": "cppdbg",
      "request": "launch",
      "program": "${workspaceFolder}/build/calculator_test.exe",
      "args": [],
      "stopAtEntry": false,
      "cwd": "${workspaceFolder}",
      "environment": [],
      "externalConsole": false,
      "MIMode": "gdb",
      "miDebuggerPath": "gdb",
      "setupCommands": [
        {
          "description": "Enable pretty-printing for gdb",
          "text": "-enable-pretty-printing",
          "ignoreFailures": true
        }
      ]
    },
    {
      "name": "Python: pytest debug",
      "type": "debugpy",
      "request": "launch",
      "module": "pytest",
      "args": ["-v", "--no-header"],
      "console": "integratedTerminal"
    }
  ]
}
```

---

## 설치 확인 체크리스트

- [ ] `cmake --version` 출력 확인 (3.20 이상)
- [ ] `g++ --version` 출력 확인
- [ ] C++ 프로젝트: `cmake -S . -B build -G "MinGW Makefiles"` 성공
- [ ] C++ 프로젝트: `cmake --build build` 성공
- [ ] C++ 테스트: `cd build && ctest --output-on-failure` Green 확인
- [ ] `pytest --version` 출력 확인
- [ ] Python 테스트: `pytest` Green 확인
- [ ] VSCode tasks.json 태스크 실행 확인 (`Ctrl+Shift+B`)
- [ ] VSCode launch.json 디버거 실행 확인 (`F5`)

---

## 공통 오류 및 해결

| 오류 | 원인 | 해결 |
|---|---|---|
| `cmake: command not found` | CMake PATH 미등록 | CMake 설치 후 시스템 PATH에 `cmake/bin` 추가 |
| `-- The CXX compiler identification is unknown` | MinGW64 PATH 미등록 | MinGW64 `bin` 폴더를 시스템 PATH에 추가 후 PowerShell 재시작 |
| FetchContent 다운로드 실패 | 네트워크/방화벽 차단 | 방법 B(vcpkg) 사용 또는 Google Test zip을 수동 다운로드하여 로컬 경로 지정 |
| `gtest_discover_tests` 실행 파일 미발견 | 빌드 경로 불일치 | `cmake --build build` 완료 후 `build/` 안에 `.exe` 파일 존재 확인 |
| `ModuleNotFoundError: No module named 'src'` | `src/__init__.py` 없음 또는 PYTHONPATH 미설정 | `src/__init__.py` 생성 또는 `pytest.ini`에 `pythonpath = .` 추가 |
| `pytest` 명령 미인식 | WinPython pip로 설치했으나 PATH 문제 | `python -m pytest` 로 실행 또는 WinPython Command Prompt 사용 |
| VSCode C++ 디버거 `gdb` 미발견 | gdb PATH 미등록 | `miDebuggerPath`를 `gdb.exe` 절대 경로로 지정 (예: `C:/mingw64/bin/gdb.exe`) |
