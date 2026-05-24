# Chapter 21: Windows 10

## 21.1 Windows 역사와 설계 목표

### 역사적 흐름

```
MS-DOS (1981)         - 단일 태스크, 16비트
Windows 1.0 (1985)   - MS-DOS 위 GUI 셸
Windows 3.x (1990)   - 협력적 멀티태스킹
Windows NT 3.1 (1993)- 완전한 새 커널, 선점형 멀티태스킹, 32비트
Windows NT 4.0 (1996)- GUI를 커널로 이동 (GDI/USER → Ring 0)
Windows 2000 (2000)  - Active Directory, 플러그앤플레이
Windows XP (2001)    - NT+Consumer 통합
Windows Vista (2006) - UAC, 드라이버 서명, ASLR
Windows 7 (2009)     - 성능 개선
Windows 8 (2012)     - 모던 UI, ARM 지원
Windows 10 (2015)    - 서비스형 OS, Hyper-V, WSL
Windows 11 (2021)    - TPM 2.0, 재설계된 UI
```

### 설계 목표

1. **확장성 (Extensibility)**: 계층적 구조, 마이크로커널적 요소
2. **이식성 (Portability)**: HAL을 통한 하드웨어 추상화 (x86, x64, ARM)
3. **신뢰성 (Reliability)**: 드라이버 격리, 구조화된 예외 처리
4. **호환성 (Compatibility)**: Win32 API, POSIX, OS/2 서브시스템 (현재는 POSIX만 WSL로)
5. **보안 (Security)**: SID 기반 접근 제어, UAC, 코드 서명
6. **성능 (Performance)**: NTFS, 비동기 I/O, NUMA 인식

---

## 21.2 아키텍처

### 계층 구조

```
┌─────────────────────────────────────────────────────────┐
│              사용자 모드 (User Mode)                     │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐   │
│  │Win32 앱  │ │.NET 앱   │ │WSL 앱    │ │UWP 앱    │   │
│  └────┬─────┘ └────┬─────┘ └────┬─────┘ └────┬─────┘   │
│       │            │            │             │         │
│  ┌────┴────────────┴────────────┴─────────────┴──────┐  │
│  │         서브시스템 DLL (kernel32.dll, ntdll.dll)   │  │
│  └─────────────────────┬──────────────────────────────┘  │
│                        │ NT Native API (NtCreateFile 등) │
├────────────────────────┼────────────────────────────────┤
│              커널 모드 (Kernel Mode)                     │
│  ┌─────────────────────┴──────────────────────────────┐  │
│  │  Executive (실행기)                                  │  │
│  │  ┌─────────┐ ┌─────────┐ ┌──────────┐ ┌────────┐  │  │
│  │  │Process  │ │Memory   │ │I/O       │ │Cache   │  │  │
│  │  │Manager  │ │Manager  │ │Manager   │ │Manager │  │  │
│  │  ├─────────┤ ├─────────┤ ├──────────┤ ├────────┤  │  │
│  │  │Object   │ │Security │ │PnP       │ │Power   │  │  │
│  │  │Manager  │ │Reference│ │Manager   │ │Manager │  │  │
│  │  └─────────┘ └─────────┘ └──────────┘ └────────┘  │  │
│  └────────────────────────────────────────────────────┘  │
│  ┌─────────────────────────────────────────────────────┐  │
│  │  Windows Kernel (Ntoskrnl.exe - 하위 레벨)           │  │
│  │  스케줄러, 인터럽트/예외 처리, 동기화 프리미티브     │  │
│  └─────────────────────────────────────────────────────┘  │
│  ┌─────────────────────────────────────────────────────┐  │
│  │  HAL (Hardware Abstraction Layer)                    │  │
│  └─────────────────────────────────────────────────────┘  │
├─────────────────────────────────────────────────────────┤
│                    하드웨어                              │
└─────────────────────────────────────────────────────────┘
```

### Executive 구성 요소

