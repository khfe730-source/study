# Chapter 8: Exceptional Control Flow

## 핵심 주제

프로세서의 제어 흐름은 일반적으로 순차적이지만, 시스템은 내부 프로그램 변수로는 포착할 수 없는 외부 이벤트에도 반응해야 한다. **예외적 제어 흐름(ECF, Exceptional Control Flow)** 은 이를 위해 하드웨어·OS·애플리케이션 모든 계층에서 사용되는 기본 메커니즘이다.

ECF의 형태:
- **하드웨어 레벨**: 이벤트 감지 → 예외 핸들러로 제어 이전
- **OS 레벨**: 커널의 컨텍스트 스위치 (프로세스 간 전환)
- **애플리케이션 레벨**: 시그널, nonlocal jump

---

## 8.1 Exceptions (예외)

예외는 프로세서 상태의 변화(이벤트)에 반응하여 제어 흐름을 갑자기 바꾸는 것이다.

```
정상 실행:  I_prev → I_curr → I_next → ...

예외 발생:  I_curr 실행 중 → 이벤트 감지
            → 예외 테이블(exception table)을 통해 예외 핸들러로 간접 호출
            → 핸들러 처리 후: (1) I_curr 재실행 / (2) I_next로 복귀 / (3) 프로그램 종료
```

### 8.1.1 예외 처리 메커니즘

- 시스템 부팅 시: OS가 **예외 테이블(jump table)** 초기화. 항목 k = 예외 k의 핸들러 주소
- 런타임: 프로세서가 이벤트 감지 → 예외 번호 k 결정 → 예외 테이블[k]로 간접 프로시저 호출

**일반 함수 호출과의 차이:**
- 반환 주소: 현재 명령어(I_curr) 또는 다음 명령어(I_next) (예외 클래스에 따라)
- 추가 프로세서 상태 저장 (x86-64: EFLAGS 등)
- 유저 → 커널 전환 시 **커널 스택**에 저장
- 핸들러는 **커널 모드**로 실행 (모든 시스템 리소스 접근 가능)

### 8.1.2 예외의 네 가지 클래스

| 클래스 | 원인 | 동기성 | 반환 동작 |
|--------|------|--------|---------|
| **Interrupt (인터럽트)** | I/O 디바이스 신호 | 비동기 | 항상 I_next로 복귀 |
| **Trap (트랩)** | 의도적 예외 (syscall) | 동기 | 항상 I_next로 복귀 |
| **Fault (폴트)** | 복구 가능한 오류 | 동기 | I_curr 재실행 또는 종료 |
| **Abort (어보트)** | 복구 불가능한 오류 | 동기 | 반환 안 함 |

**Interrupt:** I/O 디바이스(네트워크 어댑터, 디스크 컨트롤러, 타이머)가 프로세서 핀을 통해 신호 전송. 현재 명령어 실행 완료 후 처리하고 다음 명령어로 복귀 → 프로그램에 투명하게 보임.

**Trap (System Call):** 유저 프로그램이 커널 서비스를 요청하는 제어된 인터페이스. `syscall n` 명령어로 트랩을 발생시키면 커널이 서비스 n을 처리하고 I_next로 복귀.

