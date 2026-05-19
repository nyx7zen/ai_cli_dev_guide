# 용어 정리

## Overview

- 이 문서는 Claude Code CLI 워크플로로 전환하는 과정에서 자주 접하는 용어를 정리한 참조 문서다.
- 기술 배경이 있는 개발자를 기준으로 작성했으며, 이미 알고 있는 개념은 제외했다.
- 각 용어는 독립적으로 참조할 수 있도록 작성했다.

---

## CLI vs Web 기반 AI

| 항목 | Web 기반 AI | CLI 기반 AI |
|---|---|---|
| 사용 방식 | 브라우저에서 대화 | 터미널에서 명령 실행 |
| 파일 접근 | 직접 불가 (복사-붙여넣기) | 프로젝트 파일 직접 읽기/쓰기 |
| 컨텍스트 유지 | 대화창 내에서만 | 프로젝트 전체 파일 참조 가능 |
| 반복 작업 | 수동 | 자동화 가능 |
| 적합한 작업 | 질문, 초안 작성, 개념 설명 | TDD 사이클, 리팩터링, 파일 수정 |

**선택 기준:**
- 코드를 에디터에서 복사해 AI에 붙여넣고, 응답을 다시 에디터에 붙여넣는 패턴(하드코딩 패턴)을 반복하고 있다면 CLI로 전환할 시점이다.
- 단순 질의응답이나 문서 초안 작성은 웹 기반 AI가 더 빠르다.

---

## Claude Code

Anthropic이 제공하는 AI CLI 에이전트. 터미널에서 실행하며 프로젝트 파일을 직접 읽고 쓴다. 이 문서 시리즈에서 실제로 사용하는 도구다.

**Claude Code vs Gemini CLI vs Cursor AI 비교:**

| 항목 | Claude Code | Gemini CLI | Cursor AI |
|---|---|---|---|
| 제공사 | Anthropic | Google | Anysphere |
| 형태 | CLI 에이전트 (터미널) | CLI 에이전트 (터미널) | VSCode 기반 IDE |
| 파일 수정 | 터미널 또는 VSCode Extension | 터미널 | IDE 내 인라인 편집 |
| 컨텍스트 설정 | CLAUDE.md / .claude/ 폴더 | GEMINI.md | .cursorrules |
| 컨텍스트 창 | 200K 토큰 | 1M 토큰 | 모델에 따라 다름 |
| MCP 지원 | 있음 | 있음 | 있음 |
| 주요 강점 | TDD 사이클, 에이전트 위임, MCP 통합 | 대용량 컨텍스트, Google 서비스 연동 | 실시간 자동완성, 멀티파일 편집 |
| 가격 | 사용량 기반 | 무료 티어 있음 | 월정액 |

Claude Code는 IDE가 아니다. 에디터(VSCode)와 독립적으로 실행되는 에이전트이며, VSCode Extension을 통해 에디터 내에서도 사용할 수 있다.

이 문서 시리즈는 Claude Code만 다룬다. Gemini CLI와 Cursor AI는 참조 비교 목적으로만 언급한다.

---

## Gemini CLI

Google이 제공하는 AI CLI 에이전트. Claude Code와 동일한 CLI 방식으로 동작하며 터미널에서 프로젝트 파일을 직접 읽고 쓴다.

**특징:**
- 컨텍스트 창 1M 토큰 — 대용량 코드베이스를 한 번에 처리할 수 있다
- Google Drive, Gmail 등 Google 서비스와 연동
- 컨텍스트 설정 파일: `GEMINI.md` (Claude Code의 `CLAUDE.md`에 해당)

**이 문서에서의 위치:**
Gemini CLI는 이 문서 시리즈에서 다루지 않는다. 실제 사용 도구는 Claude Code다. 도구 선택 시 참조 비교 목적으로만 언급한다.

---

## MCP (Model Context Protocol)