| 컴포넌트 | 역할 |
|----------|------|
| **Object Manager** | 모든 커널 객체(파일, 프로세스, 스레드, 뮤텍스 등)의 생성·이름·수명 관리 |
| **Process Manager** | 프로세스/스레드 생성·종료, Job 객체 |
| **Memory Manager** | 가상 주소 공간, 페이지 파일, 섹션 객체 |
| **I/O Manager** | I/O 요청 패킷(IRP), 드라이버 스택, 비동기 I/O |
| **Cache Manager** | 통합 페이지 캐시, 지연 기록(lazy writer) |
| **Security Reference Monitor** | 접근 토큰 검사, 감사 로그 |
| **PnP Manager** | 장치 열거, 드라이버 로드, 자원 할당 |
| **Power Manager** | 절전 상태(S0-S5), ACPI 연동 |
| **Configuration Manager** | 레지스트리 관리 |

---

## 21.3 객체 관리자 (Object Manager)

### 객체 모델

Windows NT의 핵심 추상화: **모든 커널 자원은 객체다.**

```
객체 구조:
┌──────────────────────────────────┐
│         Object Header            │
│  - 이름 (Name)                   │
│  - 참조 카운트 (Reference Count)  │
│  - 보안 디스크립터               │
│  - 생성 시각                     │
│  - 객체 타입 포인터              │
├──────────────────────────────────┤
│         Object Body              │
│  (타입별 데이터: EPROCESS 등)    │
└──────────────────────────────────┘
```

**객체 타입 예시:**
- `Process`, `Thread`, `Job`
- `File`, `Directory`, `SymbolicLink`
- `Mutant`(뮤텍스), `Semaphore`, `Event`, `Timer`
- `Section`(공유 메모리), `Key`(레지스트리)
- `Token`(보안 토큰), `WindowStation`, `Desktop`

### 핸들 (Handle)

```
프로세스 핸들 테이블:
┌─────┬──────────────────────────────────────────┐
│ idx │ 커널 객체 포인터 | 접근 권한 | 플래그    │
├─────┼──────────────────────────────────────────┤
│  4  │ → File Object (C:\foo.txt) | R/W | inherit│
│  8  │ → Process Object (pid 1234)│ ALL  | -     │
│ 12  │ → Mutant Object (MyMutex) │ SYNC | -     │
└─────┴──────────────────────────────────────────┘

핸들값은 4의 배수 (하위 2비트 = 플래그)
```

### 객체 네임스페이스

```
\
├── Device\
│   ├── HarddiskVolume1  → 물리 드라이브
│   └── Harddisk0\Partition1
├── GLOBAL??\
│   ├── C:  → \Device\HarddiskVolume3  (심볼릭 링크)
│   └── NUL → \Device\Null
├── KernelObjects\
│   └── LowMemoryCondition
└── Sessions\
    └── 1\BaseNamedObjects\
        └── MyMutex  (사용자 뮤텍스)
```

---

## 21.4 프로세스와 스레드

### 구조체 계층

```
Job (여러 프로세스를 그룹으로 제한)
 └── Process (EPROCESS)
      ├── Virtual Address Space (VAD 트리)
      ├── Handle Table
      ├── Token (보안 컨텍스트)
      └── Thread (ETHREAD) × N
           ├── TEB (Thread Environment Block, 사용자 모드)
           ├── TrapFrame (커널 진입 시 레지스터 저장)
           └── KTHREAD (스케줄러 엔티티)
```

**EPROCESS 주요 필드:**
```c
typedef struct _EPROCESS {
    KPROCESS Pcb;           // 스케줄러용 커널 프로세스
    EX_PUSH_LOCK ProcessLock;
    LARGE_INTEGER CreateTime;
    CLIENT_ID UniqueProcessId;
    LIST_ENTRY ActiveProcessLinks; // 전체 프로세스 연결 리스트
    SIZE_T VirtualSize;
    SIZE_T WorkingSetSize;
    PVOID SectionBaseAddress;  // 실행 이미지 베이스
    PHANDLE_TABLE ObjectTable; // 핸들 테이블
    EX_FAST_REF Token;        // 보안 토큰
    // ...
} EPROCESS;
```

### 스레드 상태

