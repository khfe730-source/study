# Chapter 3. Go's Concurrency Building Blocks

## Goroutines

모든 Go 프로그램에는 최소 하나의 고루틴이 존재한다—프로세스 시작 시 자동 생성되는 **메인 고루틴(main goroutine)**. 고루틴 시작은 `go` 키워드 하나로 충분하다.

```go
go sayHello()           // 명명 함수
go func() {             // 익명 함수 즉시 실행
    fmt.Println("hello")
}()
sayHello := func() { fmt.Println("hello") }
go sayHello()           // 함수 변수
```

### 고루틴의 정체

고루틴은 OS 스레드도 그린 스레드(green thread)도 아닌 **코루틴(coroutine)**의 특수한 형태다.

- **코루틴:** 비선점형(nonpreemptive) 동시성 서브루틴. 중단/재진입 지점을 스스로 정의
- **고루틴:** Go 런타임이 블록 시점을 관찰해 자동으로 중단/재개 → 런타임과의 협력으로 사실상 선점형처럼 동작

### M:N 스케줄러와 포크-조인 모델

```
고루틴 (M개)          OS 스레드 (N개)
   G1 ─┐
   G2 ─┤  Go 런타임  ─── T1
   G3 ─┤  스케줄러  ─── T2
   G4 ─┘
```

Go는 **포크-조인(fork-join) 모델**을 따른다.

```
메인 고루틴 ────────────────────────────────► 종료
              │ fork                 ▲
              ▼                      │ join point
          자식 고루틴 ───────────────┘
```

`go` 키워드가 포크, `sync.WaitGroup.Wait()` 등이 조인 포인트를 생성한다. 조인 포인트 없이 `time.Sleep`으로 기다리는 것은 레이스 컨디션을 만들 뿐이다.

```go
var wg sync.WaitGroup
sayHello := func() {
    defer wg.Done()
    fmt.Println("hello")
}
wg.Add(1)
go sayHello()
wg.Wait() // 조인 포인트: 메인 고루틴이 여기서 블록
```

### 클로저와 변수 캡처 주의사항

고루틴은 생성된 주소 공간(address space) 내에서 실행된다—변수 원본을 참조한다.

```go
// 위험: 루프 변수를 직접 캡처
for _, salutation := range []string{"hello", "greetings", "good day"} {
    wg.Add(1)
    go func() {
        defer wg.Done()
        fmt.Println(salutation) // 루프 종료 후 "good day"만 세 번 출력!
    }()
}

// 올바른 방법: 값 복사본을 파라미터로 전달
for _, salutation := range []string{"hello", "greetings", "good day"} {
    wg.Add(1)
    go func(salutation string) { // 복사본
        defer wg.Done()
        fmt.Println(salutation)
    }(salutation) // 현재 순회값 전달
}
```

루프 변수가 스코프를 벗어나도 Go 런타임은 해당 변수를 힙(heap)으로 이동시켜 고루틴이 계속 접근할 수 있게 한다. 하지만 루프 완료 후에는 마지막 값만 남아 있게 된다.

### 고루틴의 무게

```go
// 고루틴 크기 측정
memConsumed := func() uint64 {
    runtime.GC()
    var s runtime.MemStats
    runtime.ReadMemStats(&s)
    return s.Sys
}
// 결과: ~2.817kb per goroutine
```

| 메모리 (GB) | 가능한 고루틴 수 |
|:-----------:|:-----------:|
| 1 | ~371,800 |
| 4 | ~1,487,300 |
| 8 | ~2,974,600 |

**컨텍스트 스위치 비용 비교:**

| | 비용 |
|--|--|
| OS 스레드 컨텍스트 스위치 | ~1.467 μs |
| 고루틴 컨텍스트 스위치 | ~0.225 μs (92% 빠름) |

고루틴 생성은 저렴하다. 성능 프로파일링으로 고루틴이 병목임을 증명하기 전까지는 비용을 걱정하지 말아야 한다.

---

## The sync Package

낮은 수준의 메모리 접근 동기화 프리미티브 모음. 타이트한 스코프 또는 타입 내부에 캡슐화해서 사용하는 것이 원칙이다.

