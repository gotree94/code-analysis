# SpotBugs

FindBugs의 후속 프로젝트로, **Java 바이트코드를 정적 분석**하여 잠재적 버그를 탐지하는 오픈소스 도구입니다. 컴파일된 `.class` 파일을 분석하므로 빌드 후 자동화(CI)와 잘 어울립니다. 컴파일 타임 애너테이션과 함께 사용하면 정확도가 높아집니다.

- 공식 문서: https://spotbugs.github.io/
- 대상 언어: Java (JVM)
- 라이선스: LGPL-2.1

## 1. 환경 설정 (설치)

### 사전 요구사항

- JDK 8 이상 (JDK 17 권장)
```powershell
java -version
```

### Windows 설치

**방법 1: 공식 배포본 (GUI 포함)**
1. https://github.com/spotbugs/spotbugs/releases 에서 `spotbugs-4.x.x.zip` 다운로드
2. 압축 해제 후 `bin\spotbugs.bat` 사용, `bin` 폴더를 PATH에 추가

**방법 2: Chocolatey**
```powershell
choco install spotbugs -y
```

**방법 3: Gradle 빌드 스크립트로 사용 (가장 흔한 방식)**
`build.gradle`에 플러그인 추가 (아래 활용 예제 참고)

### 리눅스/macOS
```bash
sudo apt install spotbugs       # Ubuntu (버전 오래됨)
# 또는 공식 릴리즈 tar.gz 사용
wget https://github.com/spotbugs/spotbugs/releases/download/4.8.6/spotbugs-4.8.6.tgz
tar xzf spotbugs-4.8.6.tgz
```

설치 확인:
```powershell
spotbugs -version
```

## 2. 기본 사용법

### 2.1 CLI 분석

```powershell
# classes 디렉터리의 클래스를 분석해 결과 파일 생성
spotbugs -textui -output result.txt -project analysis.project

# 또는 단일 jar/jar 묶음 분석
spotbugs -textui -analyzeOnly -project analysis.project

# 결과를 XML로
spotbugs -textui -xml:withMessages -output result.xml -project analysis.project
```

`analysis.project` (프로젝트 파일, `<?xml version="1.0" encoding="UTF-8"?>`):
```xml
<?xml version="1.0" encoding="UTF-8"?>
<Project>
  <Jar>build/classes/java/main</Jar>
  <AuxClasspathEntry>build/libs/deps</AuxClasspathEntry>
</Project>
```

또는 CLI에서 직접 jar 지정:
```powershell
spotbugs -textui -output result.txt build/libs/myapp.jar
```

### 2.2 주요 옵션

| 옵션 | 설명 |
|------|------|
| `-textui` | 콘솔 UI 모드 |
| `-output <file>` | 결과 파일 |
| `-xml:withMessages` | XML 포맷(메시지 포함) |
| `-html:fancy.xsl` | HTML 리포트 |
| `-effort:max` | 분석 강도 (min~max) |
| `-onlyAnalyze` | 대상 클래스 제한 (정규식) |
| `-low` | 낮은 심각도까지 포함 |
| `-medium` / `-high` | 특정 심각도 이상만 |
| `-progress` | 진행률 표시 |
| `-exclude <filter>` | 제외 필터 XML |
| `-include <filter>` | 포함 필터 XML |

### 2.3 GUI 사용

```powershell
spotbugs        # GUI 실행
```
- **File → New Project** → 분석할 jar/클래스 추가 → **Analyze** 클릭
- 각 버그를 코드 위치와 함께 시각적으로 확인 가능

## 3. 활용 예제

### 예제 1: Maven 프로젝트 분석 (maven plugin)

`pom.xml`에 플러그인 추가:
```xml
<build>
  <plugins>
    <plugin>
      <groupId>com.github.spotbugs</groupId>
      <artifactId>spotbugs-maven-plugin</artifactId>
      <version>4.8.6</version>
      <configuration>
        <effort>Max</effort>
        <threshold>Medium</threshold>
        <xmlOutput>true</xmlOutput>
      </configuration>
      <executions>
        <execution>
          <goals><goal>check</goal></goals>
        </execution>
      </executions>
    </plugin>
  </plugins>
</build>
```