```
Initialize → Ready ↔ Running → Terminate
                ↑        ↓
            Standby  ← Waiting (I/O, 동기화 객체)
                         ↓
                      Transition (페이지 스택 스왑 중)
```

### Job 객체

프로세스 그룹을 하나의 단위로 관리:
```
Job 객체 제한:
- CPU 사용률 상한 (JOB_OBJECT_LIMIT_JOB_TIME)
- 메모리 상한 (JOB_OBJECT_LIMIT_JOB_MEMORY)
- 프로세스 수 (JOB_OBJECT_LIMIT_ACTIVE_PROCESS)
- I/O 제한
- 사용자 인터페이스 제한 (바탕화면 전환 금지 등)

→ 현대 Windows: Job + Silo = 컨테이너 격리
```

---

## 21.5 메모리 관리

### 가상 주소 공간 (x64)

```
사용자 공간:       0x0000000000000000 ~ 0x00007FFFFFFFFFFF (128TB)
커널 공간:         0xFFFF080000000000 ~ 0xFFFFFFFFFFFFFFFF

사용자 공간 레이아웃:
  0x0000000000000000  NULL 포인터 가드 영역
  0x0000000000010000  텍스트/데이터 (IMAGE_BASE)
       ...
  0x000000007FFE0000  KUSER_SHARED_DATA (읽기 전용)
  0x000000007FFEFFFF  유저 스택 상단
  0x00007FFFFFFEFFFF  유저 공간 끝
```

### VAD 트리 (Virtual Address Descriptor)

각 프로세스는 가상 메모리 영역을 **AVL 트리**로 관리:

```c
typedef struct _MMVAD {
    ULONG_PTR StartingVpn;    // 시작 페이지 번호
    ULONG_PTR EndingVpn;      // 끝 페이지 번호
    struct _MMVAD *Parent;
    struct _MMVAD *LeftChild;
    struct _MMVAD *RightChild;
    ULONG u;                   // 커밋/예약 상태, 보호 속성
    EX_PUSH_LOCK PushLock;
    // ...
} MMVAD;
```

**VirtualAlloc 상태:**
```
MEM_RESERVE  - 주소 공간 예약 (물리 메모리 없음)
MEM_COMMIT   - 물리/페이지파일 백킹 확보
MEM_FREE     - 미사용

보호 속성:
PAGE_EXECUTE_READ       - W^X 강제 (쓰기 불가)
PAGE_EXECUTE_READWRITE  - 실행 + 읽기/쓰기
PAGE_READONLY
PAGE_NOACCESS
PAGE_GUARD              - 스택 오버플로 감지
```

### 섹션 객체 (Section Objects)

```
DLL 코드 공유 메커니즘:

Process A          Process B
  ┌──────┐           ┌──────┐
  │ VAD  │           │ VAD  │
  └──┬───┘           └──┬───┘
     │                  │
     └──────┬───────────┘
            ▼
      Section Object (ntdll.dll 코드)
            │
            ▼
      물리 메모리 페이지 (단 한 번만 로드)
```

### Working Set & 페이지 교체

**Working Set (작업 집합):** 프로세스가 현재 물리 메모리에 갖는 페이지 집합

```
워터마크 기반 관리:
PagesAvailable > Maximum → 정상
PagesAvailable < Minimum → Working Set Trim 시작
                           (Memory Manager가 각 프로세스에서 페이지 회수)

Modified Page Writer: 수정된 페이지를 페이지 파일에 비동기 기록
Mapped Page Writer: 메모리 맵 파일의 수정 페이지를 디스크에 기록
```

---

## 21.6 I/O 시스템

### I/O 요청 패킷 (IRP)

```
IRP (I/O Request Packet):
┌────────────────────────────────────┐
│ IoStatus (상태코드, 전송 바이트)   │
│ RequestorMode (UserMode/KernelMode)│
│ Cancel (취소 플래그)               │
│ MdlAddress (메모리 디스크립터 리스트)│
│ UserBuffer (사용자 버퍼 포인터)    │
│ IO_STACK_LOCATION 배열            │
│   [0] 최상위 드라이버 (파일시스템) │
│   [1] 필터 드라이버               │
│   [2] 볼륨 드라이버               │
│   [3] 디스크 드라이버             │
└────────────────────────────────────┘
```

