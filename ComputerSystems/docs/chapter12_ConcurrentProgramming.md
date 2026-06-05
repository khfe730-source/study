# Chapter 12: Concurrent Programming

## 핵심 주제

동시성(concurrency)이 필요한 이유:
- **느린 I/O 장치 처리**: I/O 대기 중 CPU 활용
- **다수 클라이언트 서비스**: 반복적 서버(iterative server)는 느린 클라이언트 하나가 전체 블록
- **멀티코어 활용**: 병렬 실행으로 성능 향상
- **인간과의 상호작용**: 멀티태스킹 UI

동시성 프로그램 구현 **세 가지 방식:**

| 방식 | 스케줄링 | 주소 공간 | 통신 |
|------|---------|---------|------|
| 프로세스 | 커널 | 독립 (사설) | IPC (명시적) |
| I/O 다중화 | 애플리케이션 | 공유 (단일 프로세스) | 전역 변수 |
| 스레드 | 커널 | 공유 (단일 프로세스) | 공유 변수 |

---

## 12.1 Concurrent Programming with Processes

가장 단순한 방법: 클라이언트마다 `fork()`로 자식 프로세스 생성.

```c
// 프로세스 기반 동시 서버 핵심 구조
void sigchld_handler(int sig) {
    while (waitpid(-1, 0, WNOHANG) > 0) ;  // 좀비 자식 회수
}

Signal(SIGCHLD, sigchld_handler);
listenfd = open_listenfd(argv[1]);
while (1) {
    connfd = Accept(listenfd, (SA *)&clientaddr, &clientlen);
    if (Fork() == 0) {
        Close(listenfd);  // 자식: 청취 디스크립터 닫기
        echo(connfd);     // 클라이언트 서비스
        Close(connfd);    // 자식: 연결 디스크립터 닫기
        exit(0);
    }
    Close(connfd);  // 부모: 연결 디스크립터 반드시 닫기! (메모리 누수 방지)
}
```

**핵심 주의사항:**
- `SIGCHLD` 핸들러에서 `while(waitpid(-1,0,WNOHANG)>0)` — 여러 좀비를 한 번에 회수 (시그널 큐 없음)
- 부모가 `connfd`를 닫지 않으면 → 파일 테이블 항목의 refcnt가 0이 되지 않아 연결 미종료 + 메모리 누수

**디스크립터 참조 카운트 흐름:**
```
fork 직후:
  부모 & 자식 모두 listenfd + connfd 보유 (refcnt=2)
자식: Close(listenfd), 서비스 후 Close(connfd)
부모: Close(connfd) → refcnt=1 → 자식이 종료하면 refcnt=0 → 파일 테이블 항목 해제
```

**장/단점:**
- 장점: 프로세스 주소 공간 독립 → 다른 프로세스 메모리 실수로 덮어쓰기 불가
- 단점: 프로세스 간 데이터 공유 어려움 (명시적 IPC 필요), 컨텍스트 스위치 오버헤드 큼

---

## 12.2 Concurrent Programming with I/O Multiplexing

`select()` 시스템 콜로 여러 디스크립터 중 준비된 것을 한 번에 감지.

### select() API

```c
#include <sys/select.h>

int select(int n, fd_set *fdset, NULL, NULL, NULL);
// n: read set의 최대 디스크립터 + 1
// fdset: 읽기 준비 감시할 디스크립터 집합 (입력&출력, side effect로 ready set 반환)
// 반환: 준비된 디스크립터 수 / -1 오류

/* 디스크립터 집합 조작 매크로 */
FD_ZERO(fd_set *fdset);              // 집합 초기화 (비움)
FD_SET(int fd, fd_set *fdset);       // fd 추가
FD_CLR(int fd, fd_set *fdset);       // fd 제거
FD_ISSET(int fd, fd_set *fdset);     // fd 멤버 여부 확인
```

**중요:** select는 fdset을 **side effect**로 수정 → 매 호출 전 read_set을 ready_set에 복사해야 함.

### 단순 I/O 다중화 서버 (select 사용)

