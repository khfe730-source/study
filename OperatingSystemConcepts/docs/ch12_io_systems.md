# Chapter 12: I/O Systems

## 12.1 개요

OS의 I/O 서브시스템은 I/O 장치의 복잡성을 커널 나머지 부분으로부터 격리시키는 추상화 계층이다. 표준화(인터페이스 통일)와 다양성(장치 종류 증가)이라는 두 트렌드 사이에서 **장치 드라이버(device driver)** 모듈이 핵심 역할을 담당한다.

---

## 12.2 I/O 하드웨어 (I/O Hardware)

### 기본 개념

```
CPU ←─ System Bus (PCIe) ─→ Device Controller ─→ Device
              │
         Memory Bus
              │
            DRAM

Port: 장치와의 연결점
Bus: 여러 장치가 공유하는 와이어 집합 + 프로토콜
Controller: 포트/버스/장치를 운영하는 전자 회로
  - Host Controller (컴퓨터 측)
  - Device Controller (장치 내장)
```

**PCIe 구조:**
- Lane = 수신 pair + 송신 pair (4선) → full-duplex byte stream
- x1/x2/x4/x8/x16/x32 → 레인 수
- PCIe gen3 x8 → 최대 8GB/s 처리량

### 12.2.1 메모리 매핑 I/O (Memory-Mapped I/O)

**I/O 접근 방식 비교:**

| 방식 | 메커니즘 | 특징 |
|------|---------|------|
| **포트 I/O** | 전용 I/O 명령어 (in/out) | 별도 주소 공간, x86 레거시 |
| **Memory-Mapped I/O** | 장치 레지스터를 메모리 주소에 매핑 | 일반 load/store 명령 사용, 현재 주류 |

**컨트롤러 레지스터 4종:**

```
Status Register  ← CPU reads: 완료 여부, 오류 여부, 데이터 준비 여부
Control Register → CPU writes: 명령 시작, 모드 설정
Data-in Register ← CPU reads: 입력 데이터
Data-out Register → CPU writes: 출력 데이터
```

### 12.2.2 폴링 (Polling)

**핸드셰이킹 시퀀스 (출력 예시):**

```
Host                           Controller
  │                                │
  │──[while busy_bit == 1] loop────│  ← busy-waiting
  │                                │
  │──set write_bit, write data_out─→│
  │──set command_ready_bit──────────→│
  │                                │
  │                    set busy_bit│
  │                    read command│
  │                    do I/O      │
  │                    clear bits  │
  │←───────────────── done ────────│
```

- **효율적 사례**: 장치가 빠르고 대기가 짧을 때 (3 CPU 명령어로 폴링 1회)
- **비효율적 사례**: 장치가 느리고 CPU가 다른 작업을 해야 할 때 → 인터럽트로 전환

### 12.2.3 인터럽트 (Interrupts)

**인터럽트 처리 사이클:**

```
1. 장치 컨트롤러가 interrupt-request line에 신호 assert
2. CPU가 매 명령어 실행 후 interrupt-request line 확인
3. CPU: state save → 인터럽트 핸들러 주소로 점프 (interrupt vector 참조)
4. FLIH (First-Level Interrupt Handler):
   - 컨텍스트 스위치, 상태 저장, 처리 작업 큐잉
5. SLIH (Second-Level Interrupt Handler):
   - 실제 I/O 처리 (별도 스케줄링)
6. state restore → return from interrupt
```

**인터럽트 벡터 (Intel Pentium):**

| 벡터 번호 | 유형 | 예시 |
|----------|------|------|
| 0–31 | **비마스크 가능 (NMI)** | 페이지 폴트, 나눗셈 오류, 디버그 예외 |
| 32–255 | **마스크 가능** | 장치 인터럽트 (I/O 완료) |

**고급 인터럽트 기능:**

```
1. 인터럽트 마스킹: 임계 구역 처리 시 maskable 인터럽트 비활성화
2. 인터럽트 체이닝: 벡터 엔트리 → 핸들러 리스트
   (장치 수 > 벡터 엔트리 수 문제 해결)
3. 인터럽트 우선순위: 고우선순위가 저우선순위를 선점
4. 소프트웨어 인터럽트(trap): 시스템 콜 구현 메커니즘
```

**인터럽트 활용 사례:**
- 장치 I/O 완료 통지
- 페이지 폴트 처리
- 시스템 콜 (trap 명령)
- 커널 내 우선순위 관리 (디스크 읽기: 고우선순위 핸들러 → 저우선순위 인터럽트로 데이터 복사)
- Solaris: 인터럽트 핸들러를 커널 스레드로 구현 → 멀티프로세서에서 병렬 처리

