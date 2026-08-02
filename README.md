# 코드 정적 분석 솔루션 가이드

CodeSonar(상용 정적 분석 도구)의 대안이 되는 **오픈소스 코드 분석 도구**들을 솔루션별로 정리한 문서 모음입니다.

각 문서는 **환경 설정(설치)** → **기본 사용법** → **실전 활용 예제** 순서로 구성되어 있습니다.

## 목차

| # | 솔루션 | 언어 대상 | 특징 | 문서 |
|---|--------|-----------|------|------|
| 1 | Clang Static Analyzer | C/C++/Objective-C | LLVM 기반 심층 경로 분석, CodeSonar와 가장 유사 | [01-clang-static-analyzer.md](01-clang-static-analyzer.md) |
| 2 | Infer (Meta) | C/Java/Objective-C | 인터리브(interprocedural) 분석, 대규모 프로젝트에 적합 | [02-infer.md](02-infer.md) |
| 3 | Cppcheck | C/C++ | 가볍고 널리 사용되는 정적 분석 | [03-cppcheck.md](03-cppcheck.md) |
| 4 | Frama-C | C | 심층 분석 + 형식 검증(정형 검증) 지원 | [04-frama-c.md](04-frama-c.md) |
| 5 | CodeQL | 다양한 언어 | GitHub 제공, 보안 취약점 분석, 쿼리 기반 | [05-codeql.md](05-codeql.md) |
| 6 | Semgrep | 다양한 언어 | 패턴 기반, 커스텀 규칙 작성 용이 | [06-semgrep.md](06-semgrep.md) |
| 7 | SpotBugs | Java | FindBugs 후속, JVM 정적 분석 | [07-spotbugs.md](07-spotbugs.md) |

## 비교 요약

| 항목 | Clang SA | Infer | Cppcheck | Frama-C | CodeQL | Semgrep | SpotBugs |
|------|----------|-------|----------|---------|--------|---------|----------|
| 심층 경로 분석 | ★★★ | ★★★ | ★ | ★★★ | ★★ | ★ | ★ |
| 빌드 연동 | make/cmake | make/cmake/gradle | 파일 단위 | make 파일 | 빌드 전체 | 파일 단위 | 빌드 전체 |
| 보안 취약점 탐지 | 일부 | 좋음 | 일부 | 좋음 | 최상 | 좋음 | 일부 |
| Windows 네이티브 | 가능 | 제한(WSL 필요) | 가능 | 제한(WSL 권장) | 가능 | 가능 | 가능 |
| 학습 곡선 | 중 | 중 | 낮음 | 높음 | 높음 | 낮음 | 낮음 |

## 참고

- 사용 환경은 **Windows 10/11** 기준으로 작성되었으며, 필요 시 WSL(Linux) 설치 방법도 함께 안내합니다.
- 각 도구의 최신 버전은 공식 사이트를 확인하세요.
- CI(젠킨스, GitHub Actions 등)에 통합하는 방법은 각 문서의 "CI 연동" 섹션을 참고하세요.