```c
FD_ZERO(&read_set);
FD_SET(STDIN_FILENO, &read_set);  // 표준 입력
FD_SET(listenfd, &read_set);      // 청취 디스크립터

while (1) {
    ready_set = read_set;  // select가 수정하므로 매번 복사
    Select(listenfd+1, &ready_set, NULL, NULL, NULL);

    if (FD_ISSET(STDIN_FILENO, &ready_set))
        command();   // 표준 입력 명령 처리

    if (FD_ISSET(listenfd, &ready_set)) {
        connfd = Accept(listenfd, ...);
        echo(connfd);  // 클라이언트 서비스 (EOF까지 블록 - 한계)
        Close(connfd);
    }
}
```

### 이벤트 주도 서버 (Event-Driven Server)

각 클라이언트를 **상태 기계(state machine)** 로 모델링:

```
상태: "descriptor d_k 가 읽기 준비 대기 중"
이벤트: "descriptor d_k 가 읽기 준비됨"
전환: "d_k에서 텍스트 줄 읽기"
```

```c
typedef struct {
    int maxfd;
    fd_set read_set;      // 모든 활성 디스크립터
    fd_set ready_set;     // select가 반환하는 준비된 서브셋
    int nready;
    int maxi;             // clientfd 배열의 high watermark 인덱스
    int clientfd[FD_SETSIZE];
    rio_t clientrio[FD_SETSIZE];
} pool;

// 메인 루프
while (1) {
    pool.ready_set = pool.read_set;
    pool.nready = Select(pool.maxfd+1, &pool.ready_set, NULL, NULL, NULL);

    // 새 연결 요청?
    if (FD_ISSET(listenfd, &pool.ready_set))
        add_client(Accept(listenfd, ...), &pool);

    // 준비된 클라이언트 서비스 (텍스트 줄 하나씩)
    check_clients(&pool);
}
```

**장/단점:**
- 장점: 프로그래머가 흐름 제어, 단일 프로세스로 공유 데이터 용이, gdb로 디버깅 가능, 컨텍스트 스위치 없음 → 효율적
- 단점: 코드 복잡도 높음, 동시성 세분화 어려움, 멀티코어 완전 활용 불가
- 채택: Node.js, nginx, Tornado 등 고성능 서버

---

## 12.3 Concurrent Programming with Threads

**스레드**: 프로세스 컨텍스트 안에서 실행되는 논리 흐름. 커널이 자동 스케줄링. **프로세스**와 **I/O 다중화**의 하이브리드.

**스레드 컨텍스트 (스레드 고유):**
- TID (Thread ID), 스택, 스택 포인터, PC, 범용 레지스터, 조건 코드

**프로세스 컨텍스트 공유 (모든 스레드):**
- 가상 주소 공간 전체 (.text, .data, 힙, 공유 라이브러리, 열린 파일)

**프로세스 대비 특징:**
- 컨텍스트 스위치가 빠름 (스레드 컨텍스트 << 프로세스 컨텍스트)
- 부모-자식 계층 없음 → **피어(peer)** 스레드 풀

### Pthreads API

```c
#include <pthread.h>

/* 스레드 생성 */
int pthread_create(pthread_t *tid, pthread_attr_t *attr,
                   func *f, void *arg);
// f: void *(func)(void *) 형태의 스레드 루틴
// 반환: 0 성공 / 비0 오류

/* 자신의 TID 조회 */
pthread_t pthread_self(void);

/* 스레드 종료 방법 */
// 1. 스레드 루틴에서 return
// 2. 명시적 종료
void pthread_exit(void *thread_return);
// main에서 호출 시: 모든 피어 스레드 종료 대기 후 프로세스 종료

// 3. 다른 스레드에 의한 강제 종료
int pthread_cancel(pthread_t tid);

/* 스레드 회수 (joinable 스레드) */
int pthread_join(pthread_t tid, void **thread_return);
// tid 스레드가 종료할 때까지 블록
// thread_return: 스레드 루틴 반환값 저장 위치
// 특정 TID만 대기 가능 (waitpid(-1,...) 같은 임의 대기 없음)

/* 스레드 분리 (메모리 자동 해제) */
int pthread_detach(pthread_t tid);
// 분리된 스레드: 종료 시 시스템이 메모리 자동 해제 (reap 불필요)
// 실제 서버에서는 detached 스레드 사용 권장

/* 1회 초기화 */
pthread_once_t once_control = PTHREAD_ONCE_INIT;
int pthread_once(pthread_once_t *once_control, void (*init)(void));
// 여러 스레드가 호출해도 init_routine은 딱 한 번만 실행
```

