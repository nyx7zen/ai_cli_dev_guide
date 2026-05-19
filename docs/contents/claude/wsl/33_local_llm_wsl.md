# WSL 로컬 LLM 설정 (Ollama)

## Overview

- **목적**: WSL2 (Ubuntu) 환경에서 Ollama 로컬 LLM을 Claude Code와 병행 사용하는 설정 및 운용 방법
- **사전 조건**: `02_setup_wsl.md` 환경 설정 완료, NVIDIA GTX 1080 Ti, Windows 측 NVIDIA 드라이버 설치

---

## 역할 분담 (Claude Code vs Ollama)

두 도구를 작업 유형에 따라 분리하여 사용한다.

| 작업 유형 | 도구 | 이유 |
|---|---|---|
| TDD 사이클 (Red/Green/Refactor) | Claude Code | 파일 직접 수정, 전체 프로젝트 컨텍스트 |
| 리팩토링 | Claude Code | 다중 파일 변경, 코드 품질 판단 |
| GitHub 작업 (PR, issue) | Claude Code + GitHub MCP | 에이전트 기능 필요 |
| 초안 작성 / 아이디어 탐색 | Ollama | 반복 실험, 비용 없음 |
| 민감 데이터 처리 | Ollama | 외부 전송 없음 |
| 빠른 문법 확인 | Ollama | 지연 없음, 비용 없음 |
| 반복 테스트 (프롬프트 실험) | Ollama | API 비용 절감 |

---

## Option A: Windows Ollama + WSL 연결 (권장)

Windows 측에서 Ollama를 실행하고 WSL에서 `localhost:11434`로 접근한다.
드라이버 충돌 위험이 없고 설정이 단순하다.

### Windows 측 Ollama 설치

Windows PowerShell에서 설치한다.

```powershell
winget install Ollama.Ollama
```

또는 https://ollama.com/download 에서 설치 파일을 직접 다운로드하여 실행한다.

설치 후 확인:

```powershell
ollama --version
# Expected: ollama version 0.x.x
```

### .wslconfig networkingMode 설정

`C:\Users\username\.wslconfig` 파일을 수정한다.

```powershell
notepad $env:USERPROFILE\.wslconfig
```

아래 내용을 추가 또는 확인한다.

```ini
[wsl2]
memory=16GB
processors=8

# Required for localhost forwarding
localhostForwarding=true
```

`networkingMode=mirrored`를 사용하는 경우 localhost 접근이 더 단순해진다.

```ini
[wsl2]
networkingMode=mirrored
memory=16GB
processors=8
```

설정 적용:

```powershell
wsl --shutdown
wsl
```

### WSL에서 localhost:11434 접근 확인

Windows에서 Ollama를 실행한 상태에서 WSL 터미널에서 확인한다.

```bash
# Windows에서 먼저 모델을 실행한다 (PowerShell)
# ollama run qwen2.5-coder:7b

# WSL에서 접근 테스트
curl http://localhost:11434/api/tags
# Expected: {"models":[...]}

# 또는
curl http://localhost:11434
# Expected: Ollama is running
```

접근이 안되면 Windows 방화벽에서 포트 11434를 허용한다.

```powershell
# PowerShell (관리자)
New-NetFirewallRule -DisplayName "Ollama WSL" `
  -Direction Inbound `
  -Protocol TCP `
  -LocalPort 11434 `
  -Action Allow
```

### 장점 및 주의사항

장점:
- WSL 내부에 CUDA/드라이버를 별도 설치할 필요 없음
- GPU 성능이 Windows 네이티브와 동일
- Ollama 업데이트가 Windows 단일 경로에서 처리됨

주의사항:
- Windows Ollama 서비스가 실행 중이어야 WSL에서 접근 가능
- WSL 재시작 후 localhost 접근이 안되면 `.wslconfig` `localhostForwarding=true` 확인

---

## Option B: WSL 직접 Ollama 설치

WSL 내부에서 직접 Ollama를 실행한다. GPU 드라이버 설정에 주의가 필요하다.

