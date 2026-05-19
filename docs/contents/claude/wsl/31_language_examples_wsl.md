# WSL 언어별 TDD 예제

## Overview

- **목적**: WSL2 (Ubuntu) 환경에서 Python, PyTorch, JAX, C++ 각 언어의 TDD 사이클 실행 방법
- **사전 조건**: `02_setup_wsl.md` 환경 설정 완료, Anaconda 설치, NVIDIA GTX 1080 Ti

---

## Python TDD (Anaconda / pytest)

### conda 환경 생성 및 활성화

프로젝트별 독립 환경을 생성한다.

```bash
conda create -n project-env python=3.11 -y
conda activate project-env
```

### pytest 설치 및 설정

```bash
pip install pytest pytest-cov
```

프로젝트 루트에 `pytest.ini`를 생성한다.

```ini
[pytest]
testpaths = tests
python_files = test_*.py
python_classes = Test*
python_functions = test_*
addopts = -v --tb=short
```

### 프로젝트 구조

```
my-project/
|-- src/
|   |-- calculator.py
|-- tests/
|   |-- test_calculator.py
|-- pytest.ini
```

### TDD 예제: NumPy 통계 계산기

**Red: 실패하는 테스트 먼저 작성**

```python
# tests/test_calculator.py
import pytest
import numpy as np
from src.calculator import StatCalculator

class TestStatCalculator:

    def test_mean_returns_correct_value(self):
        # Arrange
        calc = StatCalculator()
        data = np.array([1.0, 2.0, 3.0, 4.0, 5.0])
        # Act
        result = calc.mean(data)
        # Assert
        assert result == pytest.approx(3.0)

    def test_std_returns_correct_value(self):
        # Arrange
        calc = StatCalculator()
        data = np.array([2.0, 4.0, 4.0, 4.0, 5.0, 5.0, 7.0, 9.0])
        # Act
        result = calc.std(data)
        # Assert
        assert result == pytest.approx(2.0)

    def test_normalize_scales_to_zero_one(self):
        # Arrange
        calc = StatCalculator()
        data = np.array([0.0, 5.0, 10.0])
        # Act
        result = calc.normalize(data)
        # Assert
        assert result[0] == pytest.approx(0.0)
        assert result[-1] == pytest.approx(1.0)
```

테스트 실패 확인:

```bash
pytest tests/test_calculator.py
# Expected: FAILED (3 errors - module not found)
```

**Green: 테스트를 통과하는 최소 구현**

```python
# src/calculator.py
import numpy as np

class StatCalculator:

    def mean(self, data: np.ndarray) -> float:
        return float(np.mean(data))

    def std(self, data: np.ndarray) -> float:
        return float(np.std(data))

    def normalize(self, data: np.ndarray) -> np.ndarray:
        min_val = np.min(data)
        max_val = np.max(data)
        return (data - min_val) / (max_val - min_val)
```

```bash
pytest tests/test_calculator.py
# Expected: 3 passed
```

**Refactor: 엣지 케이스 처리 추가**

```python
# src/calculator.py (refactored)
import numpy as np

class StatCalculator:

    def mean(self, data: np.ndarray) -> float:
        if len(data) == 0:
            raise ValueError("Input array is empty")
        return float(np.mean(data))

    def std(self, data: np.ndarray) -> float:
        if len(data) == 0:
            raise ValueError("Input array is empty")
        return float(np.std(data))

    def normalize(self, data: np.ndarray) -> np.ndarray:
        if len(data) == 0:
            raise ValueError("Input array is empty")
        min_val = np.min(data)
        max_val = np.max(data)
        if max_val == min_val:
            raise ValueError("Cannot normalize: all values are identical")
        return (data - min_val) / (max_val - min_val)
```

```bash
pytest tests/test_calculator.py -v
# Expected: 3 passed
git add -A && git commit -m "feat: implement StatCalculator with normalize"
```

### Claude Code 요청 명령 예시

```
Read .claude/skills/tdd/SKILL.md and implement a pandas DataFrame
validator class in src/validator.py with pytest tests in tests/test_validator.py.
Requirements:
- validate_no_nulls(df): raise ValueError if any null values
- validate_column_types(df, schema): raise TypeError if column types mismatch
Follow Red-Green-Refactor cycle.
```

---

## PyTorch TDD (Anaconda / pytest)

### PyTorch conda 환경 설정