### 드라이버 스택 (Driver Stack)

```
ReadFile(handle, buf, n)
    │  NT Native: NtReadFile
    ▼
I/O Manager → IRP 생성
    │
    ▼ IoCallDriver
┌──────────────────────────┐
│ 파일시스템 드라이버      │ (ntfs.sys)
│ - 파일 위치 → LBA 변환  │
└──────────┬───────────────┘
           │ IoCallDriver
┌──────────▼───────────────┐
│ 볼륨 필터 드라이버       │ (fvevol.sys - BitLocker)
└──────────┬───────────────┘
           │ IoCallDriver
┌──────────▼───────────────┐
│ 디스크 드라이버          │ (disk.sys)
└──────────┬───────────────┘
           │ IoCallDriver
┌──────────▼───────────────┐
│ 포트/미니포트 드라이버   │ (storport.sys / nvme.sys)
└──────────────────────────┘
    │ DMA
    ▼
  NVMe SSD
```

### 비동기 I/O & I/O 완료 포트

```c
// I/O 완료 포트 (IOCP) 패턴 - 고성능 서버의 기반
HANDLE iocp = CreateIoCompletionPort(INVALID_HANDLE_VALUE,
                                     NULL, 0,
                                     0); // NumberOfConcurrentThreads

// 파일을 IOCP에 연결
CreateIoCompletionPort(fileHandle, iocp, completionKey, 0);

// 비동기 읽기 요청
OVERLAPPED ov = {0};
ReadFile(fileHandle, buffer, size, NULL, &ov);  // 즉시 반환

// 작업자 스레드: 완료 대기
DWORD transferred;
ULONG_PTR key;
OVERLAPPED *pov;
GetQueuedCompletionStatus(iocp, &transferred, &key, &pov, INFINITE);
// → 완료된 I/O 처리
```

**IOCP 장점:** 스레드 수 = CPU 수로 제한하면서 수천 개의 동시 연결 처리 가능

---

## 21.7 Windows 파일 시스템: NTFS

### NTFS 구조

```
NTFS 볼륨:
┌─────────────────────────────────────────────────┐
│ VBR (Volume Boot Record)                        │
├─────────────────────────────────────────────────┤
│ MFT (Master File Table)                         │
│  - 레코드 0: $MFT (MFT 자신)                    │
│  - 레코드 1: $MFTMirr (MFT 백업)               │
│  - 레코드 2: $LogFile (저널)                    │
│  - 레코드 3: $Volume                            │
│  - 레코드 4: $AttrDef                           │
│  - 레코드 5: . (루트 디렉토리)                  │
│  - 레코드 6: $Bitmap                            │
│  - 레코드 7: $Boot                              │
│  - 레코드 8: $BadClus                           │
│  - 레코드 9: $Secure                            │
│  - 레코드 10: $UpCase                           │
│  - 레코드 11: $Extend                           │
│  - 레코드 12-23: 예약                           │
│  - 레코드 24+: 일반 파일/디렉토리              │
└─────────────────────────────────────────────────┘
```

### MFT 레코드 구조

```
MFT Record (1KB):
┌──────────────────────────────────────────────┐
│ Signature "FILE"                             │
│ Sequence Number                              │
│ Link Count (하드링크 수)                     │
├──────────────────────────────────────────────┤
│ $STANDARD_INFORMATION (생성/수정/접근 시각)  │
│ $FILE_NAME (파일명, 부모 디렉토리 참조)       │
│ $SECURITY_DESCRIPTOR (ACL)                   │
│ $DATA (파일 내용 또는 데이터 런 목록)         │
│   - 작은 파일: MFT 레코드 내에 직접 저장     │
│     (Resident Attribute, ~700B 이하)         │
│   - 큰 파일: 클러스터 런 목록               │
│     (Non-resident: LCN + 길이 쌍의 목록)    │
└──────────────────────────────────────────────┘
```

