# 개발 환경 개요

## Overview

- 이 문서는 Claude Code CLI 워크플로를 시작하기 전에 전체 환경 구성을 파악하기 위한 참조 문서다.
- Windows와 WSL 두 환경의 도구 구성, 역할 분담, 폴더 구조를 한 곳에서 확인할 수 있다.
- 각 환경의 설치 절차는 `02_setup_win.md` 및 `02_setup_wsl.md`를 참조한다.

**사전 조건:**
- Windows 10/11 환경에 VSCode, Git이 설치되어 있다
- WSL2(Ubuntu) 환경에 Anaconda, VSCode Remote-WSL, Git이 설치되어 있다
- GitHub 계정이 있다

---

## 사전 설치 도구

두 환경 모두 아래 도구가 이미 설치되어 있다고 가정한다.

| 도구 | Windows | WSL (Ubuntu) | 용도 |
|---|---|---|---|
| VSCode | 설치됨 | Remote-WSL 연결 | 코드 에디터 |
| Git | 설치됨 | 설치됨 | 버전 관리 |
| GitHub 계정 | 있음 | 있음 | 원격 저장소 |
| Python | WinPython (portable) | Anaconda | 언어 런타임 |
| C++ 컴파일러 | MinGW64 | build-essential (gcc) | 언어 런타임 |
| GPU | 사용 안함 (CPU only) | NVIDIA GTX 1080 Ti | 딥러닝 연산 |

---

## 추가 설치 도구와 역할

이 문서 시리즈에서 새로 설치하는 도구 목록이다.

| 도구 | 설치 환경 | 역할 |
|---|---|---|
| Node.js / npm | Windows, WSL | Claude Code 실행 런타임 |
| Claude Code | Windows, WSL | AI CLI 에이전트 (핵심 도구) |
| VSCode Claude Code Extension | Windows, WSL | VSCode 내 Claude Code GUI |
| GitHub CLI (gh) | Windows, WSL | 터미널에서 GitHub 작업 |
| Ollama | Windows (네이티브) | 로컬 LLM 런타임 |
| MCP 서버 | Windows, WSL | Claude Code 외부 도구 연동 |

**Ollama 설치 위치:**
Ollama는 Windows 네이티브로 설치하는 것을 권장한다. WSL에서 `localhost:11434`로 접근할 수 있으며 성능 차이는 5% 이내다.

---

## 하드코딩 패턴 vs CLI 패턴

**하드코딩 패턴 (현재):**
```
에디터에서 코드 복사
  -> 브라우저 AI(Claude.ai / Gemini)에 붙여넣기
  -> 응답 복사
  -> 에디터에 붙여넣기
  -> 반복
```

**CLI 패턴 (전환 후):**
```
Claude Code가 프로젝트 파일 직접 읽기
  -> TDD 사이클 진행 (테스트 작성 -> 구현 -> 리팩터링)
  -> 파일 직접 수정
  -> git commit
  -> 반복
```

| 항목 | 하드코딩 패턴 | CLI 패턴 |
|---|---|---|
| 파일 접근 | 수동 복사-붙여넣기 | 직접 읽기/쓰기 |
| 컨텍스트 범위 | 붙여넣은 코드만 | 프로젝트 전체 |
| 반복 작업 | 매번 수동 | 자동화 가능 |
| Git 연동 | 수동 | 세션 내 자동 커밋 가능 |
| 작업 기록 | 없음 | sessions/ 폴더에 누적 |

**전환 기준:**
같은 코드를 두 번 이상 복사해 AI에 붙여넣고 있다면 CLI 패턴으로 전환할 시점이다.

---

## Windows vs WSL 선택 기준

두 환경은 독립적으로 운영한다. 작업 목적에 따라 환경을 선택한다.

| 항목 | Windows | WSL (Ubuntu) |
|---|---|---|
| Python 환경 | WinPython (portable) | Anaconda |
| C++ 환경 | MinGW64 | gcc / build-essential |
| GPU 사용 | 사용 안함 (CPU only) | NVIDIA GTX 1080 Ti |
| 주요 용도 | NumPy, Matplotlib, C++ 코딩 테스트 | PyTorch, JAX, 딥러닝 |
| 셸 | PowerShell | bash |
| Ollama 접근 | 직접 실행 | localhost:11434 (Windows 네이티브 연결) |

**선택 원칙:**
- PyTorch / JAX / GPU 연산이 필요한 작업 → WSL
- NumPy / Matplotlib / C++ 알고리즘 작업 → Windows
- 두 환경 간 파일 공유는 `/mnt/c/` 경로를 통해 가능하지만 WSL 홈 디렉터리(`~/`)에서 작업하는 것을 권장한다

---

## Claude Code vs Ollama 역할 분담

| 항목 | Claude Code | Ollama |
|---|---|---|
| 실행 위치 | Anthropic 서버 | 로컬 (GTX 1080 Ti) |
| 파일 직접 수정 | 가능 | 불가 |
| TDD 사이클 | 지원 | 미지원 |
| MCP 통합 | 지원 | 미지원 |
| 프로젝트 컨텍스트 | 전체 참조 | 요청 내용만 |
| 데이터 외부 전송 | 있음 | 없음 |
| 적합한 작업 | TDD, 리팩터링, 파일 수정, 코드 리뷰 | 초안 작성, 단순 생성, 민감 데이터 처리 |

