# Chapter 5: CPU Scheduling

> CPU 스케줄링 기본 개념(CPU–I/O 버스트 사이클, 선점/비선점, 디스패처), 스케줄링 기준 5종, 핵심 알고리즘 6가지(FCFS/SJF/RR/Priority/Multilevel Queue/MLFQ), 멀티프로세서 스케줄링(SMP/CMT/부하 균형/프로세서 친화성/HMP), 실시간 스케줄링(Rate-Monotonic/EDF), Linux CFS/Windows/Solaris 구현 사례, 알고리즘 평가 기법까지 다룬다.

---

## 5.1 기본 개념 (Basic Concepts)

### CPU–I/O 버스트 사이클 (CPU–I/O Burst Cycle)

프로세스 실행은 **CPU 버스트**와 **I/O 버스트**의 반복으로 이루어진다.

```
프로세스 실행 흐름:
CPU burst → I/O burst → CPU burst → I/O burst → ... → terminate

특성:
• I/O-bound 프로세스: 짧은 CPU 버스트 다수
• CPU-bound 프로세스: 긴 CPU 버스트 소수
• 빈도 분포: 지수(exponential) 또는 초지수(hyperexponential) 형태
```

### CPU 스케줄러 (CPU Scheduler)

CPU가 유휴 상태가 될 때마다 준비 큐(ready queue)에서 실행할 프로세스를 선택한다. 준비 큐는 FIFO 큐, 우선순위 큐, 트리, 비정렬 연결 리스트 등으로 구현 가능하다.

### 5.1.3 선점(Preemptive) vs. 비선점(Nonpreemptive) 스케줄링

스케줄링 결정이 발생하는 4가지 상황:

| 상황 | 전환 | 선택 여부 |
|------|------|----------|
| 1. running → waiting (I/O 요청 등) | 자발적 | 선택 없음 |
| 2. running → ready (인터럽트 발생) | 강제 | **선택 가능** |
| 3. waiting → ready (I/O 완료) | 자발적 | **선택 가능** |
| 4. 프로세스 종료 | - | 선택 없음 |

- **비선점(Nonpreemptive/Cooperative)**: 상황 1·4에서만 스케줄링 → 프로세스가 CPU를 자발적으로 반납할 때까지 유지
- **선점(Preemptive)**: 상황 2·3에서도 스케줄링 가능 → Windows, macOS, Linux, UNIX 모두 선점 방식

선점 스케줄링의 문제점:
- 공유 데이터 접근 시 **경쟁 조건(race condition)** 발생 가능 (Ch.6 참조)
- 커널 자료구조 수정 중 선점 시 커널 데이터 불일치 위험 → mutex 락 필요

### 5.1.4 디스패처 (Dispatcher)

스케줄러가 선택한 프로세스에 CPU를 실제로 넘겨주는 모듈:
1. 컨텍스트 스위치 수행
2. 사용자 모드로 전환
3. 사용자 프로그램의 재시작 지점으로 점프

**디스패치 지연(Dispatch Latency)**: 한 프로세스를 멈추고 다른 프로세스를 시작하는 데 걸리는 시간.

```bash
# Linux에서 컨텍스트 스위치 횟수 확인
vmstat 1 3

# 특정 프로세스의 컨텍스트 스위치 확인
cat /proc/<PID>/status
# voluntary_ctxt_switches    150  ← 자발적 (I/O 대기 등)
# nonvoluntary_ctxt_switches   8  ← 비자발적 (타임 슬라이스 만료 등)
```

---

## 5.2 스케줄링 기준 (Scheduling Criteria)

| 기준 | 방향 | 설명 |
|------|------|------|
| **CPU 이용률(CPU utilization)** | 최대화 | 실제 시스템: 40~90% |
| **처리량(Throughput)** | 최대화 | 단위 시간당 완료 프로세스 수 |
| **반환 시간(Turnaround time)** | 최소화 | 제출~완료까지의 전체 시간 |
| **대기 시간(Waiting time)** | 최소화 | 준비 큐에서 대기한 시간의 합 |
| **응답 시간(Response time)** | 최소화 | 요청 제출~첫 응답 시작까지의 시간 |

> 대화형 시스템에서는 평균 응답 시간보다 **응답 시간의 분산(variance)** 최소화가 더 중요하다.

---

## 5.3 스케줄링 알고리즘 (Scheduling Algorithms)

