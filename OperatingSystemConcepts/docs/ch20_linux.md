# Chapter 20: The Linux System

## 20.1 Linux 역사

### 탄생 배경

Linux는 1991년 Linus Torvalds가 Helsinki 대학 학부생으로서 시작한 프로젝트에서 비롯됐다. 당시 Minix(교육용 Unix 클론)를 사용하던 Torvalds는 터미널 에뮬레이터가 필요해 커널 작성을 시작했고, 이것이 Linux의 기원이 됐다.

```
역사적 흐름:
1969 - Unix (AT&T Bell Labs, Thompson & Ritchie)
1983 - GNU 프로젝트 (Stallman, FSF)
1987 - Minix 1.0 (Tanenbaum, 교육용)
1991 - Linux 0.01 (Torvalds, 취미 프로젝트)
1994 - Linux 1.0 (GPL 라이선스)
1996 - Linux 2.0 (SMP 지원)
2003 - Linux 2.6 (대규모 리팩토링)
2011 - Linux 3.0
2015 - Linux 4.0
2019 - Linux 5.0
```

### GNU/Linux

Linux는 커널만을 의미하며, 완전한 OS는 GNU 도구들(gcc, glibc, bash 등)과 결합된 **GNU/Linux**다. GPL(GNU General Public License)이 적용되어 소스코드 공개 및 자유로운 배포가 보장된다.

**주요 배포판:**
- Red Hat Enterprise Linux(RHEL), CentOS, Fedora
- Debian, Ubuntu, Linux Mint
- SUSE Linux, openSUSE
- Arch Linux, Gentoo

---

## 20.2 Linux 설계 원칙

### 핵심 철학

1. **Unix 호환성 (POSIX 준수)**: 기존 Unix 애플리케이션 이식성 보장
2. **모놀리식 커널 + 모듈**: 성능을 위해 모놀리식이지만 LKM으로 유연성 확보
3. **모든 것은 파일**: 장치, 소켓, 파이프 모두 파일 인터페이스로 통일
4. **소규모, 단일 목적 도구**: 조합(파이프)을 통한 복잡한 작업

### 커널 구조

```
사용자 공간 (User Space)
─────────────────────────────────────────────
    앱/라이브러리    │   GNU C Library (glibc)
─────────────────────────────────────────────
         시스템 콜 인터페이스 (syscall)
─────────────────────────────────────────────
  프로세스 관리  │  메모리 관리  │  파일 시스템
  CPU 스케줄러   │  가상 메모리  │  VFS
─────────────────────────────────────────────
  네트워킹      │  IPC         │  장치 드라이버
  TCP/IP 스택   │  소켓/신호   │  블록/문자/NIC
─────────────────────────────────────────────
         하드웨어 추상화 계층 (HAL)
─────────────────────────────────────────────
커널 공간 (Kernel Space)
```

---

## 20.3 커널 모듈 (Loadable Kernel Modules)

### LKM 메커니즘

Linux는 런타임에 커널 기능을 동적으로 추가/제거할 수 있는 **Loadable Kernel Module(LKM)** 시스템을 지원한다.

```c
// 모듈 기본 구조
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/init.h>

static int __init my_module_init(void) {
    printk(KERN_INFO "Module loaded\n");
    return 0;
}

static void __exit my_module_exit(void) {
    printk(KERN_INFO "Module unloaded\n");
}

module_init(my_module_init);
module_exit(my_module_exit);
MODULE_LICENSE("GPL");
MODULE_DESCRIPTION("Example LKM");
```

**모듈 관리 명령어:**
```bash
lsmod          # 로드된 모듈 목록
modprobe ext4  # 모듈 로드 (의존성 자동 처리)
rmmod ext4     # 모듈 언로드
modinfo ext4   # 모듈 정보
```

### 모듈 타입
- **장치 드라이버**: 블록(디스크), 문자(키보드), 네트워크(NIC)
- **파일 시스템**: ext4, XFS, btrfs, NTFS
- **시스템 콜**: 필요시 새로운 syscall 추가
- **네트워크 프로토콜**: 새로운 프로토콜 스택

---

## 20.4 프로세스 관리

### task_struct: 프로세스 디스크립터

Linux에서 모든 프로세스/스레드는 `task_struct`로 표현된다.

