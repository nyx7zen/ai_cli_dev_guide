# MCP 통합

## Overview

- 이 문서는 Claude Code에서 MCP 서버를 설정하고 사용하는 방법을 다룬다.
- Sequential Thinking, Context7, GitHub MCP 세 가지 서버의 설치와 사용법을 포함한다.
- Windows와 WSL 환경 모두에 적용된다.

**사전 조건:**
- Claude Code가 설치되어 있다
- Node.js / npm이 설치되어 있다
- `.claude/` 폴더 구조가 생성되어 있다

---

## MCP 개념과 동작 방식

MCP(Model Context Protocol)는 Claude Code와 외부 도구/서비스를 연결하는 표준 프로토콜이다.

**동작 흐름:**
```
Claude Code (MCP Host)
  └── MCP Client
        └── MCP Server (로컬 프로세스 또는 원격 서비스)
              └── 외부 도구 / 데이터
```

**MCP가 필요한 경우:**
- 단계별 복잡한 추론이 필요한 작업 (Sequential Thinking)
- 라이브러리 최신 문서를 실시간으로 확인해야 하는 작업 (Context7)
- GitHub 이슈, PR, 저장소를 Claude Code에서 직접 조작해야 하는 작업 (GitHub MCP)

**MCP가 불필요한 경우:**
- 단순 코드 생성 및 수정
- 로컬 파일만 다루는 작업
- 이미 알고 있는 라이브러리 버전을 사용하는 작업

---

## .claude/.mcp.json 설정

MCP 서버 설정 파일이다. 프로젝트별로 관리하며 Git으로 공유한다.

**기본 구조:**
```json
{
  "mcpServers": {
    "server-name": {
      "command": "npx",
      "args": ["-y", "package-name"],
      "env": {
        "ENV_VAR": "value"
      }
    }
  }
}
```

**세 가지 서버를 모두 등록한 예시:**
```json
{
  "mcpServers": {
    "sequential-thinking": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-sequential-thinking"]
    },
    "context7": {
      "command": "npx",
      "args": ["-y", "@upstash/context7-mcp"]
    },
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_PERSONAL_ACCESS_TOKEN": "${GITHUB_TOKEN}"
      }
    }
  }
}
```

**환경 변수 참조:**
`.mcp.json`에 토큰을 직접 입력하지 않는다. `.env` 파일에 등록하고 `${변수명}` 형식으로 참조한다.

```bash
# .env
GITHUB_TOKEN=ghp_xxxxxxxxxxxxxxxxxxxx
```

---

## Sequential Thinking MCP

복잡한 문제를 단계별로 분해하여 추론하는 MCP 서버다. 설치 없이 `npx`로 실행된다.

**설정 (.claude/.mcp.json):**
```json
{
  "mcpServers": {
    "sequential-thinking": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-sequential-thinking"]
    }
  }
}
```

**적합한 작업:**
- 여러 파일에 걸친 복잡한 리팩터링 계획 수립
- 버그 원인 분석 (증상 → 원인 → 해결책 단계적 추론)
- 아키텍처 설계 결정

**사용 예시:**
```
"Sequential Thinking을 사용해서 이 버그의 원인을 단계적으로 분석해라.
 증상: pytest 실행 시 특정 조건에서만 assertion 실패
 관련 파일: src/calculator.py, tests/test_calculator.py"
```

**동작 확인:**
```bash
# Claude Code 실행 후 MCP 서버 목록 확인
/mcp
```
출력 예시:
```
Connected MCP servers:
  sequential-thinking  (running)
```

---

## Context7 MCP

라이브러리의 최신 공식 문서를 실시간으로 조회하는 MCP 서버다. 구버전 API 사용을 방지한다.

**설정 (.claude/.mcp.json):**
```json
{
  "mcpServers": {
    "context7": {
      "command": "npx",
      "args": ["-y", "@upstash/context7-mcp"]
    }
  }
}
```

