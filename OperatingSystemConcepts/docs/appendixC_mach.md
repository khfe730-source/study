# Appendix C: The Mach System

## C.1 Mach 소개

### 배경과 목표

Mach는 1985년 Richard Rashid(현 Microsoft Research VP) 주도로 Carnegie Mellon University에서 시작된 마이크로커널 프로젝트다. 4.3BSD를 기반으로 시작했으나 점차 완전히 새로운 설계로 발전했다.

```
개발 동기:
1. 멀티프로세서 지원: SMP, NUMA, 이기종 멀티프로세서
2. 분산 시스템: 메시지 패싱 기반으로 분산 OS 구현 용이
3. OS 이식성: 하드웨어 추상화로 다양한 아키텍처 지원
4. 확장성: 마이크로커널 위에 다양한 OS 인터페이스 구현

Mach 계보:
Mach 1.0 (1986): 4.3BSD 기반, BSD + Mach 공존
Mach 2.0 (1987): VM 서브시스템 독립화, 4.4BSD에 통합
Mach 3.0 (1990): 순수 마이크로커널 (BSD는 사용자 공간 서버)

상용화:
- NeXTSTEP → macOS/iOS (XNU = Mach + BSD)
- OSF/1 (DEC Alpha) → Tru64 UNIX
- GNU Hurd (미완성)
- MkLinux (Linux on Mach)
```

### 핵심 철학

```
"OS의 정책을 커널 밖으로 내보낸다"

마이크로커널에 남기는 것 (최소한):
  - 메시지 패싱 (IPC)
  - 가상 메모리 관리
  - 태스크/스레드 추상화
  - 포트(Port) 기반 이름 공간

커널 밖으로 내보내는 것 (사용자 서버):
  - 파일 시스템
  - 네트워크 스택
  - 장치 드라이버 (일부)
  - POSIX 인터페이스
```

---

## C.2 핵심 추상화

Mach는 5가지 핵심 추상화를 제공한다:

```
1. Task      : 주소 공간 + 포트 권리 집합
2. Thread    : Task 내의 실행 흐름
3. Port      : 통신 채널 엔드포인트
4. Message   : 포트를 통해 전송되는 데이터 단위
5. Memory Object: 가상 메모리의 백킹 스토어 추상화
```

---

## C.3 Task와 Thread

### Task

```c
// Task: UNIX 프로세스와 유사하지만 더 일반적
task_t task;
kern_return_t kr = task_create(
    mach_task_self(),   // 부모 태스크
    FALSE,              // 상속 여부 (FALSE = 새 주소 공간)
    &task
);

// Task 속성:
// - 독립적인 가상 주소 공간
// - 포트 네임스페이스 (port right set)
// - CPU 시간 할당
// - 예외 핸들러
```

### Thread

```c
// Thread: Task 내의 실행 스트림
thread_t thread;
kr = thread_create(task, &thread);
kr = thread_resume(thread);  // 생성 후 suspend 상태

// 스레드 속성:
// - 실행 상태 (레지스터, PC, SP)
// - 스택
// - 예외 포트
// - 스케줄링 우선순위

// 스레드 상태 조작 (디버깅, 이주에 활용):
mach_msg_type_number_t count = x86_THREAD_STATE64_COUNT;
x86_thread_state64_t state;
thread_get_state(thread, x86_THREAD_STATE64,
                 (thread_state_t)&state, &count);
```

### Task 계층과 상속

```
부모 Task
  │
  ├── fork() 시:
  │     새 Task 생성 (새 주소 공간)
  │     부모 포트 권리 일부 상속 가능
  │
  └── Thread 생성:
        같은 Task, 같은 주소 공간
        새로운 실행 흐름

Thread 풀 vs 새 Task:
- IPC 비용: 같은 Task 내 스레드 > 다른 Task (마이크로커널에서는 IPC가 느릴 수 있음)
- XNU에서는 Mach IPC를 최적화해 오버헤드 최소화
```

---

## C.4 포트 (Port)

포트는 Mach IPC의 핵심 개념으로, 단방향 메시지 큐의 커널 관리 엔드포인트다.

