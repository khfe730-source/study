# Chapter 4. Concurrency Patterns in Go

## Confinement (한정)

공유 메모리 없이 동시성 안전성을 보장하는 방법. 정보를 단 하나의 동시성 프로세스에서만 접근 가능하게 만들면 동기화가 필요 없다.

### Ad hoc Confinement vs Lexical Confinement

| | Ad hoc | Lexical |
|--|--------|---------|
| 방식 | 관례(convention)에 의존 | 렉시컬 스코프로 컴파일러가 강제 |
| 안정성 | 팀 규모가 커지면 취약 | 잘못된 접근이 컴파일 에러 |
| 권장 | 정적 분석 도구 필요 | **권장** |

**Lexical Confinement 예시:**

```go
// 채널 소유권을 렉시컬 스코프로 한정
chanOwner := func() <-chan int {
    results := make(chan int, 5)        // 쓰기 채널은 이 스코프 내에만 존재
    go func() {
        defer close(results)
        for i := 0; i <= 5; i++ {
            results <- i
        }
    }()
    return results                      // 읽기 전용으로만 노출
}

consumer := func(results <-chan int) { // 읽기만 가능, 쓰기 불가
    for result := range results {
        fmt.Printf("Received: %d\n", result)
    }
}
```

**동시성 안전하지 않은 타입도 한정으로 보호:**

```go
printData := func(wg *sync.WaitGroup, data []byte) {
    defer wg.Done()
    var buff bytes.Buffer             // bytes.Buffer는 동시성 안전하지 않음
    for _, b := range data {          // 하지만 이 고루틴만 접근하므로 안전
        fmt.Fprintf(&buff, "%c", b)
    }
    fmt.Println(buff.String())
}

data := []byte("golang")
go printData(&wg, data[:3])           // 앞 3바이트만
go printData(&wg, data[3:])           // 뒤 3바이트만 — 각자 다른 서브슬라이스
```

한정의 이점: 동기화 비용 제거, 크리티컬 섹션 제거, 인지 부하 감소.

---

## The for-select Loop

Go 프로그램 전반에 등장하는 기본 패턴. 채널 작업과 종료 신호를 결합한다.

```go
// 패턴 1: 이터레이션 값을 채널로 전송
for _, s := range []string{"a", "b", "c"} {
    select {
    case <-done:
        return
    case stringStream <- s:
    }
}

// 패턴 2a: done 확인을 select 바깥에 (작업이 선점 불가능)
for {
    select {
    case <-done:
        return
    default:
    }
    // 선점 불가능한 작업 수행
}

// 패턴 2b: done 확인을 default 안에 (스타일 선호도 차이)
for {
    select {
    case <-done:
        return
    default:
        // 작업 수행
    }
}
```

---

## Preventing Goroutine Leaks (고루틴 누수 방지)

고루틴은 GC되지 않는다. 프로세스가 살아있는 한 블록된 고루틴은 메모리를 계속 점유한다.

고루틴이 종료될 수 있는 경로:
1. 작업 완료
2. 복구 불가능한 에러
3. **종료 신호 수신** ← 반드시 명시적으로 설계해야 함

**관례: `done <-chan interface{}`를 첫 번째 파라미터로 전달**

```go
// 누수 발생: done 채널 없음
doWork := func(strings <-chan string) <-chan interface{} {
    completed := make(chan interface{})
    go func() {
        defer close(completed)
        for s := range strings { // nil 채널 → 영원히 블록
            fmt.Println(s)
        }
    }()
    return completed
}
doWork(nil) // 고루틴이 프로세스 종료까지 메모리 점유

// 올바른 방법: done 채널로 취소 신호 전달
doWork := func(
    done <-chan interface{},
    strings <-chan string,
) <-chan interface{} {
    terminated := make(chan interface{})
    go func() {
        defer fmt.Println("doWork exited.")
        defer close(terminated)
        for {
            select {
            case s := <-strings:
                fmt.Println(s)
            case <-done: // 종료 신호
                return
            }
        }
    }()
    return terminated
}

done := make(chan interface{})
terminated := doWork(done, nil)
go func() {
    time.Sleep(1 * time.Second)
    fmt.Println("Canceling doWork goroutine...")
    close(done)
}()
<-terminated // 조인 포인트
```

