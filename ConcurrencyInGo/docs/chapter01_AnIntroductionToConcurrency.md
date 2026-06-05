# Chapter 1. An Introduction to Concurrency

## Moore's Law, Web Scale, and the Mess We're In

무어의 법칙(Moore's Law)은 1965년 고든 무어(Gordon Moore)가 집적 회로(integrated circuit)의 트랜지스터 수가 매년 두 배씩 증가한다고 예측한 법칙이다. 1975년에 2년마다 두 배로 완화되었고, 이 예측은 2012년경까지 유효했다.

클럭 속도 향상이 한계에 부딪히자 업계는 멀티코어 프로세서(multicore processor)로 방향을 틀었다. 하지만 멀티코어를 활용하려면 코드 자체가 병렬(parallel)로 실행될 수 있어야 한다는 새로운 문제에 봉착했다.

**암달의 법칙(Amdahl's Law):** 병렬화를 통한 성능 향상의 상한은 순차적으로 실행되어야 하는 코드의 비율에 의해 결정된다. 프로그램의 어떤 부분도 병렬화할 수 없는 순차 코드가 병목이 된다.

클라우드 컴퓨팅(cloud computing)의 등장으로 수평 확장(horizontal scaling)이 현실화되었고, **웹 스케일(web scale)** 소프트웨어는 수십만 개의 동시 워크로드를 처리해야 한다는 기대치가 생겼다. 결국 현대 개발자에게 동시성(concurrency)은 피할 수 없는 과제가 되었다.

---

## Why Is Concurrency Hard?

동시성 코드가 어려운 이유는 구조적이다. 아래 네 가지 핵심 문제를 이해하면 대부분의 동시성 버그 패턴이 분류된다.

### Race Conditions (경쟁 조건)

두 개 이상의 연산이 올바른 순서로 실행되어야 하는데, 그 순서가 보장되지 않을 때 발생한다. 가장 흔한 형태는 **데이터 레이스(data race)**: 한 고루틴(goroutine)이 변수를 읽는 동안 다른 고루틴이 동시에 쓰는 상황이다.

```go
var data int
go func() {
    data++
}()
if data == 0 {
    fmt.Printf("the value is %v.\n", data)
}
```

이 코드의 실행 결과는 세 가지가 가능하다:
- 아무것도 출력되지 않음 (고루틴이 `if` 검사 전에 실행)
- `"the value is 0"` 출력 (고루틴이 `fmt.Printf` 후에 실행)
- `"the value is 1"` 출력 (`if` 검사 후, `fmt.Printf` 전에 고루틴 실행)

**`time.Sleep`으로 해결 시도는 틀렸다.** 확률을 높일 뿐이며, 논리적 정확성(logical correctness)을 달성하지 못한다. 항상 올바른 동기화 기법을 사용해야 한다.

레이스 컨디션은 수년간 코드에 잠재하다가 타이밍 변화(디스크 부하 증가, 동시 사용자 증가 등)에 의해 표면화되는 가장 교활한 유형의 버그다.

---

### Atomicity (원자성)

어떤 연산이 정의된 **컨텍스트(context)** 내에서 분리 불가능하고(indivisible) 중단 불가능한(uninterruptible) 것을 **원자적(atomic)**이라 한다.

핵심: **원자성은 컨텍스트에 상대적이다.**

- 프로세스 컨텍스트에서 원자적 → OS 컨텍스트에서는 아닐 수 있음
- OS 컨텍스트에서 원자적 → 하드웨어 컨텍스트에서는 아닐 수 있음

```go
i++
```

단순해 보이는 `i++`는 실제로 세 단계로 구성된다:
1. `i`의 값 읽기 (Retrieve)
2. `i`의 값 증가 (Increment)
3. `i`의 값 저장 (Store)

각 단계는 원자적이지만, **세 단계의 조합은 원자적이지 않다.** 원자적 연산을 조합한다고 해서 전체가 원자적이 되지는 않는다.

원자성이 중요한 이유: 원자적인 연산은 암묵적으로 동시성 컨텍스트에서 안전하다. 따라서 논리적으로 정확한 프로그램을 구성하는 기반이 된다.

> **역사적 사례:** 2006년 Blizzard vs MDY Industries 소송에서 문제가 된 치트 프로그램 "Glider"는 하드웨어 인터럽트를 이용해 Warden의 메모리 스캔(프로세스 컨텍스트에서 원자적) 사이에 자신을 숨겼다. 원자성 컨텍스트의 차이를 실제로 악용한 사례다.

---

### Memory Access Synchronization (메모리 접근 동기화)

데이터 레이스가 있을 때, **크리티컬 섹션(critical section)**—공유 자원에 대한 배타적 접근이 필요한 코드 구간—을 보호해야 한다.

```go
var memoryAccess sync.Mutex
var value int

go func() {
    memoryAccess.Lock()
    value++
    memoryAccess.Unlock()
}()

memoryAccess.Lock()
if value == 0 {
    fmt.Printf("the value is %v.\n", value)
} else {
    fmt.Printf("the value is %v.\n", value)
}
memoryAccess.Unlock()
```

`sync.Mutex`로 크리티컬 섹션을 보호하면 데이터 레이스는 해결된다. 하지만 **레이스 컨디션(실행 순서의 비결정성)은 여전히 존재한다.** 고루틴이 먼저 실행될지, `if-else`가 먼저 실행될지는 알 수 없다.

메모리 접근 동기화의 트레이드오프:
- **잠금 범위가 너무 넓으면** → 성능 저하, 기아 상태(starvation) 위험
- **잠금 범위가 너무 좁으면** → 공정성(fairness) 향상, 하지만 잠금/해제 오버헤드 증가
- 규칙이 아닌 **관례(convention)**에 의존하므로 팀 전체가 따르지 않으면 무용지물

---

### Deadlocks, Livelocks, and Starvation

앞의 세 가지가 **정확성(correctness)** 문제라면, 이 세 가지는 **진행성(progress)** 문제다.

#### Deadlock (교착 상태)

모든 동시성 프로세스가 서로를 기다리며 영원히 블록된 상태. Go 런타임은 모든 고루틴이 블록된 경우를 감지해 `fatal error: all goroutines are asleep - deadlock!`을 출력하지만, 부분 교착 상태는 감지하지 못한다.

```go
type value struct {
    mu    sync.Mutex
    value int
}

printSum := func(v1, v2 *value) {
    defer wg.Done()
    v1.mu.Lock()
    defer v1.mu.Unlock()
    time.Sleep(2 * time.Second) // 교착 상태 유발
    v2.mu.Lock()
    defer v2.mu.Unlock()
    fmt.Printf("sum=%v\n", v1.value+v2.value)
}

var a, b value
wg.Add(2)
go printSum(&a, &b) // a 잠금 → b 대기
go printSum(&b, &a) // b 잠금 → a 대기 → 교착!
wg.Wait()
```

**코프만 조건(Coffman Conditions):** 교착 상태가 발생하려면 아래 네 조건이 모두 충족되어야 한다. 하나라도 제거하면 교착 상태를 예방할 수 있다.

| 조건 | 설명 |
|------|------|
| 상호 배제 (Mutual Exclusion) | 한 번에 하나의 프로세스만 자원을 점유 |
| 점유 대기 (Wait For Condition) | 자원을 점유한 채로 다른 자원을 대기 |
| 비선점 (No Preemption) | 점유한 자원은 해당 프로세스만 해제 가능 |
| 순환 대기 (Circular Wait) | 프로세스 간 순환 대기 사슬 형성 |

---

#### Livelock (라이브락)

프로세스들이 활발히 연산을 수행하지만 프로그램 상태가 앞으로 나아가지 않는 상태. CPU는 바쁘지만 유용한 작업이 없다.

```
Alice is trying to scoot: left right left right left right left right left right
Alice tosses her hands up in exasperation!
Barbara is trying to scoot: left right left right left right left right left right
Barbara tosses her hands up in exasperation!
```

교착 상태를 피하려다 **조율 없이** 서로 양보하는 패턴이 전형적인 라이브락 원인이다. 라이브락은 교착 상태보다 발견하기 어렵다—CPU 사용률이 높아 시스템이 작동 중인 것처럼 보이기 때문이다.

라이브락은 기아(starvation)의 특수한 경우: 모든 프로세스가 균등하게 굶주린다.

---

#### Starvation (기아 상태)

일부 프로세스가 필요한 자원을 얻지 못해 작업을 진행하지 못하는 상태.

```go
// 탐욕적 워커: 락을 크리티컬 섹션 밖까지 유지
greedyWorker := func() {
    for begin := time.Now(); time.Since(begin) <= runtime; {
        sharedLock.Lock()
        time.Sleep(3 * time.Nanosecond)
        sharedLock.Unlock()
        count++
    }
}

// 예의 바른 워커: 필요할 때만 락
politeWorker := func() {
    for begin := time.Now(); time.Since(begin) <= runtime; {
        sharedLock.Lock(); time.Sleep(1 * time.Nanosecond); sharedLock.Unlock()
        sharedLock.Lock(); time.Sleep(1 * time.Nanosecond); sharedLock.Unlock()
        sharedLock.Lock(); time.Sleep(1 * time.Nanosecond); sharedLock.Unlock()
        count++
    }
}
```

결과: 탐욕적 워커가 예의 바른 워커보다 약 **1.6배** 더 많은 작업을 수행한다. 탐욕적 워커가 크리티컬 섹션을 불필요하게 확장해 상대방을 기아 상태로 만든다.

**기아 상태 감지:** 메트릭(metrics) 수집이 효과적이다. 작업 완료율을 기록하고 기대치와 비교하면 기아 상태를 발견할 수 있다.

**동기화 범위 설정 원칙:**
- 처음에는 크리티컬 섹션만 잠금
- 성능 문제가 생기면 범위를 넓히는 방향으로 조정
- 반대 방향(넓은 범위→좁은 범위)은 훨씬 어렵다

---

### Determining Concurrency Safety (동시성 안전성 명시)

가장 어렵고 근본적인 문제는 **사람**이다. 동시성 코드를 작성할 때 API 설계 단계에서 동시성 의도를 명확히 전달해야 한다.

```go
// 나쁜 예: 동시성 여부, 동기화 책임 불명확
func CalculatePi(begin, end int64, pi *Pi)

// 좋은 예: 명확한 주석
// CalculatePi는 begin~end 사이의 Pi 자릿수를 계산한다.
// 내부적으로 FLOOR((end-begin)/2)개의 고루틴을 생성한다.
// Pi 구조체에 대한 쓰기 동기화는 Pi 내부에서 처리한다.
func CalculatePi(begin, end int64, pi *Pi)

// 더 좋은 예: 시그니처 자체가 의도를 전달
func CalculatePi(begin, end int64) <-chan uint
```

채널(channel)을 반환 타입으로 사용하면 함수가 고루틴을 사용하고 호출자가 동기화를 걱정할 필요 없다는 신호를 준다.

동시성 API 설계 시 주석에 포함해야 할 세 가지:
1. **누가** 동시성을 책임지는가?
2. **어떻게** 문제가 동시성 프리미티브에 매핑되는가?
3. **누가** 동기화를 책임지는가?

---

## Simplicity in the Face of Complexity

Go가 동시성을 더 쉽게 만드는 이유:

**GC (Garbage Collector):** Go 1.8 이후 GC 중단 시간이 10~100마이크로초 수준으로 감소했다. 동시 프로세스 간 메모리 관리를 직접 하지 않아도 된다.

**고루틴 멀티플렉싱:** Go 런타임이 동시성 연산을 OS 스레드에 자동으로 다중화(multiplex)한다. 개발자는 스레드 풀(thread pool) 관리, 스레드 간 부하 분산을 신경 쓰지 않아도 된다.

```go
// 다른 언어: 스레드 풀 생성, 연결 매핑, 페어 스케줄링 구현 필요
// Go: 그냥 go 키워드 하나면 됨
go handleConnection(conn)
```

**채널(Channel):** 동시성 프로세스 간 통신을 위한 조합 가능하고(composable) 동시성 안전한(concurrent-safe) 프리미티브를 제공한다.

Go의 동시성 모델은 복잡성을 숨기는 게 아니라 올바른 추상화를 제공해서, 잘못 사용하기 어렵고 올바르게 사용하기 쉬운 구조를 만든다.
