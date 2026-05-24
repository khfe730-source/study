# Operating System Concepts (10th Edition)

> Abraham Silberschatz, Peter B. Galvin, Greg Gagne 저  
> 전문가 수준의 OS 핵심 개념 정리. 인터럽트, 프로세스, 메모리, 파일 시스템, 동기화, 보안까지.

---

## 📂 챕터 목록

| 챕터 | 제목 | 핵심 내용 | 링크 |
|------|------|----------|------|
| Ch01 | Introduction | OS 정의, 인터럽트, 이중 모드, 자원 관리, 커널 자료구조 | [바로가기](./docs/ch01_introduction.md) |
| Ch02 | OS Structures | OS 서비스, 시스템 콜, API/ABI, 링커·로더, 모놀리식/마이크로커널/LKM/하이브리드, 부팅, BCC | [바로가기](./docs/ch02_os_structures.md) |
| Ch03 | Processes | 프로세스 개념/상태/PCB, 컨텍스트 스위치, fork/exec, 좀비/고아, IPC(공유메모리/메시지패싱), 파이프, 소켓, RPC, Android Binder | [바로가기](./docs/ch03_processes.md) |
| Ch04 | Threads & Concurrency | 스레드 개념, Concurrency vs Parallelism, Amdahl's Law, M:1/1:1/M:M 모델, Pthreads/Windows/Java 스레드 라이브러리, Thread Pool/Fork-Join/OpenMP/GCD/TBB, 스레딩 이슈, Linux clone()/Windows ETHREAD | [바로가기](./docs/ch04_threads.md) |
| Ch05 | CPU Scheduling | FCFS/SJF/RR/Priority/MLQ/MLFQ 알고리즘, SMP/CMT/부하균형/프로세서친화성/HMP, 실시간 스케줄링(RM/EDF/비례공유), Linux CFS(vruntime/레드블랙트리), Windows 32단계 우선순위, Solaris 6클래스 | [바로가기](./docs/ch05_cpu_scheduling.md) |
| Ch06 | Synchronization Tools | 경쟁 조건/임계 구역, Peterson's Solution, 메모리 배리어, test_and_set/CAS/원자 변수, 뮤텍스 락/스핀락, 세마포(계수/이진), 모니터+조건 변수, 교착·우선순위 역전, 도구별 성능 비교 | [바로가기](./docs/ch06_synchronization.md) |
| Ch07 | Synchronization Examples | Bounded-Buffer(mutex/empty/full 세마포), Readers-Writers(rw_mutex/read_count), 식사하는 철학자(세마포 데드락+모니터 해법), Windows 디스패처 객체/임계구역, Linux atomic_t/preempt_count, POSIX pthread_mutex/sem_open/pthread_cond, Java synchronized/ReentrantLock/Condition/Semaphore, 트랜잭셔널 메모리(STM/HTM), OpenMP critical, 함수형 언어 불변 상태 | [바로가기](./docs/ch07_synchronization_examples.md) |
| Ch08 | Deadlocks | 데드락 정의/시스템 모델, 라이브락, 4가지 필요 조건(상호배제/점유대기/비선점/순환대기), 자원 할당 그래프/사이클 판단, 3가지 처리 전략(무시/예방/회피+감지복구), 예방(순환대기 전순서 F:R→N/동적 락 한계/Linux lockdep), 안전 상태/안전 순서, 자원 할당 그래프 알고리즘(클레임 간선), 은행원 알고리즘(Available/Max/Allocation/Need, Safety O(mn²), Resource-Request), 단일 인스턴스 Wait-For 그래프 O(n²), 다중 인스턴스 감지 알고리즘, 복구(프로세스 종료/자원 선점+롤백+기아 방지) | [바로가기](./docs/ch08_deadlocks.md) |
| Ch09 | Main Memory | base/limit 레지스터 메모리 보호, 주소 바인딩(컴파일/적재/실행 시간), MMU/논리·물리 주소, 동적 적재/DLL, 연속 할당(first/best/worst fit, 외부·내부 단편화, 50% 규칙, 압축), 페이징(프레임/페이지, 주소 변환, PTBR, TLB/ASID/EAT 공식, 보호 비트/valid-invalid, 공유 페이지/재진입 코드), 계층적 페이징(2단계 32비트→64비트 문제), 해시 페이지 테이블/클러스터, 역 페이지 테이블(공유 제한), SPARC Solaris TSB/TTE, 스와핑(표준/페이지 단위), 모바일 iOS/Android 전략, IA-32(LDT/GDT/PAE 64GB), x86-64(4단계 48비트VA/52비트PA), ARMv8(4단계/3단위/마이크로+메인 TLB) | [바로가기](./docs/ch09_main_memory.md) |
| Ch10 | Virtual Memory | 가상 메모리 배경/희소 주소 공간, 요구 페이징(valid-invalid bit/폴트 처리 6단계/명령어 재시작), 빈 프레임 목록/zero-fill-on-demand, EAT 공식(p<0.0000025), COW(fork 최적화/vfork), 페이지 교체(수정 비트, FIFO/Belady's 이상, OPT, LRU-카운터·스택, 2차 기회·개선된 2차 기회, LFU/MFU, 페이지 버퍼링), 프레임 할당(균등/비례, 전역·지역, 메이저·마이너 폴트, reaper/OOM killer, NUMA·lgroup), 스래싱(지역성 모델, 작업 집합 Δ 윈도우, PFF), 메모리 압축(Android/iOS/Windows/macOS), 커널 메모리(Buddy System/Slab-SLAB·SLUB·SLOB), TLB 도달 범위/Huge Page, 프로그램 구조 지역성, I/O 인터록·락 비트, Linux(active·inactive list/kswapd), Windows(클러스터링/작업집합 트리밍), Solaris(두 손 클록/scanrate) | [바로가기](./docs/ch10_virtual_memory.md) |
| Ch11 | Mass-Storage Structure | HDD 구조(플래터/트랙/섹터/RPM/탐색시간/회전지연), NVM/NAND 플래시(FTL/읽기·쓰기·소거/가비지 컬렉션/웨어 레벨링/DWPD), HDD 스케줄링(FCFS/SCAN/C-SCAN), Linux I/O 스케줄러(Deadline/NOOP/CFQ), 오류 검출·수정(패리티/CRC/ECC), 스토리지 관리(파티셔닝/볼륨/부트 블록/불량 블록), 스왑 공간, 스토리지 연결(Host-attached/NAS/Cloud/SAN), RAID(0/1/4/5/6/0+1/1+0, MTBF, ZFS 체크섬), 객체 스토리지(HDFS/Ceph) | [바로가기](./docs/ch11_mass_storage.md) |
| Ch12 | I/O Systems | 버스/포트/컨트롤러/HBA 하드웨어, 메모리 맵 I/O vs 포트 I/O, 4개 장치 레지스터, 폴링(핸드셰이킹/바쁜 대기), 인터럽트(벡터/체인/NMI/마스크/FLIH·SLIH/우선순위), DMA(PIO 문제/스캐터-개더/사이클 훔치기/DVMA), I/O 인터페이스 추상화(블록/문자/네트워크/타이머), 블로킹·논블로킹·비동기 I/O, 커널 I/O 서브시스템(스케줄링/버퍼링/이중버퍼/캐싱/스풀링/오류처리/I/O 보호), Android 전력 관리/wakelocks/ACPI, 10단계 블로킹 read() 라이프사이클, STREAMS, I/O 성능 원칙 | [바로가기](./docs/ch12_io_systems.md) |
| Ch13 | File-System Interface | 파일 개념/7개 속성/7가지 기본 연산, 오픈 파일 테이블(2계층), 파일 잠금(공유/배타, 강제/조언), 파일 타입(확장자/매직 넘버), 접근 방법(순차/직접/인덱스), 디렉토리 구조(단일/2단계/트리/비순환 그래프/일반 그래프), 보호(ACL/UNIX 9비트 rwx), 메모리 맵 파일(mmap/요구 페이징/공유 메모리/Windows CreateFileMapping) | [바로가기](./docs/ch13_file_system_interface.md) |
| Ch14 | File-System Implementation | 계층형 파일 시스템(I/O 제어/기본 FS/파일 조직 모듈/논리 FS), 온-스토리지 구조(부트 제어 블록/볼륨 제어 블록/FCB·inode), 인-메모리 구조(마운트 테이블/디렉토리 캐시/시스템 전체·프로세스별 오픈 파일 테이블), 디렉토리 구현(선형 리스트/해시 테이블), 할당 방법(연속·extents/연결·FAT/인덱스·UNIX inode 12직접+3간접), 자유 공간 관리(비트맵/연결 리스트/그루핑/카운팅/ZFS 메타슬랩·space map/TRIM), 효율성·성능(클러스터링/통합 버퍼 캐시/free-behind·read-ahead), 복구(fsck/저널링/never-overwrite), WAFL(root inode 트리/스냅샷/클론/write-anywhere) | [바로가기](./docs/ch14_file_system_implementation.md) |
| Ch15 | File-System Internals | 특수 파일 시스템 유형(tmpfs/procfs/objfs 등), 마운트 절차·마운트 포인트, 파티션(raw vs cooked), 부트 로더·멀티부트, VFS 계층(vnode/4대 객체/함수 포인터 테이블), 원격 파일 시스템(ftp→DFS→WWW→Cloud), 클라이언트-서버 모델, 분산 정보 시스템(DNS/NIS/LDAP/Active Directory/Kerberos), 장애 모드, 일관성 의미론(UNIX/Session·AFS/Immutable), NFS(마운트 프로토콜/NFS 프로토콜·stateless·멱등성/경로명 변환/캐스케이딩 마운트/원격 연산·캐싱) | [바로가기](./docs/ch15_file_system_internals.md) |
| Ch16 | Security | 보안 침해 유형(기밀성/무결성/가용성/서비스도용/DoS), 4계층 보안 모델(물리/네트워크/OS/앱), 악성 소프트웨어(트로이목마/스파이웨어/랜섬웨어/트랩도어/논리폭탄/최소권한 원칙), 코드 인젝션(버퍼 오버플로우/shellcode/NOP-sled/ASLR), 바이러스·웜 분류(파일/부트/매크로/루트킷/다형성 등), 네트워크 위협(좀비/스니핑/스푸핑/DoS·DDoS/포트스캐닝), 암호화(대칭-DES·3DES·AES/스트림, 비대칭-RSA/공개키), 인증(해시함수/MAC/디지털서명/인증서·CA/X.509), TLS 핸드셰이크, 사용자 인증(패스워드취약점/salt+hash/OTP/생체인식/2FA·MFA), 보안 방어(보안정책/침투테스트/IPS-시그니처·이상탐지/Bayes 분석/방화벽-DMZ/ASLR/코드서명), Windows 10 보안(SID/접근토큰/DACL/SACL/무결성레이블/UAC) | [바로가기](./docs/ch16_security.md) |
| Ch17 | Protection | 보호 목표(정책·메커니즘 분리), 최소 권한 원칙·구획화·심층 방어, 보호 링(Bell-LaPadula/Intel-ring-1~3/ARM-TrustZone/ARMv8-EL0~3/Samsung RKP/Apple KPP), 보호 도메인(접근 권리/need-to-know/정적·동적/UNIX-setuid/Android-UID), 접근 행렬(행=도메인/열=객체/copy·transfer·owner·control 권리/감금 문제), 접근 행렬 구현(전역테이블/ACL/캐퍼빌리티-태그·주소분리/Lock-Key/혼합-UNIX), 접근 권리 철회(재획득/역방향포인터/간접참조/키 방식), RBAC(Solaris 10 권한·역할), MAC(레이블/SELinux/TrustedBSD/Windows MIC) vs DAC, 캐퍼빌리티 기반(Linux POSIX.1e 비트마스크/Darwin 엔타이틀먼트·XML plist), SIP·시스템콜 필터링(SECCOMP-BPF)·샌드박싱(Seatbelt/SELinux+SECCOMP)·코드서명, Java 스택 검사(doPrivileged/checkPermissions/타입안전성) | [바로가기](./docs/ch17_protection.md) |
| Ch18 | Virtual Machines | 가상 머신 개요(Host/VMM/Guest/VCPU), 역사(IBM VM/370/Popek-Goldberg 요구사항-Fidelity·Performance·Safety), 이점(격리·스냅샷·통합·라이브 마이그레이션·클라우드), 구성 요소(Trap-and-Emulate/Binary Translation-캐싱·VMware 5% 오버헤드/NPT/하드웨어 지원-VT-x·AMD-V·EPT·VT-d·인터럽트 리매핑·ARM EL2), 하이퍼바이저 유형(Type0-LPAR/Type1-ESX·KVM/Type2-VirtualBox/Paravirt-Xen 순환버퍼·Hypercall/JVM·에뮬레이션/컨테이너-Docker·Kubernetes), OS 구성 요소(CPU 과다할당·클록 오차, 메모리-balloon·dedup·double paging, I/O-DMA pass-through·NAT/bridging, 스토리지-이미지파일·P-to-V·V-to-P), 라이브 마이그레이션 7단계(R/O→R/W→dirty 페이지 반복→freeze→전송), 연구(Unikernel/Partitioning Hypervisor-Quest-V·Jailhouse) | [바로가기](./docs/ch18_virtual_machines.md) |
| Ch19 | Networks & Distributed Systems | 분산 시스템 장점(자원 공유·계산 속도·신뢰성), 네트워크 구조(LAN-Ethernet 802.3/WiFi 802.11/WAN-ARPANET→인터넷·광케이블 백본), 통신 구조(DNS 계층 해석·캐싱, OSI 7계층 모델, TCP/IP 4계층, ARP IP→MAC, Ethernet 패킷, UDP-비신뢰·무연결, TCP-ACK·시퀀스·3-way handshake·흐름제어·혼잡제어), 네트워크OS(SSH/FTP/SFTP/클라우드) vs 분산OS(데이터·계산·프로세스 마이그레이션), 설계 이슈(견고성-heartbeat 감지·재구성·복구, 투명성-위치 노출 없음·LDAP 사용자 이동성, 확장성-압축·중복제거), DFS(NFS-무상태·멱등성, OpenAFS-전체 파일 캐싱·확장성, 클러스터 기반-GFS·HDFS 메타데이터+데이터서버+청크 3중복제·MapReduce), DFS 이름 지정(위치 투명성 vs 위치 독립성, Amazon S3) | [바로가기](./docs/ch19_networks_distributed_systems.md) |
| Ch20 | Linux System | task_struct/clone()/네임스페이스/cgroup, CFS(vruntime·레드-블랙트리·스케줄링 클래스), Buddy+Slab(SLUB)/active·inactive 리스트/kswapd, VFS·ext4(extents·저널링·지연할당), 블록 I/O 스택(bio·I/O 스케줄러), 소켓/netfilter/WFP, LKM, RCU 동기화, 보안(DAC·LSM·SELinux·AppArmor·Capabilities), 컨테이너(네임스페이스+cgroup) | [바로가기](./docs/ch20_linux.md) |
| Ch21 | Windows 10 | Executive(Object Manager·Process Manager·Memory Manager·I/O Manager·Security Reference Monitor), 객체 모델(핸들·포트 네임스페이스), EPROCESS/ETHREAD/Job, VAD 트리·섹션객체·Working Set, IRP·드라이버 스택·IOCP, NTFS(MFT·Resident Attribute·저널링·ReFS), SID·ACL·무결성레이블(MIC)·UAC, 32단계 우선순위·동적 부스트, Hyper-V·WSL2, WFP | [바로가기](./docs/ch21_windows.md) |

---

## 📖 책 구성 (전체 25챕터 + 4부록)

| 파트 | 챕터 | 제목 |
|------|------|------|
| Part 1: Overview | 1–2 | Introduction, OS Structures |
| Part 2: Process Management | 3–8 | Processes, Threads, CPU Scheduling, Synchronization, Deadlocks |
| Part 3: Memory Management | 9–10 | Main Memory, Virtual Memory |
| Part 4: Storage Management | 11–15 | Mass-Storage, I/O, File System (x3) |
| Part 5: Security & Protection | 16–17 | Security, Protection |
| Part 6: Advanced Topics | 18–19 | Virtual Machines, Networks & Distributed Systems |
| Part 7: Case Studies | 20–21 | Linux, Windows |
| Appendix A | Influential OS | Atlas·CTSS·Multics·THE·RC4000·Unix·CP/M·Mach 역사적 계보 | [바로가기](./docs/appendixA_influential_os.md) |
| Appendix B | BSD UNIX | BSD 역사, FFS/UFS2, Soft Updates, BSD 소켓·mbuf, PF, ASLR/W^X, Pledge/Unveil, Jail | [바로가기](./docs/appendixB_bsd_unix.md) |
| Appendix C | The Mach System | 마이크로커널, Task/Thread/Port/Message/Memory Object, COW VM, 외부 페이저, MIG, XNU | [바로가기](./docs/appendixC_mach.md) |