**쓰기 블록 고루틴의 누수 방지:**

```go
// 누수: 수신자가 없어지면 생산자가 영원히 블록
newRandStream := func(done <-chan interface{}) <-chan int {
    randStream := make(chan int)
    go func() {
        defer close(randStream)
        for {
            select {
            case randStream <- rand.Int():
            case <-done: // 수신자 없어도 종료 가능
                return
            }
        }
    }()
    return randStream
}
```

> **원칙:** 고루틴을 생성하는 책임이 있는 존재는 그 고루틴을 중단시킬 수 있어야 할 책임도 진다.

---

## The or-channel

임의 개수의 `done` 채널 중 **하나라도** 닫히면 닫히는 복합 채널을 생성하는 패턴.

```go
var or func(channels ...<-chan interface{}) <-chan interface{}
or = func(channels ...<-chan interface{}) <-chan interface{} {
    switch len(channels) {
    case 0:
        return nil
    case 1:
        return channels[0]
    }
    orDone := make(chan interface{})
    go func() {
        defer close(orDone)
        switch len(channels) {
        case 2:
            select {
            case <-channels[0]:
            case <-channels[1]:
            }
        default:
            select {
            case <-channels[0]:
            case <-channels[1]:
            case <-channels[2]:
            case <-or(append(channels[3:], orDone)...): // 재귀
            }
        }
    }()
    return orDone
}
```

**사용 예:**

```go
sig := func(after time.Duration) <-chan interface{} {
    c := make(chan interface{})
    go func() { defer close(c); time.Sleep(after) }()
    return c
}

<-or(
    sig(2*time.Hour),
    sig(5*time.Minute),
    sig(1*time.Second),   // 이것이 가장 먼저 닫힘
    sig(1*time.Hour),
)
// "done after 1.000216772s"
```

생성 고루틴 수: `⌊N/2⌋`. 런타임에 채널 수를 알 수 없을 때 유일한 조합 방법이다.

---

## Error Handling (에러 처리)

동시성 프로세스에서 에러를 어디서 처리할지가 핵심 질문이다. 고루틴 내부에서 에러를 삼키거나 출력만 하는 것은 잘못된 설계다.

**나쁜 패턴:**

```go
go func() {
    resp, err := http.Get(url)
    if err != nil {
        fmt.Println(err) // 고루틴이 할 수 있는 게 없음. 호출자가 제어권을 잃음
        continue
    }
    responses <- resp
}()
```

**올바른 패턴: 에러를 결과 타입에 포함시켜 반환**

```go
type Result struct {
    Error    error
    Response *http.Response
}

checkStatus := func(done <-chan interface{}, urls ...string) <-chan Result {
    results := make(chan Result)
    go func() {
        defer close(results)
        for _, url := range urls {
            resp, err := http.Get(url)
            result := Result{Error: err, Response: resp}
            select {
            case <-done:
                return
            case results <- result: // 에러와 응답을 함께 전달
            }
        }
    }()
    return results
}

// 호출자가 에러 정책을 결정
errCount := 0
for result := range checkStatus(done, urls...) {
    if result.Error != nil {
        errCount++
        if errCount >= 3 {
            fmt.Println("Too many errors, breaking!")
            break
        }
        continue
    }
    fmt.Printf("Response: %v\n", result.Response.Status)
}
```

**원칙:** 에러는 동기 함수와 동일하게 반환값의 일급 시민으로 처리한다. 고루틴이 에러를 생성할 수 있으면, 에러는 결과 타입에 포함되어 동일한 통신 경로(채널)를 통해 전달되어야 한다.

---

## Pipelines

파이프라인(pipeline)은 데이터를 받아 변환하고 내보내는 일련의 스테이지(stage)로 구성된 추상화다. 각 스테이지를 독립적으로 수정, 조합, 병렬화할 수 있다.

### 파이프라인 스테이지의 조건

1. 동일한 타입을 소비하고 반환한다
2. 언어에 의해 구체화(reified)되어 전달 가능하다 (Go의 함수는 일급 객체)