```c
struct task_struct {
    volatile long   state;          // 프로세스 상태
    void            *stack;         // 커널 스택
    atomic_t        usage;          // 참조 카운트
    unsigned int    flags;          // PF_KTHREAD 등
    unsigned int    ptrace;

    struct sched_entity     se;     // CFS 스케줄링 엔티티
    struct sched_rt_entity  rt;     // RT 스케줄링
    struct sched_dl_entity  dl;     // Deadline 스케줄링

    struct task_struct __rcu *parent;
    struct list_head children;
    struct list_head sibling;

    pid_t   pid;                    // 프로세스 ID
    pid_t   tgid;                   // 스레드 그룹 ID (= 메인 스레드 pid)

    struct mm_struct    *mm;        // 메모리 맵
    struct mm_struct    *active_mm;

    struct fs_struct    *fs;        // 파일시스템 정보
    struct files_struct *files;     // 오픈 파일 테이블
    struct signal_struct *signal;   // 시그널 핸들러

    // ... 수백 개의 필드
};
```

### 프로세스 상태

```
TASK_RUNNING        - 실행 중 또는 실행 대기 (런큐에 있음)
TASK_INTERRUPTIBLE  - 인터럽트 가능한 대기 (신호로 깨어남)
TASK_UNINTERRUPTIBLE- 인터럽트 불가 대기 (I/O 완료 대기, D state)
__TASK_STOPPED      - SIGSTOP/SIGTSTP으로 정지
__TASK_TRACED       - ptrace로 추적 중
EXIT_ZOMBIE         - 종료됐지만 부모가 wait() 미호출
EXIT_DEAD           - wait() 호출 후 제거 중
```

### fork() vs clone()

Linux에서 `fork()`는 내부적으로 `clone()` 시스템 콜로 구현된다.

```c
// fork() 내부: clone(SIGCHLD, 0)
// pthread_create() 내부:
clone(CLONE_VM | CLONE_FS | CLONE_FILES | CLONE_SIGHAND |
      CLONE_THREAD | CLONE_SYSVSEM | CLONE_SETTLS |
      CLONE_PARENT_SETTID | CLONE_CHILD_CLEARTID, ...);
```

| 플래그 | 의미 |
|--------|------|
| `CLONE_VM` | 가상 메모리 공유 |
| `CLONE_FS` | 파일시스템 정보 공유 |
| `CLONE_FILES` | 파일 디스크립터 테이블 공유 |
| `CLONE_SIGHAND` | 시그널 핸들러 공유 |
| `CLONE_THREAD` | 동일 스레드 그룹 (tgid 공유) |
| `CLONE_NEWPID` | 새로운 PID 네임스페이스 (컨테이너) |

### 네임스페이스 (Namespaces)

컨테이너 격리의 핵심 메커니즘:

```
네임스페이스 유형:
- Mount (mnt)   : 파일시스템 마운트 포인트 격리
- UTS           : 호스트명/도메인명 격리
- IPC           : SysV IPC, POSIX 메시지 큐 격리
- PID           : 프로세스 ID 공간 격리
- Network (net) : 네트워크 인터페이스/라우팅 격리
- User          : UID/GID 매핑 격리
- Cgroup        : Cgroup 루트 격리
- Time          : CLOCK_REALTIME 오프셋 격리
```

---

## 20.5 스케줄링

### Completely Fair Scheduler (CFS)

Linux 기본 스케줄러. O(log n) 복잡도의 레드-블랙 트리 기반.

**핵심 개념: vruntime (Virtual Runtime)**

```
vruntime += delta_exec × (NICE_0_LOAD / task_weight)

task_weight: nice값에 따른 가중치
  nice -20 → weight 88761
  nice   0 → weight  1024  (NICE_0_LOAD)
  nice +19 → weight    15
```

```
Red-Black Tree (CFS 런큐)
         [40]
        /    \
      [20]   [60]
      /  \
   [10]  [30]
   
← 가장 작은 vruntime (leftmost node) = 다음 실행 태스크
```

**CFS 알고리즘:**
1. 새 프로세스 `vruntime = min_vruntime`으로 초기화
2. 실행 시 `vruntime` 증가 (낮은 nice → 느리게 증가)
3. 선점: 현재 태스크 vruntime > 런큐 최소 vruntime + sched_latency/n