### Pthreads "Hello, World!"

```c
void *thread(void *vargp) {
    printf("Hello, world!\n");
    return NULL;
}

int main() {
    pthread_t tid;
    Pthread_create(&tid, NULL, thread, NULL);
    Pthread_join(tid, NULL);   // 피어 스레드 종료 대기
    exit(0);
}
```

### 스레드 기반 동시 에코 서버

```c
int main(int argc, char **argv) {
    int listenfd, *connfdp;
    pthread_t tid;

    listenfd = open_listenfd(argv[1]);
    while (1) {
        connfdp = Malloc(sizeof(int));     // ← 레이스 방지: 각 스레드에 독립 메모리
        *connfdp = Accept(listenfd, ...);
        Pthread_create(&tid, NULL, thread, connfdp);
    }
}

void *thread(void *vargp) {
    int connfd = *((int *)vargp);
    Pthread_detach(pthread_self());  // 분리: 메모리 자동 해제
    Free(vargp);                      // malloc한 메모리 해제
    echo(connfd);
    Close(connfd);
    return NULL;
}
```

**connfd를 malloc으로 전달하는 이유:**
```c
// 잘못된 방법 (레이스 발생!)
*connfdp = Accept(...);
Pthread_create(&tid, NULL, thread, &connfd);
// 문제: 피어 스레드가 *vargp를 읽기 전에 메인 스레드가
//       다음 accept로 connfd를 덮어쓸 수 있음!

// 올바른 방법
connfdp = Malloc(sizeof(int));  // 각 연결마다 별도 메모리 할당
*connfdp = Accept(...);          // 새 메모리에 저장
Pthread_create(&tid, NULL, thread, connfdp);
// 스레드 루틴에서 반드시 Free(vargp) 호출
```

---

## 12.4 Shared Variables in Threaded Programs

### 스레드 메모리 모델

```
각 스레드 고유:  레지스터 (절대 공유 안 됨)
                 스택 (보통 독립, 포인터로 접근 가능하므로 완전 보호는 아님)

모든 스레드 공유: 가상 메모리 전체 (항상 공유)
                  = 코드(.text), 읽기/쓰기 데이터(.data), 힙, 공유 라이브러리, 열린 파일
```

### 변수 인스턴스와 공유 판별

```c
char **ptr;           // 전역 변수: 인스턴스 1개, 모든 스레드가 접근 → 공유

int main() {
    int i;             // 로컬 자동 변수: 메인 스레드 스택에 인스턴스 1개 (i.m)
    pthread_t tid;
    char *msgs[N] = {"Hello from foo", "Hello from bar"};  // msgs.m (스택)
    ptr = msgs;
    for (i = 0; i < N; i++)
        Pthread_create(&tid, NULL, thread, (void *)i);
}

void *thread(void *vargp) {
    int myid = (int)vargp;    // 로컬 자동 변수: 피어마다 독립 인스턴스 (myid.p0, myid.p1)
    static int cnt = 0;       // 로컬 static 변수: 인스턴스 1개, 모든 스레드가 공유
    printf("[%d]: %s (cnt=%d)\n", myid, ptr[myid], ++cnt);
}
```

**공유 변수 판별 규칙:** 변수의 인스턴스가 둘 이상의 스레드에 의해 참조되면 공유.

| 변수 종류 | 메모리 위치 | 공유 여부 |
|---------|----------|---------|
| 전역 변수 | 읽기/쓰기 영역 (1개 인스턴스) | **공유** |
| 로컬 static 변수 | 읽기/쓰기 영역 (1개 인스턴스) | **공유** |
| 로컬 자동 변수 | 각 스레드 스택 (스레드마다 독립) | **비공유** (보통) |

---

## 12.5 Synchronizing Threads with Semaphores

### 동기화 문제 — badcnt.c

```c
volatile long cnt = 0;

void *thread(void *vargp) {
    for (i = 0; i < niters; i++)
        cnt++;  // 비원자적! Load-Update-Store 3단계
}
// 두 스레드 동시 실행 시 cnt가 예측 불가
// 예: niters=1,000,000 → cnt ≠ 2,000,000
```