### 12.2.4 DMA (Direct Memory Access)

**PIO (Programmed I/O)의 문제:**
- CPU가 바이트 단위로 컨트롤러 레지스터에 데이터 공급 → 대용량 전송 시 CPU 낭비

**DMA 동작:**

```
① CPU가 DMA Command Block을 메모리에 작성:
   - 소스 주소 / 목적지 주소 / 전송 바이트 수
   - Scatter-Gather: 불연속 주소 목록 지원 → 단일 DMA 명령으로 다중 전송

② CPU가 DMA 컨트롤러에 Command Block 주소 전달 후 다른 작업 수행

③ DMA 컨트롤러가 메모리 버스를 직접 접근:
   DMA-request → DMA controller seizes bus → DMA-acknowledge → 데이터 전송

④ 전송 완료 후 DMA 컨트롤러가 CPU에 인터럽트 발생
```

**Cycle Stealing**: DMA가 메모리 버스를 점유하는 동안 CPU는 캐시만 접근 가능 → CPU 계산 속도 일시 감소, but 전체 시스템 처리량 증가

**DVMA (Direct Virtual Memory Access)**: 물리 주소 대신 가상 주소 사용 → 주소 변환 포함, 두 장치 간 전송 시 메인 메모리/CPU 개입 없이 직접 전송 가능

---

## 12.3 애플리케이션 I/O 인터페이스 (Application I/O Interface)

### I/O 계층 구조

```
┌─────────────────────────────────┐
│         Application             │
├─────────────────────────────────┤
│       System Call Interface     │  ← read/write/ioctl/select
├─────────────────────────────────┤
│     Kernel I/O Subsystem        │  ← 스케줄링, 버퍼링, 캐싱, 스풀링
├────────┬────────┬───────┬───────┤
│ Block  │ Char   │ Net   │ Timer │  ← 장치 유형별 표준 인터페이스
│ Driver │ Driver │ Driver│ Driver│
├────────┴────────┴───────┴───────┤
│     Device Controllers          │  ← 하드웨어
└─────────────────────────────────┘
```

**장치 특성 분류:**

| 차원 | 유형 | 예시 |
|------|------|------|
| 데이터 전송 | 문자 스트림 / 블록 | 키보드 / 디스크 |
| 접근 방식 | 순차 / 랜덤 | 테이프 / 디스크 |
| 동기성 | 동기 / 비동기 | 터미널 / 디스크 |
| 공유 가능성 | 전용 / 공유 | 테이프 / 키보드 |
| I/O 방향 | 읽기전용 / 쓰기전용 / 읽기쓰기 | CD-ROM / 프린터 / 디스크 |

**`ioctl()` 시스템 콜:** 장치 드라이버가 구현한 임의 명령을 애플리케이션에서 직접 호출 → 새 시스템 콜 추가 없이 드라이버 확장 가능

```c
ioctl(fd, COMMAND, &arg);  // fd: 장치, COMMAND: 명령, arg: 데이터 구조체
```

**Unix/Linux 장치 번호:**
```
major 번호: 장치 유형 (드라이버 선택)
minor 번호: 장치 인스턴스 (드라이버 내 인덱스)

$ ls -l /dev/sda*
brw-rw---- 1 root disk 8, 0 /dev/sda   ← major=8, minor=0
brw-rw---- 1 root disk 8, 1 /dev/sda1  ← major=8, minor=1
```

### 12.3.1 블록 장치와 문자 장치

**블록 장치 인터페이스:**
- `read()`, `write()`, `seek()` → 블록 단위 접근
- **Raw I/O**: 파일시스템 우회, 블록 장치를 선형 배열로 직접 접근 (DB 시스템)
- **Direct I/O**: UNIX에서 `O_DIRECT` 플래그 → 버퍼링/락킹 비활성화
- **Memory-mapped I/O**: 파일을 가상 메모리 주소에 매핑 → demand paging과 동일 메커니즘으로 효율적

**문자 스트림 인터페이스:**
- `get()`, `put()` 한 문자씩 → 키보드, 마우스, 모뎀, 프린터

### 12.3.2 네트워크 장치