### WaitGroup

결과를 수집할 다른 수단이 없을 때, 동시 작업 집합의 완료를 기다리는 데 사용한다.

```go
var wg sync.WaitGroup
wg.Add(1)               // 고루틴 시작 전에 Add 호출 (레이스 컨디션 방지)
go func() {
    defer wg.Done()     // defer로 항상 Done 보장
    fmt.Println("1st goroutine sleeping...")
    time.Sleep(1)
}()
wg.Add(1)
go func() {
    defer wg.Done()
    fmt.Println("2nd goroutine sleeping...")
    time.Sleep(2)
}()
wg.Wait()               // 카운터가 0이 될 때까지 블록
```

> **주의:** `Add`는 반드시 고루틴 외부에서, 고루틴 시작 전에 호출해야 한다. 내부에서 호출하면 `Wait`가 블록 없이 리턴할 수 있다.

### Mutex와 RWMutex

`Mutex`(Mutual Exclusion): 크리티컬 섹션을 보호하는 가장 기본적인 동기화 프리미티브.

```go
var count int
var lock sync.Mutex

increment := func() {
    lock.Lock()
    defer lock.Unlock() // defer로 Unlock 보장 (패닉 시에도)
    count++
}
decrement := func() {
    lock.Lock()
    defer lock.Unlock()
    count--
}
```

`sync.RWMutex`: 읽기와 쓰기를 구분해 크리티컬 섹션의 단면적을 줄인다.

```go
var m sync.RWMutex

// 읽기 전용 잠금: 다수의 고루틴이 동시에 읽기 잠금 보유 가능
m.RLock()
defer m.RUnlock()

// 쓰기 잠금: 배타적
m.Lock()
defer m.Unlock()
```

**독자 수 증가에 따른 성능 비교 (실측):**

```
Readers   RWMutex       Mutex
1         38μs          15μs
128       176μs         157μs
1024      459μs         850μs     ← RWMutex 우세 시작
8192      2.1ms         4.1ms
131072    71ms          62ms      ← 역전 (경합 증가)
```

독자 수가 약 2¹³(~8192) 이상일 때 `RWMutex`가 유리하다. 논리적으로 읽기/쓰기를 구분할 수 있으면 `RWMutex`를 사용하는 것이 일반적으로 권장된다.

### Cond

이벤트 발생을 기다리거나 알리는 **랑데부 포인트(rendezvous point)**. 조건이 충족될 때까지 고루틴을 효율적으로 대기시킨다.

```go
// 나쁜 방법: CPU 낭비
for conditionTrue() == false {
    time.Sleep(1 * time.Millisecond) // 얼마나 자야 하나?
}

// 올바른 방법: Cond 사용
c := sync.NewCond(&sync.Mutex{})
c.L.Lock()
for conditionTrue() == false {
    c.Wait() // Unlock → 고루틴 중단 → (신호 수신 시) Lock → 재개
}
c.L.Unlock()
```

`Wait()` 진입 시 내부적으로 `Unlock`, 리턴 시 `Lock`을 호출한다는 것을 반드시 기억해야 한다.

**Signal vs Broadcast:**

| 메서드 | 동작 |
|--------|------|
| `Signal()` | FIFO 순서로 가장 오래 기다린 고루틴 하나에게 알림 |
| `Broadcast()` | 대기 중인 모든 고루틴에게 알림 |

`Broadcast`는 채널로 재현하기 어렵고, `Cond`가 채널보다 성능적으로 우수하다.

```go
// Broadcast 활용 예: 버튼 클릭 이벤트
type Button struct {
    Clicked *sync.Cond
}
button := Button{Clicked: sync.NewCond(&sync.Mutex{})}

subscribe := func(c *sync.Cond, fn func()) {
    var goroutineRunning sync.WaitGroup
    goroutineRunning.Add(1)
    go func() {
        goroutineRunning.Done()
        c.L.Lock()
        defer c.L.Unlock()
        c.Wait()
        fn()
    }()
    goroutineRunning.Wait()
}

subscribe(button.Clicked, func() { fmt.Println("Maximizing window.") })
subscribe(button.Clicked, func() { fmt.Println("Displaying dialog box!") })
subscribe(button.Clicked, func() { fmt.Println("Mouse clicked.") })

button.Clicked.Broadcast() // 세 핸들러 모두 동시에 실행
```

