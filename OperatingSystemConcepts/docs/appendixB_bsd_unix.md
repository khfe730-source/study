# Appendix B: BSD UNIX

## B.1 BSD 역사

### 탄생 배경

BSD(Berkeley Software Distribution)는 UC Berkeley에서 AT&T Unix를 기반으로 발전시킨 Unix 계열이다.

```
역사적 흐름:
1BSD (1977)  - Bill Joy의 Pascal 인터프리터 + 에디터 (ex)
2BSD (1978)  - vi(visual editor), csh(C shell)
3BSD (1979)  - 가상 메모리 (VAX에서) → demand paging
4BSD (1980)  - 신뢰할 수 있는 신호, 작업 제어(job control)
4.1BSD (1981)- 성능 개선, 자동 설정
4.2BSD (1983)- TCP/IP 소켓 API, FFS(Fast File System)
4.3BSD (1986)- 성능/신뢰성, 라우팅 소켓
4.4BSD (1993)- Mach VM 통합, NFS, ISO 프로토콜

1994: AT&T와의 법적 분쟁 해결 → 자유롭게 배포 가능
      → FreeBSD, NetBSD, OpenBSD, DragonFly BSD로 분기
      
현재 BSD 계열:
- FreeBSD: 서버, 임베디드 (PlayStation 4/5 OS 기반)
- NetBSD: 최대 이식성 ("Of course it runs NetBSD")
- OpenBSD: 보안 최우선 (LibreSSL, OpenSSH 발원)
- macOS/iOS: Darwin(XNU) = Mach + BSD 레이어
```

### BSD의 Unix 생태계 기여

```
인터넷 인프라의 상당 부분이 BSD에서 나왔다:

소프트웨어:
- vi/ex          - 오늘날 vim의 직접 원형
- csh/tcsh       - C 셸 (현재도 맥OS 기본 셸이었음)
- sendmail       - 최초의 실용적 MTA
- named (BIND)   - DNS 서버 구현체
- routed         - RIP 라우팅 데몬

프로토콜/API:
- BSD Socket API - TCP/IP 프로그래밍의 표준 (POSIX 채택)
- NFS client     - Sun과 공동 개발
- IPv4/IPv6 구현체

보안:
- OpenSSH        - OpenBSD 개발 (SSH 표준 구현체)
- LibreSSL       - OpenSSL 포크 (OpenBSD)
```

---

## B.2 BSD 커널 아키텍처

### 전체 구조

```
사용자 공간
──────────────────────────────────────────────────────
시스템 콜 인터페이스 (POSIX + BSD 확장)
──────────────────────────────────────────────────────
┌─────────────┬──────────────┬───────────────────────┐
│ 프로세스    │ 가상 메모리  │ 파일 시스템 (VFS)     │
│ 관리        │ 서브시스템   │ FFS/UFS, NFS, devfs  │
├─────────────┴──────────────┴───────────────────────┤
│            소켓 / 네트워킹                          │
│   TCP/IP, UDP, IPv6, IPsec, PF 방화벽              │
├────────────────────────────────────────────────────┤
│               장치 드라이버                         │
│   블록, 문자, 네트워크, USB, PCI                   │
├────────────────────────────────────────────────────┤
│               HAL / 플랫폼 코드                     │
│   x86, ARM, MIPS, RISC-V                           │
└────────────────────────────────────────────────────┘
```

### 모놀리식이지만 모듈 가능

```
FreeBSD KLD (Kernel Loadable Module):
# 모듈 로드
kldload linux        # Linux ABI 호환 레이어
kldload if_tap       # TUN/TAP 네트워크

# 모듈 목록
kldstat

# 모듈 언로드
kldunload if_tap
```

---

## B.3 프로세스 관리

### proc 구조체