AI 모델과 외부 도구/데이터 소스를 연결하는 표준 프로토콜. Anthropic이 제안했으며 오픈 표준으로 공개되어 있다.

**등장 배경:**
MCP 이전에는 AI 도구마다 GitHub, Slack, DB 등 외부 서비스와의 연결을 각각 별도로 구현해야 했다 (N개 AI x M개 서비스 = N*M개 커스텀 연결). MCP는 이 문제를 표준 프로토콜 하나로 해결한다.

**동작 방식:**
```
Claude Code (MCP Host)
  └── MCP Client (프로토콜 처리)
        └── MCP Server (GitHub / Sequential Thinking / Context7 등)
              └── 외부 데이터 / 도구
```

**Claude Code에서의 사용:**
`.claude/.mcp.json` 파일에 사용할 MCP 서버를 등록한다. 이 문서에서 다루는 MCP 서버는 Sequential Thinking, Context7, GitHub MCP 세 가지다.

---

## MCP vs agents

혼동하기 쉬운 두 개념이지만 역할이 다르다.

| 항목 | MCP | agents |
|---|---|---|
| 역할 | 외부 도구/서비스 연결 | 독립된 작업 단위 위임 |
| 실행 주체 | Claude Code가 MCP 서버를 호출 | Claude Code가 sub-agent를 실행 |
| 설정 위치 | `.claude/.mcp.json` | `.claude/agents/` |
| 대화 컨텍스트 | 메인 대화와 공유 | 격리된 독립 컨텍스트 |
| 예시 | GitHub에서 이슈 조회, 문서 최신 버전 확인 | 코드 리뷰 전담, 보안 감사 전담 |

MCP는 Claude Code가 외부 서비스와 통신하는 수단이고, agents는 특정 작업을 독립적으로 처리하는 역할 분리 수단이다.

---

## rules vs skills vs agents

`.claude/` 폴더 하위의 세 가지 구성 요소. 로딩 시점과 실행 방식이 다르다.

| 항목 | `.claude/rules/` | `.claude/skills/` | `.claude/agents/` |
|---|---|---|---|
| 로딩 시점 | 매 세션 (항상) | 필요 시 (선택적) | 명시적 위임 시 |
| 실행 컨텍스트 | 메인 대화 | 메인 대화 | 독립 컨텍스트 |
| 대화 공유 | 예 | 예 | 아니오 (격리) |
| 파일 형식 | 단일 `.md` | 폴더 + `SKILL.md` | 단일 `.md` |
| 적합한 내용 | 명명 규칙, 금지 패턴 | TDD 절차, 리팩터링 단계 | 코드 리뷰, 보안 감사 |

**사용 원칙:**
- 항상 적용해야 하는 제약 조건 → `rules/`
- 특정 작업에서만 참조하는 절차적 지식 → `skills/`
- 독립적으로 처리할 수 있는 전담 작업 → `agents/`

---

## CLAUDE.md / CLAUDE.local.md

**CLAUDE.md:**
Claude Code가 매 세션 시작 시 자동으로 읽는 프로젝트 컨텍스트 파일. 프로젝트 루트에 위치한다. Git으로 관리하며 팀원 전체가 공유한다.

포함할 내용: 프로젝트 개요, 기술 스택, 디렉터리 구조, 주요 명령어, 코딩 규칙.
포함하지 않을 내용: API 키, 개인 경로, 민감 정보.

200줄 이하로 유지하는 것을 권장한다. 길어질수록 Claude Code의 컨텍스트 창을 불필요하게 소모한다.

**CLAUDE.local.md:**
개인 설정용 파일. `.gitignore`에 등록해 원격 저장소에 올리지 않는다.

포함할 내용: 로컬 경로, 개인 선호 설정, 테스트용 API 키.

---

## Session