### 포트의 특성

```
포트 = 커널이 관리하는 보호된 메시지 큐

특성:
- 커널 내부에 존재 (사용자는 "port right"만 보유)
- 단방향: Send 권리 → 포트 → Receive 권리
- 용량 유한: 큐가 가득 차면 송신자 블록
- 보호: Receive 권리 보유자만 메시지 수신 가능
```

### 포트 권리 (Port Rights)

```
권리 종류:
MACH_PORT_RIGHT_RECEIVE  - 메시지 수신 (오직 한 태스크만 보유)
MACH_PORT_RIGHT_SEND     - 메시지 송신 (여러 태스크 공유 가능)
MACH_PORT_RIGHT_SEND_ONCE - 한 번만 송신 가능 (reply port에 사용)
MACH_PORT_RIGHT_DEAD_NAME - 포트가 소멸됨을 표시
MACH_PORT_RIGHT_PORT_SET  - 포트 집합 (여러 포트를 하나로)

권리 이전:
- 권리 자체를 메시지에 담아 전달 가능
- 커널이 포트 이름 변환 처리
  (전달 전: 보내는 Task의 이름 → 전달 후: 받는 Task의 새 이름)
```

### 포트 생성과 사용

```c
// 포트 생성 (Receive 권리 획득)
mach_port_t port;
mach_port_allocate(mach_task_self(),
                   MACH_PORT_RIGHT_RECEIVE,
                   &port);

// Send 권리 삽입 (자신에게)
mach_port_insert_right(mach_task_self(),
                       port, port,
                       MACH_MSG_TYPE_MAKE_SEND);

// 메시지 수신
mach_msg(&msg.header,
         MACH_RCV_MSG,
         0,           // 송신 크기 (수신이므로 0)
         sizeof(msg),
         port,
         MACH_MSG_TIMEOUT_NONE,
         MACH_PORT_NULL);
```

### 특수 포트

```
Task Special Ports:
  TASK_KERNEL_PORT     - 태스크 자신에 대한 포트 (mach_task_self())
  TASK_HOST_PORT       - 호스트 포트 (시스템 정보)
  TASK_BOOTSTRAP_PORT  - 서비스 이름 등록/조회

Thread Special Ports:
  THREAD_KERNEL_PORT   - 스레드 자신 (mach_thread_self())
  THREAD_EXCEPTION_PORT - 예외 처리 포트

Host Special Ports:
  HOST_PORT            - 비특권 호스트 정보
  HOST_PRIV_PORT       - 특권 호스트 작업 (루트 전용)

Bootstrap Server:
  launchd(macOS)가 bootstrap server 역할
  → 서비스 이름 → 포트 매핑 (이름 서버)
  bootstrap_look_up(bootstrap_port, "com.apple.securityd", &port)
```

---

## C.5 메시지 패싱 (Message Passing)

### 메시지 구조

```c
// Mach 메시지 헤더
typedef struct mach_msg_header_t {
    mach_msg_bits_t   msgh_bits;       // 메시지 타입, 포트 권리 타입
    mach_msg_size_t   msgh_size;       // 전체 메시지 크기
    mach_port_t       msgh_remote_port;// 목적지 포트
    mach_port_t       msgh_local_port; // 송신자 포트 (reply port)
    mach_msg_voucher_t msgh_voucher_port;
    mach_msg_id_t     msgh_id;         // 메시지 ID (서비스 선택자)
} mach_msg_header_t;

// 실제 사용 예:
typedef struct {
    mach_msg_header_t header;
    // 바디 (인라인 데이터 또는 descriptor)
    int               data;
    char              name[64];
} MyMessage;
```

### mach_msg() 트랩

