# Windows 로컬 LLM 설정 (Ollama)

## Overview

- 목적: Windows 네이티브 Ollama 설치, GTX 1080 Ti CUDA 설정, Claude Code와의 역할 분담
- 사전 조건: `02_setup_win.md` 완료, NVIDIA GTX 1080 Ti 드라이버 설치

---

## 역할 분담 (Claude Code vs Ollama)

| 기준 | Claude Code (Anthropic 서버) | Ollama (로컬 GTX 1080 Ti) |
|---|---|---|
| 주요 용도 | TDD 사이클, 파일 직접 수정, 리팩터링 | 초안 작성, 아이디어 탐색, 반복 테스트 |
| 컨텍스트 | 프로젝트 전체 파일 인식 | 요청별 독립 컨텍스트 |
| 에이전트 기능 | 전체 지원 | 미지원 |
| 비용 | Claude Pro 구독 소진 | 무료 (전기세만) |
| 응답 속도 | 네트워크 의존 | VRAM 용량에 따라 다름 |
| 외부 전송 | Anthropic 서버로 전송 | 로컬 처리, 외부 전송 없음 |
| 적합한 작업 | 복잡한 리팩터링, 테스트 생성, 코드 리뷰 | 민감 데이터 처리, 반복적 텍스트 생성, 빠른 초안 |

**기본 원칙:** Claude Code를 1차 도구로 사용하고, 비용 절감이 필요하거나 민감 데이터가 포함된 경우 Ollama로 분기한다.

---

## Ollama Windows 네이티브 설치

WSL 내부 설치 대비 Windows 네이티브 설치를 권장한다. WSL에서 `localhost:11434`로 접근이 가능하고, 성능 차이는 5% 이내다.

### 설치