```c
// BSD 프로세스 디스크립터
struct proc {
    LIST_ENTRY(proc) p_list;    // 전체 프로세스 리스트
    pid_t           p_pid;
    struct proc    *p_pptr;     // 부모 프로세스
    
    struct pcred   *p_cred;     // 자격증명 (UID, GID, 그룹 목록)
    struct filedesc *p_fd;      // 파일 디스크립터 테이블
    struct vmspace *p_vmspace;  // 가상 주소 공간
    
    struct pstats  *p_stats;    // 프로세스 통계
    struct sigacts *p_sigacts;  // 시그널 핸들러
    
    int             p_stat;     // 상태: SSLEEP, SRUN, SZOMB 등
    char            p_comm[MAXCOMLEN+1]; // 프로세스 이름
    
    // kthread: 커널 스레드 (BSD의 LWP)
    struct thread  *p_threads;  // FreeBSD 5.0+에서 분리됨
};
```

### 상태 전이

```
SIDL     초기화 중 (fork 완료 전)
    ↓
SRUN     실행 가능 (런큐에 있음)
    ↓
SONPROC  현재 CPU에서 실행 중
    ↓
SSLEEP   이벤트 대기 중 (인터럽트 가능)
SSTOP    SIGSTOP으로 정지
SZOMB    좀비 (부모의 wait() 대기)
SDEAD    완전히 종료, 자원 회수 중
```

### fork/vfork/rfork

```c
// fork: 전통적 복사 (COW)
pid_t pid = fork();

// vfork: 부모 중지, 자식이 부모 주소 공간 공유
// exec() 또는 exit() 호출 전까지만 사용
pid_t pid = vfork();

// rfork: BSD 고유, 클론 세부 제어
// Linux clone()의 BSD 버전
rfork(RFPROC | RFMEM | RFFDG);

// rfork 플래그:
RFPROC   // 새 프로세스 생성 (없으면 스레드처럼)
RFMEM    // 메모리 공유
RFFDG    // 파일 디스크립터 테이블 복사
RFCFDG   // 파일 디스크립터 테이블 비우기
RFSIGSHARE // 시그널 핸들러 공유
```

---

## B.4 메모리 관리

### Mach VM 기반 (4.4BSD+)

```
4.4BSD가 Mach 2.0의 VM 서브시스템을 채택:

vm_map: 가상 주소 공간 (연결 리스트 + RB 트리)
  └── vm_map_entry: 연속 가상 주소 영역
       └── vm_object: 파일 또는 익명 메모리 백킹
            └── vm_page: 물리 페이지

COW (Copy-on-Write):
- fork() 시 페이지 복사 안 함
- 쓰기 시도 → 페이지 폴트 → 그 때 복사
```

### UMA (Universal Memory Allocator) - FreeBSD

```c
// Slab 할당자의 FreeBSD 구현
uma_zone_t zone;
zone = uma_zcreate("myzone",
                   sizeof(struct my_obj),
                   my_ctor,   // 생성자
                   my_dtor,   // 소멸자
                   my_init,   // 초기화
                   my_fini,   // 정리
                   UMA_ALIGN_PTR,
                   0);

struct my_obj *obj = uma_zalloc(zone, M_WAITOK);
// ... 사용 ...
uma_zfree(zone, obj);
```

### 물리 메모리 관리

```
FreeBSD 물리 메모리 큐:
- Free Queue      : 즉시 할당 가능
- Cache Queue     : 재사용 대기 (빠른 회수)
- Inactive Queue  : 교체 후보 (LRU 근사)
- Active Queue    : 최근 접근됨
- Wired Queue     : 고정 (스왑 불가)

Pagedaemon (kthread): 백그라운드 페이지 회수
  - vm.lowmem_period: 검사 주기
  - vm.v_free_target: 목표 여유 페이지 수
```

---

## B.5 파일 시스템: FFS (Fast File System)

### 역사

