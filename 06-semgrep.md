# Semgrep

r2c가 개발한 **패턴 기반 정적 분석 도구**입니다. 스캔할 패턴(규칙)을 간단한 YAML로 작성할 수 있어 커스텀이 매우 쉬우며, 빠른 속도로 다양한 언어를 지원합니다. "규칙으로 특정 코딩 패턴(보안 취약점, 안티패턴, 컨벤션 위반)을 빠르게 찾는" 용도에 최적화되어 있습니다.

- 공식 문서: https://semgrep.dev/
- 대상 언어: 30개 이상 (Python, JS/TS, Java, Go, C, C++, Ruby, PHP, Kotlin, Swift 등)
- 라이선스: LGPL-2.1 (엔진), 규칙 저장소는 별도

## 1. 환경 설정 (설치)

### Windows

**방법 1: pip (권장)**
```powershell
python -m pip install semgrep
```

**방법 2: Chocolatey**
```powershell
choco install semgrep -y
```

**방법 3: winget**
```powershell
winget install semgrep.semgrep
```

**방법 4: 도커**
```powershell
docker pull semgrep/semgrep
docker run --rm -v "${PWD}:/src" semgrep/semgrep scan /src
```

**리눅스/macOS**
```bash
pip install semgrep       # 또는
brew install semgrep      # macOS
```

설치 확인:
```powershell
semgrep --version
```

## 2. 기본 사용법

### 2.1 스캔 실행

```powershell
# 현재 디렉터리 스캔 (기본 규칙)
semgrep scan

# 특정 파일/폴더
semgrep scan main.c

# 특정 규칙 저장소 사용
semgrep scan --config p/default
semgrep scan --config p/security-audit
semgrep scan --config p/cwe-top-25

# 특정 규칙 파일/폴더
semgrep scan --config rules/myrule.yml
```

### 2.2 주요 옵션

| 옵션 | 설명 |
|------|------|
| `--config <rule>` | 규칙 소스 지정 (`p/...`, 파일, URL) |
| `--severity` | `ERROR`/`WARNING`/`INFO` 필터 |
| `--lang <lang>` | 언어 지정 (자동 감지 실패 시) |
| `--json` / `--sarif` | 결과 포맷 (CI 연동용) |
| `--include` / `--exclude` | 대상 파일 필터 |
| `--output <file>` | 결과 저장 |
| `--metrics=off` | 원격 메트릭 전송 끄기 |
| `--validate` | 규칙 파일 문법 검증 |
| `--dryrun` | 실행 없이 규칙 검증 |

## 3. 활용 예제

### 예제 1: 커스텀 규칙 작성

**`rules/no-strcpy.yml`** 파일 작성:
```yaml
rules:
  - id: no-insecure-strcpy
    message: strcpy 사용 금지 — strncpy 또는 snprintf로 대체하세요.
    languages: [c, cpp]
    severity: WARNING
    patterns:
      - pattern: strcpy($DST, $SRC)
    metadata:
      cwe: CWE-676
```

적용:
```powershell
semgrep --config rules/no-strcpy.yml --validate
semgrep --config rules/no-strcpy.yml main.c
```

### 예제 2: 보안 감사 규칙으로 스캔 (Python 예)

**`app.py`** 파일:
```python
import os
import sqlite3

def get_user(name):
    query = "SELECT * FROM users WHERE name = '" + name + "'"  # SQL 인젝션
    return sqlite3.connect("db").execute(query)

def save(token):
    with open("token.txt", "w") as f:
        f.write(os.environ.get("API_KEY"))   # 하드코딩된 환경변수 기록
```

스캔:
```powershell
semgrep scan --config p/security-audit app.py
```

결과 (예시):
```
app.py:5  WARNING  python.lang.security.audit.sql-injection  SQL injection detected
app.py:9  WARNING  python.lang.security.audit.hardcoded-password  Hardcoded secret
```

### 예제 3: 자주 쓰는 커뮤니티 규칙 저장소

```powershell
# 기본 보안
semgrep scan --config p/security-audit
# CWE 상위 25개
semgrep scan --config p/cwe-top-25
# OWASP
semgrep scan --config p/owasp-top-ten
# 프레임워크별 (예: React)
semgrep scan --config p/react
# 커뮤니티 규칙 모음 검색
semgrep scan --config r/java.spring.annotation
```

### 예제 4: 여러 규칙 + SARIF 출력 (CI)

```powershell
semgrep scan --config rules/ --config p/security-audit --sarif --output=semgrep.sarif
```

### 예제 5: GitHub Actions 연동

`.github/workflows/semgrep.yml`:
```yaml
name: Semgrep
on: [push, pull_request]
jobs:
  semgrep:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: returntocorp/semgrep-action@v2
        with:
          config: >-
            p/security-audit
            rules/no-strcpy.yml
```
이 Action은 기본적으로 발견 사항을 CI 로그에 표시하고, `--fail-on-error`처럼 동작하여 문제 발견 시 빌드를 실패시킵니다.

### 예제 6: pre-commit 훅 (커밋 시 자동 검사)

`.pre-commit-config.yaml`:
```yaml
repos:
  - repo: https://github.com/semgrep/semgrep
    rev: v1.60.0
    hooks:
      - id: semgrep
        args: [--config, p/security-audit]
```
```powershell
pip install pre-commit
pre-commit install
```

## 4. 규칙 문법 핵심

| 문법 | 설명 | 예시 |
|------|------|------|
| `pattern` | 정확히 일치하는 패턴 | `pattern: strcpy($A, $B)` |
| `pattern-either` | 여러 패턴 중 하나 | `pattern-either: [pattern: foo(...), pattern: bar(...)]` |
| `pattern-not` | 제외할 패턴 | `pattern-not: sprintf(_, "%s", $A)` |
| `metavariable-regex` | 변수명 정규식 조건 | `metavariable-regex: { metavariable: $X, regex: '.*secret.*' }` |
| `metavariable-comparison` | 값 비교 조건 | `metavariable-comparison: { metavariable: $N, comparison: $N > 10 }` |
| `fix` | 자동 수정 제안 | `fix: snprintf($DST, sizeof($DST), "%s", $SRC)` |

`--autofix` 옵션으로 `fix`를 자동 적용할 수 있습니다.

## 5. 자주 묻는 질문

**Q. Semgrep은 심층 분석이 되나요?**
- 아니요. Semgrep은 **구문/패턴 기반(lexical/structural)** 분석으로, Clang SA나 Infer 같은 데이터 흐름 분석은 제한적입니다. 심층 분석은 다른 도구와 병행하세요.

**Q. 오탐이 많은 편인가요?**
- 패턴 기반 특성상 완전 일치로 오탐이 비교적 적은 편입니다. 필요한 규칙만 선택해 사용하면 노이즈가 줄어듭니다.

**Q. 새 규칙은 어떻게 만들면 좋나요?**
- https://semgrep.dev/playground 에서 코드를 붙여넣고 실시간으로 규칙을 테스트해 보세요.
