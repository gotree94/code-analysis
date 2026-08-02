# CodeQL

GitHub(Semmle)이 개발한 **쿼리 기반 정적 분석 엔진**입니다. 소스 코드를 데이터베이스(QL)로 변환한 뒤, SQL과 유사한 **QL 쿼리 언어**로 보안 취약점·버그를 검색합니다. 커스텀 쿼리 작성이 가능해 보안 연구 및 대규모 취약점 탐지(CVE 분석)에 강점이 있습니다.

- 공식 문서: https://codeql.github.com/
- 대상 언어: C/C++, Java, JavaScript/TypeScript, Python, Ruby, Go, C#, Swift
- 라이선스: MIT (엔진+표준 라이브러리, 특정 용도 제한 사항 있음 — GitHub 보안 연구 목적 무료)

## 1. 환경 설정 (설치)

### Windows

**방법 1: winget**
```powershell
winget install GitHub.CodeQL
```

**방법 2: 공식 릴리즈 다운로드**
1. https://github.com/github/codeql-cli-binaries/releases 에서 `codeql-win64.zip` 다운로드
2. 압축 해제 후 `codeql.exe` 경로를 PATH에 추가

```powershell
# PATH 등록 예시 (압축 해제 위치가 C:\codeql 인 경우)
setx PATH "%PATH%;C:\codeql"
```

**방법 3: Chocolatey**
```powershell
choco install codeql -y
```

설치 확인:
```powershell
codeql --version
```

### CodeQL 쿼리 저장소(표준 라이브러리) 다운로드

```powershell
git clone https://github.com/github/codeql.git
# 사용 언어 쿼리만 가져올 수도 있음 (예: C/C++)
git clone https://github.com/github/codeql.git --depth 1
# 이후 codeql db create 시 --source-root와 함께,
# codeql resolve queries로 쿼리 경로 확인
```

> 참고: Java 실행이 필요합니다. `java -version` 으로 JDK 8 이상이 설치되어 있는지 확인하세요.

## 2. 기본 사용법

### 2.1 작업 흐름 (3단계)

```
1. 데이터베이스 생성   →  codeql database create
2. 쿼리 실행(분석)     →  codeql database analyze
3. 결과 해석/변환      →  SARIF/CSV 출력
```

### 2.2 데이터베이스 생성

```powershell
# C/C++ (빌드 도구를 감싸서 추출)
codeql database create cpp-db --language=cpp --command="make"

# CMake 기반
codeql database create cpp-db --language=cpp --command="cmake -B build && cmake --build build"

# Java (Maven)
codeql database create java-db --language=java --command="mvn clean compile"

# JavaScript (빌드 불필요)
codeql database create js-db --language=javascript --source-root=webapp/
```

### 2.3 분석 (쿼리 실행)

```powershell
# 표준 보안 쿼리 실행 (C/C++)
codeql database analyze cpp-db "C:\codeql\cpp\ql\src\codeql-suites\cpp-code-scanning.qls" --format=sarif-latest --output=cpp-results.sarif
```

### 2.4 결과 확인

```powershell
# SARIF를 사람이 읽기 좋게
codeql github merge-results --files=cpp-results.sarif
# 또는 CSV/BQRS 변환
codeql database analyze cpp-db --format=csv --output=cpp-results.csv
```

## 3. 활용 예제

### 예제 1: C 취약점 탐지 (버퍼 오버플로우)

**`vuln.c`** 파일 작성:
```c
#include <string.h>

void copy(char *src) {
    char buf[16];
    strcpy(buf, src);   // src 길이 검사 없음 → 오버플로우
}
```

데이터베이스 생성 후 분석:
```powershell
gcc -c vuln.c   # 사전 컴파일 확인 (선택)
codeql database create c-db --language=cpp --command="cl /c vuln.c"   # MSVC
# 또는 gcc 기반:
codeql database create c-db --language=cpp --command="gcc -c vuln.c"

codeql database analyze c-db "cpp-code-scanning.qls" --format=sarif-latest --output=vuln.sarif
```

결과에서 `BadlyBoundedWrite` 또는 `UncontrolledDataInArithmeticExpression`류 규칙이 `vuln.c:6 strcpy`를 지적합니다.

### 예제 2: 커스텀 쿼리 작성

`myrule.ql` 파일 작성:
```ql
import cpp
import semmle.code.cpp.security.DataFlow

from FunctionCall call, Expr target
where call.getTarget().getName() = "strcpy"
  and target = call.getArgument(0)
select call, "strcpy의 사용이 확인됨: " + target.toString()
```

실행:
```powershell
codeql database analyze c-db myrule.ql --format=text
```

### 예제 3: SQL 인젝션 탐지 (Java)

`UserDao.java` 파일:
```java
import java.sql.*;

public class UserDao {
    public ResultSet getUser(String name) throws Exception {
        Connection c = DriverManager.getConnection("jdbc:h2:mem:test");
        Statement s = c.createStatement();
        return s.executeQuery("SELECT * FROM users WHERE name = '" + name + "'");  // 인젝션
    }
}
```

분석:
```powershell
codeql database create java-db --language=java --command="mvn clean compile"
codeql database analyze java-db "java-code-scanning.qls" --format=sarif-latest --output=java.sarif
```

### 예제 4: CI 연동 (GitHub Code Scanning에 업로드)

`.github/workflows/codeql.yml` (CodeQL Action 사용):
```yaml
name: CodeQL
on: [push, pull_request]
jobs:
  analyze:
    runs-on: ubuntu-latest
    permissions:
      security-events: write
    steps:
      - uses: actions/checkout@v4
      - uses: github/codeql-action/init@v3
        with: { languages: cpp }
      - uses: github/codeql-action/autobuild@v3
      - uses: github/codeql-action/analyze@v3
```
결과는 저장소의 **Security → Code scanning alerts**에 자동 등록됩니다.

### 예제 5: 로컬에서 결과를 HTML로 보기 (VS Code)

1. VS Code에서 **CodeQL 확장** 설치
2. `CodeQL: Install Pack` → 쿼리 팩 다운로드
3. `codeql database create`로 만든 db를 VS Code에서 열어 대화형 쿼리 실행
4. 결과가 에디터 내부에서 소스와 함께 표시됨

## 4. 주요 용어/명령 정리

| 용어 | 설명 |
|------|------|
| database | 추출된 코드 데이터베이스 |
| query | QL 언어로 작성된 분석 규칙 |
| query suite (.qls) | 쿼리 묶음 (전체 보안 검사용) |
| pack | 쿼리+라이브러리 묶음 (규칙 단위 배포) |
| SARIF | 결과 표준 포맷 (CI/GitHub 연동용) |
| BQRS | CodeQL 전용 결과 포맷 |

## 5. 자주 묻는 질문

**Q. C/C++ 분석 시 빌드가 반드시 필요한가요?**
- C/C++는 추출기에 빌드가 필요합니다. 빌드가 어려운 프로젝트는 `--command="true"`로 스텁할 수 있으나 분석 품질이 떨어집니다.

**Q. 무료로 사용 가능한가요?**
- GitHub 저장소의 보안 연구 목적으로는 무료입니다. 상업적/일반 용도는 별도 라이선스 확인이 필요합니다. 자세한 조건은 공식 사이트를 참고하세요.

**Q. 오픈소스지만 학습 난이도가 높아요.**
- QL 문법이 생소할 수 있습니다. 공식 튜토리얼(`codeql.github.com/docs`)과 기본 `code-scanning.qls` 쿼리부터 시작하는 것을 권장합니다.