### NVIDIA 드라이버 주의사항

**WSL 내부에서 Linux용 NVIDIA 드라이버를 설치하면 안 된다.**

WSL2는 Windows 측 NVIDIA 드라이버를 그대로 사용한다.
WSL 내부에서 `sudo apt install nvidia-driver-xxx` 또는 NVIDIA 공식 사이트의 `.run` 파일을 실행하면 WSL GPU 기능이 파괴된다.

Windows 측 드라이버 최신 버전 설치 여부만 확인한다.

```bash
# WSL에서 GPU 인식 확인 (드라이버 설치 없이 동작해야 함)
nvidia-smi
# Expected: GTX 1080 Ti 정보 출력
# 오류 시: Windows 측 NVIDIA 드라이버 업데이트 필요
```

### CUDA 툴킷 설치 주의사항

`cuda-toolkit-12-x` 패키지만 설치한다.
`cuda-drivers` 또는 `cuda` 메타패키지는 드라이버를 포함하므로 설치하지 않는다.

```bash
# Install CUDA keyring
wget https://developer.download.nvidia.com/compute/cuda/repos/ubuntu2204/x86_64/cuda-keyring_1.1-1_all.deb
sudo dpkg -i cuda-keyring_1.1-1_all.deb
sudo apt update

# Install CUDA toolkit only (NOT cuda-drivers)
sudo apt install cuda-toolkit-12-4 -y

# Verify
nvcc --version
# Expected: Cuda compilation tools, release 12.4
```

PATH 설정:

```bash
echo 'export PATH=/usr/local/cuda/bin:$PATH' >> ~/.bashrc
echo 'export LD_LIBRARY_PATH=/usr/local/cuda/lib64:$LD_LIBRARY_PATH' >> ~/.bashrc
source ~/.bashrc
```

### Ollama WSL 직접 설치

```bash
curl -fsSL https://ollama.com/install.sh | sh
```

### systemd 서비스 설정

WSL2에서 systemd를 사용하려면 `/etc/wsl.conf`에서 활성화해야 한다.

```bash
cat /etc/wsl.conf
# 출력에 systemd=true가 없으면 추가
```

```bash
sudo tee /etc/wsl.conf << 'EOF'
[boot]
systemd=true
EOF
```

WSL 재시작 후:

```powershell
# PowerShell
wsl --shutdown
wsl
```

Ollama 서비스 등록:

```bash
sudo systemctl enable ollama
sudo systemctl start ollama
sudo systemctl status ollama
# Expected: active (running)
```

### GPU 인식 확인

```bash
ollama run qwen2.5-coder:7b
# Ollama 실행 후 별도 터미널에서
nvidia-smi
# Expected: ollama 프로세스가 GPU 메모리를 사용 중
```

---

## GTX 1080 Ti 권장 모델

VRAM 10GB (실제 사용 가능 ~9.5GB) 기준이다.

| 모델 | VRAM 사용량 | 용도 | 다운로드 명령 |
|---|---|---|---|
| qwen2.5-coder:14b | ~9GB | 코딩 특화, 고품질 | `ollama pull qwen2.5-coder:14b` |
| qwen2.5-coder:7b | ~5GB | 코딩 특화, 빠름 | `ollama pull qwen2.5-coder:7b` |
| llama3.1:8b | ~5GB | 범용, 균형 | `ollama pull llama3.1:8b` |
| mistral:7b | ~4.5GB | 범용, 빠름 | `ollama pull mistral:7b` |
| deepseek-coder:6.7b | ~4GB | 코딩 특화, 경량 | `ollama pull deepseek-coder:6.7b` |

`qwen2.5-coder:14b`를 기본으로 사용하고 VRAM이 부족할 때 `7b`로 전환한다.

### 모델 다운로드 및 실행 확인

```bash
# Download model
ollama pull qwen2.5-coder:7b

# Run test
ollama run qwen2.5-coder:7b "Write a Python function to calculate fibonacci numbers"

# List downloaded models
ollama list
```