### 채널 기반 파이프라인

```go
// 제너레이터: 이산 값 → 채널 스트림
generator := func(done <-chan interface{}, integers ...int) <-chan int {
    intStream := make(chan int)
    go func() {
        defer close(intStream)
        for _, i := range integers {
            select {
            case <-done:
                return
            case intStream <- i:
            }
        }
    }()
    return intStream
}

// 변환 스테이지
multiply := func(done <-chan interface{}, intStream <-chan int, multiplier int) <-chan int {
    multipliedStream := make(chan int)
    go func() {
        defer close(multipliedStream)
        for i := range intStream {
            select {
            case <-done:
                return
            case multipliedStream <- i * multiplier:
            }
        }
    }()
    return multipliedStream
}

// 파이프라인 조합
done := make(chan interface{})
defer close(done)

intStream := generator(done, 1, 2, 3, 4)
pipeline := multiply(done, add(done, multiply(done, intStream, 2), 1), 2)

for v := range pipeline {
    fmt.Println(v) // 6 10 14 18
}
```

**파이프라인 취소 연쇄 방식:**
- 각 스테이지는 입력 채널을 `range`로 순회하며 `done` 채널과 `select`
- `done`이 닫히면 현재 스테이지가 종료 → 출력 채널도 닫힘 → 다음 스테이지 종료

### 유용한 제너레이터들

```go
// repeat: 값을 무한 반복 생성
repeat := func(done <-chan interface{}, values ...interface{}) <-chan interface{} {
    valueStream := make(chan interface{})
    go func() {
        defer close(valueStream)
        for {
            for _, v := range values {
                select {
                case <-done:
                    return
                case valueStream <- v:
                }
            }
        }
    }()
    return valueStream
}

// take: 처음 num개만 취함
take := func(done <-chan interface{}, valueStream <-chan interface{}, num int) <-chan interface{} {
    takeStream := make(chan interface{})
    go func() {
        defer close(takeStream)
        for i := 0; i < num; i++ {
            select {
            case <-done:
                return
            case takeStream <- <-valueStream:
            }
        }
    }()
    return takeStream
}

// repeatFn: 함수를 무한 반복 호출
repeatFn := func(done <-chan interface{}, fn func() interface{}) <-chan interface{} {
    valueStream := make(chan interface{})
    go func() {
        defer close(valueStream)
        for {
            select {
            case <-done:
                return
            case valueStream <- fn():
            }
        }
    }()
    return valueStream
}

// 조합 예: 1을 10번 출력
for num := range take(done, repeat(done, 1), 10) {
    fmt.Printf("%v ", num) // 1 1 1 1 1 1 1 1 1 1
}

// 랜덤 정수 10개 생성
rand := func() interface{} { return rand.Int() }
for num := range take(done, repeatFn(done, rand), 10) {
    fmt.Println(num)
}
```

**`interface{}` 채널 vs 타입별 채널:**
- `interface{}` 채널: 범용 스테이지 재사용 가능, 타입 단언 스테이지 추가 필요
- 타입별 채널: 약 2배 빠름, 하지만 파이프라인 병목은 대개 I/O나 연산 집약 스테이지

```go
// 타입 변환 스테이지 추가
toString := func(done <-chan interface{}, valueStream <-chan interface{}) <-chan string {
    stringStream := make(chan string)
    go func() {
        defer close(stringStream)
        for v := range valueStream {
            select {
            case <-done:
                return
            case stringStream <- v.(string):
            }
        }
    }()
    return stringStream
}
```

---

## Fan-Out, Fan-In

계산 집약적인 스테이지를 병렬화하는 패턴.

- **Fan-Out:** 하나의 입력 채널을 여러 고루틴이 동시에 처리
- **Fan-In:** 여러 결과 채널을 하나로 합침 (멀티플렉싱)

**Fan-Out 조건:**
1. 이전 계산 결과에 의존하지 않는다 (순서 독립)
2. 실행 시간이 길다