### 5.3.1 FCFS (First-Come, First-Served)

가장 단순한 알고리즘. FIFO 큐로 구현.

```
예시 (P1=24ms, P2=3ms, P3=3ms, 도착 순서 P1→P2→P3):

  P1          P2  P3
  ─────────────|──|──
  0           24 27 30

평균 대기 시간: (0 + 24 + 27) / 3 = 17ms

도착 순서를 P2→P3→P1로 바꾸면:
  P2|P3|P1
  ──|──|──────────
  0  3  6         30

평균 대기 시간: (6 + 0 + 3) / 3 = 3ms
```

**문제점**:
- **호위 효과(Convoy Effect)**: CPU-bound 프로세스 하나가 CPU를 독점하면 뒤의 I/O-bound 프로세스들이 대기 → CPU·장치 이용률 저하
- 비선점 방식

### 5.3.2 SJF (Shortest-Job-First)

다음 CPU 버스트가 가장 짧은 프로세스에 CPU 할당. **평균 대기 시간이 최소**임이 증명된 최적 알고리즘.

```
예시 (P1=6, P2=8, P3=7, P4=3):
SJF 순서: P4→P1→P3→P2
  P4|P1    |P3     |P2
  ──|──────|───────|──────────
  0  3     9       16        24

평균 대기 시간: (3 + 16 + 9 + 0) / 4 = 7ms  (FCFS: 10.25ms)
```

**문제점**: 다음 CPU 버스트 길이를 미리 알 수 없다 → **지수 평균(Exponential Average)** 예측:

```
τ(n+1) = α·t(n) + (1−α)·τ(n)

• t(n): n번째 CPU 버스트 실제 길이
• τ(n): 예측값
• α: 최근 이력과 과거 이력의 가중치 (보통 α = 0.5)

α=0: 최근 이력 무시 (과거만 사용)
α=1: 과거 이력 무시 (최근만 사용)
α=0.5: 최근·과거 동등 가중치
```

**선점형 SJF = SRTF(Shortest-Remaining-Time-First)**:

```
예시 (P1: 도착 0, burst 8 / P2: 도착 1, burst 4 / P3: 도착 2, burst 9 / P4: 도착 3, burst 5):

  P1|P2    |P4  |P1         |P3
  ──|──────|────|───────────|──────────
  0  1     5    10          17        26

평균 대기 시간: [(10-0) + (1-1) + (17-2) + (5-3)] / 4 = 6.5ms
```

### 5.3.3 Round-Robin (RR)

**타임 퀀텀(time quantum, time slice)**을 정의하고 준비 큐를 원형으로 순환. 선점형.

```
예시 (P1=24, P2=3, P3=3, quantum=4):

  P1  P2  P3  P1  P1  P1  P1  P1
  ────|───|───|───────────────────
  0   4   7  10  14  18  22  26  30

평균 대기 시간: (6 + 4 + 7) / 3 = 5.67ms
```

**타임 퀀텀 크기의 영향**:

```
quantum → ∞  :  FCFS와 동일
quantum → 0  :  컨텍스트 스위치 오버헤드 폭증 (processor sharing 개념)

실용적 권장: CPU 버스트의 80%가 타임 퀀텀보다 짧도록 설정
실제 시스템: 10~100ms 범위
```

| 특성 | 설명 |
|------|------|
| 공정성 | n개 프로세스에 각각 1/n의 CPU 시간 |
| 최대 대기 | (n−1) × q 이하 |
| 반환 시간 | 보통 SJF보다 길지만 응답 시간은 개선 |

### 5.3.4 우선순위 스케줄링 (Priority Scheduling)

각 프로세스에 우선순위를 부여하고 가장 높은 우선순위의 프로세스를 먼저 실행. SJF는 우선순위 = 다음 CPU 버스트의 역수인 특수 케이스.

```
예시 (priority: 낮은 숫자 = 높은 우선순위):
P1(burst=10, pri=3) / P2(burst=1, pri=1) / P3(burst=2, pri=4)
P4(burst=1, pri=5) / P5(burst=5, pri=2)

실행 순서: P2 → P5 → P1 → P3 → P4
  P2|P5   |P1          |P3  |P4
  ──|─────|────────────|────|──
  0  1     6           16  18 19

평균 대기 시간: 8.2ms
```