```bash
conda create -n torch-env python=3.11 -y
conda activate torch-env
conda install pytorch torchvision pytorch-cuda=12.1 -c pytorch -c nvidia -y
pip install pytest pytest-cov
```

### GPU 인식 확인

```python
# scripts/check_gpu.py
import torch

print(f"PyTorch version: {torch.__version__}")
print(f"CUDA available: {torch.cuda.is_available()}")
if torch.cuda.is_available():
    print(f"GPU: {torch.cuda.get_device_name(0)}")
    print(f"VRAM: {torch.cuda.get_device_properties(0).total_memory / 1e9:.1f} GB")
```

```bash
python scripts/check_gpu.py
# Expected:
# PyTorch version: 2.x.x
# CUDA available: True
# GPU: NVIDIA GeForce GTX 1080 Ti
# VRAM: 11.2 GB
```

### TDD 예제: Custom Loss Function

**Red: 테스트 먼저 작성**

```python
# tests/test_focal_loss.py
import pytest
import torch
from src.losses import FocalLoss

class TestFocalLoss:

    def test_focal_loss_perfect_prediction_is_zero(self):
        # Arrange
        loss_fn = FocalLoss(gamma=2.0)
        logits = torch.tensor([[10.0, -10.0]])
        targets = torch.tensor([0])
        # Act
        loss = loss_fn(logits, targets)
        # Assert
        assert loss.item() == pytest.approx(0.0, abs=1e-4)

    def test_focal_loss_wrong_prediction_is_positive(self):
        # Arrange
        loss_fn = FocalLoss(gamma=2.0)
        logits = torch.tensor([[-10.0, 10.0]])
        targets = torch.tensor([0])
        # Act
        loss = loss_fn(logits, targets)
        # Assert
        assert loss.item() > 0.0

    def test_focal_loss_runs_on_gpu(self):
        # Arrange
        if not torch.cuda.is_available():
            pytest.skip("GPU not available")
        loss_fn = FocalLoss(gamma=2.0)
        logits = torch.randn(4, 10).cuda()
        targets = torch.randint(0, 10, (4,)).cuda()
        # Act
        loss = loss_fn(logits, targets)
        # Assert
        assert loss.device.type == "cuda"
        assert loss.item() >= 0.0
```

```bash
pytest tests/test_focal_loss.py
# Expected: FAILED (module not found)
```

**Green: 구현**

```python
# src/losses.py
import torch
import torch.nn as nn
import torch.nn.functional as F

class FocalLoss(nn.Module):

    def __init__(self, gamma: float = 2.0, reduction: str = "mean"):
        super().__init__()
        self.gamma = gamma
        self.reduction = reduction

    def forward(self, logits: torch.Tensor, targets: torch.Tensor) -> torch.Tensor:
        ce_loss = F.cross_entropy(logits, targets, reduction="none")
        pt = torch.exp(-ce_loss)
        focal_loss = (1 - pt) ** self.gamma * ce_loss

        if self.reduction == "mean":
            return focal_loss.mean()
        elif self.reduction == "sum":
            return focal_loss.sum()
        return focal_loss
```

```bash
pytest tests/test_focal_loss.py -v
# Expected: 3 passed
git add -A && git commit -m "feat: implement FocalLoss with GPU support"
```

### Claude Code 요청 명령 예시

```
Read .claude/skills/tdd/SKILL.md and implement a DataAugmentation
class in src/augmentation.py with pytest tests in tests/test_augmentation.py.
Requirements:
- random_horizontal_flip(tensor, p=0.5): flip image tensor horizontally
- random_crop(tensor, size): randomly crop to given size
- normalize(tensor, mean, std): normalize with given statistics
Use torch.Tensor as input type. Include GPU tests with pytest.mark.skipif.
Follow Red-Green-Refactor cycle.
```

---

## JAX TDD (Anaconda / pytest)

### JAX conda 환경 설정

```bash
conda create -n jax-env python=3.11 -y
conda activate jax-env
pip install "jax[cuda12_pip]" -f https://storage.googleapis.com/jax-releases/jax_cuda_releases.html
pip install pytest pytest-cov
```

### GPU 인식 확인

```python
# scripts/check_jax.py
import jax

print(f"JAX version: {jax.__version__}")
print(f"Devices: {jax.devices()}")
print(f"Backend: {jax.default_backend()}")
```

