# Chapter 6: Synchronization Tools

> 경쟁 조건(race condition)과 임계 구역 문제(critical-section problem)를 정의하고, 소프트웨어(Peterson's)·하드웨어(메모리 배리어/CAS/원자 변수)·고수준(뮤텍스 락·세마포·모니터·조건 변수) 동기화 도구를 다룬다. 라이브니스 실패(교착·우선순위 역전)와 도구별 성능 트레이드오프도 포함.

---

## 6.1 배경 (Background)

### 경쟁 조건 (Race Condition)

생산자-소비자 모델에서 공유 변수 `count`를 동시에 조작할 때 발생하는 오류:

```c
/* 생산자: count++ 의 기계어 분해 */
register1 = count         // (1) count=5
register1 = register1 + 1 // (2) register1=6

/* 소비자: count-- 의 기계어 분해 */
register2 = count         // (3) count=5  ← 인터리빙!
register2 = register2 - 1 // (4) register2=4
count = register1         // (5) count=6  ← 잘못된 값
count = register2         // (6) count=4  ← 덮어씀
```

단계 (5)와 (6)의 순서에 따라 `count`는 4, 5, 6 중 어느 값도 될 수 있다. 올바른 결과는 5뿐이다.

**경쟁 조건(Race Condition)**: 여러 프로세스가 공유 데이터를 동시에 접근·조작할 때, 실행 순서에 따라 결과가 달라지는 상황.

---

## 6.2 임계 구역 문제 (The Critical-Section Problem)

### 임계 구역 구조

```c
while (true) {
    /* entry section */       // 진입 허가 요청
        critical section      // 공유 데이터 접근
    /* exit section */        // 임계 구역 해제
        remainder section     // 나머지 코드
}
```

### 올바른 해법의 3가지 필요충분 조건

| 조건 | 설명 |
|------|------|
| **상호 배제(Mutual Exclusion)** | 프로세스 Pi가 임계 구역 실행 중이면 다른 어떤 프로세스도 임계 구역 진입 불가 |
| **진행(Progress)** | 임계 구역 실행 중인 프로세스가 없고, 진입 대기자가 있다면 remainder section 이외의 프로세스들이 진입자를 결정하며 무한 연기 불가 |
| **한정된 대기(Bounded Waiting)** | 프로세스가 진입 요청 후 다른 프로세스들이 임계 구역에 진입하는 횟수에 상한이 존재 |

### 커널의 경쟁 조건

커널 코드도 경쟁 조건에 취약하다:
- `next_available_pid` — fork() 시 PID 할당
- 열린 파일 목록, 메모리 할당 구조체, 프로세스 목록, 인터럽트 처리 자료구조

**선점 커널(Preemptive Kernel) vs. 비선점 커널(Nonpreemptive Kernel)**:

| 구분 | 경쟁 조건 위험 | 실시간 적합성 | 응답성 |
|------|--------------|-------------|--------|
| 비선점 커널 | 낮음 (한 번에 하나만 실행) | 나쁨 | 나쁨 |
| 선점 커널 | 높음 (SMP에서 특히) | 좋음 | 좋음 |

현대 OS는 대부분 선점 커널을 채택하며, 동기화 도구로 경쟁 조건을 방어한다.

---

## 6.3 Peterson의 해법 (Peterson's Solution)

두 프로세스(P0, P1)에 대한 소프트웨어 기반 고전 해법. **현대 아키텍처에서는 명령어 재정렬(reordering)로 인해 작동 보장 불가**이지만, 알고리즘 설계 원리를 이해하는 데 유용하다.

### 공유 변수

```c
int turn;        // 임계 구역 진입 차례 (0 또는 1)
bool flag[2];    // 임계 구역 진입 의사 표시
```

### 알고리즘

```c
/* 프로세스 Pi (j = 1 - i) */
while (true) {
    flag[i] = true;    // 진입 의사 표명
    turn = j;          // 상대방에게 우선권 양보
    while (flag[j] && turn == j)
        ;              // busy wait: 상대방이 진입 의사 있고 상대 차례면 대기
    /* critical section */
    flag[i] = false;   // 의사 철회
    /* remainder section */
}
```

### 정확성 증명

1. **상호 배제**: Pi가 임계 구역 진입 조건 — `flag[j]==false` 또는 `turn==i`. `turn`은 동시에 i와 j가 될 수 없으므로 두 프로세스가 동시에 조건을 만족할 수 없다.
2. **진행**: Pj가 임계 구역에 없으면(`flag[j]==false`) Pi가 즉시 진입. Pj가 대기 중이면 `turn` 값에 따라 정확히 하나가 진입.
3. **한정된 대기**: Pj가 임계 구역을 나오면 `flag[j]=false`로 설정 → Pi는 최대 1번의 Pj 진입 후 진입 보장.

### 현대 아키텍처에서의 문제점

```
/* 명령어 재정렬 예시 */
Thread 2 원래 순서:   x = 100; flag = true;
재정렬 후:            flag = true; x = 100;  ← Thread 1이 x=0을 출력할 수 있음

Peterson 해법에서:
  flag[i] = true;
  turn = j;
두 명령이 재정렬되면 → 두 프로세스가 동시에 임계 구역 진입 가능
```

올바른 동기화는 **하드웨어 지원**이 필수적이다.

---

## 6.4 동기화를 위한 하드웨어 지원 (Hardware Support for Synchronization)

### 6.4.1 메모리 배리어 (Memory Barriers / Memory Fences)

**메모리 모델(Memory Model)**:
- **강한 순서(Strongly Ordered)**: 한 프로세서의 메모리 변경이 즉시 다른 모든 프로세서에 가시
- **약한 순서(Weakly Ordered)**: 변경이 즉시 다른 프로세서에 가시하지 않을 수 있음

**메모리 배리어**: 이후의 load/store 연산보다 이전의 load/store 연산이 먼저 완료되도록 강제하는 명령어.

```c
/* Thread 1 */
while (!flag)
    memory_barrier();   // flag 로드 완료 보장
print x;                // x는 반드시 최신 값

/* Thread 2 */
x = 100;
memory_barrier();       // x 저장 완료 보장
flag = true;            // flag는 x 이후에 갱신됨
```

> 메모리 배리어는 매우 저수준 연산으로 커널 개발자가 특수 코드를 작성할 때만 직접 사용한다.

### 6.4.2 하드웨어 명령어 (Hardware Instructions)

#### test_and_set()

```c
/* 원자적(atomic) 실행 - 인터럽트 불가 */
bool test_and_set(bool *target) {
    bool rv = *target;
    *target = true;   // 항상 true로 설정
    return rv;        // 이전 값 반환
}

/* 뮤텍스 구현 */
bool lock = false;
do {
    while (test_and_set(&lock))
        ;             // busy wait
    /* critical section */
    lock = false;
    /* remainder section */
} while (true);
```

**문제**: 한정된 대기(bounded waiting)를 만족하지 않음.

#### compare_and_swap (CAS)

```c
/* 원자적 실행 */
int compare_and_swap(int *value, int expected, int new_value) {
    int temp = *value;
    if (*value == expected)
        *value = new_value;   // 기대값과 일치할 때만 갱신
    return temp;              // 항상 이전 값 반환
}

/* 기본 뮤텍스 구현 */
int lock = 0;
while (true) {
    while (compare_and_swap(&lock, 0, 1) != 0)
        ;             // busy wait
    /* critical section */
    lock = 0;
    /* remainder section */
}
```

#### CAS + bounded waiting (한정된 대기 보장)

```c
/* 공유 변수 */
bool waiting[n];   // 초기화: false
int lock;          // 초기화: 0

while (true) {
    waiting[i] = true;
    int key = 1;
    while (waiting[i] && key == 1)
        key = compare_and_swap(&lock, 0, 1);
    waiting[i] = false;

    /* critical section */

    /* 다음 대기 프로세스를 순환 탐색 */
    j = (i + 1) % n;
    while ((j != i) && !waiting[j])
        j = (j + 1) % n;
    if (j == i)
        lock = 0;           // 대기자 없음 → 락 해제
    else
        waiting[j] = false; // j를 다음 진입자로 지정

    /* remainder section */
}
```

- 상호 배제: `waiting[i]==false` 또는 `key==0`일 때만 진입
- 한정된 대기: 순환 탐색으로 최대 n-1번 대기 후 진입 보장

> **Intel x86**: `lock cmpxchg <dest>, <src>` — `lock` 접두사로 버스를 잠가 원자성 보장

### 6.4.3 원자적 변수 (Atomic Variables)

CAS를 기반으로 만든 정수·불리언 등 기본 타입의 원자적 연산 변수.

```c
atomic_int sequence;

/* CAS 기반 원자적 증가 */
void increment(atomic_int *v) {
    int temp;
    do {
        temp = *v;
    } while (temp != compare_and_swap(v, temp, temp + 1));
}
```

**한계**: 단일 변수 갱신에는 유효하지만, 복수 변수 간 복합 조건(예: 버퍼의 count + while 루프)에는 불충분 → 더 강력한 도구 필요.

---

## 6.5 뮤텍스 락 (Mutex Locks)

임계 구역 보호를 위한 가장 단순한 고수준 동기화 도구. mutex = **mut**ual **ex**clusion.

```c
/* 구현 원리 */
bool available = true;

void acquire() {
    while (!available)
        ;              // busy wait (스핀락)
    available = false;
}

void release() {
    available = true;
}

/* 사용 패턴 */
while (true) {
    acquire();
    /* critical section */
    release();
    /* remainder section */
}
```

### 스핀락 (Spinlock)

acquire/release가 CAS로 원자적 구현된 뮤텍스 락의 별칭.

| 특성 | 설명 |
|------|------|
| **장점** | 컨텍스트 스위치 없음 → 짧은 대기 시 효율적 |
| **단점** | 긴 대기 시 CPU 사이클 낭비 |
| **적합** | 멀티코어에서 락을 짧게 보유하는 경우 |
| **부적합** | 단일 코어 / 긴 임계 구역 |

> **일반 규칙**: 락 보유 시간 < 두 번의 컨텍스트 스위치 시간이면 스핀락이 유리.

**잠금 경합(Lock Contention)**:
- **비경합(Uncontended)**: 스레드가 락 획득 시도 시 즉시 가용
- **낮은 경합(Low contention)**: 소수의 스레드가 경쟁
- **높은 경합(High contention)**: 다수의 스레드가 경쟁 → 성능 저하

---

## 6.6 세마포 (Semaphores)

정수 변수 S에 두 원자적 연산 wait()·signal()만 허용하는 동기화 도구. Dijkstra가 도입 (원래 명칭: P/V 연산).

```c
/* 기본 정의 (busy-wait 버전) */
void wait(int *S) {
    while (*S <= 0)
        ;     // busy wait
    (*S)--;
}

void signal(int *S) {
    (*S)++;
}
```

### 6.6.1 세마포 활용

**이진 세마포(Binary Semaphore, 0~1)**: 뮤텍스 락과 동일하게 동작.

**계수 세마포(Counting Semaphore)**: 유한한 자원 인스턴스 수를 제어.
```
S = 자원 인스턴스 수로 초기화
사용 전: wait(S) → S--
사용 후: signal(S) → S++
S == 0이면 모든 자원 사용 중 → 이후 wait() 블록
```

**실행 순서 제어** (S1 → S2 보장):
```c
// P1:
S1;
signal(synch);   // synch 초기값 = 0

// P2:
wait(synch);     // S1 완료 전까지 대기
S2;
```

### 6.6.2 세마포 구현 (블로킹 버전)

busy wait 대신 프로세스를 대기 큐로 suspend:

```c
typedef struct {
    int value;
    struct process *list;  // 대기 프로세스 큐
} semaphore;

void wait(semaphore *S) {
    S->value--;
    if (S->value < 0) {
        // 대기 큐에 추가 후 sleep
        add_this_process_to(S->list);
        sleep();
    }
}

void signal(semaphore *S) {
    S->value++;
    if (S->value <= 0) {
        // 대기 큐에서 하나 깨움
        process *P = remove_from(S->list);
        wakeup(P);
    }
}
```

**핵심 특성**:
- `value < 0`이면 `|value|` = 현재 대기 중인 프로세스 수
- busy wait가 wait()/signal() 내부로 제한 (수십 개 명령어) → 애플리케이션의 긴 임계 구역 busy wait 제거
- 단일 프로세서: 인터럽트 비활성화로 원자성 보장
- 멀티코어: CAS 또는 스핀락으로 원자성 보장

---

## 6.7 모니터 (Monitors)

세마포/뮤텍스를 잘못 사용하면 검출하기 어려운 타이밍 버그 발생:

```c
// 오류 1: wait/signal 순서 바꿈 → 상호 배제 위반
signal(mutex); ... critical section ... wait(mutex);

// 오류 2: signal 대신 wait → 영구 블록
wait(mutex); ... critical section ... wait(mutex);

// 오류 3: wait 또는 signal 누락
```

모니터는 이런 오류를 언어 수준에서 방지하는 **고수준 동기화 추상화**다.

### 6.7.1 모니터 사용법 (Monitor Usage)

```
monitor monitor_name {
    /* 공유 변수 선언 */

    function P1(...) { ... }
    function P2(...) { ... }
    ...
    initialization_code(...) { ... }
}
```

```
모니터 구조:
┌─────────────────────────────────────┐
│  공유 데이터                          │
│  ┌──────────────────────────────┐   │
│  │  operations (함수들)          │   │ ← 한 번에 하나의 프로세스만
│  └──────────────────────────────┘   │   모니터 내부에서 실행 가능
│  초기화 코드                          │
└─────────────────────────────────────┘
       ↑
  entry queue (진입 대기열)
```

**핵심**: 모니터는 내부에서 동시에 하나의 프로세스만 활성화됨을 자동 보장 → 상호 배제를 명시적으로 코딩할 필요 없음.

### 조건 변수 (Condition Variables)

```c
condition x, y;

x.wait();     // 현재 프로세스를 x에 suspend
x.signal();   // x에서 대기 중인 프로세스 하나를 재개 (없으면 무효)
```

세마포 signal()과의 차이: 조건 변수 signal()은 대기자가 없으면 **아무 효과 없음**.

**signal 후 처리 방식**:

| 방식 | 설명 |
|------|------|
| **Signal and Wait** | 신호를 보낸 P가 대기, 재개된 Q가 실행 (Hoare 방식) |
| **Signal and Continue** | P가 계속 실행, Q는 나중에 실행 (Mesa 방식, Java 등) |

### 6.7.2 세마포로 모니터 구현

```c
/* 모니터 진입/퇴출 */
semaphore mutex = 1;   // 모니터 상호 배제

/* signal-and-wait 구현 */
semaphore next = 0;    // 신호 보낸 프로세스 대기용
int next_count = 0;

/* 각 함수 F를 감싸는 패턴 */
wait(mutex);
... F의 본문 ...
if (next_count > 0)
    signal(next);
else
    signal(mutex);

/* 조건 변수 x */
semaphore x_sem = 0;
int x_count = 0;

/* x.wait() */
x_count++;
if (next_count > 0) signal(next); else signal(mutex);
wait(x_sem);
x_count--;

/* x.signal() */
if (x_count > 0) {
    next_count++;
    signal(x_sem);   // x 대기자 깨움
    wait(next);      // 자신은 대기
    next_count--;
}
```

### 6.7.3 우선순위 대기 (Conditional-Wait)

```c
x.wait(c);  // c: 우선순위 번호(정수), 낮을수록 먼저 재개됨
```

```c
/* 단일 자원 할당기 예시 */
monitor ResourceAllocator {
    bool busy;
    condition x;

    void acquire(int time) {
        if (busy)
            x.wait(time);   // 요청 시간이 짧은 순서로 재개
        busy = true;
    }

    void release() {
        busy = false;
        x.signal();
    }

    initialization_code() { busy = false; }
}
```

---

## 6.8 라이브니스 (Liveness)

동기화 도구 사용 시 프로세스가 영구적으로 진행하지 못하는 **라이브니스 실패(Liveness Failure)** 가 발생할 수 있다.

### 6.8.1 교착 상태 (Deadlock)

두 프로세스가 서로가 보유한 세마포를 기다리며 영구 블록:

```
P0: wait(S) → wait(Q) → ...
P1: wait(Q) → wait(S) → ...

초기 상태: S=1, Q=1
P0: wait(S) → S=0 (성공)
P1: wait(Q) → Q=0 (성공)
P0: wait(Q) → Q=0, 블록 (P1이 signal(Q) 해야 하는데...)
P1: wait(S) → S=0, 블록 (P0이 signal(S) 해야 하는데...)
→ 상호 영구 대기 = 교착 상태
```

교착 상태의 자세한 처리 방법은 Chapter 8에서 다룬다.

### 6.8.2 우선순위 역전 (Priority Inversion)

**상황**: 우선순위 L < M < H인 세 프로세스가 있을 때:

```
시간 흐름:
1. H가 세마포 S 필요 (L이 보유 중 → H 블록)
2. M이 runnable → L을 선점 (L은 S를 보유한 채 실행 못 함)
3. 결과: H < M < 우선순위임에도 M이 H보다 먼저 실행

실제 사례: 1997년 화성 Pathfinder 탐사선
  - 고우선순위 'bc_dist' 태스크 지연
  - 저우선순위 'ASI/MET' 태스크가 공유 자원 보유
  - 중간 우선순위 태스크들이 ASI/MET 선점
  - 결과: bc_dist 시간 초과 → 시스템 주기적 리셋
  - 해결: VxWorks의 우선순위 상속 프로토콜 전역 활성화
```

**해결책: 우선순위 상속 프로토콜(Priority-Inheritance Protocol)**

```
L이 S 보유 중, H가 S 요청 시:
→ L이 H의 우선순위를 임시 상속
→ M이 L을 선점 불가
→ L이 S 해제 후 원래 우선순위 복귀
→ H가 S 획득하여 실행
```

---

## 6.9 성능 평가 (Evaluation)

동기화 도구 선택 시 **경합 수준(contention level)**에 따른 성능 특성을 고려해야 한다.

### CAS 기반 vs. 전통적 락 (뮤텍스/세마포)

| 경합 수준 | CAS 기반 | 전통적 락 |
|----------|---------|----------|
| **비경합(Uncontended)** | 약간 빠름 | 빠름 |
| **중간 경합(Moderate)** | **훨씬 빠름** | 느림 (컨텍스트 스위치 발생) |
| **높은 경합(High)** | 느림 (루프 반복) | **빠름** |

**중간 경합에서 CAS가 유리한 이유**: CAS 실패 시 몇 번의 루프로 재시도 vs. 뮤텍스는 스레드 suspend + 대기 큐 + 컨텍스트 스위치 비용.

**높은 경합에서 전통 락이 유리한 이유**: CAS는 충돌이 잦으면 루프를 계속 돌지만, 전통 락은 대기 스레드가 잠들어 CPU 낭비 없음.

### 도구 선택 지침

| 상황 | 권장 도구 |
|------|----------|
| 단일 공유 변수(카운터 등) | 원자적 변수 (atomic integer) |
| 짧은 임계 구역, 멀티코어 | 스핀락 (spinlock) |
| 일반 임계 구역 보호 | 뮤텍스 락 |
| 유한 개수 자원 제어 | 계수 세마포 |
| 복수 읽기 / 단수 쓰기 | Reader-Writer 락 |
| 복잡한 동기화 조건 | 모니터 + 조건 변수 |

---

## 6.10 핵심 정리

```
동기화 도구 계층:
─────────────────────────────────────────────────────
하드웨어 지원 (기반)
  ├─ 메모리 배리어  → 명령어 재정렬 방지
  ├─ test_and_set() → 원자적 읽기-쓰기
  └─ compare_and_swap (CAS) → 조건부 원자적 교환

저수준 소프트웨어 도구
  ├─ 뮤텍스 락(Mutex) → 이진 잠금, 스핀락 구현
  └─ 세마포(Semaphore) → 계수 기반, 대기 큐 포함

고수준 언어 구조
  └─ 모니터(Monitor) + 조건 변수(Condition Variable)
       → 자동 상호 배제, 복잡한 조건 동기화
─────────────────────────────────────────────────────
```

| 도구 | 상호 배제 | 실행 순서 제어 | busy wait | 오용 위험 |
|------|---------|--------------|----------|----------|
| 스핀락 | ✓ | ✗ | ✓ | 낮음 |
| 뮤텍스 락 | ✓ | ✗ | ✗(블로킹) | 낮음 |
| 이진 세마포 | ✓ | ✗ | ✗(블로킹) | 중간 |
| 계수 세마포 | ✓(N개) | ✓ | ✗(블로킹) | 중간 |
| 모니터 | ✓(자동) | ✓(조건변수) | ✗(블로킹) | 낮음 |

**라이브니스 실패 요약**:

| 문제 | 원인 | 해결 |
|------|------|------|
| 교착(Deadlock) | 순환 자원 대기 | 자원 획득 순서 고정 / 회피 알고리즘 (Ch.8) |
| 기아(Starvation) | 불공정 스케줄링 | FIFO 대기 큐, 에이징 |
| 우선순위 역전 | 저우선순위가 락 보유 | 우선순위 상속 프로토콜 |