**소켓 인터페이스:**
```c
socket()   → 소켓 생성
bind()     → 로컬 주소 바인딩
connect()  → 원격 연결
listen()   → 연결 대기
accept()   → 연결 수락
send()/recv() → 데이터 송수신
select()   → 비차단 다중 소켓 감시 (busy waiting 제거)
```

### 12.3.3 클록과 타이머

- **Programmable Interval Timer**: 지정 시간 후 인터럽트 발생 (1회 또는 반복)
  - 스케줄러의 타임슬라이스 만료
  - 디스크 캐시 주기적 플러시
  - 네트워크 타임아웃 처리
- **가상 클록**: 단일 타이머 하드웨어로 다수의 타이머 요청 시뮬레이션 → 커널이 최소 시간 요청 우선 정렬, 타이머 완료 시 다음 최소 시간으로 재설정
- **HPET (High-Performance Event Timer)**: 10MHz 이상, 멀티플 comparator
- **NTP (Network Time Protocol)**: 클록 드리프트 보정

### 12.3.4 블로킹 vs 논블로킹 vs 비동기 I/O

```
Blocking I/O:
  syscall → [스레드 실행 중지] → I/O 완료 → [재개] → 반환값
  (구현 단순, 대부분의 기본 시스템 콜)

Nonblocking I/O:
  syscall → 즉시 반환 (전송된 바이트 수, 0 포함 가능)
  (커널이 전송 가능한 만큼만 전송)

Asynchronous I/O:
  syscall → 즉시 반환 (완료 통보 없음)
  → 나중에 완료 통보: 변수 설정 / 시그널 / 소프트웨어 인터럽트 / 콜백
  (완전한 전송 보장, 완료 시점은 미래)
```

**`select()` 동작:** 최대 대기 시간 지정 가능, 0이면 폴링 효과 → 실제 전송은 별도 read/write 필요

### 12.3.5 벡터 I/O (Vectored I/O)

```c
// UNIX readv/writev: scatter-gather I/O
struct iovec iov[3] = {
    { buf1, len1 },
    { buf2, len2 },
    { buf3, len3 }
};
readv(fd, iov, 3);  // 단일 시스템 콜로 불연속 버퍼에 읽기
```

- 컨텍스트 스위치 오버헤드 감소
- 원자성 보장 옵션 (다른 스레드의 중간 삽입 방지)

---

## 12.4 커널 I/O 서브시스템 (Kernel I/O Subsystem)

### 12.4.1 I/O 스케줄링

```
Device-Status Table:
┌────────────────┬────────┬──────────────────────────┐
│   Device       │ Status │   Request Queue           │
├────────────────┼────────┼──────────────────────────┤
│ keyboard       │ idle   │ (empty)                   │
│ laser printer  │ busy   │ → req(addr=38546, len=1K) │
│ disk unit 1    │ idle   │ (empty)                   │
│ disk unit 2    │ busy   │ → req(file=xxx, read)     │
│                │        │ → req(file=yyy, write)    │
└────────────────┴────────┴──────────────────────────┘
```

- 비동기 I/O 지원 시 커널이 다수의 I/O 요청을 동시 추적
- 가상 메모리 서브시스템 요청이 애플리케이션 요청보다 우선순위 높음

### 12.4.2 버퍼링 (Buffering)

**버퍼링의 3가지 목적:**

**1. 속도 불일치 (Speed Mismatch) 처리:**
```
네트워크 (1Gbps) → [Buffer 1] ← fill    → [Buffer 2] → SSD 쓰기
                              → SSD 쓰기     fill    ←
→ Double Buffering: 생산자/소비자 분리, 타이밍 제약 완화
```

**2. 데이터 크기 불일치 처리:**
- 네트워크 패킷 단편화/재조합

**3. Copy Semantics 보장:**
```c
// write() 호출 시 커널이 즉시 데이터를 커널 버퍼에 복사
write(fd, app_buf, n);
// 이후 app_buf 변경해도 쓰여진 데이터에 영향 없음
// 실제 디스크 쓰기는 커널 버퍼에서 수행
```
- COW(Copy-on-Write) + 가상 메모리 매핑으로 효율적 구현 가능

### 12.4.3 캐싱 (Caching)

- **버퍼 vs 캐시**: 버퍼는 유일한 사본을 보유할 수 있음, 캐시는 항상 다른 곳에 원본이 있음
- 실제로는 **버퍼 캐시(buffer cache)**: 디스크 데이터의 커널 버퍼가 캐시 역할도 겸함
  - 파일 읽기 요청 → 먼저 buffer cache 확인 → hit 시 물리 I/O 없이 반환
  - 디스크 쓰기는 수 초간 buffer cache에 축적 후 대용량 배치 쓰기