```bash
python scripts/check_jax.py
# Expected:
# JAX version: 0.4.x
# Devices: [CudaDevice(id=0)]
# Backend: gpu
```

### TDD 예제: JAX 행렬 연산

**Red: 테스트 먼저 작성**

```python
# tests/test_matrix_ops.py
import pytest
import jax
import jax.numpy as jnp
from src.matrix_ops import MatrixOps

class TestMatrixOps:

    def test_softmax_output_sums_to_one(self):
        # Arrange
        ops = MatrixOps()
        x = jnp.array([1.0, 2.0, 3.0])
        # Act
        result = ops.softmax(x)
        # Assert
        assert jnp.abs(jnp.sum(result) - 1.0) < 1e-6

    def test_softmax_preserves_order(self):
        # Arrange
        ops = MatrixOps()
        x = jnp.array([1.0, 3.0, 2.0])
        # Act
        result = ops.softmax(x)
        # Assert
        assert result[1] > result[2] > result[0]

    def test_batch_matmul_shape(self):
        # Arrange
        ops = MatrixOps()
        a = jnp.ones((4, 3, 2))
        b = jnp.ones((4, 2, 5))
        # Act
        result = ops.batch_matmul(a, b)
        # Assert
        assert result.shape == (4, 3, 5)

    def test_runs_on_gpu(self):
        # Arrange
        ops = MatrixOps()
        if jax.default_backend() != "gpu":
            pytest.skip("GPU not available")
        x = jnp.ones((1000, 1000))
        # Act
        result = ops.softmax(x)
        # Assert
        assert result.shape == (1000, 1000)
```

```bash
pytest tests/test_matrix_ops.py
# Expected: FAILED (module not found)
```

**Green: 구현**

```python
# src/matrix_ops.py
import jax.numpy as jnp
from jax import jit

class MatrixOps:

    @jit
    def softmax(self, x: jnp.ndarray) -> jnp.ndarray:
        x_max = jnp.max(x)
        exp_x = jnp.exp(x - x_max)
        return exp_x / jnp.sum(exp_x)

    @jit
    def batch_matmul(self, a: jnp.ndarray, b: jnp.ndarray) -> jnp.ndarray:
        return jnp.einsum("bik,bkj->bij", a, b)
```

```bash
pytest tests/test_matrix_ops.py -v
# Expected: 4 passed
git add -A && git commit -m "feat: implement JAX MatrixOps with jit compilation"
```

### Claude Code 요청 명령 예시

```
Read .claude/skills/tdd/SKILL.md and implement a JAX-based
gradient descent optimizer in src/optimizer.py with tests in tests/test_optimizer.py.
Requirements:
- SGDOptimizer: step(params, grads, lr) -> updated_params
- AdamOptimizer: step(params, grads, m, v, t, lr) -> (updated_params, m, v)
Use jax.numpy only. Include @jit decorators. Follow Red-Green-Refactor cycle.
```

---

## C++ TDD (gcc / Google Test)

### build-essential 설치

```bash
sudo apt update
sudo apt install build-essential cmake -y

gcc --version
# Expected: gcc (Ubuntu) 11.x.x

cmake --version
# Expected: cmake version 3.x.x
```

### Google Test 설치

#### 방법 A: apt 설치

```bash
sudo apt install libgtest-dev -y
cd /usr/src/gtest
sudo cmake CMakeLists.txt
sudo make
sudo cp lib/*.a /usr/lib
```

#### 방법 B: CMake FetchContent (권장, 버전 독립적)

`CMakeLists.txt`에서 FetchContent로 자동 다운로드한다. 별도 설치 불필요.

### 프로젝트 구조

```
my-project/
|-- src/
|   |-- string_utils.cpp
|   |-- string_utils.h
|-- tests/
|   |-- test_string_utils.cpp
|-- CMakeLists.txt
```

### CMakeLists.txt 설정

```cmake
cmake_minimum_required(VERSION 3.14)
project(MyProject CXX)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# Source library
add_library(string_utils src/string_utils.cpp)
target_include_directories(string_utils PUBLIC src/)

# Google Test via FetchContent
include(FetchContent)
FetchContent_Declare(
  googletest
  URL https://github.com/google/googletest/archive/v1.14.0.zip
)
set(gtest_force_shared_crt ON CACHE BOOL "" FORCE)
FetchContent_MakeAvailable(googletest)

enable_testing()

# Test executable
add_executable(test_string_utils tests/test_string_utils.cpp)
target_link_libraries(test_string_utils string_utils GTest::gtest_main)

include(GoogleTest)
gtest_discover_tests(test_string_utils)
```