**적합한 작업:**
- 처음 사용하는 라이브러리의 API 확인
- 버전 업그레이드 후 변경된 API 확인
- pytest fixture, JAX API 등 자주 바뀌는 API 사용 시

**사용 예시:**
```
"Context7로 pytest 8.x의 fixture 사용법을 확인하고
 테스트 코드를 작성해라"
```

```
"Context7로 JAX 최신 버전의 jit 컴파일 방법을 확인해라"
```

**동작 확인:**
```bash
/mcp
```
출력 예시:
```
Connected MCP servers:
  context7  (running)
```

---

## GitHub MCP

GitHub 이슈, PR, 저장소를 Claude Code에서 직접 조작하는 MCP 서버다.

**GitHub Personal Access Token 발급:**
1. GitHub → Settings → Developer settings → Personal access tokens → Tokens (classic)
2. Generate new token 클릭
3. 권한 선택: `repo`, `read:org`
4. 발급된 토큰을 `.env`에 저장

```bash
# .env
GITHUB_TOKEN=ghp_xxxxxxxxxxxxxxxxxxxx
```

**설정 (.claude/.mcp.json):**
```json
{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_PERSONAL_ACCESS_TOKEN": "${GITHUB_TOKEN}"
      }
    }
  }
}
```

**적합한 작업:**
- 이슈 목록 조회 및 생성
- PR 생성 및 리뷰 요청
- 저장소 파일 조회

**사용 예시:**
```
"GitHub MCP로 현재 저장소의 open 이슈 목록을 가져와라"
```

```
"현재 브랜치의 변경사항을 기반으로 PR을 생성해라.
 제목과 설명은 커밋 메시지를 참고해라"
```

**동작 확인:**
```bash
/mcp
```
출력 예시:
```
Connected MCP servers:
  github  (running)
```

---

## MCP 서버 동작 확인 방법

**Claude Code 내에서 확인:**
```bash
# 연결된 MCP 서버 목록
/mcp
```

**개별 서버 테스트:**
```bash
# Sequential Thinking 테스트
"Sequential Thinking을 사용해서 1+1=2를 단계별로 설명해라"

# Context7 테스트
"Context7로 pytest 최신 버전을 확인해라"

# GitHub MCP 테스트
"GitHub MCP로 현재 저장소 이름을 확인해라"
```

**서버가 연결되지 않을 때 수동 실행 테스트 (Windows):**
```powershell
npx -y @modelcontextprotocol/server-sequential-thinking
```

**서버가 연결되지 않을 때 수동 실행 테스트 (WSL):**
```bash
npx -y @modelcontextprotocol/server-sequential-thinking
```
정상이면 서버가 실행되고 대기 상태가 된다. `Ctrl+C`로 종료한다.

---

## Verification Checklist

- `.claude/.mcp.json` 파일이 프로젝트 루트의 `.claude/` 폴더 안에 있다
- `.env`에 `GITHUB_TOKEN`이 등록되어 있다
- `.gitignore`에 `.env`가 등록되어 있다
- `/mcp` 명령으로 세 서버가 모두 `running` 상태인지 확인한다
- `.mcp.json`에 토큰이 직접 입력되어 있지 않다

---

## Common Errors and Solutions

| 오류 | 원인 | 해결 |
|---|---|---|
| `/mcp` 실행 시 서버가 보이지 않는다 | `.mcp.json` 경로 오류 | `.claude/.mcp.json` 위치 확인 |
| GitHub MCP 인증 실패 | 토큰 만료 또는 권한 부족 | GitHub에서 토큰 재발급, `repo` 권한 확인 |
| `npx` 명령을 찾을 수 없다 | Node.js 미설치 | Node.js LTS 설치 후 재시도 |
| Context7가 문서를 찾지 못한다 | 라이브러리 이름 오타 | 공식 패키지 이름 확인 후 재요청 |
| WSL에서 MCP 서버 연결 실패 | Node.js PATH 미설정 | `~/.bashrc`에 nvm PATH 설정 확인 |