```go
// Fan-Out: CPU 수만큼 스테이지 복제
numFinders := runtime.NumCPU()
finders := make([]<-chan interface{}, numFinders)
for i := 0; i < numFinders; i++ {
    finders[i] = primeFinder(done, randIntStream) // 같은 입력 채널 공유
}

// Fan-In: 여러 채널을 하나로 합침
fanIn := func(done <-chan interface{}, channels ...<-chan interface{}) <-chan interface{} {
    var wg sync.WaitGroup
    multiplexedStream := make(chan interface{})

    multiplex := func(c <-chan interface{}) {
        defer wg.Done()
        for i := range c {
            select {
            case <-done:
                return
            case multiplexedStream <- i:
            }
        }
    }

    wg.Add(len(channels))
    for _, c := range channels {
        go multiplex(c) // 각 채널마다 고루틴 1개
    }

    go func() {
        wg.Wait()
        close(multiplexedStream) // 모든 채널이 소진되면 닫기
    }()
    return multiplexedStream
}

// 결과: 소수 탐색 23초 → 5초 (78% 단축, 8코어 기준)
for prime := range take(done, fanIn(done, finders...), 10) {
    fmt.Printf("\t%d\n", prime)
}
```

```
Fan-Out/Fan-In 구조:

입력 채널
    │
    ├──► primeFinder (고루틴 1)──┐
    ├──► primeFinder (고루틴 2)──┤
    ├──► primeFinder (고루틴 3)──┼──► fanIn ──► 출력 채널
    ├──► primeFinder (고루틴 4)──┤
    └──► ...                    ┘
```

> **주의:** Fan-Out/Fan-In은 결과 순서를 보장하지 않는다.

---

## The or-done-channel

외부 채널을 `done` 신호와 함께 읽을 때의 보일러플레이트 제거 패턴.

**문제:** `done`을 고려하면 단순한 `range`가 복잡한 `select`로 폭발한다.

```go
// 읽기 어려운 코드
loop:
for {
    select {
    case <-done:
        break loop
    case maybeVal, ok := <-myChan:
        if ok == false {
            return
        }
        // val 처리
    }
}

// orDone으로 캡슐화
orDone := func(done, c <-chan interface{}) <-chan interface{} {
    valStream := make(chan interface{})
    go func() {
        defer close(valStream)
        for {
            select {
            case <-done:
                return
            case v, ok := <-c:
                if !ok {
                    return
                }
                select {
                case valStream <- v:
                case <-done:
                }
            }
        }
    }()
    return valStream
}

// 깔끔한 사용
for val := range orDone(done, myChan) {
    // val 처리
}
```

---

## The tee-channel

Unix `tee` 명령에서 이름을 딴 패턴. 하나의 채널에서 오는 값을 두 개의 채널로 동시에 전달한다.

```go
tee := func(done <-chan interface{}, in <-chan interface{}) (_, _ <-chan interface{}) {
    out1 := make(chan interface{})
    out2 := make(chan interface{})
    go func() {
        defer close(out1)
        defer close(out2)
        for val := range orDone(done, in) {
            var out1, out2 = out1, out2 // 로컬 섀도잉
            for i := 0; i < 2; i++ {
                select {
                case <-done:
                case out1 <- val:
                    out1 = nil  // 이미 전송, 다음 select에서 블록
                case out2 <- val:
                    out2 = nil  // 이미 전송, 다음 select에서 블록
                }
            }
        }
    }()
    return out1, out2
}

out1, out2 := tee(done, take(done, repeat(done, 1, 2), 4))
for val1 := range out1 {
    fmt.Printf("out1: %v, out2: %v\n", val1, <-out2)
}
```

> `out1`과 `out2`로의 쓰기는 타이트하게 결합되어 있다. 두 채널 모두에 값이 전달되기 전까지 다음 입력을 읽지 않는다.

---

## The bridge-channel

채널의 채널(`<-chan <-chan interface{}`)을 단일 채널로 평탄화하는 패턴.

