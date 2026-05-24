# Chapter 4: Threads & Concurrency

> 멀티스레딩의 개념, 멀티코어 프로그래밍 과제, 스레딩 모델, 스레드 라이브러리(Pthreads/Windows/Java), 암묵적 스레딩(Thread Pool/Fork-Join/OpenMP/GCD/TBB), 스레딩 이슈, OS별 구현까지 다룬다.

---

## 4.1 개요 (Overview)

### 스레드(Thread)의 구성 요소

스레드는 CPU 활용의 기본 단위로 다음 4가지 요소로 구성된다:

```
Thread = { Thread ID, Program Counter, Register Set, Stack }
```

같은 프로세스에 속한 스레드들은 **코드 섹션, 데이터 섹션, 열린 파일, 시그널** 등 프로세스 자원을 공유한다.

```
┌─────────────────────────────────────────────────────┐
│              단일 스레드 프로세스                        │
│  ┌────────┐  ┌────────┐  ┌────────┐  ┌────────┐   │
│  │  Code  │  │  Data  │  │ Files  │  │ Stack  │   │
│  └────────┘  └────────┘  └────────┘  └────────┘   │
└─────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────┐
│              멀티스레드 프로세스                         │
│  ┌────────┐  ┌────────┐  ┌────────┐                │
│  │  Code  │  │  Data  │  │ Files  │  ← 공유         │
│  └────────┘  └────────┘  └────────┘                │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐            │
│  │Thread 1  │ │Thread 2  │ │Thread 3  │            │
│  │PC/Reg/   │ │PC/Reg/   │ │PC/Reg/   │            │
│  │Stack     │ │Stack     │ │Stack     │            │
│  └──────────┘ └──────────┘ └──────────┘            │
└─────────────────────────────────────────────────────┘
```

### 4.1.1 동기 (Motivation)

현대 애플리케이션은 대부분 멀티스레드로 구현된다:
- 웹 서버: 요청마다 별도 스레드로 서비스 (프로세스 fork 대비 훨씬 저비용)
- 이미지 처리: 각 이미지마다 썸네일 생성 스레드
- 워드 프로세서: UI 렌더링 / 키입력 처리 / 맞춤법 검사 스레드 분리
- Linux 커널: `kthreadd(PID=2)` 가 모든 커널 스레드의 부모

### 4.1.2 멀티스레딩의 이점 (Benefits)

| 이점 | 설명 |
|------|------|
| **응답성(Responsiveness)** | 일부 스레드가 블로킹되어도 다른 스레드가 계속 실행 |
| **자원 공유(Resource Sharing)** | 프로세스 내 메모리·자원을 기본 공유 (shared memory/msg passing 불필요) |
| **경제성(Economy)** | 스레드 생성·컨텍스트 스위치 비용 << 프로세스 생성·스위치 비용 |
| **확장성(Scalability)** | 멀티코어 시스템에서 병렬 실행으로 처리량 향상 |

---

## 4.2 멀티코어 프로그래밍 (Multicore Programming)

### 동시성(Concurrency) vs. 병렬성(Parallelism)

```
단일 코어 - 동시성(Concurrency): 시분할로 진행
─────────────────────────────────────────────────▶ time
 T1  T2  T2  T3  T3  T4  T4  T1  T1  ...

멀티코어 - 병렬성(Parallelism): 물리적 동시 실행
core 0: T1  T1  T1  T3  T3  ...
core 1: T2  T2  T4  T4  ...
```

- **동시성**: 여러 태스크가 "진전(progress)"을 만드는 상태 → 단일 코어에서도 가능
- **병렬성**: 여러 태스크가 "동시에(simultaneously)" 실행되는 상태 → 멀티코어 필수

### 4.2.1 프로그래밍 도전 과제 (Programming Challenges)

멀티코어 시스템을 위한 5가지 주요 과제:

1. **태스크 식별(Identifying tasks)**: 독립적으로 분리 가능한 병렬 태스크 발굴
2. **균형(Balance)**: 태스크 간 균등한 작업량 분배 (가치가 낮은 태스크에 코어 낭비 금지)
3. **데이터 분할(Data splitting)**: 태스크가 접근하는 데이터도 코어별로 분할
4. **데이터 의존성(Data dependency)**: 태스크 간 데이터 의존관계 분석 → 실행 순서 동기화
5. **테스트·디버깅(Testing and debugging)**: 병렬 실행 경로가 복수 → 재현 어려움

### Amdahl의 법칙 (Amdahl's Law)

직렬(serial) 부분 S가 존재할 때, N개 코어 추가로 얻을 수 있는 최대 속도 향상:

```
           1
speedup ≤ ─────────────
          S + (1−S)/N
```

- S=0.25(직렬 25%)이면, N=∞일 때 최대 **4배** 속도 향상
- S=0.50(직렬 50%)이면, N=∞일 때 최대 **2배** 속도 향상
- **핵심**: 직렬 부분이 성능 향상의 근본적 병목

### 4.2.2 병렬성의 유형 (Types of Parallelism)

```
데이터 병렬성(Data Parallelism)          태스크 병렬성(Task Parallelism)
─────────────────────────────────        ──────────────────────────────────
  같은 연산, 다른 데이터 서브셋              다른 연산, 같은/다른 데이터

  core 0: array[0..N/2-1] sum            core 0: statistical op A on array
  core 1: array[N/2..N-1] sum            core 1: statistical op B on array
```

두 방식은 상호 배타적이지 않으며 하이브리드로 조합 가능.

---

## 4.3 멀티스레딩 모델 (Multithreading Models)

스레드 지원은 **사용자 수준(user threads)** 또는 **커널 수준(kernel threads)**으로 제공된다. 최종적으로 두 수준 간 매핑이 필요하며 세 가지 모델이 존재한다.

### 4.3.1 Many-to-One 모델

```
user space  [T] [T] [T] [T]  ← 여러 사용자 스레드
                  │
kernel space     [K]          ← 단 하나의 커널 스레드
```

- 스레드 관리가 user space 라이브러리에서 이뤄져 효율적
- **단점**: 하나라도 blocking 시스템 콜 → 전체 프로세스 블록
- **단점**: 멀티코어에서 진정한 병렬 실행 불가 (커널 스레드 1개만 실행)
- 사례: Solaris Green Threads, 초기 Java → 현재는 거의 미사용

### 4.3.2 One-to-One 모델

```
user space  [T1] [T2] [T3] [T4]
             │    │    │    │
kernel space [K1] [K2] [K3] [K4]
```

- 더 높은 동시성: 한 스레드 블로킹해도 다른 스레드 계속 실행
- 멀티프로세서에서 병렬 실행 가능
- **단점**: 사용자 스레드 생성 시 커널 스레드도 생성 → 대량 스레드 시 오버헤드
- 사례: **Linux, Windows** (현재 가장 보편적)

### 4.3.3 Many-to-Many 모델

```
user space  [T1][T2][T3][T4][T5][T6]  ← 사용자 스레드 (많음)
                  │  │  │  │
kernel space    [K1][K2][K3][K4]       ← 커널 스레드 (≤ 사용자 스레드)
```

- 사용자 스레드 수에 제한 없음
- blocking 시 커널이 다른 스레드 스케줄 가능
- **단점**: 구현 복잡도가 높음
- **Two-level model**: M:M + 특정 사용자 스레드를 커널 스레드에 고정(bind) 가능
- 실제로는 코어 수 증가로 커널 스레드 제한이 덜 중요해져 One-to-One이 대세

| 모델 | 병렬성 | blocking 문제 | 구현 복잡도 | 사용 OS |
|------|--------|--------------|------------|---------|
| Many-to-One | ✗ | 전체 블록 | 낮음 | 구식 시스템 |
| One-to-One | ✓ | 개별 블록 | 중간 | Linux, Windows |
| Many-to-Many | ✓ | 개별 블록 | 높음 | 일부 UNIX |

