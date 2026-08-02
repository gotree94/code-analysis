# Frama-C

CEA List가 개발한 **C 언어용 오픈소스 정적 분석 플랫폼**입니다. 경로 민감 분석(EVA 플러그인)뿐 아니라 **정형 검증(WP 플러그인)**까지 지원하여, 상용 도구 중에서도 CodeSonar와 가장 유사한 "심층 분석 + 검증" 수준을 제공합니다.

- 공식 문서: https://frama-c.com/
- 대상 언어: C (C99/C11 일부, C17 일부)
- 라이선스: LGPL-2.1 (코어는 LGPL, 일부 플러그인은 GPL)

> **Windows 참고:** Frama-C는 OCaml 기반으로 Windows 네이티브 빌드가 가능하지만 초보자에게는 **WSL2(Ubuntu) 환경을 강력히 권장**합니다.

## 1. 환경 설정 (설치)

### Windows에서 WSL2 + Ubuntu 설정

```powershell
wsl --install -d Ubuntu-22.04
wsl -d Ubuntu-22.04
```

### Ubuntu(WSL)에 설치

**방법 1: opam (권장, 최신 버전)**
```bash
sudo apt update
sudo apt install -y opam
opam init --disable-sandboxing -y
eval $(opam env)
opam install frama-c
```

**방법 2: 소스 빌드**
```bash
git clone https://git.frama-c.com/pub/frama-c.git
cd frama-c
make && sudo make install
```

**방법 3: 패키지 (일반적으로 오래된 버전)**
```bash
sudo apt install frama-c
```

설치 확인:
```bash
frama-c -version
# 예: Frama-C 28.0 (Nickel) 2024-01-19
```

## 2. 기본 사용법

### 2.1 핵심 플러그인

| 플러그인 | 명령 | 역할 |
|----------|------|------|
| **EVA** | `-eva` | 경로 민감 정적 분석 (경고/버그 탐지) — 실무 핵심 |
| **WP** | `-wp` | Hoare 논리 기반 정형 검증 (수학적 증명) |
| **RTE** | `-rte` | 런타임 에러(0나눗셈, 오버플로우, 배열 초과) 삽입 |
| **Aorai** | `-aorai` | 오토마타 기반 (콜백 검증) |
| **Slicing** | `-slice-calls` | 프로그램 슬라이싱 |

### 2.2 기본 명령 형태

```bash
# EVA 분석 (기본)
frama-c -eva file.c

# EVA + 런타임 에러 검사
frama-c -eva -rte file.c

# GUI 실행 (그래픽 확인)
frama-c-gui -eva -rte file.c
```

## 3. 활용 예제

### 예제 1: EVA를 이용한 배열 오버런 탐지

**`array.c`** 파일 작성:
```c
int main(void) {
    int tab[5];
    int i;
    for (i = 0; i < 10; i++)   // i는 0..9 → tab[5..9] 초과
        tab[i] = i;
    return tab[0];
}
```

분석:
```bash
frama-c -eva -rte array.c
```

결과 (예시):
```
[rte] annotating function main
[eva] array.c:7: Warning:
      out of bounds read. assert tab[i] < 5;
      cannot prove validity of the memory access at line 7
```

### 예제 2: 산술 오버플로우 탐지

**`overflow.c`** 파일 작성:
```c
int f(int x, int y) {
    return x + y;    // 오버플로우 가능성
}
```

분석:
```bash
frama-c -eva -rte overflow.c
```

결과:
```
[eva] Warning: signed overflow. assert (int)(x + y) in { -2147483648 .. 2147483647 }
```

### 예제 3: 정형 검증 (WP) — 함수 계약 작성

**`max.c`** 파일 작성:
```c
/*@
  requires 0 <= a <= 1000;
  requires 0 <= b <= 1000;
  ensures \result == a || \result == b;
  ensures \result >= a && \result >= b;
*/
int max(int a, int b) {
    if (a > b) return a;
    return b;
}
```

분석 (계약 검증):
```bash
frama-c -wp -wp-rte max.c
```

결과:
```
[wp] Running WP plugin...
[wp] 7 goals scheduled
[wp] [Alt-Ergo] Goal typed_max_post_2 : Valid
[wp] [Alt-Ergo] Goal typed_max_post_1 : Valid
```

모든 goal이 `Valid`이면 함수가 계약을 만족함을 **수학적으로 증명**한 것입니다. 프로버(증명기)가 필요하며, `-wp-prover alt-ergo` 옵션으로 지정합니다.

### 예제 4: EVA로 함수 분석 — 0나눗셈

**`div.c`** 파일 작성:
```c
#include <stdio.h>

int div10(int n) {
    return 10 / n;    // n이 0이면 런타임 에러
}

int main(void) {
    return div10(0);  // 0 전달 → 에러
}
```

분석:
```bash
frama-c -eva -rte div10.c
```

### 예제 5: RTE 애노테이션 삽입 및 결과 확인

```bash
# 원본에 런타임 에러 검증을 위한 assert 삽입한 소스 출력
frama-c -rte -then-last -print div10.c

# 각 assert의 만족 여부를 EVA로 확인
frama-c -rte -eva div10.c
```

### 예제 6: 값 집합 분석 결과 출력

```bash
frama-c -eva -eva-slevel 10 -no-eva-warn-undefined-arith max.c
# -eva-slevel: 병합 수준을 높여 정밀도 향상
# 값 범위 확인: -eva-print-values
```

## 4. 성능/정밀도 관련 주요 옵션

| 옵션 | 설명 |
|------|------|
| `-eva-slevel N` | 상태 병합 수준 (정밀도 ↔ 성능 트레이드오프) |
| `-eva-precision N` | 정밀도 레벨 0~11 (기본 11, 클수록 느림) |
| `-slevel N` | 경로 민감도 |
| `-wp-prover PROVER` | 증명기 선택 (alt-ergo, z3, cvc4 등) |
| `-wp-timeout N` | 증명 타임아웃(초) |
| `-j N` | 병렬 증명 |
| `-no-...` | 특정 경고 억제 |

프로버 설치 예시:
```bash
sudo apt install alt-ergo   # 기본 증명기
sudo apt install z3         # SMT 솔버
```

## 5. 자주 묻는 질문

**Q. Windows 네이티브 설치는 불가능한가요?**
- 가능하지만 OCaml 도구체인 구성이 복잡합니다. Windows에서 편하게 쓰려면 WSL2 + Ubuntu를 권장합니다.

**Q. 정형 검증(WP)은 어떻게 배우면 되나요?**
- ACSL(ANSI/ISO C Specification Language) 계약 문법(`requires`, `ensures`, `loop invariant`)을 먼저 익히세요. Frama-C 공식 튜토리얼(https://frama-c.com/html/tute.html) 참고.

**Q. EVA와 WP 중 무엇을 먼저 써야 하나요?**
- EVA는 자동으로 경고를 잡아주고, WP는 계약을 직접 작성해 증명합니다. 실무에서는 **EVA 먼저 → 중요한 함수는 WP** 순으로 적용하는 것이 일반적입니다.