**주요 파라미터:**
```bash
# /proc/sys/kernel/sched_*
sched_latency_ns        = 24000000  # 24ms (전체 스케줄링 주기)
sched_min_granularity_ns = 3000000  # 3ms (최소 실행 시간)
sched_wakeup_granularity_ns = 4000000
```

### 스케줄링 클래스 계층

```
stop_sched_class    (최고 우선순위, CPU 마이그레이션용)
    ↓
dl_sched_class      (Deadline: EDF 기반)
    ↓
rt_sched_class      (Real-Time: FIFO/RR, 우선순위 0-99)
    ↓
fair_sched_class    (CFS: 일반 프로세스)
    ↓
idle_sched_class    (idle 스레드)
```

**실시간 정책:**
```c
SCHED_FIFO  // 선점 없음, 높은 RT 우선순위가 낮은 것 선점
SCHED_RR    // 라운드로빈 (time quantum 후 동일 우선순위 큐 끝으로)
SCHED_DEADLINE // 주기적 태스크, CBS(Constant Bandwidth Server)
```

### NUMA 인식 스케줄링

```
NUMA 토폴로지 예시:
  Node 0: CPU 0-7,  RAM 0-31GB
  Node 1: CPU 8-15, RAM 32-63GB

  같은 노드 내 접근: ~100ns
  원격 노드 접근:   ~300ns (3x 느림)
```

CFS는 `sched_domain` 계층을 통해 NUMA 토폴로지를 인식하고, 부하 분산 시 원격 NUMA 접근을 최소화한다.

---

## 20.6 메모리 관리

### 물리 메모리 구조

```
물리 메모리 Zone:
┌─────────────────────────────────────────────────────┐
│ ZONE_DMA     (0~16MB)   - ISA DMA 장치용             │
│ ZONE_DMA32   (0~4GB)    - 32비트 DMA 장치용          │
│ ZONE_NORMAL  (~128GB)   - 일반 커널 매핑              │
│ ZONE_HIGHMEM (>128GB)   - 32비트에서 동적 매핑 필요  │
│ ZONE_MOVABLE            - 메모리 핫플러그/defrag      │
└─────────────────────────────────────────────────────┘
```

### 가상 주소 공간 (x86-64)

```
주소 공간 레이아웃 (48비트 VA):
0x0000000000000000 - 0x00007FFFFFFFFFFF  사용자 공간 (128TB)
  └── 텍스트, 데이터, BSS, 힙, 스택, 공유 라이브러리
  
(canonical hole: non-canonical 주소 구간)

0xFFFF800000000000 - 0xFFFFFFFFFFFFFFFF  커널 공간 (128TB)
  ├── direct mapping (물리 메모리 직접 매핑)
  ├── vmalloc 영역
  ├── kernel text/data
  └── modules
```

### Buddy System + Slab Allocator

**Buddy System (페이지 단위 할당):**

```
order 0: 4KB  (2^0 pages)
order 1: 8KB  (2^1 pages)
order 2: 16KB (2^2 pages)
...
order 11: 8MB (2^11 pages, MAX_ORDER)

할당: 2^k 크기 블록 → 없으면 더 큰 블록 분할(split)
해제: 버디 블록과 병합(merge) 가능하면 상위 order로
```

**Slab Allocator (객체 단위 할당):**

```
kmem_cache (예: task_struct_cache)
  └── Slab 1 [obj][obj][obj][obj][free][free]
  └── Slab 2 [obj][obj][free][free][free][free]
  └── Slab 3 (new slab, all free)

장점:
- 객체 재사용 (초기화 비용 절감)
- 내부 단편화 최소화
- 캐시 라인 정렬
- Per-CPU 캐시로 잠금 경합 감소

Linux 구현체:
- SLAB: 원래 Bonwick 설계
- SLUB: 현재 기본 (오버헤드 감소, 디버깅 용이)
- SLOB: 임베디드용 (최소 메모리)
```

### 가상 메모리 (vm_area_struct)

```c
struct mm_struct {
    struct vm_area_struct *mmap;  // VMA 연결 리스트
    struct rb_root mm_rb;         // VMA 레드-블랙 트리
    unsigned long mmap_base;      // mmap 시작 주소
    unsigned long total_vm;       // 총 페이지 수
    unsigned long locked_vm;      // 잠긴 페이지
    // ...
};

struct vm_area_struct {
    unsigned long vm_start;
    unsigned long vm_end;
    struct vm_area_struct *vm_next, *vm_prev;
    struct rb_node vm_rb;
    pgprot_t vm_page_prot;        // 접근 권한
    unsigned long vm_flags;       // VM_READ, VM_WRITE, VM_EXEC 등
    struct vm_operations_struct *vm_ops;  // fault, open, close
    // ...
};
```

