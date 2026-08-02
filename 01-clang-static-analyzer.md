# Clang Static Analyzer

LLVM 프로젝트에서 제공하는 **오픈소스 정적 분석 도구**입니다. 심볼릭 실행(symbolic execution) 기반으로 코드의 실행 경로를 심층 분석하여, 잠재적 널 포인터 역참조, 메모리 누수, 죽은 코드 등을 탐지합니다. 상용 도구인 CodeSonar와 가장 유사한 수준의 심층 분석을 무료로 제공합니다.

- 공식 문서: https://clang.llvm.org/docs/ClangStaticAnalyzer.html
- 대상 언어: C, C++, Objective-C
- 라이선스: Apache 2.0 / LLVM Release License

## 1. 환경 설정 (설치)

### Windows

**방법 1: winget 사용 (권장)**
```powershell
winget install LLVM.LLVM
```

**방법 2: Chocolatey 사용**
```powershell
choco install llvm -y
```

**방법 3: 공식 설치 프로그램**
- https://releases.llvm.org 에서 `LLVM-*.exe` 다운로드 후 설치
- 설치 시 `PATH에 추가` 옵션을 반드시 체크

**방법 4: 리눅스 (WSL)**
```bash
sudo apt update
sudo apt install clang
```

설치 확인:
```powershell
clang --version
clang --analyze --version   # 분석 관련 기능 확인
```

> 참고: Windows에서는 `scan-build`(Perl 기반) 대신 파이썬 기반의 **`scan-build-py`** 또는 `clang --analyze`를 주로 사용합니다. LLVM 12 이상에서는 `scan-build` 명령이 함께 제공됩니다.

## 2. 기본 사용법

### 2.1 단일 파일 분석 (가장 간단)

```powershell
# C 파일
clang --analyze -Xanalyzer -analyzer-output=text main.c

# C++ 파일 (헤더 경로 필요 시 -I 지정)
clang++ --analyze main.cpp
```

### 2.2 scan-build 으로 빌드 연동 분석

`scan-build`는 기존 빌드(make/cmake/ninja 등)를 감싸서 그동안 컴파일된 코드를 자동으로 분석합니다.

```powershell
# Makefile 기반
scan-build make

# CMake + make
scan-build cmake --build . --clean-first
scan-build make

# ninja 기반
scan-build ninja
```

### 2.3 HTML 리포트 생성

```powershell
scan-build -o reports make
```

결과: `reports/<timestamp>/index.html` — 웹 브라우저에서 시각적으로 확인 가능.

### 2.4 상세 옵션

```powershell
# 특정 체커(분석기)만 활성화
clang --analyze -Xanalyzer -analyzer-checker=core,unix,security main.c

# 모든 체커 나열
clang -cc1 -analyzer-checker-help

# 경고를 에러로 승격
scan-build -o reports -status-bugs make
# -status-bugs: 분석 중 버그 발견 시 비정상 종료 코드 반환 (CI에서 유용)
```

## 3. 활용 예제

### 예제 1: 메모리 누수 탐지

**`leak.c`** 파일 작성:
```c
#include <stdlib.h>

void f(void) {
    int *p = (int*)malloc(sizeof(int));
    if (*p > 0) {        // p가 초기화되지 않음 + malloc 누수
        return;
    }
}                        // free(p) 없이 함수 종료 → 누수
```

분석:
```powershell
clang --analyze -Xanalyzer -analyzer-checker=unix.Malloc leak.c
```

결과 (예시):
```
leak.c:5:18: warning: Dereference of undefined pointer value (initializer)
leak.c:7:1: warning: Potential leak of memory pointed to by 'p'
```

### 예제 2: 널 포인터 역참조 탐지

**`null.c`** 파일 작성:
```c
#include <stdlib.h>

int g(int *p) {
    return *p;        // p가 NULL일 가능성
}

void h(void) {
    int *q = NULL;
    g(q);             // NULL 전달
}
```

분석:
```powershell
clang --analyze -Xanalyzer -analyzer-output=text null.c
```

### 예제 3: CMake 프로젝트 통합

```powershell
# 소스 준비
mkdir myproj && cd myproj
# CMakeLists.txt, main.c 등 작성

# 빌드 + 분석 한번에
scan-build -o report_dir cmake --build . --clean-first
scan-build -o report_dir make
```

### 예제 4: CI(GitHub Actions) 연동 예시

`.github/workflows/analyze.yml`:
```yaml
name: Static Analysis
on: [push, pull_request]
jobs:
  analyze:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Install clang
        run: sudo apt-get install -y clang
      - name: Run scan-build
        run: |
          scan-build -o reports --status-bugs make
      - name: Upload report
        uses: actions/upload-artifact@v4
        with:
          name: static-analysis-report
          path: reports/
```

## 4. 주요 체커(Checker) 모음

| 체커 그룹 | 예시 | 설명 |
|-----------|------|------|
| `core` | `core.NullDereference`, `core.UndefinedBinaryOperatorResult` | 핵심 경로 분석 (기본 활성) |
| `unix` | `unix.Malloc`, `unix.MallocSizeof` | 메모리 관리 문제 |
| `security` | `security.insecureAPI.strcpy`, `security.insecureAPI.rand` | 보안에 취약한 API 사용 |
| `cplusplus` | `cplusplus.NewDelete`, `cplusplus.Move` | C++ 메모리 관리 |
| `alpha` | `alpha.security.*` | 실험 단계 체커 (오탐 가능) |

## 5. 자주 묻는 질문

**Q. Windows에서 `scan-build`가 동작하지 않아요.**
- `scan-build`는 Perl 스크립트입니다. Windows에서는 `clang --analyze` 직접 사용 또는 WSL에서 실행을 권장합니다.

**Q. 오탐(False Positive)이 너무 많아요.**
- `-Xanalyzer -analyzer-werror` 대신 리포트만 확인하거나, 특정 체커만 활성화해서 실행해보세요.

**Q. 실행 속도가 느려요.**
- 심층 분석 특성상 빌드보다 수 배 느립니다. CI에서 전체 분석 시 캐시 활용(빌드 캐시)과 병렬 빌드(`scan-build make -j4`)를 조합하세요.