### TDD 예제: 문자열 유틸리티

**Red: 테스트 먼저 작성**

```cpp
// tests/test_string_utils.cpp
#include <gtest/gtest.h>
#include "string_utils.h"

class StringUtilsTest : public ::testing::Test {
protected:
    StringUtils utils;
};

// Given-When-Then: trim
TEST_F(StringUtilsTest, TrimRemovesLeadingAndTrailingSpaces) {
    // Given
    std::string input = "  hello world  ";
    // When
    std::string result = utils.trim(input);
    // Then
    EXPECT_EQ(result, "hello world");
}

TEST_F(StringUtilsTest, TrimEmptyStringReturnsEmpty) {
    // Given
    std::string input = "";
    // When
    std::string result = utils.trim(input);
    // Then
    EXPECT_EQ(result, "");
}

// Given-When-Then: split
TEST_F(StringUtilsTest, SplitByCommaReturnsCorrectParts) {
    // Given
    std::string input = "a,b,c";
    // When
    auto result = utils.split(input, ',');
    // Then
    ASSERT_EQ(result.size(), 3u);
    EXPECT_EQ(result[0], "a");
    EXPECT_EQ(result[1], "b");
    EXPECT_EQ(result[2], "c");
}

// Given-When-Then: to_upper
TEST_F(StringUtilsTest, ToUpperConvertsAllCharacters) {
    // Given
    std::string input = "hello";
    // When
    std::string result = utils.to_upper(input);
    // Then
    EXPECT_EQ(result, "HELLO");
}
```

빌드 후 실패 확인:

```bash
cmake -B build -S .
cmake --build build
# Expected: linker error (symbol not found)
```

**Green: 구현**

```cpp
// src/string_utils.h
#pragma once
#include <string>
#include <vector>

class StringUtils {
public:
    std::string trim(const std::string& s);
    std::vector<std::string> split(const std::string& s, char delimiter);
    std::string to_upper(const std::string& s);
};
```

```cpp
// src/string_utils.cpp
#include "string_utils.h"
#include <algorithm>
#include <sstream>

std::string StringUtils::trim(const std::string& s) {
    size_t start = s.find_first_not_of(" \t\n\r");
    if (start == std::string::npos) return "";
    size_t end = s.find_last_not_of(" \t\n\r");
    return s.substr(start, end - start + 1);
}

std::vector<std::string> StringUtils::split(const std::string& s, char delimiter) {
    std::vector<std::string> tokens;
    std::stringstream ss(s);
    std::string token;
    while (std::getline(ss, token, delimiter)) {
        tokens.push_back(token);
    }
    return tokens;
}

std::string StringUtils::to_upper(const std::string& s) {
    std::string result = s;
    std::transform(result.begin(), result.end(), result.begin(), ::toupper);
    return result;
}
```

```bash
cmake --build build
cd build && ctest --output-on-failure
# Expected: 4 tests passed
cd ..
git add -A && git commit -m "feat: implement StringUtils with trim/split/to_upper"
```

---

## VSCode Remote-WSL 빌드 설정

### .vscode/tasks.json

```json
{
  "version": "2.0.0",
  "tasks": [
    {
      "label": "cmake: configure",
      "type": "shell",
      "command": "cmake -B build -S .",
      "group": "build",
      "presentation": {
        "reveal": "always",
        "panel": "shared"
      }
    },
    {
      "label": "cmake: build",
      "type": "shell",
      "command": "cmake --build build --parallel",
      "group": {
        "kind": "build",
        "isDefault": true
      },
      "dependsOn": "cmake: configure",
      "presentation": {
        "reveal": "always",
        "panel": "shared"
      }
    },
    {
      "label": "ctest: run",
      "type": "shell",
      "command": "cd build && ctest --output-on-failure",
      "group": {
        "kind": "test",
        "isDefault": true
      },
      "dependsOn": "cmake: build",
      "presentation": {
        "reveal": "always",
        "panel": "shared"
      }
    },
    {
      "label": "pytest: run",
      "type": "shell",
      "command": "pytest tests/ -v",
      "group": {
        "kind": "test",
        "isDefault": false
      },
      "presentation": {
        "reveal": "always",
        "panel": "shared"
      }
    }
  ]
}
```