```go
bridge := func(done <-chan interface{}, chanStream <-chan <-chan interface{}) <-chan interface{} {
    valStream := make(chan interface{})
    go func() {
        defer close(valStream)
        for {
            var stream <-chan interface{}
            select {
            case maybeStream, ok := <-chanStream:
                if !ok {
                    return
                }
                stream = maybeStream
            case <-done:
                return
            }
            for val := range orDone(done, stream) { // 현재 채널 소진
                select {
                case valStream <- val:
                case <-done:
                }
            }
        }
    }()
    return valStream
}

// 사용 예: 채널 10개를 단일 스트림으로
genVals := func() <-chan <-chan interface{} {
    chanStream := make(chan (<-chan interface{}))
    go func() {
        defer close(chanStream)
        for i := 0; i < 10; i++ {
            stream := make(chan interface{}, 1)
            stream <- i
            close(stream)
            chanStream <- stream
        }
    }()
    return chanStream
}

for v := range bridge(nil, genVals()) {
    fmt.Printf("%v ", v) // 0 1 2 3 4 5 6 7 8 9
}
```

---

## Queuing (큐잉)

파이프라인에 버퍼를 추가해 스테이지를 디커플링하는 최적화 기법. **성능 최적화의 마지막 수단**이다.

### 큐잉의 효과

큐잉은 전체 런타임을 줄이지 않는다—스테이지가 블록 상태에 있는 시간을 줄일 뿐이다.

```
버퍼 없는 파이프라인 (전체 13초):
Short 스테이지가 9초 소요

버퍼 2 추가 후 (전체 여전히 13초):
Short 스테이지가 3초만 소요 (블록 시간 2/3 단축)
```

진정한 큐잉의 가치: **스테이지 간 디커플링**. 수락 스테이지(acceptConnection)가 처리 스테이지(processRequest)의 속도에 영향받지 않도록.

### 큐잉이 성능을 실제로 향상시키는 경우

1. **청킹(chunking):** 배치로 처리하면 단건 처리보다 효율적인 경우 (예: `bufio.Writer`)
   - 언버퍼드 쓰기 ~3,969 ns/op vs 버퍼드 쓰기 ~1,356 ns/op (2.9배 빠름)

2. **부정적 피드백 루프(negative feedback loop) 방지:** 처리 지연이 더 많은 입력을 유발하는 데스 스파이럴(death-spiral) 차단

### Little's Law (리틀의 법칙)

파이프라인 처리량을 예측하는 공식: **L = λW**

- `L` = 시스템 내 평균 단위 수
- `λ` = 평균 도착률 (단위/시간)
- `W` = 단위가 시스템에서 보내는 평균 시간

**응용 1: 처리량 계산**
- 3스테이지 파이프라인, 요청 1개가 1초 → 3r = λ × 1s → **λ = 3 req/s**

**응용 2: 큐 크기 결정**
- 3스테이지, 처리 1ms, 목표 100,000 req/s → L-3 = 100,000 × 0.0001 = 10 → **큐 크기 = 7**

**중요 결론:** 큐를 추가하면 L이 증가 → 결국 W(지연) 증가. 큐는 처리량이 아닌 블로킹 시간을 줄인다.

### 큐를 배치할 위치

- **파이프라인 입구**: 피드백 루프 차단
- **배치 처리로 효율이 높아지는 스테이지**: 청킹 효과

> 비용이 큰 스테이지 뒤에 큐를 추가하려는 유혹을 피하라. Little's Law가 효과 없음을 증명한다.

---

## The context Package

`done` 채널 패턴을 확장해 취소 이유, 데드라인, 요청 범위 데이터를 함께 전달하는 표준 패키지 (Go 1.7 이후 표준 라이브러리 포함).

### Context 인터페이스

```go
type Context interface {
    Deadline() (deadline time.Time, ok bool) // 데드라인 조회
    Done() <-chan struct{}                    // 취소 신호 채널
    Err() error                              // 취소 이유 (Canceled or DeadlineExceeded)
    Value(key interface{}) interface{}       // 요청 범위 데이터
}
```

### Context 생성 함수

```go
context.Background()                              // 루트 Context (빈 Context)
context.TODO()                                    // 임시 플레이스홀더

context.WithCancel(parent)                        // cancel() 호출 시 done 닫힘
context.WithDeadline(parent, time.Time)           // 지정 시각에 자동 닫힘
context.WithTimeout(parent, time.Duration)        // 지정 시간 후 자동 닫힘
context.WithValue(parent, key, val)               // 데이터 첨부
```