### Once

`sync.Once`는 `Do`에 전달된 함수가 **단 한 번만 호출됨을 보장**한다—여러 고루틴에서 동시에 호출해도.

```go
var count int
increment := func() { count++ }

var once sync.Once
var wg sync.WaitGroup
wg.Add(100)
for i := 0; i < 100; i++ {
    go func() {
        defer wg.Done()
        once.Do(increment) // 100번 호출되지만 increment는 1번만 실행
    }()
}
wg.Wait()
fmt.Printf("Count is %d\n", count) // Count is 1
```

**주의 1:** `Once`는 호출된 함수의 종류가 아니라 `Do`가 호출된 횟수를 카운트한다.

```go
once.Do(increment) // 실행됨
once.Do(decrement) // 실행 안 됨 (Do가 이미 한 번 호출됐으므로)
```

**주의 2:** 순환 참조 시 교착 상태 발생.

```go
var onceA, onceB sync.Once
var initB func()
initA := func() { onceB.Do(initB) }
initB = func() { onceA.Do(initA) }
onceA.Do(initA) // 교착 상태!
```

`sync.Once`는 항상 작은 렉시컬 스코프 내에 캡슐화해서 사용하라.

### Pool

**오브젝트 풀 패턴(object pool pattern)**의 동시성 안전 구현체. 생성 비용이 높은 객체(DB 연결 등)를 재사용하거나, 메모리 할당을 제한하는 데 사용한다.

```go
myPool := &sync.Pool{
    New: func() interface{} {
        fmt.Println("Creating new instance.")
        return struct{}{}
    },
}

myPool.Get()           // New 호출 → "Creating new instance."
instance := myPool.Get() // New 호출 → "Creating new instance."
myPool.Put(instance)   // 풀에 반환
myPool.Get()           // 재사용 → New 호출 안 됨
```

**성능 효과 (네트워크 서버 예시):**

| 방식 | 성능 |
|------|------|
| 매 요청마다 새 연결 | ~1,000,385,643 ns/op |
| Pool로 연결 재사용 | ~2,904,307 ns/op (**346배 빠름**) |

**Pool 사용 원칙:**
1. `New`는 스레드 안전해야 한다
2. `Get`으로 받은 객체의 상태를 가정하지 말 것 (이전 사용자가 상태를 남겼을 수 있음)
3. 사용 완료 후 반드시 `Put` 호출 (보통 `defer`로)
4. 풀의 객체들은 형태가 대략 균일해야 한다 (가변 크기 슬라이스 등에는 비효율적)

---

## Channels

CSP에서 유래한 고루틴 간 통신 프리미티브. 메모리를 직접 공유하는 대신 **값을 통신**한다.

### 선언과 생성

```go
var dataStream chan interface{}          // 선언
dataStream = make(chan interface{})      // 인스턴스화 (양방향)

var recvOnly <-chan interface{}          // 수신 전용 (read-only)
var sendOnly chan<- interface{}          // 송신 전용 (write-only)

intStream := make(chan int)             // 타입 채널
```

Go는 필요 시 양방향 채널을 단방향으로 암묵적 변환한다. 단방향 채널은 주로 함수 파라미터/반환값에서 API 의도 표현에 사용된다.

### 기본 연산

```go
stringStream := make(chan string)
go func() {
    stringStream <- "Hello channels!" // 송신: <- 오른쪽
}()
fmt.Println(<-stringStream) // 수신: <- 왼쪽
```

채널은 **블로킹(blocking)**이다:
- 빈 채널에서 읽기 → 값이 들어올 때까지 블록
- 가득 찬 채널에 쓰기 → 공간이 생길 때까지 블록

이 특성이 조인 포인트를 자연스럽게 만들어 준다.

### 채널 닫기 (close)

