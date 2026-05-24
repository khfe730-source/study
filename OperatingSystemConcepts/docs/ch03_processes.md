# Chapter 3: Processes

> **핵심 한 줄 요약**: 프로세스는 실행 중인 프로그램이며 OS는 PCB로 프로세스 상태를 관리하고, fork/exec 메커니즘으로 프로세스를 생성하며, 공유 메모리(shared memory)와 메시지 패싱(message passing)이라는 두 IPC 모델을 통해 프로세스 간 협력을 가능하게 한다.

---

## 목차

- [3.1 프로세스 개념](#31-프로세스-개념)
- [3.2 프로세스 스케줄링](#32-프로세스-스케줄링)
- [3.3 프로세스 연산](#33-프로세스-연산)
- [3.4 프로세스 간 통신 (IPC)](#34-프로세스-간-통신-ipc)
- [3.5 공유 메모리 기반 IPC](#35-공유-메모리-기반-ipc)
- [3.6 메시지 패싱 기반 IPC](#36-메시지-패싱-기반-ipc)
- [3.7 IPC 시스템 사례](#37-ipc-시스템-사례)
- [3.8 클라이언트-서버 통신](#38-클라이언트-서버-통신)

---

## 3.1 프로세스 개념

### 3.1.1 프로세스의 메모리 구조

프로세스는 단순한 프로그램이 아닌 **실행 중인 프로그램**이다. 디스크에 저장된 실행 파일(수동적 개체)이 메모리에 로드되면 비로소 프로세스(능동적 개체)가 된다.

```
        고주소 (max)
┌──────────────────┐
│      Stack       │  ← 함수 호출 시 활성화 레코드 push/pop
│   (↓ grows)      │     (파라미터, 지역 변수, 반환 주소)
├ ─ ─ ─ ─ ─ ─ ─ ─ ┤
│   (free space)   │
├ ─ ─ ─ ─ ─ ─ ─ ─ ┤
│      Heap        │  ← 런타임 동적 할당 (malloc/free)
│   (↑ grows)      │
├──────────────────┤
│      Data        │  ← 전역 변수 (초기화/비초기화)
├──────────────────┤
│      Text        │  ← 실행 코드 (read-only, 고정 크기)
└──────────────────┘
        저주소 (0)
```

- **Text / Data**: 프로그램 실행 중 크기 고정
- **Stack / Heap**: 서로를 향해 동적으로 증가. OS는 두 영역이 겹치지 않도록 보장

> **C 프로그램의 메모리 구조**: `size` 명령으로 확인 가능. `bss`는 초기화된 데이터 섹션(Block Started by Symbol)

동일한 프로그램의 두 인스턴스는 Text 섹션을 공유할 수 있지만, Data/Heap/Stack 섹션은 독립적이다.

### 3.1.2 프로세스 상태 (Process State)

```
                   admitted
   new ──────────────────────► ready
                                  │
                    scheduler     │
                    dispatch      ▼
                 ◄────────── running
   terminated ◄──── exit            │
                                     │ I/O or event wait
                 interrupt           ▼
   ready ◄────────────────── waiting
              I/O or event
              completion
```

| 상태 | 의미 |
|------|------|
| **New** | 프로세스가 생성 중 |
| **Running** | 명령어가 실행 중. CPU core 당 하나만 가능 |
| **Waiting** | I/O 완료, 시그널 수신 등 이벤트 대기 |
| **Ready** | 프로세서 할당을 기다리는 상태. 여러 개 동시 존재 가능 |
| **Terminated** | 실행 완료 |

### 3.1.3 프로세스 제어 블록 (PCB)

OS에서 각 프로세스는 **PCB(Process Control Block)** — Linux에서는 `task_struct` — 로 표현된다.

```
┌─────────────────────────┐
│     Process State       │  (running, ready, waiting, ...)
├─────────────────────────┤
│     Process Number      │  (PID)
├─────────────────────────┤
│     Program Counter     │  (다음 실행할 명령어 주소)
├─────────────────────────┤
│     CPU Registers       │  (인터럽트 발생 시 저장/복원)
├─────────────────────────┤
│  CPU Scheduling Info    │  (우선순위, 스케줄링 큐 포인터)
├─────────────────────────┤
│  Memory Management Info │  (base/limit 레지스터, 페이지 테이블)
├─────────────────────────┤
│   Accounting Info       │  (CPU 사용 시간, 시간 제한, 계정)
├─────────────────────────┤
│   I/O Status Info       │  (할당된 I/O 장치, 열린 파일 목록)
└─────────────────────────┘
```

**Linux `task_struct` 주요 필드**:
```c
struct task_struct {
    long              state;      // 프로세스 상태
    struct sched_entity se;       // 스케줄링 정보
    struct task_struct *parent;   // 부모 프로세스
    struct list_head  children;   // 자식 프로세스 리스트
    struct files_struct *files;   // 열린 파일 목록
    struct mm_struct  *mm;        // 주소 공간
};
```

Linux 커널은 모든 활성 프로세스를 **이중 연결 리스트(doubly linked list)**로 관리하며, `current` 포인터가 현재 실행 중인 프로세스를 가리킨다.

### 3.1.4 스레드 (Threads)

지금까지의 프로세스 모델은 단일 실행 흐름을 가정한다. 현대 OS는 **멀티스레드(multithreaded)** 프로세스를 지원하여 하나의 프로세스가 여러 작업을 동시에 수행한다. 멀티코어 시스템에서는 스레드들이 실제 병렬로 실행된다. 자세한 내용은 4장에서 다룬다.

---

## 3.2 프로세스 스케줄링

### 스케줄링 목표
- **멀티프로그래밍**: CPU가 항상 실행할 프로세스를 갖도록 → CPU 이용률 극대화
- **시분할**: 프로세스를 빈번하게 전환 → 사용자가 각 프로그램과 상호작용 가능한 응답 시간 확보

**I/O 바운드 프로세스**: 컴퓨팅보다 I/O에 더 많은 시간 소요. I/O 요청 짧게 반복.
**CPU 바운드 프로세스**: I/O 요청이 드물고 컴퓨팅 시간이 길다.

### 3.2.1 스케줄링 큐 (Scheduling Queues)

```
새 프로세스 → [Ready Queue] → CPU 실행
                                  │
               ┌──────────────────┼──────────────────┐
               ▼                  ▼                  ▼
          I/O 요청            자식 종료 대기      인터럽트/타임슬라이스
               │                  │                  │
         [I/O Wait Q]       [Wait Queue]        [Ready Queue로 복귀]
               │                  │
          I/O 완료          자식 프로세스 종료
               │                  │
         [Ready Queue]      [Ready Queue]
```

- **Ready Queue**: 실행 준비 완료. 연결 리스트로 구현. 헤더가 첫 번째 PCB를 가리킴
- **Wait Queue**: 특정 이벤트(I/O 완료 등) 대기 중인 프로세스 집합

### 3.2.2 CPU 스케줄링

**CPU 스케줄러**: Ready Queue에서 다음 실행 프로세스를 선택하여 CPU core를 할당. 최소 100ms마다 실행되며 실제로는 더 자주 실행.

**스와핑(Swapping)**: 메모리가 과부하일 때 프로세스를 메모리 → 디스크로 내보내고(swap out), 나중에 다시 불러오는(swap in) 기법. 멀티프로그래밍 정도(degree of multiprogramming)를 조절. 9장에서 상세히 다룸.

### 3.2.3 컨텍스트 스위치 (Context Switch)

CPU core가 다른 프로세스로 전환할 때 현재 프로세스의 **컨텍스트를 PCB에 저장**하고 새 프로세스의 **컨텍스트를 PCB에서 복원**하는 작업.

```
Process P0                  OS                   Process P1
   │                         │                        │
   │  인터럽트/시스템콜        │                        │
   ├────────────────────────►│                        │
   │                    P0 상태 저장 (PCB0)            │
   │                         │                        │
   │                    P1 상태 복원 (PCB1)            │
   │                         ├───────────────────────►│
   │                         │              실행 중    │
   │                         │  인터럽트/시스템콜       │
   │                         │◄───────────────────────┤
   │                    P1 상태 저장 (PCB1)            │
   │                    P0 상태 복원 (PCB0)            │
   │◄────────────────────────┤                        │
   │  재개                   │                        │
```

- **컨텍스트 스위치 시간 = 순수 오버헤드**: 전환 중에는 유용한 작업 없음
- 일반적으로 수 마이크로초. 복사해야 할 레지스터 수, 메모리 속도, 특수 명령어 유무에 의존
- **다중 레지스터 세트(multiple register sets)**: 포인터만 변경하면 되어 컨텍스트 스위치 가속 가능. 활성 프로세스 수 > 레지스터 세트 수이면 메모리로 복사 필요

---

## 3.3 프로세스 연산

### 3.3.1 프로세스 생성

프로세스는 실행 중에 새 프로세스를 생성할 수 있다. 생성한 프로세스가 **부모(parent)**, 새로 생성된 프로세스가 **자식(child)**이다. 자식은 다시 자식을 생성하여 **프로세스 트리(process tree)**를 형성.

모든 프로세스는 고유 **PID(process identifier)**로 식별된다.

**Linux 프로세스 트리 예**:
```
systemd (pid=1)
├── logind (pid=8415)
│   └── bash (pid=8416)
│       ├── ps (pid=9298)
│       └── vim (pid=9204)
└── sshd (pid=3028)
    └── sshd (pid=3610)
        └── tcsh (pid=4005)
```

`systemd`(또는 구 UNIX의 `init`)는 항상 PID 1을 가지며 모든 사용자 프로세스의 루트 부모.

**자식 프로세스 생성 시 두 가지 선택**:

| 실행 방식 | 주소 공간 |
|----------|---------|
| 부모가 자식과 병행(concurrent) 실행 | 자식이 부모의 복사본 |
| 부모가 자식 종료까지 대기 | 자식에 새 프로그램 로드 |

#### UNIX: `fork()` + `exec()`

```c
#include <sys/types.h>
#include <stdio.h>
#include <unistd.h>

int main() {
    pid_t pid = fork();

    if (pid < 0) {          // 오류
        fprintf(stderr, "Fork Failed");
        return 1;
    } else if (pid == 0) { // 자식 프로세스 (fork() 반환값 = 0)
        execlp("/bin/ls", "ls", NULL);
        // exec()는 성공 시 반환하지 않음
    } else {               // 부모 프로세스 (fork() 반환값 = 자식 PID)
        wait(NULL);        // 자식 종료까지 대기
        printf("Child Complete");
    }
    return 0;
}
```

**`fork()` 동작 원리**:
```
parent
  │
  │  fork()
  ├──────────────────► child (부모 주소 공간의 복사본)
  │  반환값: child PID  │  반환값: 0
  │                     │
  │  wait()             │  exec("/bin/ls")  ← 주소 공간을 새 프로그램으로 덮어씀
  │  (대기 중)          │  exec()는 성공 시 반환하지 않음
  │                     │
  │                     │  exit(0)
  │◄────────────────────┤
  │  wait() 반환        │
  │  재개               │
```

- `fork()` 후 두 프로세스 모두 `fork()` 이후 명령어부터 실행
- 자식은 부모의 권한, 스케줄링 속성, 열린 파일 등을 상속
- `exec()` 성공 시 현재 프로세스의 메모리 이미지를 새 프로그램으로 완전히 교체. 실패 시에만 반환

#### Windows: `CreateProcess()`

```c
STARTUPINFO si;
PROCESS_INFORMATION pi;
ZeroMemory(&si, sizeof(si));
ZeroMemory(&pi, sizeof(pi));

CreateProcess(NULL,
    "C:\\WINDOWS\\system32\\mspaint.exe",
    NULL, NULL, FALSE, 0, NULL, NULL,
    &si, &pi);

WaitForSingleObject(pi.hProcess, INFINITE); // wait()에 해당
CloseHandle(pi.hProcess);
CloseHandle(pi.hThread);
```

| | `fork()` (UNIX) | `CreateProcess()` (Windows) |
|--|---------|------------|
| 주소 공간 | 부모 복사본 | 지정 프로그램 바로 로드 |
| 파라미터 수 | 0 | 10개 이상 |
| 프로그램 교체 | 별도 `exec()` 필요 | 생성 시 지정 |

### 3.3.2 프로세스 종료

프로세스는 마지막 명령어 실행 후 `exit()` 시스템 콜로 자발적으로 종료하거나, 부모가 `TerminateProcess()` 등으로 강제 종료할 수 있다.

**좀비 프로세스(Zombie Process)**: 종료되었지만 부모가 아직 `wait()`를 호출하지 않은 프로세스. 프로세스 테이블에 exit status 유지 목적으로 엔트리 잔존. 부모가 `wait()` 호출 후 비로소 PID와 테이블 엔트리 해제.

**고아 프로세스(Orphan Process)**: 부모가 먼저 종료되어 부모 없이 남겨진 프로세스. 전통 UNIX는 `init`이 새 부모로 입양. Linux는 `systemd` 또는 다른 지정 프로세스가 고아 프로세스를 관리.

**계단식 종료(Cascading Termination)**: 부모 종료 시 모든 자식도 강제 종료하는 시스템. OS가 주도.

```c
// UNIX 프로세스 종료 패턴
pid_t pid;
int status;
pid = wait(&status);  // 자식 PID 반환, 종료 상태 획득
```

#### Android 프로세스 중요도 계층

리소스 부족 시 Android가 우선 종료하는 순서 (낮을수록 먼저 종료):

```
1. Foreground  ← 현재 화면에 보이는 앱 (가장 중요)
2. Visible     ← 직접 보이지는 않지만 포그라운드가 참조하는 프로세스
3. Service     ← 백그라운드지만 음악 스트리밍 등 사용자 인지 가능
4. Background  ← 사용자에게 보이지 않는 백그라운드
5. Empty       ← 활성 컴포넌트 없음 (가장 먼저 종료)
```

Android는 프로세스 종료 전 상태를 저장하고, 사용자가 앱으로 돌아올 때 저장된 상태로 재개.

---

## 3.4 프로세스 간 통신 (IPC)

### 독립 vs 협력 프로세스

- **독립 프로세스(Independent)**: 다른 프로세스와 데이터 공유 없음
- **협력 프로세스(Cooperating)**: 다른 프로세스에 영향을 주거나 받음

### 프로세스 협력의 이유

- **정보 공유**: 여러 앱이 동일 정보 동시 접근 (예: 클립보드)
- **계산 가속화**: 작업을 서브태스크로 분할하여 병렬 실행 (멀티코어 필요)
- **모듈화**: 시스템 기능을 별도 프로세스/스레드로 분리

### 두 가지 IPC 모델

```
(a) 공유 메모리                    (b) 메시지 패싱

Process A │ Process B         Process A        Process B
          │                       │                 │
    [공유 메모리 영역]              │  send(message)  │
          │                       ▼                 │
       kernel                 [message queue]       │
                                  │    receive()     │
                               kernel               ▼
```

| | 공유 메모리 | 메시지 패싱 |
|--|-----------|-----------|
| **속도** | 빠름 (메모리 접근 속도) | 느림 (시스템 콜 필요) |
| **적합 데이터** | 대량 데이터 | 소량 데이터 |
| **분산 시스템** | 구현 어려움 | 용이함 |
| **동기화** | 프로그래머 책임 | 충돌 없음 |
| **초기 설정** | 공유 영역 설정 후 커널 불필요 | 매번 커널 개입 |

---

## 3.5 공유 메모리 기반 IPC

공유 메모리 영역은 일반적으로 생성한 프로세스의 주소 공간에 위치하며, 다른 프로세스들은 이 영역을 자신의 주소 공간에 **첨부(attach)**하여 사용한다.

### 생산자-소비자 문제 (Producer-Consumer Problem)

협력 프로세스의 전형적 패러다임. 생산자는 정보를 생성하고 소비자는 소비한다.

**원형 버퍼(Circular Buffer)를 이용한 유한 버퍼(Bounded Buffer)**:

```c
#define BUFFER_SIZE 10
typedef struct { ... } item;

item buffer[BUFFER_SIZE];
int in = 0;   // 다음 빈 위치 (생산자 포인터)
int out = 0;  // 다음 소비 위치 (소비자 포인터)

// 비어있음: in == out
// 가득참: (in + 1) % BUFFER_SIZE == out
// 최대 BUFFER_SIZE - 1개 아이템 저장 가능 (슬롯 하나는 항상 낭비)
```

```c
// 생산자
item next_produced;
while (true) {
    while (((in + 1) % BUFFER_SIZE) == out)
        ; // 가득 찼으면 대기
    buffer[in] = next_produced;
    in = (in + 1) % BUFFER_SIZE;
}

// 소비자
item next_consumed;
while (true) {
    while (in == out)
        ; // 비어있으면 대기
    next_consumed = buffer[out];
    out = (out + 1) % BUFFER_SIZE;
}
```

> 이 구현은 동기화 없이 생산자와 소비자가 동시에 접근하는 문제가 있다. 6, 7장에서 동기화 메커니즘을 다룬다.

---

## 3.6 메시지 패싱 기반 IPC

주소 공간을 공유하지 않고 프로세스 간 통신/동기화. 분산 환경에 특히 유용.

기본 연산:
```
send(message)
receive(message)
```

### 3.6.1 명명 (Naming)

#### 직접 통신 (Direct Communication)

각 프로세스가 수신자/발신자를 명시적으로 지명:
```
send(P, message)    // 프로세스 P에 메시지 전송
receive(Q, message) // 프로세스 Q에서 메시지 수신
```

- 프로세스 쌍 당 **자동으로 하나의 링크** 생성
- 하드코딩된 식별자 → **모듈성 낮음**. 프로세스 ID 변경 시 모든 참조 수정 필요

#### 간접 통신 (Indirect Communication)

메일박스(mailbox) 또는 포트(port)를 통해 통신:
```
send(A, message)    // 메일박스 A로 전송
receive(A, message) // 메일박스 A에서 수신
```

- 두 프로세스가 **공유 메일박스**를 가질 때만 링크 성립
- 하나의 링크가 2개 이상의 프로세스와 연결 가능
- 프로세스 쌍 간에 여러 링크(메일박스) 존재 가능
- **프로세스 소유 메일박스**: 소유자만 수신 가능. 프로세스 종료 시 메일박스 소멸
- **OS 소유 메일박스**: 독립적 존재. 생성/삭제/소유권 이전을 시스템 콜로 관리

### 3.6.2 동기화 (Synchronization)

| 방식 | 동작 |
|------|------|
| **Blocking send** | 수신자 또는 메일박스가 메시지를 받을 때까지 발신자 블록 |
| **Nonblocking send** | 발신자가 메시지 전송 후 즉시 재개 |
| **Blocking receive** | 메시지가 있을 때까지 수신자 블록 |
| **Nonblocking receive** | 유효한 메시지 또는 null 즉시 반환 |

**랑데뷰(Rendezvous)**: send와 receive 모두 blocking. 생산자-소비자 문제가 단순해짐:

```c
// 생산자 (blocking send)
while (true) {
    /* produce item */
    send(next_produced);  // 소비자가 받을 때까지 블록
}

// 소비자 (blocking receive)
while (true) {
    receive(next_consumed);  // 메시지가 있을 때까지 블록
    /* consume item */
}
```

### 3.6.3 버퍼링 (Buffering)

| 용량 | 동작 |
|------|------|
| **Zero capacity** | 큐 없음. 발신자는 수신자가 받을 때까지 무조건 블록 (no buffering) |
| **Bounded capacity** | 큐 크기 n. 가득 차면 발신자 블록 |
| **Unbounded capacity** | 큐 무제한. 발신자 절대 블록 안 함 |

---

## 3.7 IPC 시스템 사례

### 3.7.1 POSIX 공유 메모리

POSIX 공유 메모리는 **메모리 매핑 파일(memory-mapped file)**로 구현된다.

```c
// 생산자: 공유 메모리 생성 및 쓰기
const int SIZE = 4096;
const char *name = "OS";

int fd = shm_open(name, O_CREAT | O_RDWR, 0666);  // 공유 메모리 객체 생성
ftruncate(fd, SIZE);                                // 크기 설정 (4096 bytes)
char *ptr = (char *)mmap(0, SIZE,                   // 메모리 매핑
    PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);

sprintf(ptr, "%s", "Hello");      // 공유 메모리에 쓰기
ptr += strlen("Hello");
sprintf(ptr, "%s", "World!");

// 소비자: 공유 메모리 읽기
fd = shm_open(name, O_RDONLY, 0666);
ptr = (char *)mmap(0, SIZE, PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);
printf("%s", (char *)ptr);
shm_unlink(name);                  // 공유 메모리 제거
```

**핵심 API**:
| 함수 | 설명 |
|------|------|
| `shm_open(name, flags, mode)` | 공유 메모리 객체 생성/오픈. 파일 디스크립터 반환 |
| `ftruncate(fd, size)` | 공유 메모리 크기 설정 |
| `mmap(...)` | 메모리 매핑 설정. 포인터 반환 |
| `shm_unlink(name)` | 공유 메모리 객체 제거 |

### 3.7.2 Mach 메시지 패싱

Mach는 분산 시스템을 위해 설계된 OS로, macOS/iOS의 커널 컴포넌트(Darwin)에 포함. **모든 태스크 간 통신이 메시지로 이루어진다**.

**포트(Port)**: Mach에서의 메일박스. 유한 크기, 단방향. 양방향 통신에는 요청/응답 포트 쌍 사용.

```
┌──────────────────────────────────────┐
│  Task T1 (port P1 소유)              │
│      send → P2 (T2 소유)             │
│      T2가 P1으로 응답하려면           │
│      T1이 T2에게 P1의 SEND right 부여│
└──────────────────────────────────────┘
```

**포트 권한(Port Rights)**:
- `MACH_PORT_RIGHT_RECEIVE`: 해당 포트에서 메시지 수신 가능 (포트 소유자만)
- `MACH_PORT_RIGHT_SEND`: 해당 포트로 메시지 전송 가능

특수 포트:
- **Task Self Port**: 커널에 메시지를 보내기 위한 포트
- **Notify Port**: 커널이 태스크에 이벤트를 알리는 포트

```c
// 포트 생성
mach_port_t port;
mach_port_allocate(
    mach_task_self(),      // 자기 자신을 가리키는 task
    MACH_PORT_RIGHT_RECEIVE,
    &port);

// 메시지 구조
struct message {
    mach_msg_header_t header;  // 고정 크기 헤더 (크기, 소스/목적지 포트)
    int data;                   // 가변 크기 바디
};

// 메시지 전송
mach_msg(&message.header, MACH_SEND_MSG, sizeof(message),
         0, MACH_PORT_NULL, MACH_MSG_TIMEOUT_NONE, MACH_PORT_NULL);

// 메시지 수신
mach_msg(&message.header, MACH_RCV_MSG, 0, sizeof(message),
         server, MACH_MSG_TIMEOUT_NONE, MACH_PORT_NULL);
```

**성능 최적화**: Mach는 메시지 복사 대신 **가상 메모리 매핑** 기법을 사용. 발신자의 메시지를 수신자의 주소 공간에 매핑 → 실제 데이터 복사 없음. 단, 같은 호스트 내에서만 적용.

### 3.7.3 Windows ALPC

Windows는 **ALPC(Advanced Local Procedure Call)**를 통해 같은 머신의 프로세스 간 메시지 패싱을 제공.

```
Client                  Server
  │  연결 포트 오픈        │
  ├──────────────────────►│  Connection Port (공개)
  │  연결 요청             │
  │◄───────────────────── │  채널(통신 포트 쌍) 생성 및 핸들 반환
  │                        │
  │  Client ←──► Server   │  Communication Port (private)
```

메시지 크기에 따른 세 가지 전송 방식:
1. **≤ 256 bytes**: 포트의 메시지 큐 직접 사용
2. **큰 메시지**: 채널과 연결된 **Section Object** (공유 메모리 영역) 경유
3. **매우 큰 데이터**: 서버가 직접 클라이언트 주소 공간 읽기/쓰기 API 사용

> ALPC는 Windows API에 노출되지 않음. 개발자는 RPC를 통해 간접 사용.

### 3.7.4 파이프 (Pipes)

파이프는 초기 UNIX의 가장 오래된 IPC 메커니즘 중 하나.

#### 일반 파이프 (Ordinary Pipes / Anonymous Pipes)

```
부모 프로세스                     자식 프로세스
   fd[1] ──────────────────────► fd[0]
  (write end)        파이프       (read end)
```

특성:
- **단방향(unidirectional)**: 양방향 통신 시 파이프 2개 필요
- **부모-자식 관계 필요**: `fork()` 후 자식이 파이프를 상속받아 사용
- 같은 머신의 프로세스끼리만 사용 가능

```c
// UNIX 일반 파이프
int fd[2];
pipe(fd);       // fd[0]: 읽기 끝, fd[1]: 쓰기 끝

pid_t pid = fork();
if (pid > 0) {  // 부모: 쓰기
    close(fd[0]);                           // 읽기 끝 닫기
    write(fd[1], write_msg, strlen(write_msg)+1);
    close(fd[1]);
} else {        // 자식: 읽기
    close(fd[1]);                           // 쓰기 끝 닫기
    read(fd[0], read_msg, BUFFER_SIZE);
    close(fd[0]);
}
```

> 미사용 끝을 반드시 닫아야 한다. 그래야 쓰기 프로세스가 쓰기 끝을 닫았을 때 읽기 프로세스가 EOF(read() 반환값 0)를 감지할 수 있다.

Windows의 **익명 파이프(Anonymous Pipes)**: UNIX 일반 파이프와 유사. `CreatePipe()`로 생성. `ReadFile()`/`WriteFile()`로 I/O.

#### 명명 파이프 (Named Pipes / FIFOs)

| 특성 | 일반 파이프 | 명명 파이프 |
|------|-----------|-----------|
| 부모-자식 관계 | 필요 | 불필요 |
| 통신 방향 | 단방향 | UNIX: 반이중, Windows: 전이중 |
| 프로세스 종료 후 | 소멸 | 파일 시스템에 존재 유지 |
| 여러 Writer | 불가 | 가능 |
| 네트워크 통신 | 불가 | Windows만 가능 |

```bash
# UNIX FIFO 생성 (mkfifo 시스템 콜)
mkfifo my_fifo
# 이후 open/read/write/close로 일반 파일처럼 사용
# 명시적 삭제 전까지 파일 시스템에 존재
```

**실무 활용**: UNIX 셸에서 `|` 연산자가 바로 파이프.
```bash
ls | less    # ls의 출력을 less의 입력으로
```

---

## 3.8 클라이언트-서버 통신

### 3.8.1 소켓 (Sockets)

소켓은 **통신의 끝점(endpoint)**이다. 네트워크상의 두 프로세스가 통신하려면 각자 하나씩, 총 두 개의 소켓이 필요하다.

**소켓 식별**: `IP 주소 + 포트 번호`
- **잘 알려진 포트(Well-known ports)**: 1024 미만. 표준 서비스 전용
  - SSH: 22, FTP: 21, HTTP: 80
- 클라이언트 포트: 1024 이상의 임시 포트 할당

```
Host X (146.86.5.20)          Web Server (161.25.19.8)
socket: 146.86.5.20:1625 ◄──► socket: 161.25.19.8:80
```

모든 연결은 소켓 쌍이 유일해야 한다.

**Java 소켓 예시**:

```java
// 서버 (DateServer.java)
ServerSocket sock = new ServerSocket(6013);
while (true) {
    Socket client = sock.accept();           // 연결 대기 (블록)
    PrintWriter pout = new PrintWriter(client.getOutputStream(), true);
    pout.println(new java.util.Date().toString());
    client.close();
}

// 클라이언트 (DateClient.java)
Socket sock = new Socket("127.0.0.1", 6013); // 127.0.0.1 = loopback
InputStream in = sock.getInputStream();
BufferedReader bin = new BufferedReader(new InputStreamReader(in));
String line;
while ((line = bin.readLine()) != null)
    System.out.println(line);
sock.close();
```

Java 소켓 타입:
- **`Socket`**: TCP (연결 지향, 신뢰성)
- **`DatagramSocket`**: UDP (비연결형, 빠름)
- **`MulticastSocket`**: 멀티캐스트 (다중 수신자)

> 소켓은 **저수준(low-level)** 통신. 바이트 스트림만 교환 → 데이터 구조 해석은 앱 책임.

### 3.8.2 원격 프로시저 호출 (RPC)

RPC는 **함수 호출 메커니즘을 네트워크로 추상화**한다. 클라이언트가 원격 서버의 함수를 마치 로컬 함수처럼 호출하는 것처럼 보이게 한다.

**핵심 컴포넌트**:

```
Client Side                              Server Side
┌────────────────────────┐         ┌──────────────────────────┐
│  Client Application    │         │  Server Application      │
│         │              │         │         ▲                │
│    (로컬 함수처럼 호출) │         │    (실제 함수 실행)       │
│         ▼              │         │         │                │
│  Client Stub           │         │  Server Stub             │
│  - 파라미터 마샬링      │         │  - 파라미터 언마샬링     │
│  - 서버 포트 로케이션  │  메시지  │  - 함수 실행 후 결과 반환│
│  - 메시지 전송 ────────┼─────────┼─────────►               │
└────────────────────────┘         └──────────────────────────┘
```

**파라미터 마샬링(Parameter Marshaling)**: 클라이언트와 서버의 데이터 표현 방식이 다를 수 있어 표준 형식으로 변환.
- **빅 엔디안(big-endian)**: 최상위 바이트 먼저 저장
- **리틀 엔디안(little-endian)**: 최하위 바이트 먼저 저장
- **XDR(External Data Representation)**: 머신 독립적 표준 데이터 표현

**RPC 의미론(Semantics)**:

| 의미론 | 설명 | 구현 |
|--------|------|------|
| **At most once** | 최대 1회 실행. 중복 실행 방지 | 타임스탬프 + 이력 관리. 중복 메시지 무시 |
| **Exactly once** | 정확히 1회 실행. 분실/중복 모두 방지 | `at most once` + ACK 메커니즘. 클라이언트가 ACK 받을 때까지 재전송 |

**포트 바인딩(Port Binding)**:
1. **정적 바인딩**: 컴파일 타임에 고정 포트 번호
2. **동적 바인딩**: **랑데뷰 데몬(matchmaker)** 경유. 클라이언트가 matchmaker에 RPC 이름으로 포트 질의 → 포트 번호 반환 → 해당 포트로 RPC 호출

```
Client ──► matchmaker (고정 RPC 포트)
           "RPC X의 포트는?" ──► 포트 P 반환
Client ──► 서버 포트 P로 실제 RPC 호출
```

#### Android RPC (Binder 프레임워크)

Android는 같은 시스템 내 프로세스 간 IPC로 RPC를 활용하는 **Binder 프레임워크**를 제공한다.

```
Client App                          Service
    │  bindService()                    │
    ├───────────────────────────────────►│
    │  onBind() 호출                     │
    │◄────────────────────────────────── │ stub 반환
    │                                    │
    │  service.remoteMethod(3, 0.14)    │
    ├───────────────────────────────────►│
    │  (Binder가 파라미터 마샬링/전달)   │
    │◄────────────────────────────────── │ 결과 반환
```

**AIDL (Android Interface Definition Language)**: 원격 서비스 인터페이스 정의. `.aidl` 파일에서 stub 코드 자동 생성.

```java
// RemoteService.aidl
interface RemoteService {
    boolean remoteMethod(int x, double y);
}
```

Binder는 파라미터 마샬링, 프로세스 간 데이터 전달, 서비스 구현 호출, 반환값 전달을 모두 자동으로 처리한다.

---

## 핵심 정리

| 개념 | 설명 |
|------|------|
| **프로세스 메모리 구조** | Text(코드) + Data(전역) + Heap(동적) + Stack(함수 호출). Stack↓Heap↑ |
| **5가지 프로세스 상태** | New → Ready ↔ Running → Waiting → Terminated |
| **PCB / task_struct** | OS가 프로세스를 표현하는 자료구조. 상태, PC, 레지스터, 스케줄링/메모리/I/O 정보 |
| **컨텍스트 스위치** | PCB에 현재 상태 저장 후 다른 프로세스 PCB 복원. 순수 오버헤드 |
| **fork() + exec()** | fork: 부모 복사 생성. exec: 주소 공간을 새 프로그램으로 교체 |
| **좀비 프로세스** | 종료되었지만 부모가 wait() 미호출. 테이블 엔트리 잔존 |
| **고아 프로세스** | 부모가 먼저 종료. systemd/init이 입양 |
| **공유 메모리 IPC** | 빠름. 동기화는 프로그래머 책임. 분산 환경에 불리 |
| **메시지 패싱 IPC** | 시스템 콜 필요. 소량 데이터. 분산 환경에 유리. 충돌 없음 |
| **직접/간접 통신** | 직접: 프로세스 명시적 지명. 간접: 메일박스/포트 경유 |
| **POSIX 공유 메모리** | shm_open → ftruncate → mmap. 메모리 매핑 파일 방식 |
| **Mach 포트** | 단방향 메일박스. 포트 권한으로 접근 제어. 가상 메모리 매핑으로 복사 최소화 |
| **일반 파이프** | 단방향, 부모-자식 관계 필요. 프로세스 종료 시 소멸 |
| **명명 파이프 (FIFO)** | 부모-자식 불필요. 파일 시스템에 지속. UNIX: 반이중, Windows: 전이중 |
| **소켓** | IP:포트로 식별. 분산 환경의 저수준 통신 |
| **RPC** | 원격 함수 호출 추상화. 스텁이 마샬링 처리. XDR로 데이터 표현 통일 |
| **Android Binder** | 같은 시스템 내 프로세스 간 RPC. AIDL로 인터페이스 정의, stub 자동 생성 |