**카운터 루프의 어셈블리 분해:**
```
H_i : 루프 헤드 명령들
L_i : movq cnt(%rip), %rdx_i   (공유 변수 로드)
U_i : addq $1, %rdx_i           (레지스터 증가)
S_i : movq %rdx_i, cnt(%rip)   (공유 변수 저장)
T_i : 루프 테일 명령들
```

{L_i, U_i, S_i} = **임계 구역(critical section)** — 상호 배타적 접근 필요.

### 진행 그래프 (Progress Graph)

- 2D 직교 좌표계: 각 축 = 스레드 진행도
- 안전하지 않은 영역(unsafe region): 두 임계 구역의 교집합
- 안전 궤적: unsafe region을 건드리지 않는 궤적

### 세마포어 (Semaphore)

음수가 아닌 정수 + 두 가지 원자적 연산:

```
P(s): if s > 0: s-- (즉시 반환)
      if s == 0: 스레드 일시 중단, 다른 스레드의 V 연산 대기
V(s): s++, 대기 스레드 하나 재시작
→ 불변식: s ≥ 0 항상 유지
```

```c
#include <semaphore.h>

int sem_init(sem_t *sem, 0, unsigned int value);
int sem_wait(sem_t *s);   /* P(s) */
int sem_post(sem_t *s);   /* V(s) */

// csapp.h 래퍼 (오류 처리 포함)
void P(sem_t *s);
void V(sem_t *s);
```

### 12.5.3 뮤텍스 (Mutual Exclusion)

**이진 세마포어(binary semaphore)** = **뮤텍스(mutex)**: 초기값 1.

```c
volatile long cnt = 0;
sem_t mutex;
Sem_init(&mutex, 0, 1);

void *thread(void *vargp) {
    for (i = 0; i < niters; i++) {
        P(&mutex);   // 뮤텍스 잠금 (임계 구역 진입)
        cnt++;
        V(&mutex);   // 뮤텍스 해제 (임계 구역 탈출)
    }
}
```

**진행 그래프 해석:**
```
P(s)와 V(s) 연산이 s < 0인 금지 구역(forbidden region) 생성
금지 구역이 안전하지 않은 구역을 완전히 둘러쌈
→ 어떤 실행 궤적도 안전하지 않은 구역에 진입 불가
→ 항상 올바른 결과 보장
```

### 12.5.4 세마포어를 이용한 자원 스케줄링

#### 생산자-소비자 문제 (Producer-Consumer)

```
생산자 → [bounded buffer (n slots)] → 소비자
         ↑                    ↑
    버퍼 꽉 차면 대기    버퍼 비면 대기
```

**Sbuf 패키지 (동기화된 원형 버퍼):**

```c
typedef struct {
    int *buf;      // 버퍼 배열
    int n;         // 최대 슬롯 수
    int front;     // 첫 번째 항목
    int rear;      // 마지막 항목
    sem_t mutex;   // 버퍼 접근 뮤텍스
    sem_t slots;   // 사용 가능한 슬롯 수 (초기값 n)
    sem_t items;   // 사용 가능한 항목 수 (초기값 0)
} sbuf_t;

void sbuf_insert(sbuf_t *sp, int item) {
    P(&sp->slots);   // 빈 슬롯 대기
    P(&sp->mutex);   // 뮤텍스 잠금
    sp->buf[(++sp->rear) % sp->n] = item;
    V(&sp->mutex);   // 뮤텍스 해제
    V(&sp->items);   // 항목 추가 알림
}

int sbuf_remove(sbuf_t *sp) {
    P(&sp->items);   // 항목 대기
    P(&sp->mutex);   // 뮤텍스 잠금
    int item = sp->buf[(++sp->front) % sp->n];
    V(&sp->mutex);   // 뮤텍스 해제
    V(&sp->slots);   // 슬롯 복원 알림
    return item;
}
```

#### 독자-저자 문제 (Readers-Writers)

```
독자(reader): 공유 객체 읽기만 → 여러 독자 동시 접근 허용
저자(writer): 공유 객체 수정 → 독점적 접근 필요
```