**우선순위 종류**:
- **내부 우선순위**: 시간 제한, 메모리 요구량, I/O버스트 비율 등 측정값 기반
- **외부 우선순위**: 비용, 부서 등 OS 외부 기준

**핵심 문제: 기아(Starvation)**
- 낮은 우선순위 프로세스가 무한정 대기 가능
- **해결책: 에이징(Aging)** — 대기 시간에 따라 우선순위를 점진적으로 높임

**Priority + RR 조합**: 같은 우선순위의 프로세스 사이에서 RR로 실행.

### 5.3.5 다단계 큐 스케줄링 (Multilevel Queue Scheduling)

우선순위마다 별도의 큐를 두어 O(n) 탐색 없이 O(1)로 최고 우선순위 프로세스 선택.

```
highest ┌──────────────────┐ ← Real-time processes
        ├──────────────────┤ ← System processes
        ├──────────────────┤ ← Interactive processes
        └──────────────────┘ ← Batch processes
lowest
```

- 각 큐는 자체 스케줄링 알고리즘 보유 (예: foreground=RR, background=FCFS)
- 큐 간 스케줄링: 고정 우선순위 선점 또는 타임 슬라이스 분배 (예: foreground 80%, background 20%)
- **단점**: 프로세스가 큐 간에 이동하지 못함 → 유연성 부족

### 5.3.6 다단계 피드백 큐 스케줄링 (Multilevel Feedback Queue, MLFQ)

프로세스가 **큐 사이를 이동**할 수 있는 가장 유연한 알고리즘.

```
큐 0 (quantum=8ms)   ──→ 초과 시 큐 1로 강등
큐 1 (quantum=16ms)  ──→ 초과 시 큐 2로 강등
큐 2 (FCFS)          ──→ 큐 0, 1이 비어있을 때만 실행
```

- 짧은 CPU 버스트: 높은 우선순위 큐에 머뭄 → I/O-bound·인터랙티브 프로세스 유리
- 긴 CPU 버스트: 낮은 우선순위 큐로 자동 강등 → CPU-bound 프로세스
- 오래 대기한 프로세스: 상위 큐로 승격(aging) → 기아 방지

**MLFQ 정의 파라미터**:
1. 큐의 개수
2. 각 큐의 스케줄링 알고리즘
3. 프로세스를 상위 큐로 올리는 조건
4. 프로세스를 하위 큐로 내리는 조건
5. 프로세스가 처음 진입할 큐 결정 방법

---

## 5.4 스레드 스케줄링 (Thread Scheduling)

현대 OS에서는 프로세스가 아닌 **커널 스레드**를 스케줄링한다.

### 경쟁 범위 (Contention Scope)

| 범위 | 약어 | 설명 | 적용 모델 |
|------|------|------|----------|
| **프로세스 경쟁 범위** | PCS | 같은 프로세스 내 스레드 간 경쟁 → LWP 스케줄링 | Many-to-One, Many-to-Many |
| **시스템 경쟁 범위** | SCS | 시스템 전체 스레드 간 경쟁 → 실제 CPU 스케줄링 | One-to-One (Linux, Windows) |

### Pthreads 스케줄링 API

```c
/* 경쟁 범위 설정 */
pthread_attr_setscope(&attr, PTHREAD_SCOPE_SYSTEM);  /* SCS */
pthread_attr_setscope(&attr, PTHREAD_SCOPE_PROCESS); /* PCS */

/* 현재 범위 조회 */
pthread_attr_getscope(&attr, &scope);
```

Linux/macOS는 `PTHREAD_SCOPE_SYSTEM`만 지원 (One-to-One 모델).

---

## 5.5 멀티프로세서 스케줄링 (Multi-Processor Scheduling)

### 5.5.1 멀티프로세서 접근 방식

| 방식 | 설명 | 단점 |
|------|------|------|
| **비대칭 멀티프로세싱(Asymmetric)** | 마스터 프로세서가 모든 스케줄링 담당 | 마스터가 병목 |
| **대칭 멀티프로세싱(SMP)** | 각 프로세서가 자체 스케줄링 | 공유 큐 동기화 필요 |

**SMP 준비 큐 구성 두 가지**:

```
(a) 공통 준비 큐 (common ready queue)
    ┌────────────────────┐
    │  T0 T1 T2 T3 ...  │
    └──┬───┬───┬────────┘
      core0 core1 coren
    → 경쟁 조건 위험, 락 오버헤드

(b) 코어별 사설 큐 (per-core run queues) ← 현대 OS 표준
    core0: [T0,T1]  core1: [T2,T3]  coren: [Tn]
    → 캐시 친화성 이점, 부하 균형 알고리즘 필요
```

### 5.5.2 멀티코어 프로세서

**메모리 스톨(Memory Stall)**: 프로세서 속도 >> 메모리 속도 → 캐시 미스 시 CPU가 최대 50% 대기.

**해결책: 칩 멀티스레딩(Chip Multithreading, CMT)**

```
코어당 하드웨어 스레드 2개 이상 할당:
  thread0: C  M  C  M  C  M    ← M(메모리 스톨) 동안
  thread1:  C  M  C  M  C  M   ← 다른 스레드가 실행

Intel 용어: 하이퍼스레딩(Hyper-Threading, HT) 또는 SMT
  - i7: 코어당 2 하드웨어 스레드
  - Oracle Sparc M7: 코어당 8 스레드, 코어 8개 → 64 논리 CPU
```

**CMT 멀티스레딩 방식**:

| 방식 | 스위치 시점 | 파이프라인 플러시 비용 |
|------|------------|----------------------|
| **거친 입도(Coarse-grained)** | 메모리 스톨 등 긴 지연 이벤트 발생 시 | 높음 |
| **세밀한 입도(Fine-grained)** | 명령어 사이클 경계 | 낮음 (스위치 로직 내장) |

**2단계 스케줄링**:
- Level 1 (OS): 소프트웨어 스레드 → 논리 CPU (하드웨어 스레드) 할당
- Level 2 (코어 내부): 하드웨어 스레드 → 물리 코어 할당 (RR 또는 urgency 기반)

### 5.5.3 부하 균형 (Load Balancing)

SMP에서 프로세서 간 작업 부하를 균등하게 유지:

| 방식 | 설명 |
|------|------|
| **Push 마이그레이션** | 주기적으로 부하를 감시하다가 과부하 프로세서에서 유휴 프로세서로 스레드를 밀어넣음 |
| **Pull 마이그레이션** | 유휴 프로세서가 바쁜 프로세서에서 대기 스레드를 당겨옴 |

Linux CFS 스케줄러와 FreeBSD ULE 스케줄러는 push/pull을 동시에 구현.

### 5.5.4 프로세서 친화성 (Processor Affinity)

특정 프로세서에서 실행 중인 스레드는 해당 프로세서의 캐시를 워밍(warm cache)함. 스레드를 다른 프로세서로 이전하면 캐시 무효화 비용 발생 → 같은 프로세서 유지 선호.

| 종류 | 설명 |
|------|------|
| **소프트 친화성(Soft affinity)** | OS가 가능한 같은 프로세서 유지 시도 (보장 없음) |
| **하드 친화성(Hard affinity)** | 시스템 콜로 실행 가능 CPU 집합 명시적 지정 |

```c
/* Linux 하드 친화성 설정 */
sched_setaffinity(pid, cpusetsize, mask);
```

**NUMA와 스케줄링**:

```
CPU 0 ────────── Local Memory 0 (fast)
  │                     │
  interconnect ─────────┼──── (slow cross-NUMA access)
  │                     │
CPU 1 ────────── Local Memory 1 (fast)
```

부하 균형과 프로세서 친화성은 **상충(tradeoff)** 관계 — 스레드 이동으로 부하는 균등해지지만 캐시/메모리 접근 비용 증가.

### 5.5.5 이기종 멀티프로세싱 (Heterogeneous Multiprocessing, HMP)

같은 ISA이지만 성능·전력이 다른 코어를 혼합:

```
ARM big.LITTLE:
  big 코어 (고성능, 고전력)  ← 인터랙티브, 단시간 집중 작업
  LITTLE 코어 (저성능, 저전력) ← 백그라운드, 장시간 실행
```

Windows 10은 스레드가 자신의 전력 요구에 맞는 스케줄링 정책을 선택 가능.

---

## 5.6 실시간 CPU 스케줄링 (Real-Time CPU Scheduling)

| 구분 | 설명 |
|------|------|
| **소프트 실시간(Soft real-time)** | 실시간 프로세스를 우선시하지만 데드라인 보장 없음 |
| **하드 실시간(Hard real-time)** | 데드라인 내 서비스 필수 — 지난 서비스는 무서비스와 동일 |

