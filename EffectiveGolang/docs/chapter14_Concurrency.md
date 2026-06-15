# Chapter 14. Concurrency

> 원문: [Effective Go — Concurrency](https://go.dev/doc/effective_go#concurrency)

---

## 1. Share by communicating — 메모리 공유 대신 통신

많은 환경에서 동시성 프로그래밍은 **공유 변수에 대한 올바른 접근**을 구현하는 미묘함 때문에 어렵다. Go는 다른 접근을 권장한다: **공유 값을 채널(channel)로 주고받고, 실제로는 별도의 실행 스레드가 그것을 능동적으로 공유하지 않는다.** 어느 순간에든 **단 하나의 고루틴만** 값에 접근한다 → **데이터 레이스(data race)가 설계상 발생할 수 없다.**

이 사고방식을 슬로건으로 압축하면:

> **메모리를 공유함으로써 통신하지 말고, 통신함으로써 메모리를 공유하라.**
> *(Do not communicate by sharing memory; instead, share memory by communicating.)*

이 접근을 과하게 밀어붙일 수도 있다 — 예컨대 참조 카운트는 정수 변수에 뮤텍스를 두르는 게 최선일 수 있다. 그러나 **고수준 접근으로서 채널로 접근을 제어**하면 명료하고 올바른 프로그램을 쓰기 쉽다.

> 모델 직관: 단일 스레드 프로그램은 동기화 프리미티브가 필요 없다. 그런 인스턴스 둘을 통신시키고 **통신 자체가 동기화 장치**라면 다른 동기화가 여전히 필요 없다. Unix 파이프라인이 이 모델에 완벽히 들어맞는다. Go의 동시성은 Hoare의 **CSP(Communicating Sequential Processes)**에서 유래했으며, **타입 안전한 Unix 파이프의 일반화**로도 볼 수 있다.

---

## 2. Goroutines

기존 용어(스레드, 코루틴, 프로세스)가 부정확한 함의를 줘서 **고루틴(goroutine)**이라 부른다. 고루틴은 **같은 주소 공간에서 다른 고루틴과 동시에 실행되는 함수**다.

- **경량(lightweight)** — 스택 공간 할당 정도의 비용뿐.
- 스택은 작게 시작해 싸고, 필요에 따라 힙을 할당/해제하며 커진다.
- 여러 **OS 스레드에 다중화(multiplex)**되므로, 하나가 I/O 대기로 블록돼도 나머지는 계속 실행된다.

함수/메서드 호출 앞에 **`go` 키워드**를 붙이면 새 고루틴에서 실행된다. 호출이 끝나면 고루틴은 조용히 종료된다(Unix 셸의 `&`와 유사).

```go
go list.Sort() // list.Sort를 동시 실행; 기다리지 않음
```

함수 리터럴이 유용하다.

```go
func Announce(message string, delay time.Duration) {
    go func() {
        time.Sleep(delay)
        fmt.Println(message)
    }() // 괄호 주의 — 함수를 호출해야 한다
}
```

Go의 함수 리터럴은 **클로저(closure)**다 — 참조하는 변수가 활성인 동안 살아남도록 구현이 보장한다. 다만 위 예제들은 **완료를 알릴 방법이 없어** 실용적이지 않다. 그래서 채널이 필요하다.

---

## 3. Channels

채널도 맵처럼 `make`로 할당하며, 결과 값은 기반 자료구조에 대한 참조다. 선택적 정수 인자는 **버퍼 크기**를 정한다. 기본은 0 = **버퍼 없는(unbuffered)/동기(synchronous)** 채널.

```go
ci := make(chan int)            // 버퍼 없는 int 채널
cj := make(chan int, 0)         // 버퍼 없는 int 채널
cs := make(chan *os.File, 100)  // 버퍼 있는 *os.File 채널
```

버퍼 없는 채널은 **통신(값 교환)과 동기화(두 고루틴이 알려진 상태임을 보장)를 결합**한다.

```go
c := make(chan int)
go func() {
    list.Sort()
    c <- 1 // 신호 전송; 값은 중요치 않음
}()
doSomethingForAWhile()
<-c // 정렬 완료 대기; 받은 값은 버림
```

블로킹 규칙:

- **수신자는 받을 데이터가 있을 때까지 항상 블록**된다.
- 버퍼 없는 채널: **송신자는 수신자가 받을 때까지 블록**.
- 버퍼 있는 채널: 송신자는 값이 버퍼에 복사될 때까지만 블록(버퍼가 차 있으면 누군가 받을 때까지 대기).

### 세마포어로서의 버퍼 채널

버퍼 채널을 **세마포어(semaphore)**처럼 써서 처리량을 제한할 수 있다. 채널 버퍼 용량이 `process` 동시 호출 수를 제한한다.

```go
var sem = make(chan int, MaxOutstanding)

func handle(r *Request) {
    sem <- 1   // 활성 큐가 빌 때까지 대기
    process(r) // 오래 걸릴 수 있음
    <-sem      // 완료; 다음 요청 허용
}

func Serve(queue chan *Request) {
    for {
        req := <-queue
        go handle(req) // handle 완료를 기다리지 않음
    }
}
```

문제: `Serve`가 요청마다 새 고루틴을 만들어, 동시에 `MaxOutstanding`개만 돌 수 있는데도 요청이 너무 빨리 오면 자원을 무한 소비한다. **고루틴 생성을 게이팅(gating)**해 해결:

```go
func Serve(queue chan *Request) {
    for req := range queue {
        sem <- 1
        go func() {
            process(req)
            <-sem
        }()
    }
}
```

> Go 1.22 이전에는 루프 변수가 모든 고루틴에 공유되어 버그가 있었다(1.22부터 반복마다 새 변수).

자원을 잘 관리하는 또 다른 방법: **고정된 수의 `handle` 고루틴**이 모두 요청 채널에서 읽게 한다. 고루틴 수가 동시 `process` 호출 수를 제한한다.

```go
func handle(queue chan *Request) {
    for r := range queue {
        process(r)
    }
}

func Serve(clientRequests chan *Request, quit chan bool) {
    for i := 0; i < MaxOutstanding; i++ {
        go handle(clientRequests)
    }
    <-quit // 종료 신호 대기
}
```

---

## 4. Channels of channels — 채널의 채널

Go의 중요한 성질 하나는 **채널이 일급 값(first-class value)**이라 다른 값처럼 할당·전달된다는 것이다. 흔한 용도는 **안전한 병렬 디멀티플렉싱(demultiplexing)**이다. 요청 객체 안에 **응답용 채널**을 넣으면 각 클라이언트가 자기만의 응답 경로를 제공할 수 있다.

```go
type Request struct {
    args       []int
    f          func([]int) int
    resultChan chan int
}

func sum(a []int) (s int) {
    for _, v := range a {
        s += v
    }
    return
}

request := &Request{[]int{3, 4, 5}, sum, make(chan int)}
clientRequests <- request                       // 요청 전송
fmt.Printf("answer: %d\n", <-request.resultChan) // 응답 대기
```

서버 측은 핸들러만 바뀐다.

```go
func handle(queue chan *Request) {
    for req := range queue {
        req.resultChan <- req.f(req.args)
    }
}
```

> 이것은 **속도 제한·병렬·논블로킹 RPC 시스템의 골격**이다 — 그리고 **뮤텍스가 하나도 없다.**

---

## 5. Parallelization — 다중 코어 병렬화

계산을 독립적으로 실행 가능한 조각으로 나눌 수 있으면, 각 조각 완료를 알리는 채널과 함께 **여러 CPU 코어에 병렬화**할 수 있다.

```go
type Vector []float64

// v[i] .. v[n-1]에 연산 적용
func (v Vector) DoSome(i, n int, u Vector, c chan int) {
    for ; i < n; i++ {
        v[i] += u.Op(v[i])
    }
    c <- 1 // 이 조각 완료 신호
}

const numCPU = 4

func (v Vector) DoAll(u Vector) {
    c := make(chan int, numCPU) // 버퍼는 선택이지만 합리적
    for i := 0; i < numCPU; i++ {
        go v.DoSome(i*len(v)/numCPU, (i+1)*len(v)/numCPU, u, c)
    }
    // 채널을 비우며 완료 카운트
    for i := 0; i < numCPU; i++ {
        <-c
    }
    // 전부 완료
}
```

`numCPU`를 상수로 두는 대신 런타임에 물어볼 수 있다.

```go
var numCPU = runtime.NumCPU()        // 하드웨어 코어 수
var numCPU = runtime.GOMAXPROCS(0)   // 사용자가 지정한 동시 실행 코어 수(0은 조회만)
```

> **동시성(concurrency) ≠ 병렬성(parallelism)을 혼동하지 말 것.**
> - **동시성** = 프로그램을 독립적으로 실행되는 컴포넌트로 **구조화**하는 것.
> - **병렬성** = 효율을 위해 여러 CPU에서 실제로 **동시에 계산**하는 것.
>
> Go의 동시성 기능이 일부 문제를 병렬 계산으로 구조화하기 쉽게 하지만, **Go는 동시성 언어이지 병렬 언어가 아니며**, 모든 병렬화 문제가 Go 모델에 맞는 것은 아니다.

---

## 6. A leaky buffer — 누수 버퍼 (free list)

동시성 도구는 비동시성 아이디어도 표현하기 쉽게 만든다. RPC 패키지에서 추상한 예 — 버퍼 할당/해제를 피하려 **버퍼 채널로 free list**를 표현한다.

```go
var freeList = make(chan *Buffer, 100)
var serverChan = make(chan *Buffer)

func client() {
    for {
        var b *Buffer
        select {
        case b = <-freeList:
            // 재고 있음; 끝
        default:
            // 없으면 새로 할당
            b = new(Buffer)
        }
        load(b)         // 네트워크에서 다음 메시지 읽기
        serverChan <- b // 서버로 전송
    }
}

func server() {
    for {
        b := <-serverChan // 작업 대기
        process(b)
        select {
        case freeList <- b:
            // free list에 반환; 끝
        default:
            // free list 가득; 그냥 진행 (버퍼는 GC가 회수)
        }
    }
}
```

`select`의 **`default` 절은 다른 케이스가 준비되지 않았을 때 실행**되므로, 이 `select`들은 **절대 블록하지 않는다(non-blocking)**. 버퍼 채널과 가비지 컬렉터에 부기를 맡겨 **몇 줄로 leaky bucket free list**를 구현했다.

---

## 7. 정리

| 항목 | 핵심 |
|---|---|
| 철학 | 메모리 공유로 통신하지 말고 **통신으로 메모리 공유** (CSP) |
| 고루틴 | 경량, OS 스레드에 다중화, `go f()`, 함수 리터럴은 클로저 |
| 채널 | `make(chan T, n)`, unbuffered=동기, 수신자는 항상 블록 |
| 세마포어 | 버퍼 채널 용량으로 동시성 제한, 고루틴 게이팅/고정 워커 풀 |
| 채널의 채널 | 일급 값 → 응답 채널로 뮤텍스 없는 RPC |
| 병렬화 | 조각마다 고루틴 + 완료 채널, `runtime.GOMAXPROCS(0)` |
| 동시성 vs 병렬성 | 구조화 vs 동시 실행 — **Go는 동시성 언어** |
| select | `default`로 논블로킹, leaky free list |

**한 줄 요약:** Go 동시성은 채널로 통신하며 메모리를 공유해 데이터 레이스를 설계상 제거한다. 고루틴은 경량이고, 버퍼 채널로 세마포어·워커 풀·논블로킹 free list를 만들며, 동시성(구조)과 병렬성(실행)을 구분해 이해해야 한다.
