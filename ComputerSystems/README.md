# Computer Systems: A Programmer's Perspective (CS:APP, 3rd Edition)
**저자**: Randal E. Bryant, David R. O'Hallaron

컴퓨터 시스템의 하드웨어와 소프트웨어 동작 원리를 프로그래머 관점에서 깊이 탐구하는 책.  
C 프로그램이 어떻게 기계어로 변환되고, 프로세서/메모리/OS/네트워크와 어떻게 상호작용하는지를 다룬다.

---

## 📑 챕터 목록

| 챕터 | 제목 | 핵심 내용 | 링크 |
|------|------|---------|------|
| Chapter 1 | A Tour of Computer Systems | 비트와 컨텍스트, 컴파일 시스템, 하드웨어 구조, 캐시, 메모리 계층, OS 추상화, 암달의 법칙, 병렬성 | [보기](./docs/chapter01_ATourOfComputerSystems.md) |

---

## 📚 책 구성

| 파트 | 챕터 | 주제 |
|------|------|------|
| Part I: Program Structure and Execution | 2~6 | 정수/부동소수점 표현, x86-64 어셈블리, 프로세서 아키텍처, 성능 최적화, 메모리 계층 |
| Part II: Running Programs on a System | 7~9 | 링킹, 예외 제어 흐름, 가상 메모리 |
| Part III: Interaction and Communication | 10~12 | 시스템 I/O, 네트워크 프로그래밍, 동시성 |

---

## 핵심 학습 목표

- 수치 표현의 한계와 오버플로우/언더플로우 방지
- x86-64 어셈블리와 컴파일러 최적화 이해
- 캐시 친화적(cache-friendly) 코드 작성
- 링크 타임 오류 디버깅
- 버퍼 오버플로우 취약점 이해와 방어
- 프로세스/스레드/동기화 프리미티브 활용
- 동적 메모리 할당기 구현 원리