```c
kern_return_t mach_msg(
    mach_msg_header_t *msg,        // 메시지 버퍼
    mach_msg_option_t  option,     // SEND, RECEIVE, SEND+RECEIVE
    mach_msg_size_t    send_size,  // 송신 크기
    mach_msg_size_t    rcv_size,   // 수신 버퍼 크기
    mach_port_t        rcv_name,   // 수신할 포트
    mach_msg_timeout_t timeout,    // MACH_MSG_TIMEOUT_NONE
    mach_port_t        notify      // 알림 포트
);

// 옵션 조합:
MACH_SEND_MSG                  // 메시지 송신
MACH_RCV_MSG                   // 메시지 수신
MACH_SEND_MSG | MACH_RCV_MSG   // RPC 패턴 (송신 후 즉시 수신)
```

### 메시지 전달 최적화

```
소형 메시지 (인라인):
  송신자 → 커널 복사 → 수신자 복사 (2회 복사)
  
대형 메시지 (Out-of-Line = OOL):
  mach_msg_ool_descriptor_t 사용
  → 가상 메모리 리매핑 (COW)
  → 물리 복사 없음! (페이지 테이블 조작만)
  
  send: 송신자 페이지 → 공유 매핑 생성
  recv: 수신자 주소 공간에 페이지 매핑
  
  대용량 데이터 (수MB)도 O(1) 비용에 전달 가능
```

### MIG (Mach Interface Generator)

```
MIG: Mach의 RPC 인터페이스 정의 언어 + 코드 생성기

.defs 파일 → MIG → client stub + server stub (C 코드)

예: 파일시스템 서버 인터페이스
subsystem filesystem 1000;

routine fs_open(
    server : mach_port_t;
    path   : string_t;
    flags  : int;
    out fd : int;
    out kr : kern_return_t
);

→ 컴파일 시 fs_open() 클라이언트 함수와
  server dispatch 루틴 자동 생성
  
macOS의 System Call 상당수가 MIG로 구현됨
(예: vm_allocate, task_info, thread_get_state 등)
```

---

## C.6 가상 메모리 (Virtual Memory)

Mach의 VM 서브시스템은 가장 혁신적인 부분 중 하나로, 현재 macOS/iOS VM의 직접 원형이다.

### 핵심 추상화

```
vm_map: 가상 주소 공간
  └── vm_map_entry (범위, 보호, 공유 여부)
       └── vm_object (메모리 오브젝트 참조)
            └── vm_page (물리 페이지)

vm_object: 메모리의 백킹 스토어 추상화
  - 파일 (vnode object)
  - 익명 메모리 (anon object)
  - 디바이스 메모리
  - 외부 메모리 오브젝트 (사용자 공간 페이저)
```

### COW (Copy-on-Write) 구현

```
fork() 시 VM 복사:

부모 vm_map:      자식 vm_map:
  [entry A] ──────── [entry A'] 
       ↓                  ↓
    vm_object(cow) ← 공유! (reference count = 2)
       ↓
    물리 페이지들 (모두 read-only로 보호)

쓰기 시도 → 페이지 폴트 → 폴트 핸들러:
  1. vm_object에서 해당 페이지 찾음
  2. refcount > 1 → COW 필요
  3. 새 vm_object 생성
  4. 새 물리 페이지 할당 + 내용 복사
  5. 페이지 테이블 업데이트 (쓰기 허용)
```

### 외부 페이저 (External Pager)

Mach의 가장 독특한 기능: 페이지 인/아웃 정책을 사용자 공간 서버에 위임

```
외부 페이저 프로토콜:

[애플리케이션]         [커널]           [외부 페이저 서버]
     │                   │                    │
  vm_map() ──────────→  │                    │
  (with pager port)      │ memory_object_init──→
                         │                    │ (초기화)
  페이지 접근 →          │                    │
  [페이지 폴트]          │ memory_object_data_request──→
                         │                    │
                         │ ←── memory_object_data_provided ──
                         │ (데이터 수신)      │
  [정상 실행 재개] ←─────│                    │

활용:
- 파일 시스템 서버: 파일 페이지 관리
- 데이터베이스: 직접 페이지 캐시 제어
- 분산 공유 메모리: 원격 노드에서 페이지 fetch
```

### vm_allocate / vm_map