### 취소 전파

```go
func main() {
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()

    go func() {
        if err := printGreeting(ctx); err != nil {
            fmt.Printf("cannot print greeting: %v\n", err)
            cancel() // 에러 시 전체 컨텍스트 트리 취소
        }
    }()
    go func() {
        if err := printFarewell(ctx); err != nil {
            fmt.Printf("cannot print farewell: %v\n", err)
        }
    }()
}

func genGreeting(ctx context.Context) (string, error) {
    ctx, cancel := context.WithTimeout(ctx, 1*time.Second) // 하위에만 적용
    defer cancel()
    // ...
}

func locale(ctx context.Context) (string, error) {
    // 데드라인을 미리 체크해 불필요한 작업 방지 (fail fast)
    if deadline, ok := ctx.Deadline(); ok {
        if deadline.Sub(time.Now().Add(1*time.Minute)) <= 0 {
            return "", context.DeadlineExceeded
        }
    }
    select {
    case <-ctx.Done():
        return "", ctx.Err() // "context deadline exceeded" or "context canceled"
    case <-time.After(1 * time.Minute):
    }
    return "EN/US", nil
}
```

**결과:**
```
cannot print greeting: context deadline exceeded
cannot print farewell: context canceled
```

`genGreeting`이 자신만의 타임아웃 Context를 만들어도 부모의 Context에 영향 없음 → 조합 가능성.

### Context를 이용한 데이터 전달

```go
// 패키지 내 비공개 키 타입으로 충돌 방지
type ctxKey int
const (
    ctxUserID    ctxKey = iota
    ctxAuthToken
)

// 타입 안전한 접근자 함수
func UserID(c context.Context) string {
    return c.Value(ctxUserID).(string)
}

func ProcessRequest(userID, authToken string) {
    ctx := context.WithValue(context.Background(), ctxUserID, userID)
    ctx = context.WithValue(ctx, ctxAuthToken, authToken)
    HandleResponse(ctx)
}
```

### Context 데이터 저장 가이드라인

Context에 저장할 데이터의 5가지 기준:

| 기준 | 설명 |
|------|------|
| 프로세스/API 경계를 가로지르는 데이터 | 프로세스 내부에서만 사용하면 부적합 |
| 불변 데이터 | 변경된다면 요청 범위 데이터가 아님 |
| 단순 타입 | 복잡한 패키지 그래프 임포트 불필요 |
| 메서드 없는 데이터 | 로직은 데이터를 소비하는 쪽에 있어야 함 |
| 동작을 장식하는 데이터 | 알고리즘 분기를 좌우하면 선택적 파라미터가 되어버림 |

**Context에 적합한 데이터 예시:**

| 데이터 | 적합성 |
|--------|--------|
| Request ID | ✓ 적합 |
| User ID | ✓ 적합 |
| Authorization Token | △ 논쟁 여지 있음 |
| API Server Connection | ✗ 부적합 |

> **핵심:** 취소 기능(`WithCancel`, `WithTimeout`, `WithDeadline`)은 항상 유용하다. 데이터 저장(`WithValue`)은 팀과 함께 가이드라인을 정하고 신중하게 사용하라.

---

## 패턴 요약

```
Confinement       — 동기화 없이 안전성 확보 (컴파일러가 강제하는 렉시컬 한정 권장)
for-select loop   — 취소 가능한 반복 작업의 기본 골격
Goroutine Leak    — done 채널로 모든 고루틴에 취소 신호 전달
or-channel        — N개 done 채널을 하나로 합침 (재귀 + 고루틴)
Error Handling    — 에러를 결과 타입에 포함해 채널로 전달
Pipeline          — 제너레이터 + 변환 스테이지 체인 (채널 기반)
Fan-Out/Fan-In    — 느린 스테이지를 병렬화 (순서 무보장)
or-done           — done + 채널 읽기 보일러플레이트 캡슐화
tee-channel       — 값을 두 채널로 분기
bridge-channel    — 채널의 채널을 단일 채널로 평탄화
Queuing           — 스테이지 디커플링 (Little's Law 기반 설계)
context           — 취소 + 데드라인 + 데이터 전달의 표준 패턴
```