### 페이지 교체: active/inactive 리스트

```
active list   ←→   inactive list
  (hot)             (cold → 교체 대상)

페이지 접근 시:
  inactive → active (promote)
  
메모리 부족 시:
  active → inactive (demote)
  inactive → 스왑/해제 (evict)

kswapd: 백그라운드 페이지 회수 데몬
  - 워터마크: pages_min, pages_low, pages_high
  - pages_low 이하 → kswapd 깨어남
  - pages_min 이하 → direct reclaim (동기)
```

---

## 20.7 파일 시스템

### Virtual File System (VFS)

```
사용자: open("/etc/passwd", O_RDONLY)
         │
         ▼
    VFS 계층 (sys_open → do_sys_open → vfs_open)
    ┌─────────────────────────────────────────┐
    │  inode 캐시 / dentry 캐시              │
    │  superblock 객체                        │
    └─────────────────────────────────────────┘
         │              │             │
         ▼              ▼             ▼
       ext4           tmpfs          XFS
    (디스크)         (RAM)          (디스크)
```

**VFS 4대 객체:**

| 객체 | 설명 | 핵심 연산 |
|------|------|-----------|
| `superblock` | 마운트된 파일시스템 메타데이터 | `read_inode`, `write_inode`, `sync_fs` |
| `inode` | 파일 메타데이터 (크기, 권한, 블록 위치) | `create`, `lookup`, `link`, `mkdir` |
| `dentry` | 디렉토리 엔트리 (이름 → inode 매핑) | `d_revalidate`, `d_delete` |
| `file` | 열린 파일 인스턴스 | `read`, `write`, `mmap`, `ioctl` |

### ext4 파일 시스템

**구조:**
```
ext4 디스크 레이아웃:
┌──────────┬──────────────────────────────────────────┐
│ Boot Blk │ Block Group 0 │ Group 1 │ ... │ Group N  │
└──────────┴──────────────────────────────────────────┘

Block Group:
┌────────────┬──────────┬────────────┬──────────────────┐
│ Superblock │ GDT      │ Block/Inode│ Data Blocks      │
│ (복사본)   │          │ Bitmap     │                  │
└────────────┴──────────┴────────────┴──────────────────┘
```

**ext4 주요 특징:**
- **Extents**: 연속 블록 범위 표현 (최대 128MB), B-tree 구조
- **Journaling**: ordered(기본), writeback, journal 모드
- **Delayed Allocation**: 쓰기 시 즉시 블록 할당 않고 flush 시 일괄 할당
- **64-bit 지원**: 최대 1EB 파일시스템
- **Checksums**: 저널 및 메타데이터 체크섬

### 저널링 (Journaling)

```
Ordered 모드 (기본):
1. 데이터 블록 디스크에 기록
2. 저널에 메타데이터 기록 (commit record)
3. 체크포인트: 저널 → 실제 위치로 메타데이터 이동

충돌 복구:
- 저널의 commit record 있으면 → 재실행(redo)
- 없으면 → 불완전한 트랜잭션 무시
```

---

## 20.8 I/O 시스템

### 블록 I/O 스택

```
사용자 read(fd, buf, n)
      │
      ▼
   VFS 레이어
      │
      ▼
   Page Cache      ← 히트 시 바로 반환
      │ 미스
      ▼
   파일시스템 (ext4, XFS ...)
      │
      ▼
   블록 레이어 (bio 구조체)
      │
      ▼
   I/O 스케줄러
   ┌─────────────────────────────┐
   │ None (NVMe 기본)            │
   │ mq-deadline (SCSI, SATA)   │
   │ BFQ (데스크톱)              │
   │ Kyber (저지연)              │
   └─────────────────────────────┘
      │
      ▼
   블록 드라이버 (SCSI/NVMe/virtio)
      │
      ▼
   하드웨어
```