### 5.6.1 지연 최소화 (Minimizing Latency)

**이벤트 지연(Event Latency)**: 이벤트 발생부터 서비스까지의 시간.

```
이벤트 → [인터럽트 지연] → ISR 시작 → [디스패치 지연] → 실시간 태스크 실행
```

- **인터럽트 지연(Interrupt Latency)**: 인터럽트 도착 → ISR 시작. 커널이 인터럽트를 비활성화하는 시간을 최소화해야 함
- **디스패치 지연(Dispatch Latency)**: 기존 프로세스 정지 → 새 프로세스 시작. 선점 커널로 최소화 (하드 실시간: 수 마이크로초 단위)

디스패치 지연의 충돌 단계(Conflict phase):
1. 커널에서 실행 중인 프로세스 선점
2. 저우선순위 프로세스가 보유한 자원을 고우선순위 프로세스에게 해제

### 5.6.2 우선순위 기반 스케줄링

실시간 OS는 반드시 **선점형 우선순위 스케줄링**을 지원해야 한다.

**주기적(Periodic) 태스크 파라미터**:
```
t: CPU 처리 시간 (processing time)
d: 데드라인 (deadline)
p: 주기 (period)
조건: 0 ≤ t ≤ d ≤ p

CPU 이용률: t/p
```

**입장 제어(Admission Control)**: 스케줄러가 새 태스크의 데드라인 보장 가능성을 판단해 수락/거절.

### 5.6.3 단조 속도 스케줄링 (Rate-Monotonic Scheduling, RM)

**정적 우선순위 + 선점**: 주기가 짧을수록 높은 우선순위 (주기의 역수에 비례).

```
예시: P1(p=50, t=20), P2(p=100, t=35)
CPU 이용률: 20/50 + 35/100 = 0.75 (75%)

RM 우선순위: P1 > P2 (짧은 주기 = 높은 우선순위)

시간:  0    20   50   70   75   100  120  ...
       P1    P2   P1→  P1   P2
              ↑P1 선점
```

**RM의 한계 — CPU 이용률 상한**:
```
N개 프로세스의 최악 CPU 이용률 = N(2^(1/N) − 1)
  N=1: 100%
  N=2: ~83%
  N=∞: ~69.3% (ln 2 ≈ 0.693)
```

정적 우선순위 알고리즘 중 **최적**이지만 69% 이상의 CPU 이용률을 보장할 수 없음.

### 5.6.4 최단 데드라인 우선 스케줄링 (Earliest-Deadline-First, EDF)

**동적 우선순위**: 데드라인이 빠를수록 높은 우선순위. 태스크가 runnable 상태가 될 때마다 데드라인을 선언해야 한다.

```
특성:
• 주기적 태스크 불필요, 버스트 크기도 가변 가능
• 이론적으로 CPU 이용률 100% 달성 가능 (최적 알고리즘)
• 실제로는 컨텍스트 스위치·인터럽트 처리 비용으로 100% 불가
• RM이 실패하는 고이용률(94%) 케이스도 스케줄 가능
```

| 알고리즘 | 우선순위 | 적용 조건 | CPU 이용률 상한 |
|---------|---------|----------|----------------|
| Rate-Monotonic | 정적 (주기 역수) | 주기적 태스크 | ~69% (N→∞) |
| EDF | 동적 (데드라인) | 제약 없음 | 이론적 100% |

### 5.6.5 비례 공유 스케줄링 (Proportional Share Scheduling)

T 공유를 전체 애플리케이션에 분배. 프로세스 A가 N 공유를 받으면 N/T의 CPU 시간 보장.

```
예시: T=100
  A: 50 shares (50% CPU)
  B: 15 shares (15% CPU)
  C: 20 shares (20% CPU)
  D가 30 shares 요청 → 85+30=115 > 100 → 거절 (입장 제어)
```

### 5.6.6 POSIX 실시간 스케줄링

```c
/* POSIX 실시간 스케줄링 클래스 */
SCHED_FIFO   /* 같은 우선순위: 타임 슬라이싱 없음, FCFS */
SCHED_RR     /* 같은 우선순위: 타임 슬라이싱(RR) */
SCHED_OTHER  /* 구현 정의 */

/* API */
pthread_attr_setschedpolicy(&attr, SCHED_FIFO);
pthread_attr_getschedpolicy(&attr, &policy);
```