---

## 4.4 스레드 라이브러리 (Thread Libraries)

스레드 라이브러리 구현 방식 두 가지:
1. **완전 user-space 라이브러리**: 커널 지원 없음 → 함수 호출이 시스템 콜 아님
2. **커널 수준 라이브러리**: OS가 직접 지원 → 함수 호출이 시스템 콜

### 비동기(Asynchronous) vs. 동기(Synchronous) 스레딩

- **비동기 스레딩**: 부모가 자식 스레드 생성 후 즉시 독립 실행 → 데이터 공유 거의 없음
- **동기 스레딩**: 부모가 자식 완료를 기다린 후(join) 재개 → 결과 취합 패턴

### 4.4.1 Pthreads

POSIX 표준(IEEE 1003.1c)으로 **명세(specification)**이지 구현이 아니다. Linux/macOS에서 지원.

```c
#include <pthread.h>
#include <stdio.h>
#include <stdlib.h>

int sum;  /* 스레드 간 공유 전역 변수 */

void *runner(void *param) {
    int i, upper = atoi(param);
    sum = 0;
    for (i = 1; i <= upper; i++)
        sum += i;
    pthread_exit(0);
}

int main(int argc, char *argv[]) {
    pthread_t tid;
    pthread_attr_t attr;

    pthread_attr_init(&attr);            /* 기본 속성 초기화 */
    pthread_create(&tid, &attr, runner, argv[1]);  /* 스레드 생성 */
    pthread_join(tid, NULL);             /* 스레드 종료 대기 */
    printf("sum = %d\n", sum);
}
```

여러 스레드 join 패턴:
```c
pthread_t workers[NUM_THREADS];
for (int i = 0; i < NUM_THREADS; i++)
    pthread_join(workers[i], NULL);
```

### 4.4.2 Windows Threads

```c
#include <windows.h>

DWORD Sum;  /* 공유 전역 변수 */

DWORD WINAPI Summation(LPVOID Param) {
    DWORD Upper = *(DWORD*)Param;
    for (DWORD i = 1; i <= Upper; i++)
        Sum += i;
    return 0;
}

int main(int argc, char *argv[]) {
    DWORD ThreadId;
    HANDLE ThreadHandle;
    int Param = atoi(argv[1]);

    ThreadHandle = CreateThread(
        NULL,        /* 기본 보안 속성 */
        0,           /* 기본 스택 크기 */
        Summation,   /* 스레드 함수 */
        &Param,      /* 파라미터 */
        0,           /* 기본 생성 플래그 */
        &ThreadId);  /* 스레드 ID 반환 */

    WaitForSingleObject(ThreadHandle, INFINITE);  /* pthread_join 상당 */
    CloseHandle(ThreadHandle);
    printf("sum = %d\n", Sum);
}
```

복수 스레드 대기:
```c
WaitForMultipleObjects(N, THandles, TRUE, INFINITE);
```

### 4.4.3 Java Threads

Java는 `Runnable` 인터페이스 구현이 권장 패턴이다:

```java
class Task implements Runnable {
    public void run() {
        System.out.println("I am a thread.");
    }
}

Thread worker = new Thread(new Task());
worker.start();  /* run()을 직접 호출하지 않는다 */
worker.join();   /* pthread_join 상당 */
```

Lambda 표현식(Java 8+)으로 간결하게:
```java
Runnable task = () -> { System.out.println("I am a thread."); };
Thread worker = new Thread(task);
worker.start();
```

#### Java Executor Framework

`java.util.concurrent` 패키지의 `Executor` 인터페이스는 스레드 생성과 실행을 분리한다. `Callable`/`Future`로 반환값도 처리:

```java
import java.util.concurrent.*;

class Summation implements Callable<Integer> {
    private int upper;
    public Summation(int upper) { this.upper = upper; }

    public Integer call() {
        int sum = 0;
        for (int i = 1; i <= upper; i++) sum += i;
        return sum;
    }
}

public class Driver {
    public static void main(String[] args) {
        int upper = Integer.parseInt(args[0]);
        ExecutorService pool = Executors.newSingleThreadExecutor();
        Future<Integer> result = pool.submit(new Summation(upper));
        try {
            System.out.println("sum = " + result.get());
        } catch (InterruptedException | ExecutionException ie) { }
    }
}
```

| Pthreads | Windows | Java |
|----------|---------|------|
| `pthread_create()` | `CreateThread()` | `new Thread(); .start()` |
| `pthread_join()` | `WaitForSingleObject()` | `.join()` |
| 전역 변수로 공유 | 전역 변수로 공유 | 명시적 공유 배열 필요 |
| POSIX 표준 | Windows 전용 | JVM 위 어느 OS든 |

---

## 4.5 암묵적 스레딩 (Implicit Threading)

수천 개의 스레드를 직접 관리하는 것은 복잡하다. **암묵적 스레딩**은 스레드 생성·관리를 컴파일러나 런타임 라이브러리에 위임하고, 개발자는 **태스크(task)**만 식별하는 전략이다.

### 4.5.1 스레드 풀 (Thread Pool)

**문제**: 요청마다 스레드를 생성/소멸하면 생성 오버헤드 + 무제한 스레드로 자원 고갈 위험.

**해법**: 시작 시 일정 수의 스레드를 미리 생성해 풀(pool)에 대기시킨다.

```
요청 도착
    │
    ▼
풀에 유휴 스레드 있음? ─Yes─▶ 스레드 깨워 즉시 처리
    │
    No
    ▼
큐에 요청 저장 → 스레드 완료 시 큐에서 꺼내 처리
```

**이점**:
1. 기존 스레드 재사용 → 스레드 생성 대기 시간 제거
2. 동시 스레드 수 상한 설정 가능 → 자원 보호
3. 태스크 실행 전략을 분리 (지연 실행, 주기적 실행 등)

**Windows 스레드 풀 API**:
```c
DWORD WINAPI PoolFunction(PVOID Param) { /* 별도 스레드로 실행 */ }

// 스레드 풀에 작업 제출
QueueUserWorkItem(&PoolFunction, NULL, 0);
```

**Java 스레드 풀**:
```java
// 3가지 팩토리 메서드
ExecutorService pool1 = Executors.newSingleThreadExecutor();   // 크기 1
ExecutorService pool2 = Executors.newFixedThreadPool(4);       // 고정 크기
ExecutorService pool3 = Executors.newCachedThreadPool();       // 무제한(재사용)

ExecutorService pool = Executors.newCachedThreadPool();
for (int i = 0; i < numTasks; i++)
    pool.execute(new Task());
pool.shutdown();
```

### 4.5.2 Fork-Join

분할 정복(divide-and-conquer) 알고리즘에 적합한 동기적 병렬화 패턴.

```
           main thread
          /     fork    \
    subtask1           subtask2
      fork/join          fork/join
    /      \           /      \
 task      task     task      task
          ↑   join   ↑
           main thread (combine results)
```

**Java Fork-Join 프레임워크**:
```java
import java.util.concurrent.*;

public class SumTask extends RecursiveTask<Integer> {
    static final int THRESHOLD = 1000;
    private int begin, end;
    private int[] array;

    public SumTask(int begin, int end, int[] array) {
        this.begin = begin; this.end = end; this.array = array;
    }

    protected Integer compute() {
        if (end - begin < THRESHOLD) {
            // 직접 계산
            int sum = 0;
            for (int i = begin; i <= end; i++) sum += array[i];
            return sum;
        } else {
            int mid = (begin + end) / 2;
            SumTask left  = new SumTask(begin, mid, array);
            SumTask right = new SumTask(mid + 1, end, array);
            left.fork();
            right.fork();
            return right.join() + left.join();
        }
    }
}

// 사용
ForkJoinPool pool = new ForkJoinPool();
int sum = pool.invoke(new SumTask(0, SIZE - 1, array));
```