**bio 구조체:**
```c
struct bio {
    sector_t        bi_sector;    // 시작 섹터
    struct block_device *bi_bdev;
    unsigned int    bi_opf;       // REQ_OP_READ, REQ_OP_WRITE 등
    unsigned short  bi_vcnt;      // bio_vec 수
    struct bio_vec  bi_inline_vecs[]; // scatter-gather I/O 벡터
};
```

### 네트워크 스택

```
소켓 API (send/recv)
      │
      ▼
   BSD Socket Layer (AF_INET, AF_UNIX ...)
      │
      ▼
   Transport Layer (TCP/UDP)
      │ TCP: 3-way handshake, 혼잡 제어, 재전송
      ▼
   Network Layer (IP, ICMP, ARP)
      │
      ▼
   Netfilter (iptables/nftables 훅 포인트)
      │
      ▼
   링크 레이어 (Ethernet, WiFi ...)
      │
      ▼
   NIC 드라이버 → DMA → 하드웨어
```

**netfilter 훅 포인트:**
```
PREROUTING → [routing] → FORWARD → POSTROUTING → 송신
                 ↓
              INPUT → 로컬 프로세스 → OUTPUT
```

---

## 20.9 프로세스 간 통신 (IPC)

### 시그널 (Signals)

```c
// 시그널 핸들러 등록
struct sigaction sa = {
    .sa_handler = my_handler,
    .sa_flags   = SA_RESTART,
};
sigaction(SIGTERM, &sa, NULL);

// 주요 시그널
SIGHUP  (1)  - 터미널 연결 해제
SIGINT  (2)  - Ctrl+C
SIGKILL (9)  - 강제 종료 (무시/차단 불가)
SIGSEGV (11) - 세그멘테이션 폴트
SIGTERM (15) - 종료 요청
SIGCHLD (17) - 자식 프로세스 상태 변경
SIGSTOP (19) - 프로세스 정지 (무시 불가)
```

### 파이프와 FIFO

```c
// 익명 파이프
int pipefd[2];
pipe(pipefd);  // pipefd[0]=읽기, pipefd[1]=쓰기

// 명명 파이프 (FIFO)
mkfifo("/tmp/myfifo", 0666);
```

### POSIX 공유 메모리

```c
// 생성
int fd = shm_open("/myshm", O_CREAT | O_RDWR, 0666);
ftruncate(fd, SIZE);
void *ptr = mmap(NULL, SIZE, PROT_READ | PROT_WRITE,
                 MAP_SHARED, fd, 0);

// 사용 후
munmap(ptr, SIZE);
shm_unlink("/myshm");
```

### Cgroups (Control Groups)

자원 제한 및 계층적 관리 메커니즘 (Docker/Kubernetes의 핵심):

```
Cgroup v2 계층 (단일 트리):
/sys/fs/cgroup/
├── memory.max          # 메모리 상한
├── cpu.weight          # CPU 상대 비중 (CFS weight)
├── cpu.max             # CPU bandwidth: "quota period"
├── io.max              # I/O bandwidth 제한
└── pids.max            # 프로세스 수 제한

예시: 컨테이너 메모리 512MB 제한
echo "536870912" > /sys/fs/cgroup/mycontainer/memory.max
```

---

## 20.10 보안 모델

### DAC: 전통적 Unix 권한

```
파일 권한: rwxrwxrwx
           │││││││││
           ││││││└┘└── others (3비트)
           │││└┘└───── group  (3비트)
           └┘└──────── owner  (3비트)

특수 비트:
setuid (4000): 실행 시 파일 소유자 권한으로
setgid (2000): 실행 시 파일 그룹 권한으로
sticky (1000): 디렉토리 내 타인 파일 삭제 방지
```

### MAC: Linux Security Modules (LSM)

커널 훅을 통한 강제 접근 제어:

```c
// LSM 훅 예시
security_inode_permission(inode, mask)
security_file_open(file)
security_socket_connect(sock, address, addrlen)
```

**SELinux (Security-Enhanced Linux):**
```
모든 주체/객체에 보안 레이블(context) 부여:
  user:role:type:level

예: unconfined_u:unconfined_r:httpd_t:s0

정책: httpd_t 타입은 httpd_log_t 파일에만 쓰기 가능
→ httpd 프로세스가 /etc/passwd 쓰기 시도 → 거부
```