**첫 번째 독자-저자 문제 (독자 우선):**
```c
int readcnt = 0;        // 현재 임계 구역에 있는 독자 수
sem_t mutex, w;         // 각각 1로 초기화

void reader() {
    P(&mutex);          // readcnt 보호
    readcnt++;
    if (readcnt == 1)   // 첫 번째 독자: 저자 차단
        P(&w);
    V(&mutex);
    /* 임계 구역: 읽기 수행 */
    P(&mutex);
    readcnt--;
    if (readcnt == 0)   // 마지막 독자: 저자 허용
        V(&w);
    V(&mutex);
}

void writer() {
    P(&w);              // 모든 독자/저자 차단
    /* 임계 구역: 쓰기 수행 */
    V(&w);
}
```

> **기아(starvation)**: 독자 우선 해결책에서 독자가 계속 오면 저자는 무한 대기 가능.

### 12.5.5 사전 스레딩 서버 (Prethreaded Server)

매 클라이언트마다 스레드 생성 대신 **미리 생성된 작업자 스레드 풀** 사용 → 스레드 생성 오버헤드 제거.

```
메인 스레드 (생산자):
  accept() → connfd → sbuf에 삽입

작업자 스레드들 (소비자, 4개):
  sbuf에서 connfd 제거 → echo_cnt() 서비스
```

```c
#define NTHREADS 4
#define SBUFSIZE 16

sbuf_t sbuf;  // 공유 버퍼

int main() {
    sbuf_init(&sbuf, SBUFSIZE);
    for (i = 0; i < NTHREADS; i++)
        Pthread_create(&tid, NULL, thread, NULL);  // 작업자 스레드 사전 생성

    while (1) {
        connfd = Accept(listenfd, ...);
        sbuf_insert(&sbuf, connfd);  // 생산자
    }
}

void *thread(void *vargp) {
    Pthread_detach(pthread_self());
    while (1) {
        int connfd = sbuf_remove(&sbuf);  // 소비자
        echo_cnt(connfd);
        Close(connfd);
    }
}
```

**pthread_once 패턴 (패키지 초기화):**
```c
static pthread_once_t once = PTHREAD_ONCE_INIT;

void echo_cnt(int connfd) {
    Pthread_once(&once, init_echo_cnt);  // 첫 번째 호출 시만 실행
    // ...
}
```

---

## 12.6 Using Threads for Parallelism

**순차(Sequential) ⊃ 동시(Concurrent) ⊃ 병렬(Parallel)**

병렬 프로그램은 멀티코어에서 동시에 실행되는 동시 프로그램이다.

### 병렬 덧셈 비교

```c
// 잘못된 방법: 공유 전역 변수 + 뮤텍스 (매우 느림!)
void *sum_mutex(void *vargp) {
    for (i = start; i < end; i++) {
        P(&mutex);
        gsum += i;    // 뮤텍스 경합이 심각한 오버헤드
        V(&mutex);
    }
}

// 개선 1: 스레드별 전용 배열 원소 (훨씬 빠름)
void *sum_array(void *vargp) {
    for (i = start; i < end; i++)
        psum[myid] += i;  // 스레드 고유 원소, 동기화 불필요
}

// 개선 2: 로컬 변수 사용 후 배열에 저장 (가장 빠름)
void *sum_local(void *vargp) {
    long sum = 0;
    for (i = start; i < end; i++)
        sum += i;   // 레지스터 수준 연산, 메모리 참조 최소화
    psum[myid] = sum;
}
```

**성능 비교 (4코어, n=2³¹):**
```
             스레드 수:  1       2       4       8      16
psum-mutex:          68.0   432.0   719.0   552.0   599.0 (뮤텍스로 역효과)
psum-array:           7.3     3.6     1.9     1.9     1.8
psum-local:           1.1     0.5     0.3     0.3     0.3
```

### 병렬 성능 지표

```
강한 스케일링 (Strong Scaling, 고정 문제 크기):
  스피드업: Sp = T1 / Tp   (T1: 1개 코어 시간, Tp: p개 코어 시간)
  효율성:   Ep = Sp / p    (범위: 0~100%)

약한 스케일링 (Weak Scaling):
  코어 수만큼 문제 크기도 늘림 → 코어당 작업량 일정
  더 현실적 측정 (더 많은 코어로 더 큰 문제 해결)
```