---

## 5.7 OS별 스케줄링 구현 사례

### 5.7.1 Linux: Completely Fair Scheduler (CFS)

Linux 커널 2.6.23부터 기본 스케줄러. **스케줄링 클래스(Scheduling Class)** 기반 아키텍처:
- **CFS 클래스** (기본): nice 값 기반 비례 CPU 시간 할당
- **실시간 클래스**: SCHED_FIFO / SCHED_RR

#### CFS 핵심 메커니즘

```
vruntime (가상 실행 시간):
  • 각 태스크마다 유지되는 누적 실행 시간 (우선순위 보정)
  • nice=0 (기본): vruntime = 실제 실행 시간
  • 낮은 우선순위: vruntime 증가율 > 실제 시간
  • 높은 우선순위: vruntime 증가율 < 실제 시간

→ 다음 실행 태스크 = vruntime이 가장 작은 태스크
```

**targeted latency**: 모든 runnable 태스크가 적어도 한 번씩 실행되는 목표 시간 간격. 이 간격 안에서 CPU 시간 비례 할당.

**nice 값과 우선순위**:
```
nice 범위: −20(높은 우선순위) ~ +19(낮은 우선순위)
Linux 전역 우선순위:
  0  ~  99: 실시간 (SCHED_FIFO/RR), 숫자 작을수록 높은 우선순위
  100 ~ 139: 일반 태스크 (nice -20 = 100, nice +19 = 139)
```

#### CFS의 자료구조: 레드-블랙 트리

```
Red-Black Tree (키: vruntime):
      T4(50)
     /       \
  T1(20)    T7(80)
  /    \    /    \
T0(10) T2(30) T5(70) T9(90)

가장 왼쪽 노드 = vruntime 최소 = 다음 실행 태스크
  → O(log N) 탐색, 단 rb_leftmost 캐시로 O(1) 조회
```

#### CFS 부하 균형

- 부하 = 우선순위 × 평균 CPU 이용률 (I/O-bound 고우선순위 스레드도 낮은 부하)
- **NUMA 인식 스케줄링 도메인(Scheduling Domain)** 계층:

```
코어0 ─┬─ domain0 (L2 공유)
코어1 ─┘           ├─ 물리 프로세서 도메인 (L3 공유)
코어2 ─┬─ domain1 ─┘
코어3 ─┘

균형 전략: 낮은 도메인부터 균형 → NUMA 경계 이동은 심각한 불균형에만 허용
```

### 5.7.2 Windows 스케줄링

**우선순위 기반 선점형 스케줄링**, 32단계 우선순위.

```
우선순위 구성:
  0     : 메모리 관리 전용
  1~15  : 가변(Variable) 클래스 (우선순위 동적 조정)
  16~31 : 실시간(Real-time) 클래스

API 우선순위 클래스:
  IDLE < BELOW_NORMAL < NORMAL < ABOVE_NORMAL < HIGH < REALTIME

스레드 상대 우선순위:
  IDLE / LOWEST / BELOW_NORMAL / NORMAL / ABOVE_NORMAL / HIGHEST / TIME_CRITICAL
```

**Windows 스레드 우선순위 매트릭스 (일부)**:

| 상대 우선순위 | ABOVE_NORMAL | NORMAL | BELOW_NORMAL | IDLE |
|-------------|-------------|--------|-------------|------|
| TIME_CRITICAL | 15 | 15 | 15 | 15 |
| HIGHEST | 12 | 10 | 8 | 6 |
| NORMAL | 10 | 8 | 6 | 4 |
| IDLE | 1 | 1 | 1 | 1 |

**우선순위 동적 조정 (가변 클래스)**:
- 타임 퀀텀 소진: 우선순위 **낮춤** (최소 기본 우선순위까지)
- 대기 완료(I/O 등): 우선순위 **올림** (키보드 I/O > 디스크 I/O)
- 포그라운드 프로세스: 타임 퀀텀 3배 증가

**Windows 7+ UMS(User-Mode Scheduling)**: 커널 개입 없이 user-space에서 스레드 생성·관리. Microsoft ConcRT(Concurrency Runtime)이 내부 구현에 활용.