실행:
```powershell
mvn compile spotbugs:check          # 발견 시 빌드 실패
mvn spotbugs:spotbugs               # 리포트만 생성 (target/spotbugsXml.xml)
mvn spotbugs:gui                    # GUI로 확인
```

### 예제 2: Gradle 프로젝트 분석

`build.gradle`:
```gradle
plugins {
    id 'com.github.spotbugs' version '6.0.10'
}

spotbugs {
    effort = 'max'
    reportLevel = 'medium'
    ignoreFailures = false
}

tasks.withType(com.github.spotbugs.snom.SpotBugsTask) {
    reports {
        html { enabled = true }
        xml  { enabled = true }
    }
}
```

실행:
```powershell
gradle spotbugsMain        # 분석 실행
# 결과: build/reports/spotbugs/main.html
```

### 예제 3: 버그 유발 샘플 코드

**`Buggy.java`** 파일:
```java
public class Buggy {
    private String value;

    public void set(String v) {
        value = v;
    }

    public String get() {
        // equals 없이 == 비교 (객체 비교 실수)
        if (value == "secret") return "ok";
        return null;
    }

    public int size() {
        return value.length();  // value가 null이면 NPE
    }
}
```

분석:
```powershell
javac Buggy.java
spotbugs -textui -output result.txt Buggy.class
```

결과 (예시):
```
H C Buggy.java:11  ES_COMPARING_STRINGS_WITH_EQ  Comparison of String objects using == or !=
H C Buggy.java:15  NP_DEREFERENCE_OF_READLINE_VALUE  Possible null pointer dereference
```

### 예제 4: 필터 파일로 오탐 제외

`exclude.xml`:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<FindBugsFilter>
  <Match>
    <Class name="com.example.generated.*" />   <!-- 생성 코드 제외 -->
  </Match>
  <Match>
    <Bug pattern="EI_EXPOSE_REP" />            <!-- 특정 패턴 제외 -->
  </Match>
</FindBugsFilter>
```

적용:
```powershell
spotbugs -textui -exclude exclude.xml -output result.txt myapp.jar
```

### 예제 5: CI 연동 (GitHub Actions, Maven 기준)

`.github/workflows/spotbugs.yml`:
```yaml
name: SpotBugs
on: [push]
jobs:
  spotbugs:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with: { distribution: 'temurin', java-version: '17' }
      - name: Run SpotBugs
        run: mvn compile spotbugs:check
```

## 4. 주요 버그 패턴(버그 종류) 요약

| 카테고리 | 예시 패턴 | 설명 |
|----------|-----------|------|
| NullPointer | `NP_NULL_ON_SOME_PATH` | 널 역참조 가능성 |
| 가시성 | `EI_EXPOSE_REP` | 객체 내부 필드 노출 (mutable) |
| 문자열 비교 | `ES_COMPARING_STRINGS_WITH_EQ` | `==`로 문자열 비교 |
| 동시성 | `UWF_FIELD_NOT_INITIALIZED_IN_CONSTRUCTOR`, `SWL_SLEEP_WITH_LOCK_HELD` | 스레드 관련 |
| 상속 | `EQ_OVERRIDING_EQUALS_NOT_SYMMETRIC` | equals 재정의 오류 |
| 리소스 | `OS_OPEN_STREAM` | 스트림 미해제 |
| 직렬화 | `SE_BAD_FIELD` | 직렬화 관련 문제 |

패턴 전체 목록: https://spotbugs.readthedocs.io/en/stable/bugDescriptions.html

## 5. 자주 묻는 질문

**Q. 소스가 아니라 바이트코드를 분석하나요?**
- 네. 컴파일된 `.class`/`.jar`를 분석하므로 빌드가 선행되어야 합니다.

**Q. FindBugs와의 차이는?**
- SpotBugs는 FindBugs의 후속 프로젝트로, 활발히 유지보수되며 최신 Java 버전을 지원합니다.

**Q. 낮은 심각도까지 보려면?**
- `-low` 옵션 또는 Maven `threshold`를 `Low`로 설정하세요. 오탐이 늘어날 수 있습니다.