**psum-local 성능:**
| 스레드 수 | 1 | 2 | 4 | 8 | 16 |
|---------|---|---|---|---|----|
| 실행 시간(s) | 1.06 | 0.54 | 0.28 | 0.29 | 0.30 |
| 스피드업 Sp | 1 | 1.9 | 3.8 | 3.7 | 3.5 |
| 효율성 Ep | 100% | 98% | 95% | 91% | 88% |

t > 4 (코어 수 초과) 시 성능 감소 → 동일 코어에서 여러 스레드 컨텍스트 스위치 오버헤드.

> **핵심 교훈**: 동기화 오버헤드는 매우 비쌈. 가능하면 피하고, 불가피하면 최대한 분산. 사소한 코드 변경이 성능에 극적 영향을 미친다.

---

## 12.7 Other Concurrency Issues

### 12.7.1 스레드 안전성 (Thread Safety)

**스레드 안전 함수:** 여러 동시 스레드가 반복 호출해도 항상 올바른 결과 반환.

**4종류의 스레드 비안전 함수:**

| 클래스 | 설명 | 해결책 |
|--------|------|--------|
| Class 1 | 공유 변수 보호 없음 | P/V로 임계 구역 보호 (호출부 변경 불필요) |
| Class 2 | 여러 호출 간 상태 유지 (static 변수) | 재작성: 상태를 호출자 인수로 전달 (호출부 변경 필요) |
| Class 3 | static 변수 포인터 반환 (`ctime`, `gethostbyname`) | lock-and-copy 기법 |
| Class 4 | 비안전 함수 호출 | g의 클래스에 따라 다름 |

**lock-and-copy 기법 (Class 3 해결):**
```c
char *ctime_ts(const time_t *timep, char *privatep) {
    char *sharedp;
    P(&mutex);
    sharedp = ctime(timep);         // 비안전 함수 호출
    strcpy(privatep, sharedp);      // 결과를 프라이빗 메모리로 복사
    V(&mutex);
    return privatep;
}
```

### 12.7.2 재진입성 (Reentrancy)

**재진입 함수(reentrant function)**: 공유 데이터를 전혀 참조하지 않는 함수. 스레드 안전 함수의 진정한 부분집합.

```
모든 함수 = 스레드 안전 ∪ 스레드 비안전
스레드 안전 ⊃ 재진입 함수 (더 강한 보장)
```

- **명시적 재진입**: 모든 인수가 값으로 전달, 모든 데이터 참조가 로컬 자동 스택 변수
- **암시적 재진입**: 포인터 인수가 있지만 비공유 데이터를 가리키는 경우

```c
// 비재진입: static 변수 사용
unsigned rand(void) {
    next_seed = next_seed * 1103515245 + 12543;  // static next_seed 공유!
    return (unsigned)(next_seed >> 16) % 32768;
}

// 재진입 버전: 상태를 호출자가 관리
int rand_r(unsigned int *nextp) {
    *nextp = *nextp * 1103515245 + 12345;  // nextp는 호출자 소유
    return (unsigned int)(*nextp / 65536) % 32768;
}
```

**재진입 표준 라이브러리 함수:** `_r` 접미사 버전 사용 권장
- `asctime_r`, `ctime_r`, `gethostbyname_r`, `rand_r`, `strtok_r` 등

### 12.7.3 비안전 라이브러리 함수 목록

| 비안전 함수 | 클래스 | 안전 대체 |
|----------|--------|---------|
| `rand` | 2 | `rand_r` |
| `strtok` | 2 | `strtok_r` |
| `asctime`, `ctime`, `localtime` | 3 | `*_r` 버전 |
| `gethostbyname`, `gethostbyaddr` | 3 | `getaddrinfo`, `getnameinfo` |
| `inet_ntoa` | 3 | `inet_ntop` |

### 12.7.4 레이스 (Races)

**레이스**: 한 스레드가 특정 지점에 도달하는 순서가 다른 스레드보다 먼저여야 한다는 가정에 의존하는 버그.

```c
// 잘못된 코드: 레이스 발생!
for (i = 0; i < N; i++)
    Pthread_create(&tid[i], NULL, thread, &i);  // &i 공유!

void *thread(void *vargp) {
    int myid = *((int *)vargp);  // 메인이 i를 증가시키기 전에 읽어야 올바름
    // 레이스: 메인의 i++ vs 피어의 *vargp 역참조
}
```