```go
intStream := make(chan int)
close(intStream)

integer, ok := <-intStream
// ok == false, integer == 0 (제로값)
```

닫힌 채널에서:
- 읽기는 제로값을 계속 반환한다 (무한히 읽을 수 있음 → 여러 수신자 지원)
- `(값, ok)` 두 번째 반환값이 `false`로 채널 종료를 알림

**`range`로 채널 순회:**

```go
intStream := make(chan int)
go func() {
    defer close(intStream) // 고루틴 종료 시 채널 닫기
    for i := 1; i <= 5; i++ {
        intStream <- i
    }
}()

for integer := range intStream { // 채널이 닫히면 자동으로 루프 종료
    fmt.Printf("%v ", integer)
}
// 출력: 1 2 3 4 5
```

**채널 닫기로 여러 고루틴 동시 언블록:**

```go
begin := make(chan interface{})
for i := 0; i < 5; i++ {
    go func(i int) {
        <-begin // 신호 대기
        fmt.Printf("%v has begun\n", i)
    }(i)
}
close(begin) // 모든 고루틴 동시에 언블록 (n번 쓰기보다 빠름)
```

### 버퍼 채널 (Buffered Channel)

```go
dataStream := make(chan interface{}, 4) // 용량 4
```

```
초기:         [ ][ ][ ][ ]
c <- 'A':     [A][ ][ ][ ]
c <- 'B','C','D': [A][B][C][D]  (가득 참)
c <- 'E':     블록! (공간이 생길 때까지)
<-c:          [B][C][D][E]  (A 수신, E 쓰기 언블록)
```

- **언버퍼드 채널** = 용량 0의 버퍼 채널 (`make(chan int)` == `make(chan int, 0)`)
- 버퍼 채널은 인메모리 FIFO 큐로 동작
- 수신자가 있고 버퍼가 비어 있으면 버퍼를 우회해 직접 전달

> **주의:** 버퍼 채널은 교착 상태를 덜 발생시켜 숨기는 효과가 있다. 개발 단계에서 교착 상태를 일찍 발견하는 게 낫다.

### 채널 상태별 연산 결과 (참조표)

| 연산 | 채널 상태 | 결과 |
|------|-----------|------|
| 읽기 | `nil` | 블록 |
| | 열림 + 비어있음 | 블록 |
| | 열림 + 값 있음 | 값 반환 |
| | 닫힘 | `<제로값>, false` |
| | 송신 전용 | 컴파일 에러 |
| 쓰기 | `nil` | 블록 |
| | 열림 + 가득 참 | 블록 |
| | 열림 + 공간 있음 | 값 쓰기 |
| | 닫힘 | **패닉** |
| | 수신 전용 | 컴파일 에러 |
| close | `nil` | **패닉** |
| | 열림 + 비어있음/있음 | 채널 닫힘 |
| | 이미 닫힘 | **패닉** |
| | 수신 전용 | 컴파일 에러 |

### 채널 소유권(Channel Ownership) 원칙

패닉과 교착 상태를 방지하는 핵심 설계 원칙.

**채널 소유자(owner)의 책임:**
1. 채널 인스턴스화
2. 쓰기 수행 또는 소유권을 다른 고루틴에 이전
3. 채널 닫기
4. 위 세 가지를 캡슐화해 읽기 전용 채널로 노출

```go
chanOwner := func() <-chan int {
    resultStream := make(chan int, 5) // 1. 인스턴스화
    go func() {
        defer close(resultStream)    // 3. 닫기 보장
        for i := 0; i <= 5; i++ {
            resultStream <- i        // 2. 쓰기
        }
    }()
    return resultStream              // 4. 읽기 전용으로 노출
}

resultStream := chanOwner()
for result := range resultStream {   // 소비자: 블로킹과 닫힘만 처리
    fmt.Printf("Received: %d\n", result)
}
```

이 패턴으로:
- `nil` 채널 쓰기/닫기 패닉 제거
- 닫힌 채널 쓰기 패닉 제거
- 중복 close 패닉 제거
- 타입 시스템이 잘못된 쓰기를 컴파일 타임에 방지