```c
// 주소 공간 예약
vm_address_t addr = 0;
vm_allocate(mach_task_self(),
            &addr,
            4096,          // 크기
            VM_FLAGS_ANYWHERE); // 아무 위치에

// 다른 태스크 주소 공간에 매핑 (디버거에서 활용)
vm_map(target_task,
       &addr,
       size,
       0,      // mask
       VM_FLAGS_ANYWHERE,
       src_object,
       offset,
       FALSE,  // copy = FALSE → 공유 매핑
       VM_PROT_READ | VM_PROT_WRITE,
       VM_PROT_ALL,
       VM_INHERIT_DEFAULT);

// 보호 속성 변경
vm_protect(mach_task_self(), addr, size,
           FALSE,                        // 최대 보호 변경 아님
           VM_PROT_READ | VM_PROT_EXECUTE); // 실행 가능으로
```

---

## C.7 예외 처리 (Exception Handling)

Mach의 예외 처리는 포트 기반으로 설계되어 디버거 구현에 매우 강력하다.

```
예외 발생 흐름:

스레드에서 예외 발생 (예: SIGSEGV)
    │
    ▼
1. Thread exception port 확인 → 있으면 메시지 전송
2. Task exception port 확인   → 있으면 메시지 전송
3. Host exception port 확인   → 있으면 메시지 전송
4. 아무도 없으면 → 스레드/태스크 종료

예외 타입:
EXC_BAD_ACCESS     - 잘못된 메모리 접근 (SIGSEGV/SIGBUS)
EXC_BAD_INSTRUCTION- 잘못된 명령 (SIGILL)
EXC_ARITHMETIC     - 산술 오류 (SIGFPE)
EXC_BREAKPOINT     - 브레이크포인트 (SIGTRAP)
EXC_SOFTWARE       - 소프트웨어 예외
EXC_CRASH          - 충돌

디버거 구현:
  task_set_exception_ports(target_task,
                           EXC_MASK_ALL,
                           debugger_port,  // 예외를 여기로!
                           EXCEPTION_DEFAULT,
                           THREAD_STATE_NONE);
  
  → 예외 발생 시 디버거 포트로 메시지 수신
  → 디버거가 처리 후 스레드 재개 또는 종료 결정

macOS lldb, gdb가 이 메커니즘을 사용
```

---

## C.8 스케줄링

### 멀티프로세서 지원

```
Mach 설계 시 핵심 목표 중 하나가 SMP 지원:

- 각 CPU마다 독립적인 런큐
- 프로세서 셋 (Processor Set): CPU를 그룹으로 관리
- 태스크를 특정 프로세서 셋에 배정
- 친화성(affinity) 힌트 지원

현대 XNU(macOS):
- 대칭형 멀티스레딩(SMT) 인식
- NUMA 인식
- Thermal 압력 기반 스케줄링 (모바일)
- QoS 기반 우선순위 (App Nap, Background Execution)
```

### QoS (Quality of Service) 클래스 - macOS/iOS

```
Mach 스케줄링 우선순위 위에 XNU가 추가한 추상화:

User Interactive    (최고): UI 갱신, 애니메이션
User Initiated      (높음): 사용자 요청 즉시 작업
Default             (기본): 명시적 지정 없음
Utility             (낮음): 백그라운드 다운로드
Background          (최저): 백업, 동기화
Unspecified

GCD (Grand Central Dispatch) 통합:
dispatch_async(dispatch_get_global_queue(QOS_CLASS_BACKGROUND, 0), ^{
    // 배경 작업
});
```

---

## C.9 XNU: Mach + BSD 하이브리드

macOS/iOS의 커널 XNU(X is Not Unix)는 Mach 위에 BSD 레이어를 쌓은 하이브리드 구조다.