1. [https://ollama.com](https://ollama.com) 에서 "Download for Windows" 클릭
2. `OllamaSetup.exe` 실행
3. 설치 완료 후 시스템 트레이에 Ollama 아이콘 확인

설치 후 Ollama는 Windows 서비스로 자동 시작된다.

### 서비스 상태 확인

```powershell
# Check if Ollama is running
Invoke-RestMethod -Uri "http://localhost:11434/api/tags"
```

예상 출력 (모델 미설치 시):

```json
{"models":[]}
```

또는:

```powershell
ollama list
```

---

## GTX 1080 Ti CUDA 설정 (Windows)

### NVIDIA 드라이버 확인

```powershell
nvidia-smi
```

예상 출력:

```
+-----------------------------------------------------------------------------------------+
| NVIDIA-SMI 5xx.xx          Driver Version: 5xx.xx    CUDA Version: 12.x               |
|-------------------------------+----------------------+----------------------+
| GPU  Name                Persistence-M | Bus-Id      Disp.A | Volatile Uncorr. ECC |
| Fan  Temp   Perf          Pwr:Usage/Cap |         Memory-Usage | GPU-Util  Compute M. |
|=========================|======================|======================|
|   0  NVIDIA GeForce GTX 1080 Ti  Off  | 00000000:01:00.0  On |                  N/A |
| 23%   38C    P8              9W / 250W |    500MiB / 11264MiB |      0%      Default |
+-----------------------------------------------------------------------------------------+
```

CUDA Version이 12.x 이상으로 표시되면 Ollama가 GPU를 자동으로 사용한다.

### CUDA Toolkit 설치 (선택)

Ollama 자체는 별도 CUDA Toolkit 설치 없이 GPU를 사용한다. Python 코드에서 CUDA를 직접 사용할 때만 설치가 필요하다.

설치가 필요한 경우: [https://developer.nvidia.com/cuda-downloads](https://developer.nvidia.com/cuda-downloads) 에서 Windows용 설치.

### Ollama GPU 인식 확인

모델 실행 중 `nvidia-smi`로 VRAM 사용량이 증가하면 GPU를 사용하는 것이다:

```powershell
# In a separate PowerShell window, watch GPU usage
while ($true) { nvidia-smi; Start-Sleep 2; Clear-Host }
```

---

## 권장 모델 (VRAM 10GB 기준)

GTX 1080 Ti는 VRAM 11GB이므로 아래 모델을 실행할 수 있다.

| 모델 | VRAM 사용량 | 용도 | 권장도 |
|---|---|---|---|
| `qwen2.5-coder:14b` | ~9GB | 코딩 특화, 고품질 | 1순위 |
| `qwen2.5-coder:7b` | ~5GB | 코딩 특화, 빠름 | 2순위 |
| `qwen2.5:7b` | ~5GB | 범용 텍스트 | 보조 |
| `deepseek-coder-v2:16b` | ~10GB | 코딩 특화 | VRAM 여유 시 |

### 모델 다운로드

```powershell
# Download coding model (primary)
ollama pull qwen2.5-coder:14b

# Download smaller model (faster)
ollama pull qwen2.5-coder:7b
```

다운로드 진행 상황이 터미널에 표시된다.

### 실행 확인

```powershell
# Test model response
ollama run qwen2.5-coder:7b "Write a Python function to compute factorial."
```

API로 확인:

```powershell
$body = @{
    model = "qwen2.5-coder:7b"
    prompt = "Write hello world in Python."
    stream = $false
} | ConvertTo-Json

Invoke-RestMethod -Uri "http://localhost:11434/api/generate" `
    -Method POST `
    -ContentType "application/json" `
    -Body $body
```

---

## WSL -> Windows Ollama 연결

WSL 내 Claude Code 또는 Python 코드에서 Windows에 설치된 Ollama에 접근한다.

### 접근 방식

WSL2는 기본적으로 Windows 호스트의 `localhost`를 `host.docker.internal` 또는 WSL의 `$(cat /etc/resolv.conf | grep nameserver | awk '{print $2}')` IP로 접근한다.

더 간단한 방법으로 Windows Ollama 설정에서 모든 인터페이스 바인딩을 설정한다.

### Windows 환경 변수 설정

```powershell
# Set Ollama to listen on all interfaces
[System.Environment]::SetEnvironmentVariable("OLLAMA_HOST", "0.0.0.0", "Machine")
```

설정 후 Ollama 서비스 재시작:

```powershell
# Restart Ollama service
Stop-Process -Name "ollama" -Force -ErrorAction SilentlyContinue
Start-Sleep 2
# Re-open Ollama from Start menu or tray
```

### .wslconfig 설정 (선택)

WSL2 네트워크 모드를 mirrored로 설정하면 `localhost`로 Windows 서비스에 직접 접근 가능하다.

`C:\Users\<username>\.wslconfig` 파일 생성 또는 수정:

```ini
[wsl2]
memory=16GB
processors=8

[experimental]
networkingMode=mirrored
```

설정 적용:

```powershell
wsl --shutdown
# Then restart WSL
```

### WSL에서 연결 확인

```bash
# Inside WSL terminal
curl http://localhost:11434/api/tags
```

응답이 오면 연결 성공.

---

## 앱 코드 분기 처리 (Python 예제)

`.env` 파일의 `LLM_PROVIDER` 값으로 Claude Code API와 Ollama를 전환한다.

### .env 설정

```
# LLM provider: claude | ollama
LLM_PROVIDER=ollama

# Anthropic
ANTHROPIC_API_KEY=sk-ant-...

# Ollama
OLLAMA_HOST=http://localhost:11434
OLLAMA_MODEL=qwen2.5-coder:7b
```

### Python 분기 코드

```python
# src/llm_client.py

import os
import json
import urllib.request


def _call_ollama(prompt: str) -> str:
    """Send a prompt to the local Ollama server and return the response."""
    host = os.environ.get("OLLAMA_HOST", "http://localhost:11434")
    model = os.environ.get("OLLAMA_MODEL", "qwen2.5-coder:7b")

    payload = json.dumps({
        "model": model,
        "prompt": prompt,
        "stream": False
    }).encode("utf-8")

    req = urllib.request.Request(
        url=os.path.join(host, "api", "generate"),
        data=payload,
        headers={"Content-Type": "application/json"},
        method="POST"
    )

    with urllib.request.urlopen(req) as resp:
        result = json.loads(resp.read().decode("utf-8"))
        return result["response"]


def _call_claude(prompt: str) -> str:
    """Send a prompt to the Anthropic API and return the response."""
    import anthropic  # pip install anthropic

    client = anthropic.Anthropic(
        api_key=os.environ.get("ANTHROPIC_API_KEY")
    )
    message = client.messages.create(
        model="claude-opus-4-5",
        max_tokens=1024,
        messages=[{"role": "user", "content": prompt}]
    )
    return message.content[0].text


def call_llm(prompt: str) -> str:
    """
    Route the prompt to the configured LLM provider.
    Provider is determined by LLM_PROVIDER env variable.
    """
    provider = os.environ.get("LLM_PROVIDER", "claude")

    if provider == "ollama":
        return _call_ollama(prompt)
    elif provider == "claude":
        return _call_claude(prompt)
    else:
        raise ValueError(f"Unknown LLM_PROVIDER: {provider}")
```

### 사용 예시

```python
# Load .env before calling
import os
from dotenv import load_dotenv  # pip install python-dotenv
load_dotenv()

from src.llm_client import call_llm

result = call_llm("Explain the difference between list and tuple in Python.")
print(result)
```

---

## .env 설정 (전체)

```
# ============================
# LLM Configuration
# ============================

# Provider selection: claude | ollama
LLM_PROVIDER=claude

# Anthropic Claude API
ANTHROPIC_API_KEY=sk-ant-...

# Ollama (Windows native, accessible from WSL via localhost)
OLLAMA_HOST=http://localhost:11434
OLLAMA_MODEL=qwen2.5-coder:7b

# ============================
# Project Settings
# ============================

# Project root (os.path style)
PROJECT_ROOT=C:\projects\my-project
```

`.env` 파일은 절대 커밋하지 않는다. `.gitignore`에 포함되어 있어야 한다.

---

## GPU 사용률 확인

### 실시간 모니터링

```powershell
# One-time snapshot
nvidia-smi

# Watch every 2 seconds
while ($true) {
    nvidia-smi --query-gpu=name,temperature.gpu,utilization.gpu,memory.used,memory.total --format=csv,noheader
    Start-Sleep 2
}
```

예상 출력 (모델 추론 중):

```
NVIDIA GeForce GTX 1080 Ti, 65, 98, 9200 MiB, 11264 MiB
```

`utilization.gpu`가 높고 `memory.used`가 모델 크기와 일치하면 GPU가 정상 동작 중이다.

### Python에서 GPU 확인

```python
# Verify Ollama GPU usage from Python
import urllib.request
import json

def get_ollama_gpu_info():
    """Check if Ollama is accessible and list loaded models."""
    req = urllib.request.Request("http://localhost:11434/api/tags")
    with urllib.request.urlopen(req) as resp:
        data = json.loads(resp.read().decode("utf-8"))
    return [m["name"] for m in data.get("models", [])]

print(get_ollama_gpu_info())
```

---

## 설치 확인 체크리스트

- [ ] `nvidia-smi` CUDA Version 12.x 이상 확인
- [ ] Ollama 시스템 트레이 아이콘 확인
- [ ] `Invoke-RestMethod -Uri "http://localhost:11434/api/tags"` 응답 확인
- [ ] `ollama pull qwen2.5-coder:7b` 다운로드 완료
- [ ] `ollama run qwen2.5-coder:7b "hello"` 응답 확인
- [ ] 모델 실행 중 `nvidia-smi` VRAM 사용량 증가 확인 (GPU 동작 검증)
- [ ] `.env` 파일 `LLM_PROVIDER`, `OLLAMA_HOST`, `OLLAMA_MODEL` 설정 확인
- [ ] Python `call_llm()` 함수 Ollama 응답 확인
- [ ] `.env` 파일이 `git status`에 표시되지 않는지 확인

---

## 공통 오류 및 해결

| 오류 | 원인 | 해결 |
|---|---|---|
| `http://localhost:11434` 연결 거부 | Ollama 서비스 미실행 | 시스템 트레이 Ollama 아이콘 확인 또는 `OllamaSetup.exe` 재실행 |
| GPU 사용 없이 CPU로만 실행 | NVIDIA 드라이버 버전 낮음 또는 CUDA 미인식 | `nvidia-smi` 실행 확인 후 드라이버 업데이트 (최신 Game Ready 또는 Studio 드라이버) |
| `ollama pull` 속도 매우 느림 | 네트워크 문제 또는 모델 크기 큼 | 14b 대신 7b 모델 먼저 테스트 |
| VRAM 부족 오류 | 모델이 VRAM 용량 초과 | `qwen2.5-coder:7b` (5GB) 로 전환, 또는 `OLLAMA_NUM_GPU=0`으로 CPU 실행 |
| WSL에서 `localhost:11434` 접근 불가 | .wslconfig networkingMode 미설정 | `.wslconfig`에 `networkingMode=mirrored` 추가 후 `wsl --shutdown` |
| `ModuleNotFoundError: anthropic` | anthropic 패키지 미설치 | `pip install anthropic` |
| `.env` 값이 Python에서 None | `load_dotenv()` 호출 누락 | `from dotenv import load_dotenv; load_dotenv()` 코드 최상단에 추가 |
| 모델 응답이 영어로만 출력 | 모델 기본 동작 | 프롬프트에 언어 명시: "Respond in Korean." 추가 |
