# 프로젝트 환경

## Python 인터프리터

- 경로: `C:\winpython\WPy64-312101\python\python.exe`
- 배포판: WinPython 3.12

## Jupyter Book 빌드

`jupyter-book.exe`는 WinPython 환경에서 직접 실행이 안 되므로 아래 명령을 사용.
빌드 전 반드시 clean 먼저 실행하고, 완료 후 브라우저로 자동 열기:

```powershell
# 1. Clean
& "C:\winpython\WPy64-312101\python\python.exe" -c "from jupyter_book.cli.main import main; import sys; sys.argv = ['jb', 'clean', 'd:/projects/nyx7zen/ai_cli_dev_guide/docs']; main()"

# 2. Build
& "C:\winpython\WPy64-312101\python\python.exe" -c "from jupyter_book.cli.main import main; import sys; sys.argv = ['jb', 'build', 'd:/projects/nyx7zen/ai_cli_dev_guide/docs']; main()"

# 3. Open browser
Start-Process "D:\projects\nyx7zen\ai_cli_dev_guide\docs\_build\html\index.html"
```

빌드 결과: `docs/_build/html/index.html`

## 구조

- 문서 루트: `docs/`
- TOC: `docs/_toc.yml`
- 설정: `docs/_config.yml`
- GitHub Actions: `.github/workflows/deploy.yml` (main 브랜치 push 시 GitHub Pages 자동 배포)
