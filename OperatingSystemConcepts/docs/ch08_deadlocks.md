# Chapter 8: Deadlocks

## 목차
1. [시스템 모델 (System Model)](#1-시스템-모델)
2. [멀티스레드 애플리케이션에서의 데드락](#2-멀티스레드-애플리케이션에서의-데드락)
3. [데드락 특성 (Deadlock Characterization)](#3-데드락-특성)
4. [데드락 처리 방법 개요](#4-데드락-처리-방법-개요)
5. [데드락 예방 (Deadlock Prevention)](#5-데드락-예방)
6. [데드락 회피 (Deadlock Avoidance)](#6-데드락-회피)
7. [데드락 감지 (Deadlock Detection)](#7-데드락-감지)
8. [데드락 복구 (Recovery from Deadlock)](#8-데드락-복구)

---

## 1. 시스템 모델

스레드는 자원을 사용하기 전에 **요청(request) → 사용(use) → 해제(release)** 순서를 반드시 따른다.

```
스레드 수명주기에서의 자원 접근:

  request()  →  [대기 또는 즉시 획득]  →  use()  →  release()
  (wait/semaphore acquire)               (임계구역)   (signal/release)
```

- 자원 타입(resource type) Rj는 동일한 인스턴스 Wj개를 보유
- 뮤텍스 락·세마포는 현대 시스템에서 데드락의 가장 흔한 원인
- 커널이 자원 할당 테이블(allocated/free 상태)을 관리

**데드락 정의**: 집합 내 모든 스레드가, 그 집합 내 다른 스레드만이 유발할 수 있는 이벤트를 기다리는 상태.

---

## 2. 멀티스레드 애플리케이션에서의 데드락

### 2.1 뮤텍스 락 데드락

```c
pthread_mutex_t first_mutex;
pthread_mutex_t second_mutex;
pthread_mutex_init(&first_mutex, NULL);
pthread_mutex_init(&second_mutex, NULL);

/* thread_one: first → second 순으로 획득 */
void *do_work_one(void *param) {
    pthread_mutex_lock(&first_mutex);
    pthread_mutex_lock(&second_mutex);
    /* 작업 */
    pthread_mutex_unlock(&second_mutex);
    pthread_mutex_unlock(&first_mutex);
    pthread_exit(0);
}

/* thread_two: second → first 순으로 획득 */
void *do_work_two(void *param) {
    pthread_mutex_lock(&second_mutex);
    pthread_mutex_lock(&first_mutex);
    /* 작업 */
    pthread_mutex_unlock(&first_mutex);
    pthread_mutex_unlock(&second_mutex);
    pthread_exit(0);
}
```

**데드락 시나리오**: thread_one이 first_mutex를 획득한 상태에서 thread_two가 second_mutex를 획득하면, 두 스레드 모두 영원히 블록된다.

---

### 2.2 라이브락 (Livelock)

데드락과 유사한 활성 실패(liveness failure). 스레드가 블록되지 않지만 진전도 없이 계속 재시도.

```c
/* trylock을 사용한 라이브락 예시 */
void *do_work_one(void *param) {
    int done = 0;
    while (!done) {
        pthread_mutex_lock(&first_mutex);
        if (pthread_mutex_trylock(&second_mutex)) {
            /* 작업 성공 */
            pthread_mutex_unlock(&second_mutex);
            pthread_mutex_unlock(&first_mutex);
            done = 1;
        } else {
            pthread_mutex_unlock(&first_mutex);
            /* 실패 → 처음부터 재시도 */
        }
    }
}
```

**발생 조건**: thread_one이 first_mutex, thread_two가 second_mutex를 획득 → 각자 trylock 실패 → 해제 후 무한 반복.

**해결책**: 재시도 전 **무작위 지연(random backoff)** 삽입 (이더넷 CSMA/CD와 동일한 원리).

---

## 3. 데드락 특성

### 3.1 필요 조건 4가지 (Coffman 조건)

데드락은 아래 4가지 조건이 **동시에** 성립할 때만 발생한다.

| 조건 | 설명 |
|------|------|
| **상호 배제 (Mutual Exclusion)** | 자원이 비공유 모드로 점유됨 (동시 접근 불가) |
| **점유 대기 (Hold and Wait)** | 최소 1개 자원을 점유한 채 다른 자원을 기다림 |
| **비선점 (No Preemption)** | 할당된 자원은 자발적으로만 해제됨 |
| **순환 대기 (Circular Wait)** | T0→T1→…→Tn→T0 형태의 대기 순환 존재 |

> 순환 대기는 점유 대기를 함의하므로 조건이 완전히 독립적이지는 않다. 하지만 예방 전략 설계 시 각각 별도로 고려한다.

---

### 3.2 자원 할당 그래프 (Resource-Allocation Graph)

유향 그래프 G = (V, E)로 데드락 상태를 정밀하게 표현.

```
노드:
  스레드 Ti   → 원(circle)으로 표현
  자원 타입 Rj → 사각형(rectangle)으로 표현, 인스턴스 수 = 내부 점(dot)

간선 종류:
  요청 간선 (request edge):  Ti → Rj  (Ti가 Rj를 요청 중)
  할당 간선 (assignment edge): Rj → Ti  (Rj의 인스턴스가 Ti에 할당됨)
```

**예시: 데드락 있는 그래프**

```
  T1 ──request──▶ [R1] ──assign──▶ T2
  ▲                                  │
  │                                  │ request
  assign                             ▼
  [R2] ◀──assign── T3 ◀──request── [R3]
   │
   └──assign──▶ T1   ← T2도 R2 할당받음 (2 instances)
```

**사이클 판단 규칙**

- 자원 타입별 인스턴스가 **1개**인 경우: 사이클 ↔ 데드락 (필요충분)
- 자원 타입별 인스턴스가 **여러 개**인 경우: 사이클 → 데드락 가능성 (필요조건만)

---

## 4. 데드락 처리 방법 개요

```
데드락 처리 전략
├── 1. 무시 (Ostrich Algorithm)
│      └── Linux, Windows 기본 채택 (빈도 낮을 때 비용 대비 효과)
├── 2. 예방/회피 (Prevention / Avoidance)
│      ├── 예방: 4가지 필요 조건 중 하나 제거
│      └── 회피: 사전 정보 활용, 항상 안전 상태(safe state) 유지
└── 3. 감지 + 복구 (Detection + Recovery)
       └── 데이터베이스 시스템에서 주로 채택
```

---

## 5. 데드락 예방

### 5.1 상호 배제 제거

공유 가능 자원(예: 읽기 전용 파일)은 데드락에 관여하지 않는다. 그러나 뮤텍스 락처럼 본질적으로 비공유인 자원은 제거 불가 → **현실적 해결책 아님**.

### 5.2 점유 대기 제거

**프로토콜 A**: 실행 시작 전 모든 자원을 한 번에 요청 → 자원 낭비, 기아(starvation) 위험

**프로토콜 B**: 현재 보유 자원이 없을 때만 추가 자원 요청 가능 (추가 요청 전 기존 자원 전부 해제)

**단점**: 자원 이용률 저하, 인기 자원 조합을 필요로 하는 스레드의 기아 가능

### 5.3 비선점 제거

스레드가 자원을 보유하고 즉시 할당받을 수 없는 추가 자원을 요청하면 → 현재 보유 자원을 **암묵적으로 모두 해제**(선점당함).

- CPU 레지스터, DB 트랜잭션처럼 상태 저장/복원이 쉬운 자원에 적합
- **뮤텍스 락·세마포에는 적용 불가** (데드락 발생 주 자원인데 역설)

### 5.4 순환 대기 제거 ← 가장 실용적

모든 자원 타입에 **전순서(total ordering)** 부여: F: R → ℕ

```c
// 예시: 자원 번호 할당
F(first_mutex)  = 1
F(second_mutex) = 5

// 프로토콜: 번호 오름차순으로만 자원 획득 가능
// → thread_two도 first_mutex(1) → second_mutex(5) 순서 강제
//   순환 대기 수학적으로 불가능함을 귀류법으로 증명:
//   F(R0) < F(R1) < ... < F(Rn) < F(R0) → 모순
```

**동적 락의 한계**: 계좌 이체 함수처럼 런타임에 락이 결정될 경우 순서 강제 어려움.

```c
// 데드락 가능 예 (동적 락 순서)
void transaction(Account from, Account to, double amount) {
    lock1 = get_lock(from);
    lock2 = get_lock(to);
    acquire(lock1);   // thread A: checking→savings
    acquire(lock2);   // thread B: savings→checking → 데드락!
    ...
}
```

Java에서는 `System.identityHashCode(Object)`를 락 획득 순서 함수로 활용하는 관례가 있다.

> **Linux lockdep**: 커널 내 락 획득 순서를 동적으로 감시하는 도구. 순서 위반·인터럽트 핸들러에서의 스핀락 사용 등을 탐지. 2006년 도입 후 데드락 보고 수가 10배 감소.

---

## 6. 데드락 회피

예방은 자원 이용률을 낮추고 처리량을 감소시킨다. 회피는 **사전 정보(maximum need)**를 활용해 항상 안전 상태를 유지한다.

### 6.1 안전 상태 (Safe State)

**안전 순서(safe sequence)** `<T1, T2, ..., Tn>`가 존재하면 시스템은 안전 상태.

조건: 각 Ti가 추가로 필요한 자원 = (현재 가용 자원) + (Tj<i가 해제할 자원)

```
안전(safe) ⊂ 불안전(unsafe) ⊂ 데드락(deadlock)

   ┌──────────────────────────────┐
   │  unsafe                      │
   │   ┌────────────────────┐     │
   │   │   safe             │     │
   │   └────────────────────┘     │
   │              ┌────────────┐  │
   │              │  deadlock  │  │
   │              └────────────┘  │
   └──────────────────────────────┘

- 안전 상태 → 데드락 없음 (보장)
- 불안전 상태 → 데드락 가능성 (반드시 데드락은 아님)
```

**예시**: 자원 12개, T0(max=10), T1(max=4), T2(max=9)

| 스레드 | 최대 요구 | 현재 할당 | 현재 필요 |
|--------|----------|----------|----------|
| T0 | 10 | 5 | 5 |
| T1 | 4 | 2 | 2 |
| T2 | 9 | 2 | 7 |

가용 자원 = 3. 안전 순서 `<T1, T0, T2>` 존재 → 안전 상태.  
T2가 자원 1개 추가 요청하면 가용 = 2 → T1만 실행 가능 → **불안전 상태**.

---

### 6.2 자원 할당 그래프 알고리즘 (단일 인스턴스)

**클레임 간선(claim edge)** Ti →Rj (점선): Ti가 미래에 Rj를 요청할 수 있음.

```
클레임 간선 ──(요청 시)──▶ 요청 간선 ──(할당 시)──▶ 할당 간선
              (실선)                    (역방향 실선)
```

**알고리즘**: 요청 처리 시 클레임 간선을 할당 간선으로 변환해도 사이클이 없으면 허용.  
사이클 감지 O(n²). 사이클 발생 → 불안전 상태 → 요청 거부.

---

### 6.3 은행원 알고리즘 (Banker's Algorithm, 다중 인스턴스)

n개 스레드, m개 자원 타입일 때 필요한 데이터 구조:

| 구조 | 크기 | 의미 |
|------|------|------|
| `Available[j]` | m | 자원 타입 Rj의 가용 인스턴스 수 |
| `Max[i][j]` | n×m | Ti의 Rj에 대한 최대 요구 수 |
| `Allocation[i][j]` | n×m | Ti에 현재 할당된 Rj 인스턴스 수 |
| `Need[i][j]` | n×m | Ti가 추가로 필요한 Rj 수 = Max[i][j] − Allocation[i][j] |

#### 안전성 알고리즘 (Safety Algorithm) — O(m × n²)

```
1. Work = Available, Finish[i] = false (모든 i)
2. Finish[i]==false이고 Need[i] ≤ Work인 i 탐색
   → 없으면 4단계로
3. Work = Work + Allocation[i]
   Finish[i] = true
   → 2단계로 반복
4. 모든 i에 대해 Finish[i]==true → 시스템 안전 상태
```

#### 자원 요청 알고리즘 (Resource-Request Algorithm)

Ti의 요청 벡터 Request_i에 대해:

```
1. Request_i ≤ Need_i?  → 아니면 에러 (최대 요구 초과)
2. Request_i ≤ Available? → 아니면 Ti 대기
3. 가상 할당 (provisional allocation):
     Available    -= Request_i
     Allocation_i += Request_i
     Need_i       -= Request_i
4. 안전성 알고리즘 실행:
     안전 → 요청 승인 (실제 할당)
     불안전 → 요청 거부, 이전 상태 복원, Ti 대기
```

#### 예시: 5스레드(T0~T4), 3자원(A:10, B:5, C:7)

**현재 상태**

| | Allocation (A B C) | Max (A B C) | Need (A B C) |
|--|--|--|--|
| T0 | 0 1 0 | 7 5 3 | 7 4 3 |
| T1 | 2 0 0 | 3 2 2 | 1 2 2 |
| T2 | 3 0 2 | 9 0 2 | 6 0 0 |
| T3 | 2 1 1 | 2 2 2 | 0 1 1 |
| T4 | 0 0 2 | 4 3 3 | 4 3 1 |

Available = (3, 3, 2). 안전 순서: **<T1, T3, T4, T2, T0>**

**T1이 (1,0,2) 요청 시:**
- Request_1 = (1,0,2) ≤ Available = (3,3,2) → 가상 할당
- 새 Available = (2,3,0)
- 안전성 검사: `<T1, T3, T4, T0, T2>` 존재 → **요청 승인**

**불승인 예시**: T4가 (3,3,0) 요청 → Available 부족으로 거부.  
T0가 (0,2,0) 요청 → 자원은 있지만 결과 상태가 불안전 → 거부.

---

## 7. 데드락 감지

예방·회피 알고리즘 미사용 시 데드락 발생 가능. 주기적으로 감지 후 복구.

### 7.1 단일 인스턴스: 대기 그래프 (Wait-For Graph)

자원 할당 그래프에서 자원 노드를 제거하고 간선을 합쳐 생성.

```
자원 할당 그래프:
  Ti → Rq → Tj  (Ti가 Rq 보유한 Tj를 대기)

대기 그래프(wait-for graph):
  Ti → Tj         (자원 노드 제거 후 직접 연결)
```

```
(a) 자원 할당 그래프            (b) 대기 그래프

  T1 → R1 → T3                   T1 → T3
  T2 → R3 → T5                   T2 → T5
  T4 → R2 → T3                   T4 → T3
  T5 → R4 → T4  → R5 → T1        T5 → T4 → T1
```

사이클 탐지 O(n²). 사이클 존재 = 데드락.

> **BCC deadlock detector**: Linux에서 `pthread_mutex_lock/unlock` 프로브를 삽입해 wait-for 그래프를 실시간 추적. 사이클 감지 시 보고.
> **Java thread dump**: JVM이 `Ctrl-L`(Unix) 또는 `Ctrl-Break`(Windows) 입력 시 wait-for 그래프에서 사이클 탐색 후 데드락 스레드 보고.

### 7.2 다중 인스턴스: 감지 알고리즘 — O(m × n²)

은행원 알고리즘과 유사한 데이터 구조 사용. `Need` 대신 `Request` (현재 실제 요청).

```
데이터 구조:
  Available[j]      : 가용 자원
  Allocation[i][j]  : 현재 할당량
  Request[i][j]     : 스레드 i의 현재 요청량

알고리즘:
1. Work = Available
   Finish[i] = (Allocation_i == 0) ? true : false
2. Finish[i]==false이고 Request_i ≤ Work인 i 탐색
   → 없으면 4단계로
3. Work = Work + Allocation_i
   Finish[i] = true  (낙관적 가정: Ti가 곧 완료해 자원 반환)
   → 2단계로
4. Finish[i]==false인 i 존재 → Ti는 데드락 상태
```

**예시**: 5스레드(T0~T4), 3자원(A:7, B:2, C:6)

| | Allocation | Request | Available |
|--|--|--|--|
| T0 | 0 1 0 | 0 0 0 | 0 0 0 |
| T1 | 2 0 0 | 2 0 2 | |
| T2 | 3 0 3 | 0 0 0 | |
| T3 | 2 1 1 | 1 0 0 | |
| T4 | 0 0 2 | 0 0 2 | |

초기: `<T0, T2, T3, T1, T4>` → 모두 완료 → **데드락 없음**.

T2가 C 1개 추가 요청하면 Request_2 = (0,0,1):
T0만 완료 가능 → Available = (0,1,0) → 나머지 T1~T4 불만족 → **T1, T2, T3, T4 데드락**.

### 7.3 감지 알고리즘 호출 시점

- **매 요청 실패 시**: 데드락 원인 스레드 즉시 식별 가능, 오버헤드 큼
- **주기적으로**: 예) 1시간마다 or CPU 이용률 40% 이하로 떨어질 때
  - 데드락이 자원을 idle 상태로 잡아두어 CPU 이용률 급감

---

## 8. 데드락 복구

### 8.1 프로세스·스레드 종료

**방법 A: 데드락 관련 프로세스 전부 종료**
- 데드락 사이클 즉시 제거
- 그러나 오래 실행된 계산 결과 손실

**방법 B: 데드락 해소될 때까지 프로세스 하나씩 종료**
- 종료 후 매번 감지 알고리즘 재실행 → 오버헤드
- **희생자(victim) 선택 기준**:
  1. 프로세스 우선순위
  2. 실행 시간(이미 얼마나 실행했는가)
  3. 점유 자원의 종류와 수
  4. 완료까지 필요한 추가 자원 수
  5. 종료 시 영향받는 프로세스 수

### 8.2 자원 선점 (Resource Preemption)

데드락 스레드에서 자원을 빼앗아 다른 스레드에 할당.

```
고려사항 3가지:

1. 희생자 선택 (Selecting a victim)
   → 최소 비용 기준: 자원 보유 수, 지금까지 소요 시간 등

2. 롤백 (Rollback)
   → 가장 단순: 전체 롤백(abort + restart)
   → 더 효율적: 데드락 해소에 필요한 최소 지점까지만 롤백
     (단, 전체 프로세스 상태 이력 보관 필요)

3. 기아 방지 (Starvation prevention)
   → 동일 프로세스가 반복 희생자로 선택되는 문제
   → 해결: 롤백 횟수를 비용 함수에 포함
           (롤백 많은 프로세스는 희생자 선택 우선순위 낮춤)
```

---

## 데드락 전략 비교

| 전략 | 자원 이용률 | 오버헤드 | 기아 위험 | 적용 대상 |
|------|-----------|---------|----------|----------|
| 예방 — 상호 배제 제거 | 높음 | 낮음 | 없음 | 읽기 전용 자원만 |
| 예방 — 점유 대기 제거 | 낮음 | 낮음 | 있음 | 정적 자원 요구 |
| 예방 — 비선점 | 중간 | 중간 | 가능 | 상태 저장 가능 자원 |
| 예방 — 순환 대기 제거 | 중간 | 낮음 | 가능 | 가장 실용적 |
| 회피 — 은행원 알고리즘 | 중간 | 높음 O(mn²) | 없음 | 최대 요구량 사전 공개 가능 시 |
| 감지 + 복구 | 높음 | 중간(주기적) | 종료 희생자 | DB 트랜잭션 |
| 무시 | 최고 | 없음 | 있음 | Linux, Windows 기본 |

---

## 핵심 정리

- **데드락 4 필요 조건**: 상호 배제, 점유 대기, 비선점, 순환 대기 — **동시** 성립 시만 발생
- **자원 할당 그래프**: 단일 인스턴스 → 사이클 = 데드락 (필충); 다중 인스턴스 → 사이클 = 가능성만
- **예방의 핵심**: 순환 대기 제거(자원 번호 전순서)가 유일하게 실용적
- **은행원 알고리즘**: 안전 상태 보장, Need = Max − Allocation, 안전성 알고리즘 O(mn²)
- **감지**: 단일 인스턴스 → wait-for graph O(n²); 다중 인스턴스 → 은행원 유사 알고리즘 O(mn²)
- **복구**: 프로세스 종료(전체 or 부분) vs 자원 선점 + 롤백 (기아 방지 필수)
- **현실**: 대부분 OS(Linux/Windows)는 데드락을 무시하고 개발자 책임으로 전가