```c
// 올바른 코드: 각 스레드마다 독립 메모리 할당
for (i = 0; i < N; i++) {
    int *ptr = Malloc(sizeof(int));
    *ptr = i;
    Pthread_create(&tid[i], NULL, thread, ptr);
}

void *thread(void *vargp) {
    int myid = *((int *)vargp);
    Free(vargp);   // 스레드가 책임지고 해제
    printf("Hello from thread %d\n", myid);
}
```

### 12.7.5 교착 상태 (Deadlock)

**교착 상태**: 스레드 집합이 영원히 참이 되지 않을 조건을 기다리며 블록된 상태.

```
교착 상태 발생 조건 (세마포어 잘못 사용):
Thread 1: P(s) → P(t) → V(s) → V(t)
Thread 2: P(t) → P(s) → V(t) → V(s)
→ Thread 1이 P(s) 후 P(t) 대기, Thread 2가 P(t) 후 P(s) 대기 → 교착
```

**진행 그래프에서 교착 분석:**
- 두 세마포어의 금지 구역이 겹치면 → 교착 구역(deadlock region) 형성
- 실행 궤적이 교착 구역에 진입하면 탈출 불가 (매우 예측하기 어려움)

**뮤텍스 기반 교착 방지 규칙:**

> **뮤텍스 잠금 순서 규칙 (Mutex Lock Ordering Rule):**  
> 모든 스레드가 뮤텍스를 **동일한 순서**로 잠근다면 교착 상태를 피할 수 있다.

```c
// 올바른 패턴: 모든 스레드가 s → t 순서로 잠금
Thread 1: P(s) → P(t) → V(t) → V(s)
Thread 2: P(s) → P(t) → V(t) → V(s)
→ 교착 없음
```

---

## 세 가지 동시성 방식 비교

| 기준 | 프로세스 | I/O 다중화 | 스레드 |
|------|---------|----------|--------|
| 주소 공간 공유 | ✗ (사설) | ✓ (단일 프로세스) | ✓ (공유) |
| 데이터 공유 | 어려움 (IPC 필요) | 쉬움 (전역 변수) | 쉬움 (공유 변수) |
| 동기화 필요 여부 | 낮음 | 낮음 | **높음 (필수)** |
| 성능 | 느림 (IPC/fork 오버헤드) | 빠름 (스위치 없음) | 중간 (컨텍스트 스위치) |
| 멀티코어 활용 | ✓ | ✗ (단일 프로세스) | ✓ |
| 디버깅 | 어려움 | 쉬움 (순차와 유사) | 어려움 |
| 코드 복잡도 | 중간 | 높음 | 중간 |

---

## 요약

| 주제 | 핵심 |
|------|------|
| 동시성 3방식 | 프로세스(독립)/I/O다중화(단일 프로세스)/스레드(하이브리드) |
| 프로세스 기반 | SIGCHLD 핸들러로 좀비 회수, 부모의 connfd close 필수 |
| select() | fd_set 비트벡터, side effect로 ready_set 반환, 매번 복사 필요 |
| 이벤트 주도 | 상태 기계 모델, pool 구조, nginx/Node.js 채택 |
| 스레드 | pthread_create/join/detach/cancel/once, joinable vs detached |
| 공유 변수 | 전역·static: 공유, 로컬 자동: 스레드별 독립 |
| 세마포어 | P(s)/V(s), 뮤텍스(초기값 1), 금지 구역→임계 구역 보호 |
| 생산자-소비자 | sbuf_t: mutex+slots+items, 3-세마포어 패턴 |
| 독자-저자 | readcnt + w 세마포어, 독자 우선 → 기아 가능 |
| 병렬 프로그래밍 | 동기화 오버헤드 최소화, 로컬 변수 활용, Sp=T1/Tp |
| 스레드 안전성 | 4가지 클래스, lock-and-copy, _r 접미사 재진입 함수 |
| 레이스 | &i 공유 버그 → malloc으로 독립 메모리 할당 |
| 교착 상태 | 겹치는 금지 구역, 뮤텍스 잠금 순서 일관성으로 방지 |