---

## The select Statement

채널들을 하나로 묶는 접착제. 여러 채널을 동시에 대기하고 준비된 하나를 처리하는 구조.

```go
var c1, c2 <-chan interface{}
var c3 chan<- interface{}
select {
case <-c1:
    // c1에서 수신
case <-c2:
    // c2에서 수신
case c3 <- struct{}{}:
    // c3에 송신
}
```

`switch`와의 차이점:
- case 문이 순서대로 평가되지 않는다
- 모든 채널 연산을 **동시에** 확인
- 준비된 채널이 없으면 전체 `select` 블록

### 복수 채널이 동시에 준비된 경우

```go
c1 := make(chan interface{}); close(c1)
c2 := make(chan interface{}); close(c2)

var c1Count, c2Count int
for i := 1000; i >= 0; i-- {
    select {
    case <-c1: c1Count++
    case <-c2: c2Count++
    }
}
// c1Count: ~505, c2Count: ~496
```

Go 런타임은 준비된 case들 중 **균일 의사난수(pseudo-random uniform selection)**로 하나를 선택한다. 의도를 알 수 없으므로 평균적으로 공정하게 처리한다.

### 타임아웃

```go
var c <-chan int
select {
case <-c:
    // 처리
case <-time.After(1 * time.Second):
    fmt.Println("Timed out.")
}
```

`time.After`는 지정 시간 후 현재 시각을 보내는 채널을 반환한다. `select`의 타임아웃 패턴에 자연스럽게 어울린다.

### default 절

모든 채널이 블록 상태일 때 즉시 실행할 코드를 지정. 논블로킹 채널 연산을 구현할 수 있다.

```go
select {
case <-c1:
case <-c2:
default:
    fmt.Printf("In default after %v\n", time.Since(start)) // ~1.4μs
}
```

**`for-select` 루프 패턴** (작업을 계속하면서 종료 신호 대기):

```go
done := make(chan interface{})
go func() {
    time.Sleep(5 * time.Second)
    close(done)
}()

workCounter := 0
loop:
for {
    select {
    case <-done:
        break loop
    default:
    }
    workCounter++
    time.Sleep(1 * time.Second)
}
// Achieved 5 cycles of work before signalled to stop.
```

**빈 select:** `select {}`는 영원히 블록한다.

---

## GOMAXPROCS

`runtime.GOMAXPROCS`는 **작업 큐를 호스팅하는 OS 스레드 수**를 제어한다. Go 1.5 이후 기본값은 호스트 머신의 논리 CPU 수.

```go
// Go 1.5 이전 관용구 (현재는 불필요)
runtime.GOMAXPROCS(runtime.NumCPU())
```

일반적으로 조정할 필요가 없다. 유용한 경우:
- 레이스 컨디션 테스트: `GOMAXPROCS`를 논리 CPU 수보다 높게 설정해 경쟁 상태를 더 자주 유발
- 특정 하드웨어/워크로드에서 실험적 최적화

> **주의:** 이 값을 조정하면 프로그램이 하드웨어에 더 가까워지지만, 추상화와 장기 성능 안정성을 희생한다. Go 버전, 하드웨어가 바뀔 때마다 재검증이 필요하다.

---

## 정리

| 프리미티브 | 사용 시점 |
|------------|-----------|
| `sync.WaitGroup` | 고루틴 그룹 완료 대기, 결과 수집 수단이 없을 때 |
| `sync.Mutex` | 구조체 내부 상태 보호, 크리티컬 섹션 직렬화 |
| `sync.RWMutex` | 다수 독자, 소수 작성자 패턴에서 성능 최적화 |
| `sync.Cond` | 조건 기반 대기/알림, 특히 Broadcast가 필요할 때 |
| `sync.Once` | 초기화를 정확히 한 번만 수행 보장 |
| `sync.Pool` | 생성 비용이 높은 객체 재사용, 메모리 할당 제한 |
| Channel | 고루틴 간 데이터/소유권 이전, 여러 컴포넌트 조율 |
| `select` | 여러 채널 동시 대기, 타임아웃, 논블로킹 연산 |