### 저널링 ($LogFile)

```
NTFS 저널링 (Write-Ahead Log):
1. 트랜잭션 시작 → $LogFile에 redo/undo 레코드 기록
2. 메타데이터 변경 실행
3. 커밋 레코드 기록

충돌 복구:
- $LogFile의 LSN(Log Sequence Number) 기준으로
  커밋된 것은 redo, 미커밋은 undo
- 파일 내용은 보호 안 됨 (메타데이터만)
```

**대안: ReFS (Resilient File System)**
- 체크섬 기반 데이터 무결성
- Integrity Streams (데이터 체크섬)
- 할당 시 기록 (Copy-on-Write) 트랜잭션
- 자동 손상 감지 및 복구 (저장소 공간과 연계)

---

## 21.8 Windows 보안 모델

### 보안 주체와 SID

모든 보안 주체(사용자, 그룹, 컴퓨터)는 **SID(Security Identifier)**로 식별:

```
SID 형식: S-R-I-SA...
S-1-5-21-3623811015-3361044348-30300820-1013

S    : SID 리터럴
1    : 리비전 레벨
5    : 식별자 기관 (5 = NT Authority)
21-... : 도메인/머신 식별자
1013 : 상대 식별자 (RID, 사용자 고유)

잘 알려진 SID:
S-1-1-0         : Everyone
S-1-5-18        : Local System
S-1-5-32-544    : BUILTIN\Administrators
S-1-16-12288    : High Mandatory Level (무결성 레이블)
```

### 접근 토큰 (Access Token)

프로세스/스레드의 보안 컨텍스트:

```
Access Token 내용:
┌─────────────────────────────────────────────┐
│ 사용자 SID                                  │
│ 그룹 SID 목록 (활성화 여부 포함)            │
│ 특권 (Privileges) 목록                      │
│   - SeDebugPrivilege                        │
│   - SeShutdownPrivilege                     │
│   - SeTakeOwnershipPrivilege                │
│ 무결성 레이블 (Integrity Level)             │
│   Untrusted/Low/Medium/High/System           │
│ 토큰 타입 (Primary/Impersonation)          │
│ 세션 ID                                     │
└─────────────────────────────────────────────┘
```

### 보안 디스크립터와 ACL

```
보안 디스크립터:
┌────────────────────────────────────────────────┐
│ Owner SID                                      │
│ Group SID                                      │
│ DACL (Discretionary ACL) - 접근 제어          │
│  ├── ACE: Allow, BUILTIN\Admins, FullControl  │
│  ├── ACE: Allow, SYSTEM, FullControl          │
│  ├── ACE: Allow, Users, ReadExecute           │
│  └── ACE: Deny, Guest, Write                  │
│ SACL (System ACL) - 감사 로그                 │
│  └── ACE: Audit, Everyone, Delete → 이벤트 로그│
└────────────────────────────────────────────────┘

접근 결정 알고리즘:
1. Deny ACE 먼저 평가 → 하나라도 일치 → 거부
2. Allow ACE 순서대로 평가 → 요청 권한 모두 충족 → 허용
3. ACL 끝까지 도달 → 거부 (묵시적 거부)
```

### 무결성 레이블 (Integrity Levels)

강제 무결성 제어(MIC): "낮은 무결성은 높은 무결성에 쓸 수 없다"

```
System  (0x4000): SYSTEM 서비스
High    (0x3000): 관리자 권한으로 실행된 프로세스, UAC 상승 후
Medium  (0x2000): 일반 사용자 프로세스 (기본)
Low     (0x1000): IE 보호 모드, 다운로드한 실행파일
Untrusted (0x0): 최저 신뢰

규칙:
- No-Write-Up: Low 프로세스는 Medium 파일에 쓰기 불가
- No-Read-Up: 설정 시 적용
- No-Execute-Up: 설정 시 적용
```

### UAC (User Account Control)

```
UAC 동작 흐름:
1. 사용자가 특권 필요 앱 실행
2. Application Information Service(AIS)가 감지
3. 동의 UI 팝업 (secure desktop에서)
4. 사용자 승인 → ShellExecute with runas verb
5. 새 프로세스: High 무결성 토큰으로 생성
```