```
원래 Unix FS (v7):
- 블록 크기: 512B
- 단편화 심각
- 성능: 약 2% 디스크 대역폭 활용

FFS (Marshall Kirk McKusick, UC Berkeley, 1983):
- 블록 크기: 4KB~8KB
- 단편화 해결: fragment(블록의 1/8)
- 실린더 그룹 (Cylinder Groups)으로 지역성
- 성능: ~40% 디스크 대역폭 활용
```

### FFS 구조

```
FFS 디스크 레이아웃:
┌──────┬──────────────────────────────────────────┐
│ Boot │  CG0   │  CG1   │  CG2   │ ... │  CGn  │
└──────┴──────────────────────────────────────────┘

Cylinder Group (CG):
┌──────────────┬───────────────────────────────────┐
│ Superblock   │ CG 디스크립터 │ inode 비트맵      │
│ (백업)       │               │ 블록 비트맵       │
├──────────────┴───────────────┴───────────────────┤
│ inode 테이블                                     │
├──────────────────────────────────────────────────┤
│ 데이터 블록                                      │
└──────────────────────────────────────────────────┘

핵심 아이디어:
- 같은 디렉토리의 파일들 → 같은 CG에 배치
- 파일의 inode와 데이터 블록 → 같은 CG에
- 디렉토리 → 분산 배치 (탐색 효율)
```

### UFS2 (Extended)

```
4.4BSD+에서 FFS를 UFS(Unix File System)로 일반화,
FreeBSD에서 UFS2로 확장:

- 64비트 블록 카운터 (8ZB 지원)
- 나노초 타임스탬프
- 확장 속성 (Extended Attributes)
- ACL 지원
- 백그라운드 fsck
- 소프트 업데이트 (Soft Updates):
  → 저널 없이 일관성 보장
  → 메타데이터 쓰기 순서 강제
```

### Soft Updates (McKusick & Ganger, 1999)

```
저널링 없이 파일시스템 일관성을 보장하는 기법:

의존성 추적:
  새 블록 할당 → inode 업데이트 순서 보장
  블록 재사용 → 이전 참조 제거 후 허용

규칙:
1. 새 포인터는 가리키는 블록이 초기화된 후에 디스크에 기록
2. 초기화된 포인터는 다시 초기화되기 전에 제거되어야 함
3. inode 링크 카운트는 실제 참조보다 보수적으로 감소

충돌 후: 약간의 공간 낭비 가능 (fsck -p로 회수)
         하지만 파일시스템 구조 손상 없음
```

---

## B.6 네트워킹: BSD 소켓

### 소켓 API의 탄생

```
4.2BSD (1983)에서 처음 등장, 현재 POSIX 표준

설계 원칙:
- 네트워크 통신을 파일 I/O처럼 다룬다
- 프로토콜 독립적 인터페이스
- 주소 패밀리: AF_INET, AF_UNIX, AF_INET6 ...
```

### 소켓 구현 내부

```c
// 커널 소켓 구조체 (간략화)
struct socket {
    int         so_type;    // SOCK_STREAM, SOCK_DGRAM
    int         so_state;   // SS_ISCONNECTED, SS_CANTSENDMORE 등
    struct sockbuf so_snd;  // 송신 버퍼
    struct sockbuf so_rcv;  // 수신 버퍼
    struct protosw *so_proto; // 프로토콜 함수 테이블
    struct socket *so_head;  // accept 큐의 head
    // ...
};

// 프로토콜 전환 테이블
struct protosw {
    int     pr_type;        // SOCK_STREAM 등
    int     pr_domain;      // AF_INET 등
    int     pr_protocol;    // IPPROTO_TCP 등
    int     pr_flags;       // PR_CONNREQUIRED 등
    
    // 함수 포인터들
    pr_input_t  *pr_input;  // 수신 처리
    pr_output_t *pr_output; // 송신 처리
    pr_ctlinput_t *pr_ctlinput; // 제어 입력 (ICMP 에러 등)
    pr_usrreqs_t  *pr_usrreqs;  // 사용자 요청 핸들러
};
```

