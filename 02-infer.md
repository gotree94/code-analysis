# Infer (Meta)

Facebook(Meta)이 개발한 오픈소스 정적 분석 도구입니다. **인터프로시저(interprocedural) 분석**을 수행하여 함수 호출 전반에 걸친 널 포인터 역참조, 리소스 누수, 교착 상태 등을 탐지합니다. 대규모 코드베이스(수백만 라인) 처리에 강점이 있습니다.

- 공식 문서: https://fbinfer.com/
- 대상 언어: C, C++, Java, Kotlin, Objective-C, Swift
- 라이선스: MIT

> **Windows 참고:** Infer는 현재 **macOS와 Linux를 공식 지원**합니다. Windows에서 사용하려면 **WSL2 + Ubuntu** 환경이 필요합니다.

## 1. 환경 설정 (설치)

### Windows에서 WSL2 설정

```powershell
# 1) WSL2 활성화
wsl --install -d Ubuntu-22.04

# 2) Ubuntu 셸로 진입
wsl -d Ubuntu-22.04
```

### Ubuntu(WSL)에 Infer 설치

**방법 1: 공식 릴리즈 tar.gz 사용 (권장)**
```bash
# 필요한 패키지
sudo apt update
sudo apt install -y curl unzip python3-pip

# 최신 버전 확인 후 다운로드
wget https://github.com/facebook/infer/releases/download/v1.1.0/infer-linux64-v1.1.0.tar.xz
tar xJf infer-linux64-v1.1.0.tar.xz
cd infer-linux64-v1.1.0
sudo ./install.sh
```

**방법 2: opam(OCaml 패키지)으로 설치**
```bash
sudo apt install -y opam
opam init --disable-sandboxing -y
opam switch create 4.14.0
eval $(opam env)
opam install infer
```

**방법 3: 소스 빌드**
```bash
git clone https://github.com/facebook/infer.git
cd infer
./build-infer.sh -y
sudo make install
```

설치 확인:
```bash
infer --version
```

## 2. 기본 사용법

### 2.1 분석 방식 개요

Infer는 빌드 명령을 감싸서 분석합니다.

| 명령 | 설명 |
|------|------|
| `infer run -- <build cmd>` | 빌드하면서 분석 (전체 파이프라인) |
| `infer analyze` | 이전 빌드 결과 재분석 |
| `infer report --issues-csv` | 리포트 생성/포맷 변환 |
| `infer clean` | 분석 결과 삭제 |

### 2.2 주요 명령 예시

```bash
# make 기반 C 프로젝트
infer run -- make

# CMake 기반
infer run -- cmake -B build && infer run -- cmake --build build

# Java (Maven)
infer run -- mvn compile

# Java (Gradle)
infer run -- gradle clean build

# 특정 분석기만 (기본: Nullsafe, ResourceLeak, etc.)
infer run --analyzer nullsafe -- javac Foo.java
```

### 2.3 분석 결과 확인

```bash
# 결과 디렉토리 확인
ls infer-out/

# 텍스트 리포트
infer report --issues-txt
cat infer-out/report.txt

# CSV 리포트
infer report --issues-csv
```

## 3. 활용 예제

### 예제 1: C 널 포인터 분석

**`null.c`** 파일 작성:
```c
#include <stdlib.h>

int deref(int* p) {
    return *p;           // p가 NULL일 수 있음
}

int main(void) {
    int* x = NULL;
    return deref(x);     // NULL 전달
}
```

분석:
```bash
infer run -- clang -c null.c
# 또는 컴파일러 없이:
infer run -- gcc -c null.c
```

결과 (예시):
```
null.c:2: NULL_DEREFERENCE: pointer `p` could be null and is dereferenced at line 2
```

### 예제 2: 리소스 누수 탐지 (C++)

**`leak.cpp`** 파일 작성:
```cpp
#include <fstream>

int main() {
    std::ifstream f("data.txt");   // close하지 않으면 RESOURCE_LEAK
    return 0;
}
```

분석:
```bash
infer run -- g++ -c leak.cpp
```

### 예제 3: Java 널 안전성(Nullsafe) 분석

**`Foo.java`** 파일 작성:
```java
import javax.annotation.Nullable;

public class Foo {
    String bar() { return null; }
    int len() {
        String s = bar();       // @Nullable 반환
        return s.length();      // NPE 가능성
    }
}
```

분석:
```bash
infer run --analyzer nullsafe -- javac Foo.java
```

### 예제 4: 보고서 필터링 및 CI 연동

```bash
# 1) CSV 생성
infer report --issues-csv > issues.csv

# 2) 특정 버그 종류만 필터 (grep)
grep -E "NULL_DEREFERENCE|RESOURCE_LEAK" issues.csv

# 3) CI에서 버그가 발견되면 실패 처리
if infer report --issues-txt | grep -q "bug"; then exit 1; fi
```

### 예제 5: GitHub Actions 연동

`.github/workflows/infer.yml`:
```yaml
name: Infer Analysis
on: [push]
jobs:
  infer:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with: { distribution: 'temurin', java-version: '17' }
      - name: Install Infer
        run: |
          wget https://github.com/facebook/infer/releases/download/v1.1.0/infer-linux64-v1.1.0.tar.xz
          tar xJf infer-linux64-v1.1.0.tar.xz
          cd infer-linux64-v1.1.0 && sudo ./install.sh
      - name: Run Infer
        run: infer run -- mvn compile
      - name: Upload report
        uses: actions/upload-artifact@v4
        with:
          name: infer-report
          path: infer-out/
```

## 4. 주요 분석기(Analyzer)

| 분석기 | 탐지 항목 |
|--------|-----------|
| `nullsafe` | 널 포인터 역참조 (Java/Objective-C) |
| `bufferoverrun` | 버퍼 오버런 (C/C++) |
| `pulse` | 리소스 누수, 언바운드 변형 분석 |
| `litho`, `racerd` | 병렬/데이터 경쟁 |
| `inferbo` | 산술 오버플로우, 버퍼 경계 |

기본 분석기 목록 확인:
```bash
infer --help | grep analyzer
```

## 5. 자주 묻는 질문

**Q. Windows에서 네이티브로 실행할 수 없나요?**
- 네. 공식 지원은 macOS/Linux입니다. Windows는 WSL2 + Ubuntu를 권장하며, WSL에서 분석 결과를 호스트의 CI에 업로드하면 됩니다.

**Q. 빌드 도구가 없는 순수 소스도 분석할 수 있나요?**
- `infer run -- clang -c file.c`처럼 단순 컴파일 명령을 직접 지정할 수 있습니다.

**Q. 오탐이 많은가요?**
- 분석기별 오탐 특성이 다릅니다. `nullsafe`는 낮은 편이고, `pulse`는 높을 수 있습니다. 주석 기반(`@Nullable` 등)으로 정확도를 높일 수 있습니다.