---

## 21.9 Windows 스케줄링

### 32단계 우선순위 시스템

```
우선순위 0-31:
  0     : Zero-page 스레드 (전용)
  1-15  : 동적 우선순위 (일반 스레드)
  16-31 : 실시간 우선순위 (RT 스레드, 고정)

기본 우선순위 = 프로세스 클래스 + 스레드 상대 우선순위
  IDLE_PRIORITY_CLASS      (4)
  BELOW_NORMAL_PRIORITY_CLASS (6)
  NORMAL_PRIORITY_CLASS    (8)  ← 기본
  ABOVE_NORMAL_PRIORITY_CLASS (10)
  HIGH_PRIORITY_CLASS      (13)
  REALTIME_PRIORITY_CLASS  (24)
```

### 동적 우선순위 부스트

```
우선순위 부스트 시나리오:
- I/O 완료: +1~8 부스트 (디스크 +1, 네트워크 +2, 키보드 +6, 마우스 +6)
- 포그라운드 프로세스: 퀀텀 1~3배 연장
- 굶주림 방지: 오랫동안 실행 안 된 스레드 → 임시 우선순위 15로 상승

부스트는 퀀텀마다 하나씩 원래 우선순위로 감소
```

### 스케줄러 자료구조

```
KiDispatcherReadyListHead[32]: 각 우선순위별 이중 연결 리스트
KiReadySummary: 32비트 비트맵 (어느 우선순위에 스레드 있는지)

선택 알고리즘:
  _BitScanReverse(KiReadySummary) → 최고 우선순위 찾기 O(1)
  해당 리스트 앞에서 스레드 꺼내기 O(1)
```

---

## 21.10 Hyper-V와 가상화

### Hyper-V 아키텍처

```
물리 하드웨어
    │
    ▼
Hyper-V Hypervisor (Type 1, Ring -1)
    │
    ├── Partition 0 (Root Partition)
    │    ├── Windows (Privileged OS)
    │    ├── VMBus (가상 버스)
    │    └── VSP (Virtualization Service Provider)
    │
    ├── Partition 1 (Guest VM)
    │    ├── Guest OS (Windows/Linux)
    │    └── VSC (Virtualization Service Client) → VMBus → VSP
    │
    └── Partition 2 (또 다른 Guest VM)
```

**Enlightenments:** Guest OS가 하이퍼바이저 존재를 인식하고 최적화:
- Hypercall (직접 호출, trap-and-emulate 대신)
- TLFS (Top-Level Functional Specification) 준수
- VMBus로 I/O 가상화 (에뮬레이션보다 빠름)

### Windows Subsystem for Linux (WSL 2)

```
WSL 2 아키텍처:
┌──────────────────────────────────────┐
│ Windows 유저 모드                    │
│  wsl.exe → Linux 프로세스 실행       │
├──────────────────────────────────────┤
│ 경량 유틸리티 VM (Hyper-V 기반)      │
│  ┌────────────────────────────────┐  │
│  │ Linux 커널 (Microsoft 컴파일)  │  │
│  │ Linux 파일시스템 (ext4)        │  │
│  │ Linux 프로세스 (bash, gcc ...) │  │
│  └────────────────────────────────┘  │
├──────────────────────────────────────┤
│ Hyper-V Hypervisor                   │
└──────────────────────────────────────┘

9P 프로토콜로 Windows ↔ Linux 파일시스템 접근
```

---

## 21.11 레지스트리

Windows 설정의 중앙 저장소:

```
레지스트리 하이브:
HKEY_LOCAL_MACHINE (HKLM)
  ├── SYSTEM\CurrentControlSet  - 드라이버, 서비스 설정
  ├── SOFTWARE                  - 설치된 앱 설정
  └── HARDWARE                  - 하드웨어 정보 (런타임 생성)

HKEY_CURRENT_USER (HKCU)
  └── Software\...              - 현재 사용자별 설정

HKEY_CLASSES_ROOT
  └── .txt → txtfile            - 파일 연결

물리 파일 위치:
C:\Windows\System32\config\SYSTEM
C:\Windows\System32\config\SOFTWARE
C:\Users\{username}\NTUSER.DAT  (HKCU)
```