### mbuf (Memory Buffer)

BSD 네트워킹의 핵심 자료구조:

```
mbuf 구조:
┌────────────────────────────────────────────────────┐
│ m_hdr: next, nextpkt, data 포인터, len, type, flags │
├────────────────────────────────────────────────────┤
│ 데이터 영역 (256B 또는 외부 클러스터 참조)         │
└────────────────────────────────────────────────────┘

mbuf 체인 (TCP 세그먼트 예):
  [IP 헤더] → [TCP 헤더] → [데이터 1] → [데이터 2]
     mbuf0  →    mbuf1   →   mbuf2    →   mbuf3

클러스터: 2KB 또는 4KB 연속 블록 (대용량 데이터용)
  mbuf.m_ext.ext_buf → 외부 클러스터 포인터

제로 카피 네트워킹:
- sendfile(2): 파일 페이지를 mbuf 외부 참조로 직접 전송
  → 커널 ↔ 사용자 복사 없음
```

### PFIL 훅 / Packet Filter (PF)

```
패킷 필터링 아키텍처:

수신 패킷 경로:
  NIC → if_input() → PFIL_IN 훅 → IP 입력 → TCP/UDP
  
PFIL 훅에서:
  PF (OpenBSD Packet Filter)
  IPFirewall (ipfw)
  IP Filter (ipfilter)

PF (OpenBSD):
  - 상태 기반 방화벽 (stateful)
  - NAT/PAT
  - QoS (ALTQ와 통합)
  - pf.conf 예시:
    pass in on em0 proto tcp to port 80 keep state
    block in quick from <bruteforce>
```

---

## B.7 스케줄링

### 전통적 BSD 스케줄러

```
다단계 피드백 큐 (MLFQ) + 우선순위 에이징:

우선순위 계산 (4초마다 재계산):
  p_pri = PUSER + p_estcpu / 4 + 2 * p_nice

p_estcpu (CPU 사용 추정치):
  decay: p_estcpu = p_estcpu × (2 × load) / (2 × load + 1)
  실행 중: p_estcpu++

→ CPU 많이 쓸수록 p_estcpu↑ → 우선순위↓ → 다른 프로세스 기회
→ 대기 중이면 p_estcpu 감소 → 우선순위↑ → I/O bound 프로세스 선호
```

### ULE 스케줄러 (FreeBSD 5.0+, 현재 기본)

```
목표: Linux CFS와 유사한 공정성 + 대화형 반응성

특징:
- 런큐: 현재(current) + 다음(next) 두 큐
- 대화형 점수(interactivity score): 수면/실행 비율 기반
- NUMA 인식
- CPU 친화성 (CPU affinity) 지원
- SMT(하이퍼스레딩) 인식

실시간:
  SCHED_FIFO, SCHED_RR (POSIX 준수)
  pthread_setschedparam()으로 설정
```

---

## B.8 BSD IPC

### UNIX 도메인 소켓

```c
// 같은 호스트 내 빠른 IPC
int sv[2];
socketpair(AF_UNIX, SOCK_STREAM, 0, sv);
// sv[0] ↔ sv[1] 양방향 통신

// 파일 디스크립터 전달:
// sendmsg/recvmsg + SCM_RIGHTS
struct msghdr msg = { ... };
struct cmsghdr *cmsg = CMSG_FIRSTHDR(&msg);
cmsg->cmsg_level = SOL_SOCKET;
cmsg->cmsg_type  = SCM_RIGHTS;
*(int *)CMSG_DATA(cmsg) = fd_to_pass;
sendmsg(sock, &msg, 0);
```

### POSIX 공유 메모리 & 메시지 큐