**Fault:** 핸들러가 오류를 수정하면 I_curr를 재실행. 대표 예: **페이지 폴트(page fault, #14)** — 가상 주소의 페이지가 메모리에 없어 디스크에서 로드한 후 명령어를 재실행.

**Abort:** DRAM/SRAM 패리티 오류 등 하드웨어 치명적 오류. 애플리케이션을 종료시키는 abort 루틴으로 제어 전달.

### 8.1.3 Linux/x86-64 예외

| 예외 번호 | 설명 | 클래스 |
|---------|------|--------|
| 0 | Divide error (SIGFPE) | Fault |
| 13 | General protection fault (Segmentation fault) | Fault |
| 14 | Page fault | Fault |
| 18 | Machine check | Abort |
| 32–255 | OS 정의 예외 (인터럽트/트랩) | Interrupt/Trap |

**시스템 콜 (Linux x86-64):**

```
# 레지스터 규약:
# %rax: 시스템 콜 번호
# %rdi, %rsi, %rdx, %r10, %r8, %r9: 인수 (최대 6개)
# 반환 후: %rax = 반환값 (음수 -4095~-1 → errno)
# %rcx, %r11 소멸
```

```asm
# write(1, "hello, world\n", 13) + _exit(0)
movq $1, %rax     # write = syscall #1
movq $1, %rdi     # arg1: stdout (fd=1)
movq $string, %rsi # arg2: 문자열 주소
movq $len, %rdx   # arg3: 길이
syscall

movq $60, %rax    # _exit = syscall #60
movq $0, %rdi     # arg1: exit status
syscall
```

| 번호 | 이름 | 설명 |
|------|------|------|
| 0 | read | 파일 읽기 |
| 1 | write | 파일 쓰기 |
| 2 | open | 파일 열기 |
| 3 | close | 파일 닫기 |
| 9 | mmap | 메모리 페이지를 파일에 매핑 |
| 57 | fork | 프로세스 생성 |
| 59 | execve | 프로그램 실행 |
| 60 | _exit | 프로세스 종료 |
| 61 | wait4 | 프로세스 종료 대기 |
| 62 | kill | 시그널 전송 |

---

## 8.2 Processes (프로세스)

**프로세스**: 실행 중인 프로그램의 인스턴스. 코드·데이터·스택·레지스터·PC·환경 변수·파일 디스크립터를 포함하는 컨텍스트(context).

프로세스가 애플리케이션에 제공하는 두 가지 핵심 추상화:
1. **독립적 논리 제어 흐름**: 프로세서를 독점 사용하는 것처럼 보임
2. **사설 주소 공간(private address space)**: 메모리를 독점 사용하는 것처럼 보임

### 8.2.1 논리 제어 흐름 & 동시성

```
시간 →
프로세스 A: ████░░░░░░░███████░░
프로세스 B: ░░░░████████░░░░░░░░
프로세스 C: ░░░░░░░░████░░░░░████

(████ = 실행 중, ░░░░ = 중단)
```

- **동시 실행(concurrent)**: A와 B는 시간적으로 겹치면 동시 실행 → A가 시작 후 B가 시작, B가 종료 전 A가 실행되면 동시
- **병렬 실행(parallel)**: 서로 다른 프로세서 코어에서 동시 실행
- **타임슬라이스(time slice)**: 프로세스가 실행되는 각 시간 구간. 멀티태스킹 = 타임슬라이싱

### 8.2.2 사설 주소 공간

```
x86-64 Linux 프로세스 주소 공간:
┌─────────────────────────────────┐ 2^48 - 1
│  Kernel virtual memory          │ (유저 코드 접근 불가)
├─────────────────────────────────┤
│  User stack (런타임 생성, ↓성장) │
│  %rsp                          │
├─────────────────────────────────┤
│  Shared libraries 영역          │
├─────────────────────────────────┤ ← brk
│  Run-time heap (malloc, ↑성장)  │
├─────────────────────────────────┤
│  .data, .bss (R/W)             │
├─────────────────────────────────┤
│  .init, .text, .rodata (R/X)   │
└─────────────────────────────────┘ 0x400000
```

### 8.2.3 사용자 모드와 커널 모드

제어 레지스터의 **모드 비트(mode bit)** 로 구분:

| 모드 | 권한 | 변경 방법 |
|------|------|---------|
| **커널 모드** | 모든 명령어 실행, 모든 메모리 접근 | 예외 발생 시 자동 전환 |
| **유저 모드** | 특권 명령어 불가, 커널 영역 직접 접근 불가 | 시스템 콜/인터럽트/폴트로만 커널 모드 진입 |

유저 모드 프로세스가 커널 데이터에 접근하는 방법: `/proc` 파일 시스템 (예: `/proc/cpuinfo`, `/proc/{pid}/maps`)

### 8.2.4 컨텍스트 스위치 (Context Switch)

커널은 **스케줄러(scheduler)** 를 통해 멀티태스킹을 구현한다:

```
컨텍스트 스위치 과정:
1. 현재 프로세스 A의 컨텍스트 저장 (레지스터, 스택 포인터, PC 등)
2. 이전에 중단된 프로세스 B의 컨텍스트 복원
3. 프로세스 B에 제어 이전

컨텍스트 = 범용 레지스터, PC, FP 레지스터, 상태 레지스터,
           유저 스택, 커널 스택, 페이지 테이블, 프로세스 테이블, 파일 테이블
```

**컨텍스트 스위치 트리거:**
- 시스템 콜 블록 (예: `read` → 디스크 I/O 대기)
- 타이머 인터럽트 (일반적으로 1ms 또는 10ms 주기)

```
프로세스 A      커널         프로세스 B
   │                            │
   │─read()─→│                  │
   │          │ 컨텍스트 스위치  │
   │          │────────────────→│
   │          │                 │ (실행)
   │          │ 디스크 인터럽트  │
   │          │←────────────────│
   │          │ 컨텍스트 스위치  │
   │←─복귀───│                  │
   │ (실행)                     │
```

---

## 8.3 System Call Error Handling

Unix 시스템 레벨 함수는 오류 시 `-1`을 반환하고 전역 변수 `errno`를 설정한다.

**패턴:**
```c
// 직접 처리
if ((pid = fork()) < 0) {
    fprintf(stderr, "fork error: %s\n", strerror(errno));
    exit(0);
}

// 에러 핸들러 함수 사용
void unix_error(char *msg) {
    fprintf(stderr, "%s: %s\n", msg, strerror(errno));
    exit(0);
}

// Stevens 스타일 Wrapper (이 책 전체에서 사용)
pid_t Fork(void) {
    pid_t pid;
    if ((pid = fork()) < 0)
        unix_error("Fork error");
    return pid;
}
// 사용: pid = Fork();  // 한 줄로 간결하게
```

---

## 8.4 Process Control

### 8.4.1 PID 조회

```c
#include <sys/types.h>
#include <unistd.h>

pid_t getpid(void);   // 현재 프로세스의 PID 반환
pid_t getppid(void);  // 부모 프로세스의 PID 반환
```

### 8.4.2 프로세스 생성: fork()

```c
#include <sys/types.h>
#include <unistd.h>

pid_t fork(void);
// 부모: 자식의 PID 반환 (양수)
// 자식: 0 반환
// 오류: -1 반환
```

`fork`의 특이한 동작 — **한 번 호출, 두 번 반환**:

```c
int x = 1;
pid = Fork();           // ← 여기서 분기
if (pid == 0) {
    printf("child: x=%d\n", ++x);  // 출력: child: x=2
    exit(0);
}
printf("parent: x=%d\n", --x);     // 출력: parent: x=0
```

**fork의 4가지 특성:**
1. **한 번 호출, 두 번 반환**: 부모에서 자식 PID, 자식에서 0
2. **동시 실행(concurrent)**: 부모와 자식의 실행 순서는 비결정적
3. **독립된 복사본**: 동일하지만 별도의 주소 공간 (코드·스택·힙·전역 변수 모두 복사)
4. **파일 공유**: 자식은 부모의 열린 파일 디스크립터를 상속

**프로세스 그래프 (process graph):**
```
main()
  │
  fork()
  │     ╲
  │      [자식]
  │        printf("child: x=2") → exit
  │
  printf("parent: x=0") → exit
```

**중첩 fork 예시 (2번 fork → 4개 프로세스):**
```c
Fork(); Fork(); printf("hello\n");
// 4개 프로세스 각각 "hello" 출력 → 총 4번 출력
```

**프로세스 상태:**
| 상태 | 설명 |
|------|------|
| Running | CPU 실행 중 또는 실행 대기 |
| Stopped | SIGSTOP/SIGTSTP/SIGTTIN/SIGTTOU 수신으로 일시 중단; SIGCONT로 재개 |
| Terminated | 영구 중단 (시그널, main 반환, exit 호출) |

### 8.4.3 자식 프로세스 회수: waitpid()

**좀비(zombie) 프로세스**: 종료됐지만 아직 부모가 회수하지 않은 프로세스. 시스템 메모리 낭비. 부모가 먼저 종료되면 `init` 프로세스(PID=1)가 입양하여 회수.

```c
#include <sys/types.h>
#include <sys/wait.h>

pid_t waitpid(pid_t pid, int *statusp, int options);
// 성공: 종료된 자식의 PID
// WNOHANG 시 자식 없으면 0
// 오류: -1

// 간단한 버전
pid_t wait(int *statusp);  // waitpid(-1, statusp, 0)과 동일
```

**pid 인수:**
- `pid > 0`: 해당 PID의 자식만 대기
- `pid == -1`: 모든 자식 중 아무나

**options:**
```c
WNOHANG    // 즉시 반환 (자식 종료 안 됐으면 0 반환)
WUNTRACED  // 종료 또는 정지(stopped)된 자식에도 반환
WCONTINUED // SIGCONT로 재개된 자식에도 반환
// 조합 가능: WNOHANG|WUNTRACED
```

**종료 상태 매크로 (statusp가 non-NULL일 때):**
```c
WIFEXITED(status)    // 정상 종료(exit/return)?
WEXITSTATUS(status)  // 정상 종료 시 exit status 반환
WIFSIGNALED(status)  // 시그널로 종료?
WTERMSIG(status)     // 종료 시그널 번호
WIFSTOPPED(status)   // 현재 정지 상태?
WSTOPSIG(status)     // 정지 시그널 번호
WIFCONTINUED(status) // SIGCONT로 재개?
```

**사용 예 — 순서 무관 회수:**
```c
while ((pid = waitpid(-1, &status, 0)) > 0) {
    if (WIFEXITED(status))
        printf("child %d exited with status %d\n", pid, WEXITSTATUS(status));
    else
        printf("child %d terminated abnormally\n", pid);
}
if (errno != ECHILD)  // 자식이 없어서 종료된 경우만 정상
    unix_error("waitpid error");
```

### 8.4.4 sleep / pause

```c
unsigned int sleep(unsigned int secs);  // secs초 동안 일시 중단. 남은 시간 반환 (시그널로 조기 깨어날 수 있음)
int pause(void);                         // 시그널을 받을 때까지 잠 (-1 반환)
```

### 8.4.5 execve()

```c
#include <unistd.h>

int execve(const char *filename, const char *argv[], const char *envp[]);
// 성공 시 반환 안 함; 오류 시 -1
```

**특성:**
- **한 번 호출, 반환 없음** (fork와 대조: 한 번 호출, 두 번 반환)
- 현재 프로세스의 주소 공간을 완전히 덮어씀 (PID는 동일하게 유지)
- 부모로부터 상속된 열린 파일 디스크립터는 유지

**실행 후 스택 레이아웃:**
```
스택 상단 (높은 주소)
  환경 변수 문자열들 (null-terminated)
  argv 문자열들 (null-terminated)
  envp[]  (null-terminated 포인터 배열)  ← %rdx
  argv[]  (null-terminated 포인터 배열)  ← %rsi
  argc                                   ← %rdi
  libc_start_main의 스택 프레임
  main의 미래 스택 프레임
스택 상단 (낮은 주소)
```

**환경 변수 조작:**
```c
char *getenv(const char *name);                               // name=value에서 value 반환
int   setenv(const char *name, const char *newval, int ow);   // 환경 변수 설정
void  unsetenv(const char *name);                             // 환경 변수 삭제
```

### 8.4.6 간단한 Unix Shell 구현

```c
// 핵심 루프
while (1) {
    printf("> ");
    fgets(cmdline, MAXLINE, stdin);
    eval(cmdline);
}

void eval(char *cmdline) {
    char *argv[MAXARGS];
    int bg = parseline(cmdline, argv);  // 인수 파싱, & 여부 확인

    if (!builtin_command(argv)) {       // quit, & 등 내장 명령 처리
        if ((pid = Fork()) == 0) {      // 자식 생성
            execve(argv[0], argv, environ);  // 요청 프로그램 실행
        }
        if (!bg)
            waitpid(pid, &status, 0);   // 포그라운드: 자식 완료 대기
        else
            printf("%d %s", pid, cmdline); // 백그라운드: 즉시 반환
    }
}
```

> **문제**: 이 단순 셸은 백그라운드 자식을 회수하지 않아 좀비가 남는다. 이를 해결하려면 시그널(8.5절)이 필요하다.

---

## 8.5 Signals

시그널(signal)은 커널 또는 다른 프로세스가 이벤트 발생을 알리는 **소프트웨어 인터럽트** 형태의 ECF다. Linux는 30가지 시그널을 지원한다.

```
번호  이름        기본 동작      이벤트
1     SIGHUP      종료          터미널 연결 끊김
2     SIGINT      종료          Ctrl+C
3     SIGQUIT     종료+코어덤프  Ctrl+\
4     SIGILL      종료          불법 명령어
6     SIGABRT     종료+코어덤프  abort() 호출
8     SIGFPE      종료+코어덤프  부동소수점 예외
9     SIGKILL     종료*         프로세스 강제 종료 (차단/무시 불가)
11    SIGSEGV     종료+코어덤프  잘못된 메모리 참조
13    SIGPIPE     종료          읽는 자 없는 파이프에 쓰기
14    SIGALRM     종료          alarm() 타이머
15    SIGTERM     종료          소프트웨어 종료 신호
17    SIGCHLD     무시          자식 프로세스 정지/종료
18    SIGCONT     무시          정지된 프로세스 재개
19    SIGSTOP     정지*         정지 신호 (차단/무시 불가)
20    SIGTSTP     정지          Ctrl+Z
```

### 8.5.1 시그널 용어

- **전송(send)**: 커널이 목적지 프로세스 컨텍스트에 비트를 설정
- **수신(receive)**: 커널이 강제로 프로세스가 시그널에 반응하게 함 (무시/종료/핸들러 실행)
- **대기 중(pending)**: 전송됐지만 아직 수신 안 됨. 동일 타입 시그널은 **큐에 쌓이지 않음** — 한 개만 대기 가능
- **차단(blocked)**: 차단된 시그널은 전송 가능하지만 차단이 해제될 때까지 수신 안 됨

커널은 각 프로세스에 대해 두 개의 비트 벡터를 유지:
- `pending`: 대기 중인 시그널 집합
- `blocked` (signal mask): 차단된 시그널 집합

### 8.5.2 시그널 전송

**프로세스 그룹:**
```c
pid_t getpgrp(void);                  // 현재 프로세스의 그룹 ID
int   setpgid(pid_t pid, pid_t pgid); // 프로세스 그룹 변경
```

**시그널 전송 방법:**

```bash
# /bin/kill 프로그램
/bin/kill -9 15213    # 프로세스 15213에 SIGKILL
/bin/kill -9 -15213   # 프로세스 그룹 15213 전체에 SIGKILL

# 키보드
Ctrl+C → SIGINT  (포그라운드 프로세스 그룹 전체)
Ctrl+Z → SIGTSTP (포그라운드 프로세스 그룹 전체, 정지)
```

```c
// kill() 함수
int kill(pid_t pid, int sig);
// pid > 0: 해당 프로세스에 전송
// pid == 0: 자신의 프로세스 그룹 전체에 전송
// pid < 0: 프로세스 그룹 |pid|에 전송

// alarm() 함수 - 자신에게 SIGALRM 전송
unsigned int alarm(unsigned int secs);
// secs 초 후 커널이 SIGALRM을 호출 프로세스에 전송
// 이전 알람 취소 후 남은 시간 반환
```

### 8.5.3 시그널 수신

커널이 프로세스를 커널 모드 → 유저 모드로 전환할 때 `pending & ~blocked`를 검사:
- 비어있으면: 정상 실행 계속
- 비어있지 않으면: 작은 번호부터 하나를 선택하여 강제 수신

**시그널 핸들러 설치:**
```c
#include <signal.h>
typedef void (*sighandler_t)(int);
sighandler_t signal(int signum, sighandler_t handler);
// handler = SIG_IGN: 시그널 무시
// handler = SIG_DFL: 기본 동작 복원
// handler = 함수 포인터: 해당 함수를 핸들러로 설치

// 예시
void sigint_handler(int sig) {
    printf("Caught SIGINT!\n");
    exit(0);
}
signal(SIGINT, sigint_handler);
```

핸들러는 다른 핸들러에 의해 중단될 수 있다 (nested handlers 가능).

### 8.5.4 시그널 차단 및 해제

```c
#include <signal.h>

// sigset_t 조작
int sigemptyset(sigset_t *set);               // 집합 초기화 (비움)
int sigfillset(sigset_t *set);                // 모든 시그널 추가
int sigaddset(sigset_t *set, int signum);     // 시그널 추가
int sigdelset(sigset_t *set, int signum);     // 시그널 제거
int sigismember(const sigset_t *set, int signum); // 멤버 여부

// 차단 집합 변경
int sigprocmask(int how, const sigset_t *set, sigset_t *oldset);
// how = SIG_BLOCK:   blocked |= set
// how = SIG_UNBLOCK: blocked &= ~set
// how = SIG_SETMASK: blocked = set

// 예시: SIGINT 일시 차단
sigset_t mask, prev;
sigemptyset(&mask);
sigaddset(&mask, SIGINT);
sigprocmask(SIG_BLOCK, &mask, &prev);     // SIGINT 차단
/* ... 인터럽트 없는 임계 구역 ... */
sigprocmask(SIG_SETMASK, &prev, NULL);    // 이전 마스크 복원
```

### 8.5.5 안전한 시그널 핸들러 작성

핸들러는 메인 프로그램과 동시에 실행되며 전역 변수를 공유한다. 다섯 가지 가이드라인:

**G0. 핸들러를 최대한 단순하게 유지**
```c
// 권장 패턴: 플래그 설정만 하고 처리는 메인에서
volatile sig_atomic_t flag = 0;
void handler(int sig) { flag = 1; }  // 핸들러는 이게 전부
// 메인: while (!flag) pause(); /* 처리 */
```

**G1. 핸들러에서 async-signal-safe 함수만 호출**
- 안전한 함수: `write`, `_exit`, `fork`, `execve`, `kill`, `sleep`, `waitpid` 등
- **안전하지 않은 함수**: `printf`, `sprintf`, `malloc`, `exit` → 핸들러에서 사용 금지
- 대신 `sio_puts()` (write 기반 Safe I/O 패키지) 사용

```c
// 안전한 핸들러
void sigint_handler(int sig) {
    sio_puts("Caught SIGINT!\n");  // write 기반 → 안전
    _exit(0);                       // exit 아닌 _exit → 안전
}
```

**G2. errno 저장 및 복원**
```c
void handler(int sig) {
    int saved_errno = errno;  // 진입 시 저장
    /* ... 처리 ... */
    errno = saved_errno;       // 반환 전 복원
}
```

**G3. 공유 전역 자료구조 접근 시 모든 시그널 차단**
```c
sigset_t mask_all;
sigfillset(&mask_all);
sigprocmask(SIG_BLOCK, &mask_all, &prev);
/* shared_data 접근 */
sigprocmask(SIG_SETMASK, &prev, NULL);
```

**G4. 전역 변수에 `volatile` 선언**
```c
volatile int g;  // 컴파일러가 레지스터 캐시 못 하도록 강제
```

**G5. 플래그 변수에 `sig_atomic_t` 사용**
```c
volatile sig_atomic_t flag;  // 읽기/쓰기가 원자적으로 보장
// 단, flag++ 같은 복합 연산은 원자적 보장 없음
```

### 시그널은 큐에 쌓이지 않는다

**핵심 원칙**: pending 비트 벡터에는 타입당 1비트뿐 → 동일 타입 시그널이 여러 번 전송돼도 최대 1개만 대기.

```c
// 잘못된 핸들러 (시그널이 큐에 쌓인다고 잘못 가정)
void handler1(int sig) {
    waitpid(-1, NULL, 0);  // 자식 1개만 회수 → 나머지 좀비로 남을 수 있음
}

// 올바른 핸들러 (모든 좀비 자식 회수)
void handler2(int sig) {
    int olderrno = errno;
    while (waitpid(-1, NULL, 0) > 0)  // 가능한 모든 자식 회수
        sio_puts("Handler reaped child\n");
    if (errno != ECHILD)
        sio_error("waitpid error");
    errno = olderrno;
}
```

### 8.5.6 동기화 레이스(Race) 방지

**문제**: `fork` 후 자식이 종료되어 SIGCHLD가 `addjob` 전에 도착하면 `deletejob`이 먼저 호출됨.

```c
// 버그 있는 코드
if ((pid = Fork()) == 0) { execve(...); }
addjob(pid);  // 자식이 이미 종료됐다면? → deletejob이 먼저 호출됨!
```

**해결책: `fork` 전에 SIGCHLD 차단**
```c
sigset_t mask_one, mask_all, prev_one;
sigemptyset(&mask_one);
sigaddset(&mask_one, SIGCHLD);
sigfillset(&mask_all);

Sigprocmask(SIG_BLOCK, &mask_one, &prev_one);  // SIGCHLD 차단
if ((pid = Fork()) == 0) {
    Sigprocmask(SIG_SETMASK, &prev_one, NULL);  // 자식: SIGCHLD 해제
    execve(...);
}
// 부모: addjob 완료 보장
Sigprocmask(SIG_BLOCK, &mask_all, NULL);
addjob(pid);
Sigprocmask(SIG_SETMASK, &prev_one, NULL);      // SIGCHLD 해제
```

### 8.5.7 sigsuspend — 시그널 원자적 대기

```c
int sigsuspend(const sigset_t *mask);
// = atomic { sigprocmask(SIG_SETMASK, mask, &prev); pause(); sigprocmask(SIG_SETMASK, &prev, NULL); }
// 시그널 수신 시 핸들러 실행 후 복귀; 항상 -1 반환
```

**스핀루프(busy waiting) 대신 sigsuspend 사용:**
```c
// 잘못된 방법: 레이스 조건
while (!pid) pause();       // SIGCHLD가 pause 전에 오면 영원히 잠

// 올바른 방법: sigsuspend
sigset_t mask, prev;
sigemptyset(&mask);
sigaddset(&mask, SIGCHLD);

Sigprocmask(SIG_BLOCK, &mask, &prev);   // SIGCHLD 차단
if (Fork() == 0) exit(0);
pid = 0;
while (!pid)
    sigsuspend(&prev);  // SIGCHLD 해제 + 잠 (원자적) → 핸들러 실행 후 복귀
Sigprocmask(SIG_SETMASK, &prev, NULL);
```

---

## 8.6 Nonlocal Jumps (비지역 점프)

C의 응용 레벨 ECF. 일반 call/return 스택 규율을 우회하여 현재 실행 중인 함수에서 다른 함수로 직접 점프.

```c
#include <setjmp.h>

int  setjmp(jmp_buf env);               // 현재 환경(PC, SP, 레지스터) 저장 → 0 반환
void longjmp(jmp_buf env, int retval);  // env로 복원 → setjmp가 retval 반환

// 시그널 핸들러용 버전 (pending/blocked 벡터도 저장)
int  sigsetjmp(sigjmp_buf env, int savesigs);
void siglongjmp(sigjmp_buf env, int retval);
```

**특성:**
- `setjmp`: **한 번 호출, 여러 번 반환** (첫 호출: 0 반환, 각 longjmp 호출: retval 반환)
- `longjmp`: **한 번 호출, 반환 없음**

**사용 사례 1 — 깊은 중첩 함수에서의 에러 복구:**
```c
jmp_buf buf;

int main() {
    switch (setjmp(buf)) {
    case 0: foo(); break;                           // 정상 실행
    case 1: printf("Error in foo\n"); break;
    case 2: printf("Error in bar\n"); break;
    }
}

void foo() { if (error1) longjmp(buf, 1); bar(); }
void bar() { if (error2) longjmp(buf, 2); }
// longjmp가 중간 스택 프레임 전체를 건너뜀
// 주의: 중간 함수에서 할당된 자원 해제 코드가 건너뛰어짐 → 메모리 누수 가능
```

**사용 사례 2 — 시그널 핸들러에서 메인으로 점프:**
```c
sigjmp_buf buf;

void handler(int sig) {
    siglongjmp(buf, 1);  // 핸들러에서 main으로 직접 점프
}

int main() {
    if (!sigsetjmp(buf, 1)) {  // 처음 실행
        Signal(SIGINT, handler);
        sio_puts("starting\n");
    } else {                   // Ctrl+C 후 재시작
        sio_puts("restarting\n");
    }
    while (1) {
        sleep(1);
        sio_puts("processing...\n");
    }
}
// 출력: starting → processing... → Ctrl+C → restarting → processing...
```

> **주의**: 핸들러 설치는 `sigsetjmp` 이후에. `siglongjmp`는 async-signal-safe 목록에 없으므로 도달 가능한 코드에서 안전 함수만 호출할 것.

**C++ / Java 예외와의 관계:**
- `try` ≈ `setjmp`
- `throw` ≈ `longjmp`
- `catch` ≈ `setjmp` 반환값 검사

---

## 8.7 Tools for Manipulating Processes

| 도구 | 기능 |
|------|------|
| `strace` | 실행 중 프로그램의 시스템 콜 추적 출력 (`-static` 옵션 권장) |
| `ps` | 현재 시스템의 프로세스 목록 (좀비 포함) |
| `top` | 프로세스 리소스 사용 정보 실시간 출력 |
| `pmap` | 프로세스 메모리 맵 표시 |
| `/proc` | 커널 데이터 구조를 ASCII 텍스트로 노출하는 가상 파일 시스템 |

```bash
strace -o trace.txt ./prog   # 시스템 콜 추적
ps aux                       # 모든 프로세스 표시
cat /proc/cpuinfo            # CPU 정보
cat /proc/$$/maps            # 현재 프로세스 메모리 맵
```

---

## 핵심 개념 정리

```
ECF 계층:
  하드웨어: Interrupt(비동기) / Trap(syscall) / Fault(재실행) / Abort(종료)
  OS: Context switch, fork/exec/wait/exit
  애플리케이션: Signal, setjmp/longjmp

fork():
  한 번 호출, 두 번 반환
  부모: 자식 PID 반환, 자식: 0 반환
  독립된 주소 공간 (복사)
  열린 파일 디스크립터 공유

waitpid():
  zombie 회수
  WNOHANG | WUNTRACED 옵션
  WIFEXITED/WEXITSTATUS 상태 확인

execve():
  한 번 호출, 반환 없음
  현재 프로세스 주소 공간 대체 (PID 유지)

Signal():
  pending/blocked 비트 벡터
  시그널은 큐에 쌓이지 않음 → 핸들러에서 while(waitpid>0) 패턴 사용
  안전한 핸들러: async-safe 함수, errno 보존, volatile/sig_atomic_t

Race 방지:
  fork 전 SIGCHLD 차단 → addjob 후 해제
  대기 시 sigsuspend (pause와 달리 원자적)

setjmp/longjmp:
  setjmp: 한 번 호출, 여러 번 반환
  longjmp: 한 번 호출, 반환 없음
  C++ try/throw/catch의 저수준 구현 기반
```

---

## 요약

| 주제 | 핵심 |
|------|------|
| 예외 4종 | Interrupt(비동기·I/O) / Trap(syscall) / Fault(재실행) / Abort(종료) |
| 시스템 콜 | syscall 명령어, %rax=번호, 레지스터로 인수 전달 |
| 프로세스 | 논리 흐름 + 사설 주소 공간, 컨텍스트 스위치로 멀티태스킹 |
| fork | 한 번 호출·두 번 반환, 독립 복사본, 동시 실행 |
| waitpid | 좀비 회수, WNOHANG·상태 매크로 |
| execve | 한 번 호출·반환 없음, 주소 공간 대체 |
| 시그널 | 소프트웨어 인터럽트, pending/blocked 비트벡터, 큐 없음 |
| 안전 핸들러 | async-safe, errno 저장, volatile/sig_atomic_t |
| 레이스 방지 | fork 전 SIGCHLD 차단, sigsuspend 사용 |
| nonlocal jump | setjmp(저장)/longjmp(복원), C++ 예외의 저수준 기반 |