**SMT 집합 기반 멀티코어 스케줄링**:
- 논리 프로세서를 SMT 집합으로 그룹화 (예: 하이퍼스레딩 쌍)
- 이상적 프로세서(ideal processor) 할당으로 코어 간 부하 분산

### 5.7.3 Solaris 스케줄링

6가지 스케줄링 클래스, 전역 우선순위로 통합 관리:

```
인터럽트 스레드          (160~169)  ← 최고
실시간 (RT)             (100~159)
시스템 (SYS)            (60~99)
공정 공유 (FSS)         
고정 우선순위 (FP)      (0~59)
시간 공유 (TS)          (0~59)      ← 기본
대화형 (IA)             (0~59)
```

**시간 공유(TS) 다단계 피드백 큐** 특성:
- 우선순위와 타임 퀀텀이 **반비례**: 높은 우선순위 → 짧은 퀀텀
- CPU 버스트가 퀀텀 초과 시 우선순위 **낮춤**
- I/O 완료(sleep → wake) 시 우선순위 **올림** (50~59)
- Solaris 9부터 One-to-One 모델로 전환

---

## 5.8 알고리즘 평가 (Algorithm Evaluation)

### 5.8.1 결정론적 모델링 (Deterministic Modeling)

특정 사전 정의 워크로드로 각 알고리즘의 성능을 수식으로 계산.

```
예시 (P1=10, P2=29, P3=3, P4=7, P5=12, 모두 시간 0 도착):

FCFS 평균 대기: (0+10+39+42+49)/5 = 28ms
SJF 평균 대기:  (10+32+0+3+20)/5  = 13ms  ← 최적
RR(q=10) 평균: (0+32+20+23+40)/5  = 23ms
```

단순하고 빠르지만 입력이 정확해야 하고 특정 워크로드에만 적용 가능.

### 5.8.2 대기열 모델 (Queueing Models)

CPU와 I/O를 서버로, 큐를 대기열로 모델링. **Little's Formula**:

```
n = λ × W

n: 큐의 평균 길이
λ: 평균 도착률 (프로세스/초)
W: 큐에서의 평균 대기 시간

예: λ=7/s, n=14  →  W = n/λ = 2초
```

수학적 가정이 현실과 괴리될 수 있어 근사 모델에 그침.

### 5.8.3 시뮬레이션 (Simulations)

컴퓨터 시스템을 소프트웨어로 모델링하여 알고리즘 성능 측정.

- **분포 기반 시뮬레이션**: 균등/지수/포아송 분포로 이벤트 생성
- **트레이스 파일 기반**: 실제 시스템 실행 이력을 캡처 → 동일 입력으로 알고리즘 비교 (가장 현실적)

### 5.8.4 구현 (Implementation)

알고리즘을 실제 OS에 코딩하여 실 운영 환경에서 평가. 가장 정확하지만 구현·테스트 비용이 가장 큼. 가상 머신 환경에서 회귀 테스트(regression testing) 수행 권장.

---

## 5.9 핵심 정리

| 알고리즘 | 선점 | 기아 | 특징 |
|---------|------|------|------|
| FCFS | ✗ | ✗ | 간단, 호위 효과 |
| SJF | ✗/✓ | ✓ | 최적 평균 대기 시간, 버스트 예측 필요 |
| SRTF | ✓ | ✓ | 선점형 SJF |
| RR | ✓ | ✗ | 타임 퀀텀, 응답 시간 개선 |
| Priority | ✗/✓ | ✓(에이징으로 해결) | 우선순위 기반 |
| MLQ | ✓ | ✓(낮은 큐) | 큐 간 이동 불가 |
| MLFQ | ✓ | ✗(에이징) | 가장 유연, 가장 복잡 |
| Rate-Monotonic | ✓ | - | 정적, 주기 역수 우선순위, 한계 ~69% |
| EDF | ✓ | - | 동적, 데드라인 기반, 이론적 100% |

| OS | 스케줄링 방식 | 특징 |
|----|-------------|------|
| **Linux** | CFS (vruntime + Red-Black Tree) | NUMA-aware, 스케줄링 도메인 계층 |
| **Windows** | 32단계 우선순위 + 동적 부스트 | SMT 집합, UMS, 포그라운드 3배 퀀텀 |
| **Solaris** | 6 클래스 + 전역 우선순위 | TS/IA 피드백 큐, RT 최고 우선순위 |