```c
// POSIX shm (4.4BSD+)
int fd = shm_open("/myshm", O_CREAT | O_RDWR, 0600);
ftruncate(fd, 4096);
void *p = mmap(NULL, 4096, PROT_READ|PROT_WRITE, MAP_SHARED, fd, 0);

// POSIX 메시지 큐
mqd_t mq = mq_open("/mymq", O_CREAT | O_RDWR, 0600, NULL);
mq_send(mq, buf, len, priority);
mq_receive(mq, buf, len, &priority);
```

---

## B.9 보안 (OpenBSD 중심)

### W^X (Write XOR Execute)

```
OpenBSD 3.3 (2003) - 최초 도입:
"쓰기 가능하면 실행 불가, 실행 가능하면 쓰기 불가"

구현:
- pmap 계층에서 PTE 쓰기 + 실행 동시 설정 거부
- JIT 컴파일러는 mprotect()로 권한 전환 필요

→ Linux의 NX 비트, Windows DEP의 선구
```

### ASLR (Address Space Layout Randomization)

```
OpenBSD 3.4 (2003) - 최초 완전 구현:
- 스택, 힙, 공유 라이브러리 무작위 배치
- 실행 파일 베이스도 무작위 (PIE - Position Independent Executable)

랜덤화 범위 (OpenBSD):
  스택: 256MB 범위 내 무작위
  mmap: 전체 주소 공간
  PIE:  전체 주소 공간
```

### Pledge & Unveil (OpenBSD 고유)

```c
// pledge: 프로세스가 앞으로 사용할 시스템 콜 집합 선언
// 선언 후 다른 syscall 시도 → SIGABRT/SIGKILL
pledge("stdio rpath inet", NULL);
// → 파일 읽기, 소켓 통신만 허용

// unveil: 프로세스가 접근할 파일시스템 경로 제한
unveil("/var/www", "r");    // 읽기만
unveil("/tmp",     "rwc");  // 읽기/쓰기/생성
unveil(NULL, NULL);          // 이후 추가 unveil 차단

// 사용 예: nginx
pledge("stdio rpath inet sendfd recvfd", NULL);
unveil("/var/www/html", "r");
```

---

## B.10 FreeBSD Jail

컨테이너의 선구자 (FreeBSD 4.0, 2000):

```c
// Jail 생성
struct jail j = {
    .version = 0,
    .path    = "/var/jail/web",  // chroot 경로
    .hostname = "web.example.com",
    .ip_number = inet_addr("192.168.1.100"),
};
jail(&j);

// Jail 내부 프로세스 제약:
// - 파일시스템: jail 루트 밖 접근 불가
// - 네트워크: 지정된 IP만 바인딩 가능
// - 프로세스: jail 밖 프로세스에 시그널 불가
// - 커널: 커널 모듈 로드 불가
```

**현대적 FreeBSD Jail (jail.conf):**
```
web {
    host.hostname = "web.example.com";
    ip4.addr = "192.168.1.100";
    path = "/var/jail/web";
    mount.devfs;
    allow.raw_sockets;
    exec.start = "/bin/sh /etc/rc";
    exec.stop  = "/bin/sh /etc/rc.shutdown";
}
```

Docker의 컨테이너 개념보다 13년 앞선 기술.

---

## 요약

```
BSD의 현대 OS 기여:
┌────────────────────────────────────────────────────────┐
│ BSD 소켓 API    - 네트워크 프로그래밍의 보편 표준      │
│ FFS/UFS         - 현대 파일시스템 설계의 기준          │
│ Soft Updates    - 저널 없는 파일시스템 일관성          │
│ mbuf            - 고성능 패킷 처리 자료구조            │
│ PF 방화벽       - 현대 방화벽의 표준 설계              │
│ ASLR / W^X      - 현대 OS 보안의 필수 기능             │
│ Pledge/Unveil   - 최소 권한의 실용적 구현              │
│ Jail            - 컨테이너 가상화의 선구               │
│ XNU(macOS)      - BSD + Mach = 세계 최대 배포          │
│ OpenSSH         - 전 세계 서버 관리의 표준             │
└────────────────────────────────────────────────────────┘
```