### .vscode/launch.json

```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Debug C++ Tests",
      "type": "cppdbg",
      "request": "launch",
      "program": "${workspaceFolder}/build/test_string_utils",
      "args": [],
      "stopAtEntry": false,
      "cwd": "${workspaceFolder}",
      "environment": [],
      "externalConsole": false,
      "MIMode": "gdb",
      "setupCommands": [
        {
          "description": "Enable pretty-printing for gdb",
          "text": "-enable-pretty-printing",
          "ignoreFailures": true
        }
      ],
      "preLaunchTask": "cmake: build"
    },
    {
      "name": "Debug Python Tests",
      "type": "debugpy",
      "request": "launch",
      "module": "pytest",
      "args": ["tests/", "-v", "--no-header"],
      "console": "integratedTerminal",
      "justMyCode": false
    }
  ]
}
```

---

## 언어별 Claude Code 요청 명령 예시

### Python / NumPy

```
Read .claude/skills/tdd/SKILL.md.
Implement a moving average calculator in src/moving_average.py.
- simple_ma(data: np.ndarray, window: int) -> np.ndarray
- exponential_ma(data: np.ndarray, alpha: float) -> np.ndarray
Write pytest tests in tests/test_moving_average.py first (Red).
Then implement (Green). Then add edge case handling (Refactor).
Commit after each Green step.
```

### PyTorch

```
Read .claude/skills/tdd/SKILL.md.
Implement a custom attention mechanism in src/attention.py.
- ScaledDotProductAttention: forward(q, k, v, mask=None)
Write tests in tests/test_attention.py including GPU tests.
Use pytest.mark.skipif for GPU-specific tests.
Follow Red-Green-Refactor cycle. Commit after each Green step.
```

### JAX

```
Read .claude/skills/tdd/SKILL.md.
Implement a neural network layer in src/layers.py using JAX.
- linear_layer(params, x): matrix multiply + bias
- relu(x): ReLU activation
Use @jax.jit where appropriate.
Write tests in tests/test_layers.py. Follow Red-Green-Refactor cycle.
```

### C++

```
Read .claude/skills/tdd/SKILL.md.
Implement a stack data structure in src/stack.h and src/stack.cpp.
- push(value), pop() -> value, peek() -> value, is_empty() -> bool
Write Google Test tests in tests/test_stack.cpp using TEST_F and Given-When-Then structure.
Update CMakeLists.txt to include the new test target.
Follow Red-Green-Refactor cycle. Commit after each Green step.
```

---

## 설치 확인 체크리스트

- [ ] `conda activate project-env` 정상 활성화
- [ ] `pytest tests/ -v` Python 테스트 실행 확인
- [ ] `python scripts/check_gpu.py` CUDA available: True 확인 (PyTorch)
- [ ] `python scripts/check_jax.py` Backend: gpu 확인 (JAX)
- [ ] `cmake -B build -S .` 빌드 구성 성공
- [ ] `cd build && ctest` C++ 테스트 실행 확인
- [ ] VSCode `Ctrl+Shift+B` 빌드 태스크 실행 확인
- [ ] VSCode 테스트 실행 (`Ctrl+Shift+P` > `Tasks: Run Test Task`) 확인

---

## 공통 오류 및 해결 방법

| 오류 | 원인 | 해결 방법 |
|---|---|---|
| `ModuleNotFoundError: src` | Python 경로 미설정 | `pip install -e .` 또는 `PYTHONPATH=$PWD pytest` |
| `CUDA not available` (PyTorch) | CUDA 버전 불일치 | `conda install pytorch pytorch-cuda=12.1 -c pytorch -c nvidia` |
| `jax.errors.UnexpectedTracerError` | JAX 추적 오류 | jit 내부에서 Python 제어 흐름 제거 |
| `FetchContent: network error` | 인터넷 접근 불가 | `cmake -DFETCHCONTENT_FULLY_DISCONNECTED=OFF -B build` |
| `gtest not found` (apt 방식) | 라이브러리 미복사 | `sudo cp /usr/src/gtest/lib/*.a /usr/lib/` |
| `conda: command not found` | Anaconda PATH 미설정 | `conda init bash && source ~/.bashrc` |
| VSCode에서 Python 인터프리터 오류 | 잘못된 conda 환경 선택 | `Ctrl+Shift+P` > `Python: Select Interpreter` > 해당 conda 환경 선택 |
