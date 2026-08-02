# Cppcheck

가장 널리 사용되는 **경량 C/C++ 정적 분석 도구**입니다. 컴파일러가 필요 없어 빠르며, 메모리 누수, 오버플로우, 버퍼 오버런, 미사용 변수 등을 탐지합니다. 심층 분석(Clang SA, Infer)보다는 "가볍고 빠른" 린트성 분석에 적합합니다.

- 공식 문서: https://cppcheck.sourceforge.io/
- 대상 언어: C, C++
- 라이선스: GPL-3.0

## 1. 환경 설정 (설치)

### Windows

**방법 1: winget**
```powershell
winget install Cppcheck.Cppcheck
```

**방법 2: Chocolatey**
```powershell
choco install cppcheck -y
```

**방법 3: 공식 설치 프로그램**
- https://cppcheck.sourceforge.io/#download 에서 Windows용 설치 파일(`cppcheck-*.x64.exe`) 다운로드
- 설치 옵션에서 `PATH에 추가` 체크

**방법 4: 소스 빌드 (요즘 버전)**
```powershell
git clone https://github.com/danmar/cppcheck.git
cd cppcheck
# Windows: cmake 사용
cmake -B build -DCMAKE_BUILD_TYPE=Release -DUSE_MATCHCOMPILER=ON
cmake --build build --config Release
```

**리눅스 (WSL/macOS)**
```bash
sudo apt install cppcheck     # Ubuntu/Debian
brew install cppcheck         # macOS
```

설치 확인:
```powershell
cppcheck --version
```

## 2. 기본 사용법

```powershell
# 단일 파일
cppcheck main.c

# 디렉터리 전체 (재귀)
cppcheck src/

# 확장자 지정
cppcheck --language=c++ main.cpp
```

### 주요 옵션

| 옵션 | 설명 |
|------|------|
| `--enable=all` | 모든 체크 활성화 (warning, style, performance, portability, unusedFunction) |
| `--enable=warning,performance,portability` | 주요 항목만 (권장) |
| `--inconclusive` | 확실하지 않은 문제도 포함 (오탐 증가 가능) |
| `--xml` / `--xml-version=2` | XML 리포트 (CI/플러그인 연동용) |
| `--output-file=result.txt` | 결과를 파일로 저장 |
| `-I <include>` | 헤더 경로 지정 |
| `-D <macro>` | 매크로 정의 |
| `--std=c++17` | 표준 지정 |
| `-j N` | 병렬 실행 (성능) |
| `--suppress=rule:id` | 특정 경고 제외 |
| `--error-exitcode=1` | 문제 발견 시 종료 코드 1 (CI용) |

## 3. 활용 예제

### 예제 1: 기본 분석

**`buggy.c`** 파일 작성:
```c
#include <stdio.h>

int main(void) {
    int arr[5];
    arr[10] = 1;          // 배열 인덱스 초과
    int x = 1;
    if (x) return 0;      // x는 항상 참 → 죽은 조건
    return 0;
}
```

분석:
```powershell
cppcheck --enable=all buggy.c
```

결과 (예시):
```
buggy.c:6:5: error: Array 'arr[5]' index 10 out of bounds
buggy.c:9:11: style: Condition 'x' is always true
```

### 예제 2: 메모리 누수 탐지

**`leak.cpp`** 파일 작성:
```cpp
#include <new>

void f() {
    int* p = new int[10];
    // delete[] 없이 반환 → 누수
}
```

분석:
```powershell
cppcheck --enable=warning leak.cpp
```

### 예제 3: 헤더/매크로가 필요한 프로젝트 분석

```powershell
cppcheck --enable=warning,performance -I include/ -DDEBUG=1 -DMAX_SIZE=100 src/ -j 4
```

### 예제 4: XML 리포트 + HTML 리포트 생성

```powershell
# 1) XML 생성
cppcheck --enable=all --xml-version=2 --xml src/ 2> cppcheck-result.xml

# 2) HTML 변환 (파이썬 스크립트 필요)
pip install jinja2
python cppcheck-htmlreport/cppcheck-htmlreport.py --file=cppcheck-result.xml --report-dir=html_report
```

### 예제 5: Suppress(경고 제외) 설정

`suppressions.txt` 파일 작성:
```
*:buggy.c:6
missingIncludeSystem:*
```
적용:
```powershell
cppcheck --enable=all --suppressions-list=suppressions.txt src/
```

### 예제 6: CI(Jenkins) 연동

```powershell
cppcheck --enable=warning,performance --xml-version=2 --xml src/ -j 4 2> cppcheck.xml
# CppCheck Plugin으로 cppcheck.xml을 리포트로 사용
```

## 4. 주요 체크 종류

| 종류 | 예시 | 설명 |
|------|------|------|
| `error` | 배열 인덱스 초과, 널 역참조 | 심각한 버그 |
| `warning` | 메모리 누수, 미해제 리소스 | 잠재 버그 |
| `style` | 미사용 변수, 코드 스타일 | 가독성 관련 |
| `performance` | 비효율적인 컨테이너 사용 | 성능 |
| `portability` | 플랫폼 의존 코드 | 이식성 |
| `unusedFunction` | 사용되지 않는 함수 | 데드 코드 |

## 5. 자주 묻는 질문

**Q. 컴파일러 없이도 분석이 가능한가요?**
- 네. Cppcheck는 자체 파서를 사용하므로 컴파일이 필요 없습니다. 다만 복잡한 매크로는 `-D`, `-I`로 힌트를 줘야 정확도가 높아집니다.

**Q. CodeSonar처럼 심층 경로 분석이 되나요?**
- 아니요. Cppcheck는 규칙/패턴 기반 분석으로, 심층 경로 분석이 필요한 경우 Clang Static Analyzer나 Infer와 병행 사용을 권장합니다.