이 문서에서 "세션"은 기술 용어가 아니라 **하나의 작업 단위**를 의미한다. Claude Code를 실행하고 특정 목표(기능 구현, 버그 수정, 리팩터링 등)를 완료한 뒤 종료하는 한 사이클이다.

세션 단위로 다음 파일을 관리한다:
```
sessions/
  01_task-name/
    task.md     # 세션 시작 전 작성하는 작업 정의
    report.md   # 세션 종료 시 Claude Code가 생성하는 결과 보고서
    prompt.md   # /export 명령으로 내보낸 대화 로그
```

세션을 나누는 기준: 작업 목표가 바뀌거나, 컨텍스트가 너무 길어져 응답 품질이 저하될 때.

---

## TDD (Red / Green / Refactor)

Test-Driven Development. 테스트를 먼저 작성하고 구현 코드를 나중에 작성하는 개발 방식.

**사이클:**

```
Red      테스트 작성 → 실패 확인 (구현 없으므로 당연히 실패)
Green    최소한의 구현 코드 작성 → 테스트 통과 확인
Refactor 코드 개선 → 테스트가 여전히 통과하는지 확인
```

**AI CLI와의 통합 방식:**
- Red 단계: Claude Code에게 테스트 코드 작성 요청
- Green 단계: Claude Code에게 테스트를 통과하는 최소 구현 요청
- Refactor 단계: Claude Code에게 코드 개선 요청, 테스트 통과 유지 확인
- 테스트 통과마다 git commit

---

## Ollama

로컬 머신에서 LLM을 실행하는 런타임. 외부 API 없이 GPU(또는 CPU)를 사용해 모델을 구동한다.

**용도:**
- 외부로 전송하면 안 되는 민감 데이터 처리
- 반복적인 단순 작업 (API 비용 절감)
- 오프라인 환경에서의 작업

**Claude Code와의 관계:**
Ollama는 Claude Code를 대체하지 않는다. TDD 사이클, 파일 직접 수정, 리팩터링 등 에이전트 기능이 필요한 작업은 Claude Code를 사용하고, 단순 텍스트 생성이나 초안 작성은 Ollama로 처리하는 역할 분담이 권장된다.

---

## OpenRouter

여러 AI 모델(Claude, GPT, Gemini 등)을 단일 API 엔드포인트로 제공하는 서비스. 모델별 API 키를 각각 관리하지 않고 OpenRouter 키 하나로 여러 모델에 접근할 수 있다.

**무료 모델 제한사항:**
- 무료 티어 모델은 컨텍스트 창이 작고 속도가 느린 경우가 많다.
- 무료 모델은 요청이 몰릴 때 응답 지연이 발생할 수 있다.
- 상업적 프로젝트에 무료 모델을 사용할 경우 해당 모델의 라이선스를 별도로 확인해야 한다.

이 문서에서 OpenRouter는 직접 다루지 않는다. Ollama 로컬 모델 또는 Claude Code(Anthropic API)를 기본으로 사용한다.

---

## Local LLM

로컬 머신에서 직접 실행하는 언어 모델. Ollama가 대표적인 실행 런타임이다.

**Cloud LLM과의 비교:**

| 항목 | Local LLM (Ollama) | Cloud LLM (Claude Code) |
|---|---|---|
| 데이터 외부 전송 | 없음 | 있음 |
| 비용 | 전기료 외 무료 | 사용량 기반 과금 |
| 성능 | GPU VRAM에 제한됨 | 대형 모델 사용 가능 |
| 에이전트 기능 | 없음 | 있음 (파일 수정, MCP 등) |
| 적합한 작업 | 단순 생성, 민감 데이터 처리 | TDD, 리팩터링, 프로젝트 전체 작업 |

**GTX 1080 Ti (VRAM 10GB) 기준 권장 모델:**
- `qwen2.5-coder:14b` — VRAM 약 9GB, 코딩 특화
- `qwen2.5-coder:7b` — VRAM 약 5GB, 빠른 응답

---

## Node.js / npm