---

## Anaconda 환경에서 Ollama 호출

### 필요 패키지 설치

```bash
conda activate project-env
pip install requests
```

### Python 예제 코드

```python
# src/llm_client.py
import os
import requests
import json

OLLAMA_HOST = os.environ.get("OLLAMA_HOST", "http://localhost:11434")

def generate(prompt: str, model: str = "qwen2.5-coder:7b") -> str:
    """Call Ollama API and return generated text."""
    url = f"{OLLAMA_HOST}/api/generate"
    payload = {
        "model": model,
        "prompt": prompt,
        "stream": False
    }
    response = requests.post(url, json=payload, timeout=120)
    response.raise_for_status()
    return response.json()["response"]

def chat(messages: list[dict], model: str = "qwen2.5-coder:7b") -> str:
    """Call Ollama chat API and return assistant message."""
    url = f"{OLLAMA_HOST}/api/chat"
    payload = {
        "model": model,
        "messages": messages,
        "stream": False
    }
    response = requests.post(url, json=payload, timeout=120)
    response.raise_for_status()
    return response.json()["message"]["content"]

if __name__ == "__main__":
    result = generate("Write a Python hello world function.")
    print(result)
```

### PyTorch와 병행 사용 예

```python
# src/pipeline.py
import torch
import os
import requests

OLLAMA_HOST = os.environ.get("OLLAMA_HOST", "http://localhost:11434")

def generate_ollama(prompt: str) -> str:
    """Generate text using Ollama local LLM."""
    response = requests.post(
        f"{OLLAMA_HOST}/api/generate",
        json={"model": "qwen2.5-coder:7b", "prompt": prompt, "stream": False},
        timeout=120
    )
    return response.json()["response"]

def run_model(input_tensor: torch.Tensor) -> torch.Tensor:
    """Run PyTorch model inference."""
    device = torch.device("cuda" if torch.cuda.is_available() else "cpu")
    input_tensor = input_tensor.to(device)
    # ... PyTorch model inference
    return input_tensor

def analyze_result(result_tensor: torch.Tensor) -> str:
    """Use Ollama to interpret model output."""
    stats = {
        "mean": float(result_tensor.mean()),
        "std": float(result_tensor.std()),
        "max": float(result_tensor.max()),
        "shape": list(result_tensor.shape)
    }
    prompt = f"Analyze these model output statistics and provide insights: {stats}"
    return generate_ollama(prompt)
```

---

## 앱 코드 분기 방법

환경 변수로 Claude Code API와 Ollama를 전환한다.

```python
# src/ai_provider.py
import os
import requests

AI_PROVIDER = os.environ.get("AI_PROVIDER", "ollama")
OLLAMA_HOST = os.environ.get("OLLAMA_HOST", "http://localhost:11434")
OLLAMA_MODEL = os.environ.get("OLLAMA_MODEL", "qwen2.5-coder:7b")

def generate_text(prompt: str) -> str:
    """Generate text using configured AI provider."""
    if AI_PROVIDER == "ollama":
        return _generate_ollama(prompt)
    elif AI_PROVIDER == "anthropic":
        return _generate_anthropic(prompt)
    else:
        raise ValueError(f"Unknown AI_PROVIDER: {AI_PROVIDER}")

def _generate_ollama(prompt: str) -> str:
    response = requests.post(
        f"{OLLAMA_HOST}/api/generate",
        json={"model": OLLAMA_MODEL, "prompt": prompt, "stream": False},
        timeout=120
    )
    response.raise_for_status()
    return response.json()["response"]

def _generate_anthropic(prompt: str) -> str:
    import anthropic
    client = anthropic.Anthropic(api_key=os.environ["ANTHROPIC_API_KEY"])
    message = client.messages.create(
        model="claude-opus-4-5",
        max_tokens=1024,
        messages=[{"role": "user", "content": prompt}]
    )
    return message.content[0].text
```

사용 예:

```bash
# Ollama 사용 (기본)
AI_PROVIDER=ollama python src/pipeline.py

# Anthropic API 사용
AI_PROVIDER=anthropic python src/pipeline.py
```

---

## .env 설정 (WSL 경로)

```bash
cat > .env << 'EOF'
# AI Provider
AI_PROVIDER=ollama

# Ollama (Windows native 사용 시)
OLLAMA_HOST=http://localhost:11434
OLLAMA_MODEL=qwen2.5-coder:7b

# Anthropic (Claude Code 사용 시)
ANTHROPIC_API_KEY=your-api-key-here

# CUDA
CUDA_VISIBLE_DEVICES=0

# Project
PROJECT_ENV=development
EOF
```

Python에서 `.env` 파일 로딩:

```bash
pip install python-dotenv
```

```python
# 파일 상단에 추가
from dotenv import load_dotenv
load_dotenv()
```

---

## GPU 활용 확인

### nvidia-smi 기본 확인

```bash
nvidia-smi
```

예상 출력:

```
+-----------------------------------------------------------------------------+
| NVIDIA-SMI 525.xx.xx    Driver Version: 525.xx.xx    CUDA Version: 12.0    |
|-------------------------------+----------------------+----------------------+
| GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp  Perf  Pwr:Usage/Cap|         Memory-Usage | GPU-Util  Compute M. |
|===============================+======================+======================|
|   0  NVIDIA GeForce ...  Off  | 00000000:01:00.0  On |                  N/A |
| 28%   45C    P8    16W / 250W |  9200MiB / 11264MiB  |      0%      Default |
+-----------------------------------------------------------------------------+
```

### watch 명령으로 실시간 모니터링

```bash
watch -n 1 nvidia-smi
```

Ctrl+C로 종료한다.

### GPU 메모리 사용량만 확인

```bash
nvidia-smi --query-gpu=memory.used,memory.free,memory.total \
  --format=csv,noheader,nounits
# Expected: 9200, 2064, 11264
```

### Ollama 실행 중 GPU 확인

```bash
# Terminal 1: Ollama 요청 실행
ollama run qwen2.5-coder:7b "Explain quicksort algorithm"

# Terminal 2: GPU 모니터링
nvidia-smi
# GPU-Util이 0% 이상이면 GPU 사용 중
```

---

## 공통 오류 및 해결 방법

| 오류 | 원인 | 해결 방법 |
|---|---|---|
| `nvidia-smi: command not found` | Windows 드라이버 미설치 또는 구버전 | Windows NVIDIA 드라이버 최신 버전 설치 후 WSL 재시작 |
| `curl localhost:11434` 연결 거부 | Windows Ollama 미실행 | Windows에서 Ollama 앱 실행 또는 `ollama serve` |
| `curl localhost:11434` 타임아웃 | `.wslconfig` localhostForwarding 미설정 | `.wslconfig`에 `localhostForwarding=true` 추가 후 `wsl --shutdown` |
| Ollama GPU 사용 안 됨 (Option B) | CUDA 미설치 또는 환경변수 오류 | `nvcc --version` 확인, PATH 재설정 |
| `cuda-drivers` 설치 후 nvidia-smi 오류 | WSL에서 금지된 패키지 설치 | WSL 인스턴스 재설치 필요 (`wsl --unregister Ubuntu`) |
| `ollama pull` 속도 느림 | 네트워크 대역폭 제한 | 백그라운드에서 실행: `nohup ollama pull model &` |
| Python `requests.exceptions.ConnectionError` | Ollama 서비스 중단 | `systemctl start ollama` 또는 Windows Ollama 재시작 |
| PyTorch와 Ollama 동시 실행 시 VRAM 부족 | 두 프로세스가 GPU 공유 | 경량 모델(7b)으로 전환 또는 PyTorch 프로세스 종료 후 Ollama 실행 |
| `ANTHROPIC_API_KEY` 미인식 | `.env` 로딩 누락 | `from dotenv import load_dotenv; load_dotenv()` 추가 |