**레지스트리 트랜잭션:**
- Windows Vista+: KTM(Kernel Transaction Manager)로 트랜잭션 레지스트리 쓰기
- NTFS 저널과 연동: 원자적 파일+레지스트리 변경 가능

---

## 21.12 Windows 커널 동기화

### 디스패처 객체 (Dispatcher Objects)

신호(signaled) 상태를 가지는 커널 객체:

```c
// 대기 가능 객체들:
Event     : Manual/Auto Reset, SetEvent/WaitForSingleObject
Semaphore : ReleaseSemaphore
Mutex     : ReleaseMutex
Timer     : SetWaitableTimer
Thread    : 스레드 종료 시 signaled
Process   : 프로세스 종료 시 signaled

// 복수 객체 동시 대기:
WaitForMultipleObjects(count, handles, waitAll, timeout);
```

### 임계 구역 (Critical Section)

```c
CRITICAL_SECTION cs;
InitializeCriticalSection(&cs);

EnterCriticalSection(&cs);  // 먼저 스핀, 그 다음 커널 뮤텍스
// ... 임계 구역 ...
LeaveCriticalSection(&cs);

DeleteCriticalSection(&cs);

// 스핀카운트 설정 (멀티프로세서에서 유리)
InitializeCriticalSectionAndSpinCount(&cs, 4000);
```

**구현 원리:** `EnterCriticalSection`은 먼저 스핀(user mode)하다가 일정 횟수 후 커널 뮤텍스(Mutant 객체)로 전환 → 커널 진입 최소화

### Slim Reader/Writer (SRW) Lock

```c
// Vista+ 추가, CRITICAL_SECTION보다 가볍고 읽기/쓰기 구분
SRWLOCK lock = SRWLOCK_INIT;

// 읽기 잠금 (여러 스레드 동시 가능)
AcquireSRWLockShared(&lock);
// ... 읽기 ...
ReleaseSRWLockShared(&lock);

// 쓰기 잠금 (독점)
AcquireSRWLockExclusive(&lock);
// ... 쓰기 ...
ReleaseSRWLockExclusive(&lock);
```

---

## 21.13 Windows 네트워킹

### 네트워크 아키텍처

```
WinSock API (사용자 모드)
    │
    ▼
AFD.sys (Ancillary Function Driver, 커널 소켓 레이어)
    │
    ▼
TDI (Transport Driver Interface) → NDIS
    │
    ▼
TCP/IP 스택 (tcpip.sys)
    │
    ├── Winsock Kernel (WSK) - 커널 모드 소켓
    │
    ▼
NDIS (Network Driver Interface Specification)
    ├── 필터 드라이버 (Windows Filtering Platform)
    │    └── WFP: 방화벽, IDS/IPS 훅 포인트
    └── 미니포트 드라이버 (NIC 드라이버)
```

**Windows Filtering Platform (WFP):** Windows Defender 방화벽, IPsec, QoS 등이 WFP 필터 계층에 구현됨

---

## 요약

```
Windows 10 핵심 설계:
┌─────────────────────────────────────────────────────┐
│ 계층적 Executive: Object Manager가 모든 자원 통합   │
│ IRP 기반 I/O: 드라이버 스택으로 기능 조합           │
│ NTFS: MFT + 저널 + 트랜잭션으로 신뢰성 확보         │
│ SID + ACL + 무결성 레이블: 3중 보안 레이어          │
│ UAC: 최소 권한 + 명시적 상승                        │
│ 32단계 우선순위 + 동적 부스트: 반응성 보장          │
│ Hyper-V + WSL2: 경량 VM으로 Linux 완전 지원         │
│ IOCP: 비동기 I/O로 고성능 서버 지원                 │
│ WFP: 커널 네트워크 정책 확장점                      │
└─────────────────────────────────────────────────────┘
```
