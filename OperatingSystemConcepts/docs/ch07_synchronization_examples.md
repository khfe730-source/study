# Chapter 7: Synchronization Examples

## 목차
1. [고전적 동기화 문제 (Classic Synchronization Problems)](#1-고전적-동기화-문제)
2. [커널 내부 동기화 (Kernel Synchronization)](#2-커널-내부-동기화)
3. [POSIX 동기화](#3-posix-동기화)
4. [Java 동기화](#4-java-동기화)
5. [대안적 접근법 (Alternative Approaches)](#5-대안적-접근법)

---

## 1. 고전적 동기화 문제

### 1.1 유한 버퍼 문제 (Bounded-Buffer Problem)

공유 버퍼(크기 n)에 생산자(producer)와 소비자(consumer)가 동시 접근하는 구조.

**세마포 구성**
```
semaphore mutex = 1;   // 버퍼 접근 상호 배제
semaphore empty = n;   // 빈 슬롯 개수
semaphore full  = 0;   // 채워진 슬롯 개수
```

**생산자 (Producer)**
```c
while (true) {
    /* 아이템 생성 */
    wait(empty);      // 빈 슬롯 확보
    wait(mutex);      // 버퍼 잠금
    /* 버퍼에 아이템 삽입 */
    signal(mutex);    // 버퍼 잠금 해제
    signal(full);     // 채워진 슬롯 +1
}
```

**소비자 (Consumer)**
```c
while (true) {
    wait(full);       // 아이템 존재 대기
    wait(mutex);      // 버퍼 잠금
    /* 버퍼에서 아이템 제거 */
    signal(mutex);    // 버퍼 잠금 해제
    signal(empty);    // 빈 슬롯 +1
    /* 아이템 소비 */
}
```

> `empty`와 `full`은 순서를 제어, `mutex`는 버퍼 접근을 보호. `wait(mutex)` 전에 반드시 resource semaphore를 먼저 wait해야 데드락을 방지한다.

---

### 1.2 독자-작가 문제 (Readers-Writers Problem)

데이터셋에 다수의 독자(reader)와 작가(writer)가 동시 접근. 여러 독자는 동시 읽기 허용, 작가는 독점 접근 요구.

**제1 독자-작가 문제**: 독자 우선 — 작가가 오래 기다릴 수 있다 (작가 기아(starvation) 가능)

**세마포 구성**
```
semaphore rw_mutex = 1;  // 작가 상호 배제 + 첫/마지막 독자용
semaphore mutex   = 1;   // read_count 보호
int       read_count = 0;
```

**작가 (Writer)**
```c
wait(rw_mutex);
/* 쓰기 작업 */
signal(rw_mutex);
```

**독자 (Reader)**
```c
wait(mutex);
read_count++;
if (read_count == 1)  // 첫 번째 독자
    wait(rw_mutex);   // 작가 진입 차단
signal(mutex);

/* 읽기 작업 */

wait(mutex);
read_count--;
if (read_count == 0)  // 마지막 독자
    signal(rw_mutex); // 작가 진입 허용
signal(mutex);
```

**독자-작가 락 (Reader-Writer Lock)**
- 읽기 모드(공유): 다수 스레드 동시 획득 가능
- 쓰기 모드(독점): 단일 스레드만 획득
- 독자 수 >> 작가 수인 애플리케이션에 적합
- POSIX: `pthread_rwlock_t`, Java: `ReentrantReadWriteLock`

---

### 1.3 식사하는 철학자 문제 (Dining-Philosophers Problem)

5명의 철학자가 원형 테이블에서 젓가락(chopstick) 공유. 생각(think) → 배고픔(hungry) → 식사(eat) 반복.

```
    철학자 0
  젓가락4  젓가락0
철학자4      철학자1
  젓가락3  젓가락1
    철학자3  철학자2
           젓가락2
```

#### 세마포 해법 — 데드락 위험

```c
semaphore chopstick[5]; // 모두 1로 초기화

// 철학자 i
while (true) {
    wait(chopstick[i]);
    wait(chopstick[(i+1) % 5]);
    /* 식사 */
    signal(chopstick[i]);
    signal(chopstick[(i+1) % 5]);
    /* 생각 */
}
```

**문제**: 모든 철학자가 동시에 왼쪽 젓가락을 집으면 데드락 발생.

**세마포 해법의 데드락 방지 대안**
- 최대 4명만 동시에 젓가락 집도록 허용
- 양쪽 젓가락이 모두 가용할 때만 집도록 (원자적으로)
- 홀수 번호 철학자는 왼쪽→오른쪽, 짝수는 오른쪽→왼쪽 순서

#### 모니터 해법 — 데드락 없음

```c
// 상태 정의
enum { THINKING, HUNGRY, EATING } state[5];
condition self[5];  // 젓가락 대기용 조건 변수

// 철학자 i가 양쪽 이웃이 식사 중이지 않을 때만 식사 가능
void test(int i) {
    if (state[(i+4)%5] != EATING &&
        state[i] == HUNGRY &&
        state[(i+1)%5] != EATING) {
        state[i] = EATING;
        self[i].signal();
    }
}

void pickup(int i) {
    state[i] = HUNGRY;
    test(i);
    if (state[i] != EATING)
        self[i].wait();  // 젓가락 가용할 때까지 대기
}

void putdown(int i) {
    state[i] = THINKING;
    test((i+4)%5);  // 왼쪽 이웃 깨우기
    test((i+1)%5);  // 오른쪽 이웃 깨우기
}
```

**특성**
- 데드락(deadlock) 없음: `state[i] != EATING`인 두 이웃이 확인되어야만 진입
- 기아(starvation) 가능: 두 이웃이 번갈아가며 식사하면 철학자 i는 영원히 대기할 수 있음

---

## 2. 커널 내부 동기화

### 2.1 Windows 동기화

Windows는 **디스패처 객체(dispatcher object)** 기반 동기화 제공.

**디스패처 객체 종류**

| 객체 | 신호(signaled) 상태 |
|------|---------------------|
| 뮤텍스(mutex) | 잠금 해제됨 |
| 세마포(semaphore) | 값 > 0 |
| 이벤트(event) | 알림 발생 |
| 타이머(timer) | 시간 만료 |

- **신호됨(signaled)**: 스레드가 블록되지 않고 객체 획득 가능
- **비신호됨(nonsignaled)**: 스레드가 객체 대기 시 블록됨

**임계 구역 객체 (Critical-Section Object)**
- 유저 모드 뮤텍스 — 커널 개입 없이 먼저 스핀락 시도
- 경쟁 발생 시에만 커널 뮤텍스 할당 (컨텍스트 스위치 비용 절감)
- 단일 프로세스 내 스레드 간 동기화에 최적화

---

### 2.2 Linux 동기화

**원자 정수 (Atomic Integer)**
```c
atomic_t counter;
atomic_set(&counter, 5);    // counter = 5
atomic_add(10, &counter);   // counter += 10
atomic_sub(4, &counter);    // counter -= 4
atomic_inc(&counter);       // counter++
int val = atomic_read(&counter);
```
- 단일 기계 명령으로 수행되므로 인터럽트·선점 없이 원자적

**뮤텍스 락 (Mutex Lock)**
```c
mutex_lock(&lock);
/* 임계 구역 */
mutex_unlock(&lock);
```
- 비재귀적(nonrecursive): 동일 태스크가 두 번 acquire하면 데드락
- 인터럽트 컨텍스트에서는 사용 불가

**스핀락 (Spinlock) vs 선점 비활성화**

| 상황 | 메커니즘 |
|------|----------|
| SMP(멀티코어) | `spin_lock()` / `spin_unlock()` |
| 단일 CPU | `preempt_disable()` / `preempt_enable()` |

- `thread_info` 구조체의 `preempt_count` 필드로 관리
- `preempt_count > 0`이면 선점 금지
- `spin_lock()` 내부에서 자동으로 `preempt_disable()` 호출

---

## 3. POSIX 동기화

### 3.1 POSIX 뮤텍스 락

```c
#include <pthread.h>

pthread_mutex_t mutex;
pthread_mutex_init(&mutex, NULL);

pthread_mutex_lock(&mutex);
/* 임계 구역 */
pthread_mutex_unlock(&mutex);

pthread_mutex_destroy(&mutex);
```

### 3.2 POSIX 세마포

**이름 있는 세마포 (Named Semaphore)** — 프로세스 간 공유 가능
```c
#include <semaphore.h>

sem_t *sem;
// O_CREAT: 없으면 생성, 0666: 권한, 1: 초기값
sem = sem_open("SEM", O_CREAT, 0666, 1);

sem_wait(sem);   // P 연산
/* 임계 구역 */
sem_post(sem);   // V 연산

sem_close(sem);
sem_unlink("SEM");
```

**이름 없는 세마포 (Unnamed Semaphore)** — 스레드 간 공유
```c
sem_t sem;
// 두 번째 인자 0: 스레드 공유 (1이면 프로세스 간)
sem_init(&sem, 0, 1);

sem_wait(&sem);
/* 임계 구역 */
sem_post(&sem);

sem_destroy(&sem);
```

### 3.3 POSIX 조건 변수

```c
pthread_mutex_t mutex;
pthread_cond_t  cond_var;

pthread_mutex_init(&mutex, NULL);
pthread_cond_init(&cond_var, NULL);

// 대기 스레드
pthread_mutex_lock(&mutex);
while (/* 조건 불만족 */)
    pthread_cond_wait(&cond_var, &mutex);
    // mutex 원자적 해제 + 조건 변수 대기
    // 깨어나면 mutex 재획득
/* 조건 만족 상태로 작업 */
pthread_mutex_unlock(&mutex);

// 신호 스레드
pthread_mutex_lock(&mutex);
/* 조건 변경 */
pthread_cond_signal(&cond_var);  // 대기 스레드 하나 깨움
pthread_mutex_unlock(&mutex);
```

> `pthread_cond_wait()`는 **spurious wakeup(허위 깨어남)** 가능성 때문에 반드시 `while` 루프로 조건 재확인해야 한다.

---

## 4. Java 동기화

### 4.1 synchronized 키워드 (모니터)

Java의 모든 객체는 단일 락(lock)을 가진다. `synchronized` 메서드/블록은 해당 객체의 모니터를 획득.

```java
public class BoundedBuffer<E> {
    private E[] buffer;
    private int count = 0, in = 0, out = 0;

    public synchronized void insert(E item) throws InterruptedException {
        while (count == buffer.length)   // 항상 while로 spurious wakeup 대비
            wait();                       // wait set으로 이동 + 락 해제
        buffer[in] = item;
        in = (in + 1) % buffer.length;
        count++;
        notifyAll();                      // wait set의 스레드를 entry set으로 이동
    }

    public synchronized E remove() throws InterruptedException {
        while (count == 0)
            wait();
        E item = buffer[out];
        out = (out + 1) % buffer.length;
        count--;
        notifyAll();
        return item;
    }
}
```

**모니터 내부 구조**
```
┌─────────────────────────────┐
│         Object Monitor      │
│  ┌───────────┐  ┌────────┐ │
│  │ Entry Set │  │Wait Set│ │
│  │ (락 대기) │  │(조건대기│ │
│  └─────┬─────┘  └───┬────┘ │
│        │ 락 획득     │ notify│
│        ▼            ▼      │
│  ┌─────────────────────┐   │
│  │    임계 구역 실행    │   │
│  └─────────────────────┘   │
└─────────────────────────────┘
```

- `wait()`: 락 해제 + wait set 이동
- `notify()`: wait set → entry set (하나)
- `notifyAll()`: wait set → entry set (전부)

**블록 동기화**
```java
// 메서드 전체 대신 필요한 부분만 동기화
synchronized(this) {
    // 임계 구역
}
```

---

### 4.2 ReentrantLock

`synchronized`보다 세밀한 제어. `try/finally` 패턴 필수.

```java
import java.util.concurrent.locks.*;

Lock lock = new ReentrantLock();

lock.lock();
try {
    /* 임계 구역 */
} finally {
    lock.unlock();  // 예외 발생 시에도 반드시 해제
}
```

**ReentrantReadWriteLock**: 독자-작가 락
```java
ReadWriteLock rwLock = new ReentrantReadWriteLock();
Lock readLock  = rwLock.readLock();
Lock writeLock = rwLock.writeLock();

// 읽기 (복수 스레드 동시 가능)
readLock.lock();
try { /* 읽기 */ } finally { readLock.unlock(); }

// 쓰기 (독점)
writeLock.lock();
try { /* 쓰기 */ } finally { writeLock.unlock(); }
```

---

### 4.3 Java 세마포

```java
import java.util.concurrent.*;

Semaphore sem = new Semaphore(1);  // 초기값 1 = 이진 세마포

try {
    sem.acquire();   // wait() — 블로킹
    /* 임계 구역 */
} catch (InterruptedException e) {
    // ...
} finally {
    sem.release();   // signal()
}
```

---

### 4.4 Java 조건 변수 (Condition)

`ReentrantLock` + `Condition` 조합으로 **이름 있는 조건 변수** 지원. 기본 모니터(`wait/notify`)는 조건 변수를 하나만 가지지만, `Condition`은 여러 개 가능.

```java
Lock lock = new ReentrantLock();
Condition notFull  = lock.newCondition();  // 버퍼 여유 있음
Condition notEmpty = lock.newCondition();  // 아이템 있음

// 생산자
lock.lock();
try {
    while (count == BUFFER_SIZE)
        notFull.await();    // 락 해제 + 대기
    /* 삽입 */
    notEmpty.signal();      // 소비자 깨움
} finally {
    lock.unlock();
}

// 소비자
lock.lock();
try {
    while (count == 0)
        notEmpty.await();
    /* 제거 */
    notFull.signal();       // 생산자 깨움
} finally {
    lock.unlock();
}
```

**synchronized vs ReentrantLock 비교**

| 특성 | synchronized | ReentrantLock |
|------|-------------|---------------|
| 조건 변수 | 1개 (wait/notify) | 복수 가능 (Condition) |
| 공정성(fairness) | 보장 없음 | `new ReentrantLock(true)` |
| 시도 획득 | 불가 | `tryLock()` |
| 인터럽트 | 불가 | `lockInterruptibly()` |
| 코드 복잡성 | 낮음 | 중간 |

---

## 5. 대안적 접근법

### 5.1 트랜잭셔널 메모리 (Transactional Memory)

데이터베이스 트랜잭션 개념을 메모리 연산에 적용. 락 없이 동기화 달성.

```c
// atomic 블록: 전부 성공하거나 전부 롤백
void update() {
    atomic {
        /* 일련의 메모리 연산 */
        /* 충돌 감지 시 자동 재시도 */
    }
}
```

**소프트웨어 트랜잭셔널 메모리 (STM, Software Transactional Memory)**
- 컴파일러가 트랜잭션 읽기/쓰기에 계측 코드(instrumentation) 삽입
- 충돌(conflict) 감지 시 롤백(rollback) 후 재시도
- 오버헤드 존재하지만 데드락 불가

**하드웨어 트랜잭셔널 메모리 (HTM, Hardware Transactional Memory)**
- CPU 캐시 일관성(cache coherency) 메커니즘을 활용
- Intel TSX, IBM POWER 등에서 지원
- 트랜잭션 중 충돌·캐시 미스 발생 시 하드웨어가 롤백
- STM보다 오버헤드 낮음

**트랜잭셔널 메모리 장점**
- 데드락 불가능
- 동시 읽기 자동 허용 (불필요한 직렬화 없음)
- 락 순서 고려 불필요

---

### 5.2 OpenMP의 임계 구역

```c
#pragma omp parallel for
for (int i = 0; i < N; i++) {
    #pragma omp critical
    {
        /* 임계 구역 — 이진 세마포처럼 동작 */
        shared_counter++;
    }
}

// 이름 있는 임계 구역 (여러 독립적 임계 구역 지원)
#pragma omp critical(section_A)
{ /* ... */ }

#pragma omp critical(section_B)
{ /* ... */ }
```

- `#pragma omp critical` 없이 `#pragma omp atomic`으로 단일 연산 원자화 가능
- OpenMP 런타임이 락 관리 — 개발자가 직접 뮤텍스 작성 불필요

---

### 5.3 함수형 언어 (Functional Languages)

**Erlang, Scala** 같은 함수형 언어는 **불변 상태(immutable state)** 를 기본으로 하여 경쟁 조건(race condition)과 임계 구역(critical section) 개념 자체를 제거.

```erlang
% Erlang: 메시지 패싱만으로 동기화
% 공유 가변 상태 없음 — 경쟁 조건 원천 차단
Counter ! {increment, 1}
```

```scala
// Scala: 불변 데이터 구조 기본
val list = List(1, 2, 3)  // 불변
val newList = 0 :: list   // 새 리스트 생성, 기존 불변
```

**핵심 원칙**: 변수를 공유하지 않으면 잠글 필요도 없다. 함수형 패러다임은 공유 메모리 대신 메시지 패싱(message passing)과 액터 모델(actor model)을 통해 동시성을 다룬다.

---

## 동기화 도구 선택 가이드

| 상황 | 권장 도구 |
|------|----------|
| 단순 정수 카운터 | 원자 변수 (`atomic_t`, `AtomicInteger`) |
| 짧은 임계 구역, SMP | 스핀락 |
| 일반 상호 배제 | 뮤텍스 락 |
| 자원 풀 개수 제어 | 세마포 |
| 복합 조건 대기 | 모니터 + 조건 변수 |
| 독자 >> 작가 | 독자-작가 락 |
| 데드락 회피 최우선 | 트랜잭셔널 메모리 |
| OpenMP 병렬 루프 | `#pragma omp critical` |
| 공유 상태 최소화 | 함수형 언어 / 액터 모델 |

---

## 핵심 정리

- **Bounded-Buffer**: `mutex(1) + empty(n) + full(0)` 세마포 3개 조합
- **Readers-Writers**: `rw_mutex`를 첫/마지막 독자만 제어 → 다수 독자 동시 접근 허용
- **Dining Philosophers**: 세마포 해법은 데드락 가능, 모니터 해법은 데드락 없지만 기아 가능
- **Windows**: 디스패처 객체(신호/비신호 상태) + 임계 구역 객체(유저 모드 최적화)
- **Linux**: `atomic_t` + `mutex_lock` + `preempt_count` 기반 스핀락
- **POSIX**: `pthread_mutex_t` / `sem_open(named)` / `sem_init(unnamed)` / `pthread_cond_t`
- **Java**: `synchronized`(단일 조건) vs `ReentrantLock+Condition`(복수 조건) 선택
- **트랜잭셔널 메모리**: 데드락 불가, 충돌 시 자동 롤백 (STM/HTM)
- **함수형 언어**: 불변 상태로 경쟁 조건 원천 제거
