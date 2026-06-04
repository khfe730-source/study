# Computer Systems: A Programmer's Perspective (CS:APP, 3rd Edition)
**저자**: Randal E. Bryant, David R. O'Hallaron

컴퓨터 시스템의 하드웨어와 소프트웨어 동작 원리를 프로그래머 관점에서 깊이 탐구하는 책.  
C 프로그램이 어떻게 기계어로 변환되고, 프로세서/메모리/OS/네트워크와 어떻게 상호작용하는지를 다룬다.

---

## 📑 챕터 목록

| 챕터 | 제목 | 핵심 내용 | 링크 |
|------|------|---------|------|
| Chapter 1 | A Tour of Computer Systems | 비트와 컨텍스트, 컴파일 시스템, 하드웨어 구조, 캐시, 메모리 계층, OS 추상화, 암달의 법칙, 병렬성 | [보기](./docs/chapter01_ATourOfComputerSystems.md) |
| Chapter 2 | Representing and Manipulating Information | 16진수, 엔디언, 부울 대수, unsigned/two's-complement 인코딩, 정수 산술(오버플로/시프트 최적화), IEEE 754 부동소수점 | [보기](./docs/chapter02_RepresentingAndManipulatingInformation.md) |
| Chapter 3 | Machine-Level Representation of Programs | x86-64 레지스터, 주소 지정 모드, MOV/LEA/산술 명령어, 조건 코드, 제어 흐름, 프로시저 호출 규약, 배열/struct/union, 버퍼 오버플로우 방어(카나리·ASLR·NX), AVX 부동소수점 | [보기](./docs/chapter03_MachineLevelRepresentationOfPrograms.md) |
| Chapter 4 | Processor Architecture | Y86-64 ISA, HCL 논리 설계, SEQ 6단계 처리, 파이프라이닝 원리(처리량/지연/해저드), PIPE 포워딩·스톨·버블, CPI 분석 | [보기](./docs/chapter04_ProcessorArchitecture.md) |
| Chapter 5 | Optimizing Program Performance | 컴파일러 한계(메모리 별칭·부작용), CPE 지표, code motion, 임시 변수 누적, 기능 유닛 지연/처리량 바운드, 루프 언롤링, 복수 어큐뮬레이터, 재결합 변환, 레지스터 스필링, gprof 프로파일링 | [보기](./docs/chapter05_OptimizingProgramPerformance.md) |
| Chapter 6 | The Memory Hierarchy | SRAM/DRAM/SSD/디스크 특성, 시간·공간 지역성, stride-k 미스율, 캐시 구조(S,E,B,m), 직접 매핑·셋 연관·완전 연관, 충돌 미스·스래싱, 쓰기 정책(write-back/write-through), Core i7 캐시 계층, 캐시 친화적 코드, 메모리 마운틴, 행렬 곱 루프 순서 최적화 | [보기](./docs/chapter06_TheMemoryHierarchy.md) |
| Chapter 7 | Linking | 컴파일러 드라이버 파이프라인, ELF 오브젝트 파일(.text/.data/.bss/.symtab/.rel), 심볼 해석(strong/weak 규칙·중복 버그), 정적 라이브러리(아카이브·순서), 재배치(PC-상대/절대), 실행 파일 로딩, 동적 링킹(.so), PIC(GOT/PLT/lazy binding), 라이브러리 인터포지셔닝 | [보기](./docs/chapter07_Linking.md) |
| Chapter 8 | Exceptional Control Flow | 예외 4종(Interrupt/Trap/Fault/Abort), 시스템 콜(syscall), 프로세스(논리 흐름·사설 주소·컨텍스트 스위치), fork/waitpid/execve, 시그널(30종·pending/blocked 비트벡터·큐 없음), 안전한 핸들러 가이드라인(G0~G5), 레이스 방지(SIGCHLD 차단·sigsuspend), nonlocal jump(setjmp/longjmp) | [보기](./docs/chapter08_ExceptionalControlFlow.md) |
| Chapter 9 | Virtual Memory | 가상/물리 주소 지정, VM 3역할(캐시/관리/보호), 페이지 테이블·PTE·페이지 폴트·demand paging, TLB, 다단계 페이지 테이블, Core i7 4단계 변환(CR3·XD/A/D 비트), Linux vm_area_struct, Memory Mapping(CoW·mmap·fork/execve 구현), malloc/free(내부·외부 단편화, 묵시적 자유 리스트, 경계 태그 합체, 분리 자유 리스트), C 메모리 버그 10종 | [보기](./docs/chapter09_VirtualMemory.md) |
| Chapter 10 | System-Level I/O | Unix I/O(파일 디스크립터·EOF·short count), 파일 타입, open/close/read/write, Rio 패키지(비버퍼/버퍼 함수·신호 안전 재시도), stat/fstat, 디렉토리 읽기, 파일 공유(디스크립터 테이블·파일 테이블·v-node), fork와 파일 위치 공유, dup2 리다이렉션, 표준 I/O vs Rio 선택 가이드라인 | [보기](./docs/chapter10_SystemLevelIO.md) |
| Chapter 11 | Network Programming | 클라이언트-서버 모델, LAN/WAN/인터넷 계층, TCP/IP·IP 주소·바이트 순서(htonl/ntohl)·inet_pton/ntop, 도메인 이름·DNS, 소켓 연결(4-tuple·임시포트·well-known 포트), 소켓 API(socket/connect/bind/listen/accept), getaddrinfo/getnameinfo, open_clientfd/open_listenfd, HTTP(요청/응답/상태코드), CGI(QUERY_STRING·dup2·fork+execve), Tiny Web Server | [보기](./docs/chapter11_NetworkProgramming.md) |

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