### 12.4.4 스풀링 (Spooling)과 장치 예약

**스풀링 구조 (프린터 예시):**
```
App1 → [spool file 1] ─┐
App2 → [spool file 2] ─┤→ Spooling Daemon → Printer (순차적 전송)
App3 → [spool file 3] ─┘
```
- 비인터리빙 장치의 동시 접근 문제 해결
- 스풀 큐 관리: 삭제, 순서 변경, 일시 정지 등

**장치 독점 예약:** Windows `OpenFile()` 공유 모드 파라미터, VMS 독점 할당

### 12.4.5 오류 처리

```
UNIX 오류 반환:
  syscall() → -1 (실패) + errno 설정
  errno: EINVAL / EBADF / ENOENT / EIO / ... (약 100개)

SCSI 3단계 오류 정보:
  ① Sense Key: 오류 일반 분류 (Hardware Error, Illegal Request...)
  ② Additional Sense Code: 오류 범주 (Bad Parameter, Self-Test Failure...)
  ③ Additional Sense Code Qualifier: 세부 정보 (어느 파라미터, 어느 서브시스템)
```

### 12.4.6 I/O 보호

- 모든 I/O 명령을 **특권 명령(privileged instruction)**으로 정의 → 사용자 프로세스는 직접 실행 불가
- 메모리 매핑 I/O 포트: 메모리 보호 시스템으로 사용자 접근 차단
- 예외: 그래픽 컨트롤러 메모리 → OS가 **잠금 메커니즘**으로 특정 영역을 특정 프로세스에 할당

### 12.4.7 커널 데이터 구조

**UNIX I/O 커널 구조 (객체 지향적 디스패치):**

```
per-process open-file table
  └─→ system-wide open-file table entry
         ├── file-system record: inode ptr, read/write/select/ioctl/close 함수 포인터
         └── socket record: network info, read/write/select/ioctl/close 함수 포인터
```
- 파일 유형(일반 파일, raw 장치, 소켓)에 따라 다른 구현을 디스패치 테이블로 추상화

**Windows**: 메시지 패싱 방식 → I/O 요청을 메시지로 변환하여 I/O 관리자 → 드라이버 체인 통과

### 12.4.8 전력 관리 (Power Management)

**Android 전력 관리 3요소:**

```
1. Power Collapse (전력 붕괴 모드):
   화면/스피커/I/O → 개별 전원 차단
   CPU → 최저 슬립 상태 (수백 mW → 수 mW)
   인터럽트(전화 수신 등) → 즉시 깨어남

2. Component-Level Power Management:
   Device Tree (물리 장치 토폴로지)
   └─ System Bus
      └─ I/O Subsystem
         ├─ Flash Storage  ← driver: 사용 중 여부 추적
         └─ USB Storage    ← driver: 사용 중 여부 추적
   → 모든 하위 컴포넌트가 미사용 → 상위 컴포넌트 전원 차단

3. Wakelocks:
   애플리케이션이 wakelock 획득 → 시스템 power collapse 방지
   앱 완료 후 wakelock 해제 → power collapse 허용
```

**ACPI (Advanced Configuration and Power Interface):**
- 업계 표준 firmware + OS 인터페이스
- 장치 상태 발견/관리, 오류 관리, 전력 관리 루틴 제공
- OS ↔ 드라이버 ↔ ACPI 루틴 ↔ 장치 체인으로 장치 제어

---

## 12.5 I/O 요청의 하드웨어 변환 (Transforming I/O Requests)

**Unix 경로명 → 디스크 섹터 변환 과정:**

```
파일명 (e.g., /usr/local/file.txt)
    │
    ▼ mount table에서 최장 매칭 prefix 탐색
장치 이름 (/dev/sda2)
    │
    ▼ 파일시스템 디렉토리 탐색
<major, minor> 번호
    │
    ▼ major → 드라이버 선택, minor → 장치 테이블 인덱스
포트 주소 / 메모리 매핑 주소
    │
    ▼
Device Controller
```

**Blocking Read() 생명 주기 (10단계):**