**Node.js:** JavaScript 런타임. 브라우저 밖에서 JavaScript를 실행하는 환경이다.

**npm:** Node.js 패키지 매니저. Node.js 설치 시 함께 설치된다.

**Claude Code에 필요한 이유:**
Claude Code는 npm 패키지로 배포된다. 설치 명령은 다음과 같다:
```bash
npm install -g @anthropic-ai/claude-code
```
Claude Code 자체가 Node.js 런타임 위에서 실행되므로 Node.js가 없으면 설치 및 실행이 불가능하다.

Python이나 C++ 개발자라도 Claude Code 사용을 위해 Node.js를 별도로 설치해야 한다. Python 개발 환경(WinPython, Anaconda)과는 독립적이다.

---

## nvm

Node Version Manager. 여러 버전의 Node.js를 설치하고 전환하는 도구.

**WSL에서 필요한 이유:**
WSL(Ubuntu)에서 `apt`로 Node.js를 설치하면 구버전이 설치되는 경우가 많다. Claude Code는 최신 LTS 버전을 요구하므로 nvm을 통해 원하는 버전을 설치하는 것이 권장된다.

**Windows에서는 불필요:**
Windows 환경에서는 nodejs.org에서 LTS 인스톨러를 직접 내려받아 설치하면 된다. nvm-windows라는 별도 도구가 있으나 이 문서에서는 다루지 않는다.

---

## WSL

Windows Subsystem for Linux. Windows 위에서 Linux(주로 Ubuntu) 환경을 실행하는 기능.

이 문서에서는 WSL이 이미 설치되어 있다고 가정한다. WSL 설치 방법은 다루지 않는다.

**이 문서에서 WSL을 사용하는 이유:**
PyTorch, JAX 등 딥러닝 프레임워크는 Linux 환경에서 더 안정적으로 동작한다. NVIDIA GPU를 WSL에서 사용하려면 Windows에 NVIDIA 드라이버가 설치되어 있어야 하며, WSL 내부에 별도의 Linux NVIDIA 드라이버를 설치해서는 안 된다.

---

## Vibe Coding

AI에게 구체적인 구현 지시 없이 의도와 결과만 설명하고, AI가 코드를 생성하도록 맡기는 방식. 프로토타이핑이나 아이디어 검증에 유용하다.

**이 문서에서의 입장:**
Vibe Coding은 빠른 초안 생성에는 효과적이지만, 프로덕션 코드 품질 관리에는 TDD 기반 워크플로가 더 적합하다. 이 문서 시리즈는 Vibe Coding이 아닌 TDD 통합 CLI 워크플로를 다룬다.

---

## Verification Checklist

- Claude Code를 처음 실행했을 때 "하드코딩 패턴을 쓰고 있는가?" 질문에 답할 수 있다
- MCP와 agents의 차이를 설명할 수 있다
- rules / skills / agents 중 어디에 무엇을 넣을지 판단할 수 있다
- CLAUDE.md에 API 키를 넣지 않는 이유를 설명할 수 있다
- WSL 환경에서 nvm이 필요한 이유를 설명할 수 있다

---

## Common Errors and Solutions

| 오류 | 원인 | 해결 |
|---|---|---|
| "Claude Code가 무엇인지 모르겠다" | Cursor AI나 GitHub Copilot과 혼동 | 이 문서의 "Claude Code" 항목 참조 |
| "MCP 서버와 agents를 언제 쓰는지 모르겠다" | 역할 구분 불명확 | "MCP vs agents" 항목 참조 |
| "CLAUDE.md가 너무 길어지고 있다" | 모든 정보를 CLAUDE.md에 넣으려 함 | rules / skills 로 분산, 200줄 이하 유지 |
| "WSL에서 Node.js 버전이 너무 낮다" | apt 기본 저장소의 구버전 설치 | nvm으로 재설치 (02_setup_wsl.md 참조) |