**운영 원칙:**
Claude Code를 기본 도구로 사용한다. Ollama는 외부 전송이 불가한 민감 데이터 처리 또는 반복적인 단순 작업에만 사용한다.

---

## .claude/ 폴더 구조 개요

Claude Code 공식 폴더 구조다. 프로젝트 루트 하위에 위치한다.

```
.claude/
├── settings.json          # Claude Code 동작 설정
├── settings.local.json    # 개인 설정 (.gitignore 등록)
├── rules/                 # 매 세션 자동 로드되는 제약 조건
│   ├── conventions.md     # 명명 규칙, 금지 패턴
│   └── tdd.md             # TDD 적용 규칙
├── skills/                # 필요 시 선택적으로 참조하는 절차
│   ├── tdd/
│   │   └── SKILL.md       # TDD 단계별 절차
│   ├── refactoring/
│   │   └── SKILL.md
│   └── code-reviewer/
│       └── SKILL.md
├── agents/                # 독립 컨텍스트로 실행되는 전담 에이전트
│   ├── code_reviewer.md
│   └── tdd_agent.md
└── .mcp.json              # MCP 서버 설정
```

| 폴더 | 로드 시점 | 용도 |
|---|---|---|
| `rules/` | 매 세션 자동 | 항상 적용할 제약 조건 |
| `skills/` | 명시적 참조 시 | 작업별 절차적 지식 |
| `agents/` | 명시적 위임 시 | 독립 실행 전담 작업 |

---

## 전체 프로젝트 폴더 구조

```
my-project/
├── CLAUDE.md                  # 매 세션 자동 로드되는 프로젝트 컨텍스트
├── CLAUDE.local.md            # 개인 설정 (.gitignore 등록)
├── README.md
├── .gitignore
├── .env                       # API 키 (절대 커밋 금지)
│
├── .claude/                   # Claude Code 공식 폴더
│   ├── settings.json
│   ├── settings.local.json
│   ├── rules/
│   │   ├── conventions.md
│   │   └── tdd.md
│   ├── skills/
│   │   ├── tdd/
│   │   │   └── SKILL.md
│   │   ├── refactoring/
│   │   │   └── SKILL.md
│   │   └── code-reviewer/
│   │       └── SKILL.md
│   ├── agents/
│   │   ├── code_reviewer.md
│   │   └── tdd_agent.md
│   └── .mcp.json
│
├── docs/                      # 프로젝트 문서
│
├── sessions/                  # 세션별 누적 파일
│   └── 01_task-name/
│       ├── task.md            # 세션 작업 정의
│       ├── report.md          # 세션 결과 보고서
│       └── prompt.md          # 대화 로그 (/export)
│
├── src/                       # 소스 코드
└── tests/                     # 테스트 코드
```

---

## 파일 생명주기 표

| 시점 | 파일 | 작업 |
|---|---|---|
| 프로젝트 시작 (최초 1회) | `CLAUDE.md` | 프로젝트 컨텍스트 작성 |
| | `CLAUDE.local.md` | 개인 설정 작성 |
| | `.gitignore` | `.env`, `CLAUDE.local.md` 등록 |
| | `.env` | API 키 등록 |
| | `.claude/rules/*.md` | 항상 적용할 규칙 작성 |
| | `.claude/skills/*/SKILL.md` | 작업별 절차 작성 |
| | `.claude/.mcp.json` | MCP 서버 등록 |
| 세션 시작 (매 세션) | `sessions/{n}_{name}/task.md` | 이번 세션 작업 정의 |
| | `CLAUDE.md` | 필요 시 컨텍스트 업데이트 |
| 작업 중 | `src/`, `tests/` | 소스 및 테스트 코드 수정 |
| | `sessions/{n}/task.md` | 수정 이력 추가 |
| 세션 종료 | `sessions/{n}/report.md` | Claude Code가 생성하는 결과 보고서 |
| | `sessions/{n}/prompt.md` | `/export` 명령으로 대화 로그 저장 |

---

## Verification Checklist

- Windows와 WSL 중 어떤 환경에서 작업할지 판단할 수 있다
- Claude Code와 Ollama의 역할 차이를 설명할 수 있다
- `.claude/rules/`, `.claude/skills/`, `.claude/agents/` 각각에 무엇을 넣을지 판단할 수 있다
- `CLAUDE.md`와 `.env`의 차이를 설명할 수 있다 (컨텍스트 vs 비밀값)
- 세션 시작과 종료 시 어떤 파일을 생성/업데이트하는지 설명할 수 있다

---

## Common Errors and Solutions

| 오류 | 원인 | 해결 |
|---|---|---|
| WSL에서 Windows 프로젝트 파일을 열었을 때 속도가 느리다 | `/mnt/c/` 경로 사용 | WSL 홈(`~/projects/`)으로 프로젝트 이동 |
| `.env` 파일이 GitHub에 올라갔다 | `.gitignore`에 등록 누락 | `.gitignore`에 `.env` 추가, `git rm --cached .env` 실행 |
| Claude Code가 프로젝트 컨텍스트를 모른다 | `CLAUDE.md` 미작성 또는 누락 | 프로젝트 루트에 `CLAUDE.md` 작성 확인 |
| Ollama가 WSL에서 연결되지 않는다 | Windows 네이티브 Ollama 미실행 | Windows에서 Ollama 서비스 실행 확인 후 `curl localhost:11434` 테스트 |