```
1.  프로세스: blocking read() 시스템 콜
2.  커널: 파라미터 검증, buffer cache 확인 → hit 시 즉시 반환
3.  물리 I/O 필요: 프로세스를 run queue → wait queue 이동
4.  디바이스 드라이버: 커널 버퍼 할당, I/O 명령을 컨트롤러 레지스터에 기록
5.  장치 컨트롤러: 하드웨어 데이터 전송 수행
6.  DMA 컨트롤러가 커널 메모리로 데이터 전송
7.  인터럽트 핸들러: I/O 완료, 상태 저장, 드라이버 시그널
8.  드라이버: 완료 확인, 상태 검사, 커널 I/O 서브시스템에 완료 보고
9.  커널: 데이터/반환값을 프로세스 주소 공간으로 복사, wait queue → ready queue
10. 스케줄러가 CPU 배정 시 프로세스 재개 (시스템 콜 반환값 수신)
```

---

## 12.6 STREAMS (UNIX System V)

**STREAMS 구조:**

```
User Process
    │ write()/putmsg() / read()/getmsg()
    ▼
Stream Head          ← 사용자 프로세스 인터페이스
  [read queue | write queue]
    │  ↕ message passing
  Stream Module A    ← ioctl()로 동적 추가 (예: 입력 편집)
  [read queue | write queue]
    │  ↕ message passing
  Stream Module B    ← 프로토콜 처리 (예: TCP/IP)
  [read queue | write queue]
    │  ↕ message passing
Driver End           ← 장치 컨트롤러 인터페이스
  [read queue | write queue]
    │
    ▼
Device Controller
```

**Flow Control**: 큐 overflow 방지 → 공간 부족 시 인접 모듈에 제어 메시지 송신
- Driver End: 흐름 제어 지원 필수, 버퍼 포화 시 수신 메시지 드롭
- 장점: 모듈 재사용 (이더넷 + 무이랄리스 NIC에 동일 네트워크 모듈 적용)
- Solaris: 소켓 메커니즘을 STREAMS로 구현

---

## 12.7 성능 (Performance)

### I/O 성능 병목

```
Context Switch 비용:
  상태 저장 → 인터럽트 핸들러 실행 → 상태 복원
  단일 사용자 PC: 초당 수천 인터럽트
  서버: 초당 수십만 인터럽트

데이터 복사 경로:
  Device → Controller Buffer → DMA → Kernel Buffer → User Space
  (각 복사 = 메모리 대역폭 소비)

원격 로그인 문자 1개 전송:
  키보드 인터럽트 → 드라이버 → 커널 → 프로세스 (컨텍스트 스위치)
  → 네트워크 시스템 콜 → 네트워크 레이어 → NIC 드라이버
  → NIC 전송 인터럽트 → 시스템 콜 완료 (컨텍스트 스위치)
  + 원격 수신 처리 + 에코 반환 = 총 수십 번 컨텍스트 스위치
```

### I/O 성능 개선 원칙

| 원칙 | 방법 |
|------|------|
| 컨텍스트 스위치 감소 | 인터럽트 배치 처리, 폴링 병행 |
| 데이터 복사 감소 | DMA + 메모리 매핑으로 커널 버퍼 우회 |
| 인터럽트 빈도 감소 | 대용량 전송, 스마트 컨트롤러, 적응형 폴링 |
| 동시성 증가 | DMA 오프로드, I/O 채널(메인프레임) |
| 하드웨어 처리 이전 | 컨트롤러 내 처리 → CPU 병렬 수행 |
| 균형 유지 | CPU/메모리/버스/I/O 중 한 곳 병목 → 다른 곳 유휴 |

### 기능 구현 위치 결정 (진화 패턴)

```
응용 프로그램 레벨     → 유연성 최고, 성능 낮음 (컨텍스트 스위치 오버헤드)
커널 / 드라이버        → 중간 성능, 개발 복잡도 증가
장치 컨트롤러 / 하드웨어 → 최고 성능, 수정 어려움 (RAID 컨트롤러 최적화 불가)
```

---

## 핵심 요약

```
I/O 하드웨어:
  폴링 → 인터럽트 → DMA  (성능 오버헤드 감소 순)
  Memory-Mapped I/O → 현대 주류

애플리케이션 인터페이스:
  블록 / 문자 / 네트워크 소켓 / 타이머 → 4대 추상화
  blocking < nonblocking < async  (응용 복잡도 증가 순)

커널 서비스:
  스케줄링 + 버퍼링(double buffer, copy semantics) +
  캐싱(buffer cache) + 스풀링 + 오류처리 + 전력관리

I/O 요청 변환:
  파일명 → mount table → major/minor → driver → controller

성능:
  DMA + 대용량 전송 + 적응형 폴링 + 하드웨어 처리 이전
```