```
XNU 아키텍처:

사용자 공간
─────────────────────────────────────────────────────
POSIX API (libSystem.dylib)
─────────────────────────────────────────────────────
        BSD 레이어 (커널 공간)
  ┌──────────────────────────────────────────────────┐
  │  POSIX syscall → BSD proc/fd/socket/vfs         │
  │  BSD 네트워킹 (TCP/IP, BPF, PF)                │
  │  VFS → HFS+/APFS/NFS...                        │
  └──────────────────────────────────────────────────┘
        Mach 레이어 (커널 공간)
  ┌──────────────────────────────────────────────────┐
  │  Task/Thread  │  IPC/Port   │  VM               │
  │  스케줄러     │  예외 처리  │  Zones (메모리)   │
  └──────────────────────────────────────────────────┘
        I/O Kit (C++ 드라이버 프레임워크)
─────────────────────────────────────────────────────
        하드웨어 (ARM, x86)

핵심: BSD와 Mach는 별도 서버가 아닌 같은 커널 주소 공간에 공존
→ 순수 마이크로커널 Mach와 달리 IPC 오버헤드 없음
→ 실용적 성능 확보
```

### Mach Trap vs BSD Syscall

```
macOS에서 syscall 두 종류 공존:

Mach Trap (음수 syscall 번호):
  __mach_msg_trap       (-31)
  __mach_reply_port     (-26)
  task_self_trap        (-28)

BSD Syscall (양수):
  read   (3)
  write  (4)
  open   (5)
  fork   (2)
  ...

// XNU syscall 분기:
switch (code & SYSCALL_CLASS_MASK) {
  case SYSCALL_CLASS_MACH: → Mach trap 테이블
  case SYSCALL_CLASS_UNIX: → BSD syscall 테이블
  case SYSCALL_CLASS_MDEP: → 머신 의존 (예: thread_fast_set_cthread_self)
}
```

---

## C.10 현대적 의의

### 순수 마이크로커널의 성능 문제

```
Mach 3.0 (순수 마이크로커널)의 성능 측정 (1990년대):
  UNIX syscall 하나 = Mach IPC + 사용자 서버 처리 + IPC 응답
  → 4BSD 대비 50~100% 느린 성능
  
원인:
1. IPC 오버헤드: 컨텍스트 스위치 × 2 + 메시지 복사
2. 데이터 복사: 커널 ↔ 사용자 공간 서버
3. 캐시 오염: 자주 있는 컨텍스트 스위치

→ macOS는 BSD를 Mach와 같은 주소 공간에 배치해 해결
  (Colocated BSD Server: 사실상 모놀리식 구조)
```

### 마이크로커널 르네상스: L4 계열

```
Mach의 교훈을 바탕으로 설계된 2세대 마이크로커널:

L4 (Jochen Liedtke, 1993):
  - IPC가 모든 것의 핵심 → IPC를 극도로 최적화
  - 초기 L4: IPC 100ns 달성 (Mach의 100배 빠름)
  - "마이크로커널은 느리다"는 신화 타파

L4 계열:
  seL4  - 수학적으로 검증된 마이크로커널
  OKL4  - Qualcomm 모뎀 칩에 사용
  Fiasco.OC - TUD-OS

seL4 (2009, NICTA/UNSW):
  - 최초로 형식 검증(formal verification)된 OS 커널
  - 7500줄 C 코드 + 200,000줄 Isabelle/HOL 증명
  - 항공우주, 자동차, 군사 용도로 채택

Google Fuchsia:
  - Zircon 마이크로커널 (L4 영향)
  - Mach와 POSIX에서 영향받은 핸들 기반 IPC
  - Android 대체를 목표로 개발 중
```

---

## 요약

```
Mach의 핵심 기여와 현대적 유산:
┌────────────────────────────────────────────────────────┐
│ 포트/메시지 IPC  → 분산 OS, 서비스 지향 아키텍처       │
│ 외부 페이저      → 사용자 공간 파일시스템, FUSE         │
│ COW VM           → 모든 현대 OS의 fork() 구현           │
│ OOL 메시지       → 제로카피 IPC의 원형                  │
│ Task/Thread 분리 → POSIX thread와 process 통합 모델    │
│ 예외 포트        → 디버거 API (ptrace 대안)             │
│ XNU              → macOS/iOS (수십억 대 기기)           │
│ seL4             → 형식 검증 OS의 실용적 구현           │
│ 마이크로커널 교훈→ "IPC 비용이 설계를 지배한다"        │
└────────────────────────────────────────────────────────┘
```