**AppArmor (Ubuntu 기본):**
```
경로 기반 정책 (SELinux보다 단순):
/etc/apparmor.d/usr.sbin.nginx:
  /var/www/** r,       # 읽기 허용
  /var/log/nginx/** w, # 쓰기 허용
  /etc/nginx/** r,
  network tcp,
```

### Capabilities

`root`를 세분화한 권한 비트:

```
CAP_NET_ADMIN    - 네트워크 설정 변경
CAP_SYS_ADMIN    - 다양한 시스템 관리 작업 (거대한 캐퍼빌리티!)
CAP_SYS_PTRACE   - ptrace 허용
CAP_KILL         - 임의 프로세스에 시그널
CAP_SETUID       - UID 변경
CAP_NET_BIND_SERVICE - 1024 이하 포트 바인딩
```

---

## 20.11 Linux 커널 동기화

### 원자적 연산

```c
atomic_t counter = ATOMIC_INIT(0);
atomic_inc(&counter);
atomic_dec_and_test(&counter);  // 0이 되면 true
int val = atomic_read(&counter);

// 64비트
atomic64_t large;
atomic64_add(1000, &large);
```

### 스핀락 vs 뮤텍스

```c
// 스핀락: 짧은 임계 구역, 인터럽트 핸들러에서 사용 가능
spinlock_t lock;
spin_lock_init(&lock);

spin_lock(&lock);           // 단순 락
spin_lock_irqsave(&lock, flags);  // 인터럽트 비활성화 + 락
// ... 임계 구역 ...
spin_unlock_irqrestore(&lock, flags);

// 뮤텍스: 긴 임계 구역, 수면 가능
struct mutex m;
mutex_init(&m);
mutex_lock(&m);   // 획득 못하면 sleep
// ... 임계 구역 ...
mutex_unlock(&m);
```

### RCU (Read-Copy-Update)

읽기 위주 공유 자료구조에서 읽기 성능을 극대화하는 동기화 메커니즘:

```c
// 읽기: 락 없음, 매우 빠름
rcu_read_lock();
struct node *p = rcu_dereference(head);
// p 사용 (p가 해제되지 않음을 RCU가 보장)
rcu_read_unlock();

// 쓰기: 새 버전 생성 후 포인터 교체
struct node *new_node = kmalloc(...);
// ... new_node 초기화 ...
rcu_assign_pointer(head, new_node);
synchronize_rcu();  // 기존 독자들이 모두 빠져나갈 때까지 대기
kfree(old_node);
```

**RCU 사용 사례:** 라우팅 테이블, 네트워크 프로토콜 핸들러, 파일시스템 dentry 캐시

---

## 20.12 주요 자료구조

### 연결 리스트 (list_head)

```c
struct task_struct {
    struct list_head children;  // 이중 연결 원형 리스트
    // ...
};

// 순회
struct task_struct *child;
list_for_each_entry(child, &current->children, sibling) {
    // ...
}
```

### 레드-블랙 트리 (rb_root)

```c
// VMA, CFS 런큐, 타이머 등에 사용
struct rb_root tree = RB_ROOT;
struct rb_node *node = rb_first(&tree);  // 최솟값
rb_insert_color(node, &tree);
rb_erase(node, &tree);
```

### Radix Tree (xarray로 대체)

페이지 캐시 인덱싱에 사용:
```c
// 파일 오프셋 → 물리 페이지 매핑
struct xarray i_pages;  // inode의 페이지 캐시
xa_store(&mapping->i_pages, index, page, GFP_KERNEL);
page = xa_load(&mapping->i_pages, index);
```

---

## 요약

```
Linux 핵심 설계:
┌─────────────────────────────────────────────────────┐
│ 모놀리식 커널 + LKM으로 유연성 확보                  │
│ task_struct로 프로세스/스레드 통합 관리              │
│ clone()으로 fork/thread 통일                         │
│ CFS: vruntime 기반 완전히 공정한 스케줄링            │
│ Buddy + Slab: 페이지·객체 이중 할당자               │
│ VFS: 모든 파일시스템에 통일된 인터페이스             │
│ Netfilter: 네트워크 정책의 훅 포인트                 │
│ LSM: SELinux/AppArmor MAC 확장점                     │
│ Namespaces + Cgroups: 컨테이너의 기반               │
│ RCU: 읽기 위주 자료구조의 락-프리 동기화            │
└─────────────────────────────────────────────────────┘
```