`ForkJoinPool`의 **work stealing** 알고리즘: 유휴 스레드가 바쁜 스레드의 큐에서 태스크를 훔쳐 부하를 균등 분산.

```
ForkJoinTask<V>  (추상)
   ├─ RecursiveTask<V>   → compute()가 값을 반환
   └─ RecursiveAction    → compute()가 void (반환값 없음)
```

### 4.5.3 OpenMP

C/C++/FORTRAN을 위한 컴파일러 지시어(directive) + API. 공유 메모리 환경에서 병렬 영역을 pragma로 표시한다.

```c
#include <omp.h>
#include <stdio.h>

int main() {
    #pragma omp parallel
    {
        /* 코어 수만큼 스레드가 이 블록을 동시에 실행 */
        printf("I am a parallel region.\n");
    }
    return 0;
}
```

루프 병렬화:
```c
#pragma omp parallel for
for (int i = 0; i < N; i++) {
    c[i] = a[i] + b[i];  /* N회 반복을 코어 수로 분할하여 병렬 실행 */
}
```

- 스레드 수를 수동 설정하거나 데이터의 공유/private 여부를 지정 가능
- Linux/Windows/macOS 지원

### 4.5.4 Grand Central Dispatch (GCD)

Apple의 macOS/iOS용 런타임 라이브러리 + API + 언어 확장. **dispatch queue**에 태스크를 등록하면 내부 스레드 풀(POSIX 스레드 기반)이 실행한다.

**두 종류의 큐**:

| 큐 종류 | 특징 |
|---------|------|
| **직렬(Serial) 큐** | FIFO, 태스크 하나씩 순서대로 실행 (main queue + 프로세스 별 private queue) |
| **병렬(Concurrent) 큐** | FIFO로 꺼내지만 복수 태스크 동시 실행 (system-wide global queue) |

**4가지 QoS 클래스**:

| QoS 클래스 | 용도 |
|-----------|------|
| `QOS_CLASS_USER_INTERACTIVE` | UI/이벤트 처리 (즉각 응답 필수) |
| `QOS_CLASS_USER_INITIATED` | 파일/URL 열기 (사용자 대기 가능) |
| `QOS_CLASS_UTILITY` | 데이터 임포트 등 긴 작업 |
| `QOS_CLASS_BACKGROUND` | 인덱싱, 백업 등 비가시적 작업 |

```swift
// Swift 클로저로 태스크 제출
let queue = DispatchQueue.global(qos: .userInitiated)
queue.async { print("I am a closure.") }
```

C/Objective-C는 블록(block, `^{ }`)으로 표현. `libdispatch`로 구현되며 Apache 라이선스로 공개 (FreeBSD 포팅 존재).

### 4.5.5 Intel Thread Building Blocks (TBB)

C++ 템플릿 라이브러리로 특별한 컴파일러/언어 지원 없이 사용 가능. TBB 태스크 스케줄러가 태스크→스레드 매핑, 부하 균형, 캐시 인식(cache-aware) 스케줄링을 담당.

```cpp
// 직렬 코드
for (int i = 0; i < n; i++) {
    apply(v[i]);
}

// TBB 병렬 for - 이터레이션 공간을 자동으로 분할하여 병렬 실행
parallel_for(size_t(0), n, [=](size_t i) { apply(v[i]); });
```

- 첫 번째, 두 번째 파라미터: 이터레이션 공간 `[0, n)`
- 세 번째 파라미터: C++ 람다 함수 (각 인덱스에 실행할 연산)
- 내부적으로 청크(chunk)로 분할 후 태스크 생성 → Java fork-join과 유사

---

## 4.6 스레딩 이슈 (Threading Issues)

### 4.6.1 fork()와 exec() 시스템 콜

멀티스레드 프로그램에서 `fork()` 의미가 달라진다:

| 상황 | 권장 동작 |
|------|----------|
| `fork()` 후 즉시 `exec()` 호출 | 호출 스레드만 복제 (모든 스레드 복제 불필요, exec이 덮어씀) |
| `fork()` 후 `exec()` 호출 안 함 | 모든 스레드 복제 (완전한 프로세스 복사) |

일부 UNIX는 두 버전의 `fork()`를 모두 제공한다.

### 4.6.2 시그널 처리 (Signal Handling)

UNIX 시그널(signal) 전달 패턴: **발생(generate) → 전달(deliver) → 처리(handle)**

- **동기 시그널**: illegal memory access, division by 0 → 해당 스레드에만 전달
- **비동기 시그널**: `Ctrl+C`, 타이머 만료 → 어느 스레드에 전달할지가 문제

멀티스레드에서의 시그널 전달 옵션:
1. 시그널이 적용되는 스레드에만 전달
2. 프로세스의 모든 스레드에 전달
3. 특정 스레드에만 전달
4. 시그널 수신 전담 스레드 지정

```c
/* POSIX: 특정 스레드에 시그널 전달 */
pthread_kill(pthread_t tid, int signal);

/* Windows: APC(Asynchronous Procedure Call)로 에뮬레이션 */
```

### 4.6.3 스레드 취소 (Thread Cancellation)

대상 스레드(target thread) 강제 종료 방법:

| 방식 | 특징 | 위험성 |
|------|------|--------|
| **비동기 취소(Asynchronous)** | 즉시 종료 | 자원 미해제 가능 → 권장하지 않음 |
| **지연 취소(Deferred)** | 취소 지점(cancellation point) 도달 시 종료 | 안전하게 정리 가능 |

**Pthreads 취소**:
```c
pthread_t tid;
pthread_create(&tid, 0, worker, NULL);

pthread_cancel(tid);       /* 취소 요청 (즉시 종료 아님) */
pthread_join(tid, NULL);   /* 종료 대기 */

/* 스레드 내부에서 취소 지점 확인 */
while (1) {
    /* 작업 수행 */
    pthread_testcancel();  /* 취소 요청 있으면 여기서 종료 */
}
```

Pthreads 취소 모드:

| 모드 | State | Type |
|------|-------|------|
| Off | Disabled | - |
| Deferred (기본) | Enabled | Deferred |
| Asynchronous | Enabled | Asynchronous |

**Java 취소** (지연 취소와 유사):
```java
worker.interrupt();  /* 인터럽트 상태를 true로 설정 */

/* 스레드 내부 */
while (!Thread.currentThread().isInterrupted()) {
    /* 작업 수행 */
}
```

### 4.6.4 스레드 로컬 스토리지 (Thread-Local Storage, TLS)

같은 프로세스 스레드들이 데이터를 공유하지만, 스레드마다 고유한 데이터가 필요한 경우.

- 로컬 변수와의 차이: TLS는 함수 호출 경계를 넘어 유지됨
- 암묵적 스레딩(스레드 풀 등)처럼 스레드 생성을 직접 제어하지 못할 때 특히 유용

| 언어/프레임워크 | TLS 선언 방식 |
|---------------|--------------|
| Java | `ThreadLocal<T>` 클래스 (`get()`/`set()`) |
| Pthreads | `pthread_key_t` 타입 |
| C# | `[ThreadStatic]` 속성 |
| GCC | `__thread` 키워드 |

```c
/* GCC TLS 예시 */
static __thread int threadID;  /* 각 스레드마다 별도 인스턴스 */
```

### 4.6.5 스케줄러 활성화 (Scheduler Activations)

Many-to-Many 및 Two-level 모델에서 커널과 스레드 라이브러리 간 통신 메커니즘.

**LWP(Lightweight Process, 경량 프로세스)**:

```
user thread library
     [T] [T] [T]
      │   │   │
     LWP LWP LWP    ← 가상 프로세서처럼 보임
      │   │   │
     [K] [K] [K]    ← 실제 커널 스레드
```

**스케줄러 활성화(Scheduler Activation)** 동작:
1. 커널이 LWP 집합(가상 프로세서)을 애플리케이션에 제공
2. 스레드가 블로킹되려 할 때 커널이 **upcall**로 라이브러리에 통보
3. 커널이 새 가상 프로세서 할당 → 라이브러리가 다른 스레드 스케줄
4. 블로킹 이벤트 완료 시 재차 upcall → 라이브러리가 블록 해제 스레드를 runnable로 표시

---

## 4.7 OS별 구현 사례 (Operating-System Examples)

### 4.7.1 Windows 스레드

Windows는 **One-to-One 모델** 사용. 각 사용자 스레드가 커널 스레드 하나에 대응.

```
                  user space
┌─────────────────────────────────────────┐
│  TEB (Thread Environment Block)          │
│  thread ID / user stack / TLS array      │
└─────────────────────────────────────────┘
                  kernel space
┌─────────────────────────────────────────┐
│  ETHREAD (Executive Thread Block)        │
│  parent process ptr / thread start addr  │
│  → KTHREAD ptr                           │
├─────────────────────────────────────────┤
│  KTHREAD (Kernel Thread Block)           │
│  scheduling & sync info / kernel stack   │
│  → TEB ptr                               │
└─────────────────────────────────────────┘
```

- **ETHREAD**: 부모 프로세스 포인터, 스레드 시작 루틴 주소 → KTHREAD 포인터
- **KTHREAD**: 스케줄링·동기화 정보, 커널 스택 → TEB 포인터
- **TEB**: 스레드 ID, 사용자 모드 스택, TLS 배열 (user space에 위치)
- ETHREAD + KTHREAD는 커널 공간에만 존재 → 커널만 접근 가능

### 4.7.2 Linux 스레드

Linux는 프로세스와 스레드를 구분하지 않고 **태스크(task)**라는 단일 개념을 사용.

```c
/* 스레드 생성: clone()에 공유 플래그 전달 */
clone(CLONE_FS | CLONE_VM | CLONE_SIGHAND | CLONE_FILES, ...);
/*
  CLONE_FS      : 파일 시스템 정보 공유
  CLONE_VM      : 메모리 공간 공유
  CLONE_SIGHAND : 시그널 핸들러 공유
  CLONE_FILES   : 열린 파일 집합 공유
*/

/* fork()는 플래그 없이 clone()을 호출하는 것과 동일 */
```

- `task_struct`가 데이터를 직접 저장하지 않고 **포인터**로 공유 자원을 참조
- `fork()`: 부모의 모든 자료구조를 복사 → 독립 프로세스
- `clone(공유 플래그)`: 자료구조를 공유(포인터 복사) → 스레드
- `clone(컨테이너 플래그)`: Linux 컨테이너 생성 가능 (Ch.18 참조)

---

## 4.8 핵심 정리

| 개념 | 요약 |
|------|------|
| 스레드 구성 | Thread ID + PC + Register Set + Stack |
| 멀티스레딩 4대 이점 | 응답성, 자원 공유, 경제성, 확장성 |
| Concurrency vs Parallelism | 동시성=진전, 병렬성=동시 실행 (멀티코어 필요) |
| Amdahl's Law | 직렬 부분 S → 최대 1/S 배 속도 향상 |
| 스레딩 모델 | Many-to-One / One-to-One / Many-to-Many |
| 주요 라이브러리 | Pthreads(POSIX), Windows API, Java(JVM) |
| 암묵적 스레딩 | Thread Pool / Fork-Join / OpenMP / GCD / TBB |
| 스레딩 이슈 | fork/exec 의미, 시그널 처리, 취소(비동기/지연), TLS, 스케줄러 활성화 |
| Linux 특징 | 프로세스/스레드 구분 없음, `clone()` + 플래그로 공유 수준 결정 |
| Windows 특징 | One-to-One, ETHREAD/KTHREAD/TEB 3계층 구조 |
