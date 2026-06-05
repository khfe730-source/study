# Chapter 5. Concurrency at Scale

## Error Propagation (에러 전파)

대규모 동시성 시스템에서 에러 전파는 흔히 이차적 관심사로 취급된다. 하지만 에러를 올바르게 설계하면 시스템의 자산이 된다.

### 잘 형성된 에러(Well-formed Error)가 담아야 할 정보

| 정보 | 내용 |
|------|------|
| **무슨 일이 발생했나** | `"disk full"`, `"socket closed"` 등 발생한 사건 |
| **언제, 어디서** | 완전한 스택 트레이스, 머신 ID, UTC 타임스탬프 |
| **사용자 친화적 메시지** | 간결하고 일시적 문제 여부를 알려주는 한 줄 메시지 |
| **더 많은 정보를 얻는 방법** | 로그 상호 참조를 위한 ID (스택 트레이스 해시 포함 가능) |

### 에러 분류

```
모든 에러
├── 버그(Bugs)             — 커스터마이즈되지 않은 "raw" 에러
│   → 예상치 못한 일이 발생했다는 일반 메시지 출력 + 버그 리포트 유도
└── 알려진 엣지 케이스    — 잘 형성된 에러
    → 커스텀 메시지를 그대로 사용자에게 표시
```

**원칙:** raw 에러를 그대로 사용자에게 전달하는 것은 항상 버그다.

### 모듈 경계에서의 에러 래핑

```go
// MyError: 잘 형성된 에러 타입
type MyError struct {
    Inner      error
    Message    string
    StackTrace string                  // debug.Stack()으로 캡처
    Misc       map[string]interface{}  // 고루틴 ID, 해시 등 추가 정보
}

func wrapError(err error, messagef string, msgArgs ...interface{}) MyError {
    return MyError{
        Inner:      err,
        Message:    fmt.Sprintf(messagef, msgArgs...),
        StackTrace: string(debug.Stack()),
        Misc:       make(map[string]interface{}),
    }
}
```

**모듈 계층 에러 처리 패턴:**

```
lowlevel.isGloballyExec()
    └─ os.Stat 에러 → LowLevelErr{wrapError(...)}으로 래핑

intermediate.runJob()
    └─ LowLevelErr 수신 → IntermediateErr{wrapError(...)}으로 재래핑
       (저수준 세부사항 추상화, 모듈 컨텍스트에 맞는 메시지로 교체)

main.main()
    └─ IntermediateErr 타입 확인
       ├─ IntermediateErr → err.Error() 그대로 사용자에게 표시
       └─ 그 외 타입 → "예상치 못한 오류, 버그 리포트 요청" + logID 표시
```

```go
func handleError(key int, err error, message string) {
    log.SetPrefix(fmt.Sprintf("[logID: %v]: ", key))
    log.Printf("%#v", err)              // 전체 에러 로깅
    fmt.Printf("[%v] %v", key, message) // 사용자에게는 요약 메시지만
}

func main() {
    err := runJob("1")
    if err != nil {
        msg := "There was an unexpected issue; please report this as a bug."
        if _, ok := err.(IntermediateErr); ok {
            msg = err.Error() // 잘 형성된 에러면 그 메시지를 사용
        }
        handleError(1, err, msg)
    }
}
```

**잘못된 에러 처리 결과:**
```
[1] There was an unexpected issue; please report this as a bug.
```
**올바른 에러 처리 결과:**
```
[1] cannot run job "1": requisite binaries not available
```

> 에러 처리의 정확성은 시스템의 창발적 속성(emergent property)이 된다. 처음부터 완벽할 필요는 없고, 타입으로 명확히 구분해 점진적으로 개선해 나간다.

---

## Timeouts and Cancellation (타임아웃과 취소)

### 타임아웃이 필요한 이유

| 이유 | 설명 |
|------|------|
| **시스템 포화(System Saturation)** | 처리 용량 초과 시, 요청을 큐에 쌓기보다 타임아웃이 나을 때 |
| **데이터 만료(Stale Data)** | 처리창이 있는 데이터: `context.WithDeadline` / `WithTimeout` 사용 |
| **교착 상태 방지** | 모든 동시성 연산에 타임아웃을 걸어 시스템이 교착 상태에서 회복 가능하게 |

> 타임아웃이 교착을 라이브락으로 바꿀 수 있음을 주의하라. 그러나 분산 시스템에서는 교착보다 라이브락이 재시도로 해결될 가능성이 더 높다.

### 취소가 발생하는 이유

- **타임아웃**: 암묵적 취소
- **사용자 개입**: 장기 실행 작업의 UI 취소
- **부모 취소**: 부모 고루틴/프로세스 종료
- **복제된 요청**: 먼저 응답한 핸들러 선택 후 나머지 취소

### 선점 가능한(Preemptable) 고루틴 설계

```go
// 문제: reallyLongCalculation이 선점 불가능
var value interface{}
select {
case <-done:
    return
case value = <-valueStream:
}
result := reallyLongCalculation(value) // 이 동안 취소 신호를 무시!
select {
case <-done:
    return
case resultStream <- result:
}

// 개선 1: 중간에 done 체크 추가
reallyLongCalculation := func(done <-chan interface{}, value interface{}) interface{} {
    intermediateResult := longCalculation(value)
    select {
    case <-done:
        return nil
    default:
    }
    return longCalculation(intermediateResult)
}

// 개선 2: 하위 함수도 선점 가능하게
reallyLongCalculation := func(done <-chan interface{}, value interface{}) interface{} {
    intermediateResult := longCalculation(done, value) // done 전달
    return longCalculation(done, intermediateResult)   // done 전달
}
```

**원칙:** 모든 비원자적 연산이 허용 기간(period) 내에 완료되도록 고루틴을 잘게 쪼갠다.

### 공유 상태 변경 시 취소 처리

```go
// 나쁜 예: 롤백 표면이 넓음
result := add(1, 2, 3)
writeTallyToState(result)   // 취소 시 1개 롤백 필요
result = add(result, 4, 5, 6)
writeTallyToState(result)   // 취소 시 2개 롤백 필요
result = add(result, 7, 8, 9)
writeTallyToState(result)

// 좋은 예: 단일 상태 변경으로 롤백 최소화
result := add(1, 2, 3, 4, 5, 6, 7, 8, 9)
writeTallyToState(result)   // 취소 시 1개만 롤백
```

### 중복 메시지 처리

취소 신호와 결과 전송이 경쟁하면 하위 소비자가 중복 메시지를 받을 수 있다.

해결 방법 (권장 순):
1. **하트비트 사용** → 부모가 취소 신호를 보내기 전에 자식이 결과를 보냈는지 확인
2. **첫 번째 또는 마지막 결과만 수락** → 멱등(idempotent) 알고리즘에 적합
3. **부모에게 전송 허가 폴링** → 가장 안전하나 복잡도 증가

---

## Heartbeats (하트비트)

동시성 프로세스가 외부에 생존을 알리는 메커니즘. 시스템 모니터링과 **결정론적 테스트** 작성에 특히 유용하다.

두 가지 종류:
1. **주기 기반 하트비트**: 외부 입력을 기다리는 장기 실행 고루틴
2. **작업 단위 시작 하트비트**: 작업 시작 직전에 펄스 전송 (테스트에 최적)

### 주기 기반 하트비트

```go
doWork := func(
    done <-chan interface{},
    pulseInterval time.Duration,
) (<-chan interface{}, <-chan time.Time) {
    heartbeat := make(chan interface{})
    results := make(chan time.Time)
    go func() {
        defer close(heartbeat)
        defer close(results)
        pulse := time.Tick(pulseInterval)
        workGen := time.Tick(2 * pulseInterval) // 작업은 펄스의 2배 간격

        sendPulse := func() {
            select {
            case heartbeat <- struct{}{}:
            default: // 아무도 듣지 않을 수 있음. 결과와 달리 펄스는 비필수적
            }
        }
        sendResult := func(r time.Time) {
            for {
                select {
                case <-done:
                    return
                case <-pulse:    // 결과 전송 대기 중에도 펄스 발송
                    sendPulse()
                case results <- r:
                    return
                }
            }
        }
        for {
            select {
            case <-done:
                return
            case <-pulse:
                sendPulse()
            case r := <-workGen:
                sendResult(r)
            }
        }
    }()
    return heartbeat, results
}

// 소비 측: timeout/2 간격으로 하트비트, timeout으로 이상 감지
const timeout = 2 * time.Second
heartbeat, results := doWork(done, timeout/2) // 펄스가 타임아웃보다 빠름
for {
    select {
    case _, ok := <-heartbeat:
        if !ok { return }
        fmt.Println("pulse")
    case r, ok := <-results:
        if !ok { return }
        fmt.Printf("results %v\n", r.Second())
    case <-time.After(timeout): // 하트비트도 결과도 없으면 비정상
        fmt.Println("worker goroutine is not healthy!")
        return
    }
}
```

**비정상 고루틴 감지 결과:**
```
pulse
pulse
worker goroutine is not healthy!   ← 교착 없이 2초 안에 감지
```

### 작업 단위 시작 하트비트 (테스트 활용)

```go
func DoWork(done <-chan interface{}, nums ...int) (<-chan interface{}, <-chan int) {
    heartbeat := make(chan interface{}, 1) // 버퍼 1: 최소 1번 펄스 보장
    intStream := make(chan int)
    go func() {
        defer close(heartbeat)
        defer close(intStream)
        time.Sleep(2 * time.Second) // 비결정론적 지연 시뮬레이션
        for _, n := range nums {
            select {               // 하트비트와 작업을 별도 select로 분리
            case heartbeat <- struct{}{}:
            default:
            }
            select {
            case <-done:
                return
            case intStream <- n:
            }
        }
    }()
    return heartbeat, intStream
}
```

**나쁜 테스트 (비결정론적):**
```go
func TestDoWork_GeneratesAllNumbers(t *testing.T) {
    _, results := DoWork(done, intSlice...)
    for i, expected := range intSlice {
        select {
        case r := <-results:
            // 검증
        case <-time.After(1 * time.Second): // ← 외부 요인에 취약
            t.Fatal("test timed out")
        }
    }
}
// 결과: --- FAIL: TestDoWork_GeneratesAllNumbers (1.00s)
//       bad_concurrent_test.go:46: test timed out
```

**좋은 테스트 (하트비트로 결정론적):**
```go
func TestDoWork_GeneratesAllNumbers(t *testing.T) {
    done := make(chan interface{})
    defer close(done)
    heartbeat, results := DoWork(done, intSlice...)
    <-heartbeat  // 고루틴이 작업을 시작할 때까지 대기 → 타임아웃 불필요
    i := 0
    for r := range results {
        if expected := intSlice[i]; r != expected {
            t.Errorf("index %v: expected %v, but received %v,", i, expected, r)
        }
        i++
    }
}
// 결과: ok command-line-arguments 2.002s (항상 통과)
```

**주기 기반 하트비트를 이용한 완전 안전 테스트:**
```go
func TestDoWork_GeneratesAllNumbers(t *testing.T) {
    const timeout = 2 * time.Second
    heartbeat, results := DoWork(done, timeout/2, intSlice...)
    <-heartbeat  // 첫 번째 하트비트 대기
    for {
        select {
        case r, ok := <-results:
            if !ok { return }
            // 검증
        case <-heartbeat:              // 하트비트로 타임아웃 리셋
        case <-time.After(timeout):    // 루프 반복이 너무 오래 걸릴 때만 실패
            t.Fatal("test timed out")
        }
    }
}
```

> **권장:** 고루틴이 시작된 이후 루프가 중단되지 않는다고 합리적으로 확신할 수 있다면, 첫 번째 하트비트만 블록하고 단순 `range`로 처리하라. 타이밍 관련 문제는 별도 테스트로 분리한다.

---

## Replicated Requests (복제된 요청)

응답 속도가 최우선일 때, 동일한 요청을 여러 핸들러에 동시에 보내고 가장 먼저 응답한 것을 사용한다.

```go
doWork := func(done <-chan interface{}, id int, wg *sync.WaitGroup, result chan<- int) {
    started := time.Now()
    defer wg.Done()
    simulatedLoadTime := time.Duration(1+rand.Intn(5)) * time.Second
    select {
    case <-done:
    case <-time.After(simulatedLoadTime):
    }
    select {
    case <-done:
    case result <- id: // 결과를 첫 번째로 전달하는 핸들러
    }
}

done := make(chan interface{})
result := make(chan int)
var wg sync.WaitGroup
wg.Add(10)
for i := 0; i < 10; i++ {
    go doWork(done, i, &wg, result) // 10개 핸들러 동시 시작
}
firstReturned := <-result // 첫 번째 결과 수신
close(done)              // 나머지 핸들러 취소
wg.Wait()
fmt.Printf("Received an answer from #%v\n", firstReturned)
```

**효과:** 핸들러 #5가 5초 걸릴 때 핸들러 #8이 1초에 응답하면 4초 절약.

**조건과 트레이드오프:**

| 항목 | 내용 |
|------|------|
| 전제 | 모든 핸들러가 동등하게 요청을 처리할 수 있어야 함 |
| 권장 조건 | 핸들러들이 다른 런타임 환경(프로세스, 머신, 데이터 스토어 경로)을 가질 때 |
| 비용 | 리소스 복제 비용 (인메모리는 저렴, 서버 복제는 고비용) |
| 부가 효과 | 자연스러운 폴트 톨러런스와 확장성 |

---

## Rate Limiting (레이트 리미팅)

단위 시간당 리소스 접근 횟수를 제한하는 기법. API 연결, 디스크 읽기/쓰기, 네트워크 패킷 등에 적용 가능.

### 레이트 리미팅이 필요한 이유

- **보안**: 무제한 접근은 브루트포스, DDoS, 로그 소진 등 공격 벡터 노출
- **안정성**: 합법적 사용자도 고부하로 시스템 성능 저하 유발 가능 → 데스 스파이럴
- **사용자 경험**: 일관된 성능 보장, 과금 실수 방지
- **예측 가능성**: 시스템이 조사된 경계 안에서 동작하도록 보장

### 토큰 버킷 알고리즘 (Token Bucket)

```
버킷 깊이 d=5, 보충률 r=0.5 토큰/초:

time │ bucket │ 요청
  0  │   5    │ ✓✓✓✓✓  (버스트: 5번 즉시 처리)
  1  │   0    │ (대기)
  2  │   1    │ ✓       (2초마다 1회)
  4  │   1    │ ✓
```

두 파라미터:
- `d` (깊이): 즉시 사용 가능한 토큰 수 = 버스트 크기
- `r` (보충률): 시간당 토큰 추가 속도 = 실질적 레이트 리미트

### golang.org/x/time/rate 패키지 활용

```go
func Per(eventCount int, duration time.Duration) rate.Limit {
    return rate.Every(duration / time.Duration(eventCount))
}

// 기본 레이트 리미터: 초당 1회
func Open() *APIConnection {
    return &APIConnection{
        rateLimiter: rate.NewLimiter(rate.Limit(1), 1),
    }
}

type APIConnection struct {
    rateLimiter *rate.Limiter
}

func (a *APIConnection) ReadFile(ctx context.Context) error {
    if err := a.rateLimiter.Wait(ctx); err != nil { // 토큰 대기
        return err
    }
    return nil
}
```

**결과:** 모든 요청이 초당 1회로 제한됨.

### 멀티 리미터 (Multi-tier Rate Limiting)

초당/분당 두 계층 제한을 조합하는 패턴:

```go
type RateLimiter interface {
    Wait(context.Context) error
    Limit() rate.Limit
}

func MultiLimiter(limiters ...RateLimiter) *multiLimiter {
    byLimit := func(i, j int) bool {
        return limiters[i].Limit() < limiters[j].Limit()
    }
    sort.Slice(limiters, byLimit) // 가장 제한적인 것부터 정렬
    return &multiLimiter{limiters: limiters}
}

func (l *multiLimiter) Wait(ctx context.Context) error {
    for _, l := range l.limiters {
        if err := l.Wait(ctx); err != nil { // 모든 리미터에 대기
            return err
        }
    }
    return nil
}

// 초당 2회 + 분당 10회 복합 제한
func Open() *APIConnection {
    secondLimit := rate.NewLimiter(Per(2, time.Second), 1)   // 초당 2회, 버스트 없음
    minuteLimit := rate.NewLimiter(Per(10, time.Minute), 10) // 분당 10회, 버스트 10
    return &APIConnection{
        rateLimiter: MultiLimiter(secondLimit, minuteLimit),
    }
}
```

**결과:**
```
22:46:10 ~ 22:46:14  → 초당 2회 속도로 처음 10건 빠르게 처리
22:46:16 ~ 22:47:10  → 분당 토큰 소진 후 6초마다 1회로 전환
```

### 다차원 레이트 리미팅

시간 외에 리소스 종류(디스크, 네트워크)도 함께 제한:

```go
func Open() *APIConnection {
    return &APIConnection{
        apiLimit: MultiLimiter(
            rate.NewLimiter(Per(2, time.Second), 2),
            rate.NewLimiter(Per(10, time.Minute), 10),
        ),
        diskLimit:    MultiLimiter(rate.NewLimiter(rate.Limit(1), 1)),
        networkLimit: MultiLimiter(rate.NewLimiter(Per(3, time.Second), 3)),
    }
}

func (a *APIConnection) ReadFile(ctx context.Context) error {
    // 파일 읽기 = API 제한 + 디스크 제한 모두 적용
    return MultiLimiter(a.apiLimit, a.diskLimit).Wait(ctx)
}

func (a *APIConnection) ResolveAddress(ctx context.Context) error {
    // DNS 조회 = API 제한 + 네트워크 제한 모두 적용
    return MultiLimiter(a.apiLimit, a.networkLimit).Wait(ctx)
}
```

```
멀티 리미터 구조:

ReadFile ──► MultiLimiter(apiLimit, diskLimit)
                 ├── apiLimit: 초당2/분당10
                 └── diskLimit: 초당1

ResolveAddress ──► MultiLimiter(apiLimit, networkLimit)
                     ├── apiLimit: 초당2/분당10
                     └── networkLimit: 초당3
```

---

## Healing Unhealthy Goroutines (비정상 고루틴 복구)

장기 실행 데몬에서 고루틴이 외부 요인(네트워크 불안정, 파일 시스템 등)으로 비정상 상태가 될 수 있다. **스튜어드(steward)**가 **워드(ward)**를 감시하고 재시작하는 패턴.

```
Steward(감시자) ──heartbeat──► Ward(피감시자)
     │                              │
     └── 타임아웃 감지 시 → close(wardDone) → 재시작
```

### 스튜어드 구현

```go
type startGoroutineFn func(
    done <-chan interface{},
    pulseInterval time.Duration,
) (heartbeat <-chan interface{})

newSteward := func(timeout time.Duration, startGoroutine startGoroutineFn) startGoroutineFn {
    return func(done <-chan interface{}, pulseInterval time.Duration) <-chan interface{} {
        heartbeat := make(chan interface{})
        go func() {
            defer close(heartbeat)
            var wardDone chan interface{}
            var wardHeartbeat <-chan interface{}

            startWard := func() {
                wardDone = make(chan interface{})
                // or(): 스튜어드가 종료되거나 스튜어드가 워드를 종료할 때 워드 중단
                wardHeartbeat = startGoroutine(or(wardDone, done), timeout/2)
            }
            startWard()
            pulse := time.Tick(pulseInterval)

        monitorLoop:
            for {
                timeoutSignal := time.After(timeout)
                for {
                    select {
                    case <-pulse:
                        select {
                        case heartbeat <- struct{}{}:
                        default:
                        }
                    case <-wardHeartbeat:   // 워드 정상: 모니터 루프 계속
                        continue monitorLoop
                    case <-timeoutSignal:   // 워드 비정상: 재시작
                        log.Println("steward: ward unhealthy; restarting")
                        close(wardDone)
                        startWard()
                        continue monitorLoop
                    case <-done:
                        return
                    }
                }
            }
        }()
        return heartbeat
    }
}
```

**핵심 설계:**
- 스튜어드 자체도 `startGoroutineFn` 시그니처를 반환 → **스튜어드도 모니터링 가능**
- 워드의 `done` 채널은 스튜어드가 종료하거나 재시작할 때 닫힘
- `or()` 패턴으로 부모(스튜어드) 종료 시 워드도 함께 종료

### 비정상 워드 복구 예시

```go
doWork := func(done <-chan interface{}, _ time.Duration) <-chan interface{} {
    log.Println("ward: Hello, I'm irresponsible!")
    go func() {
        <-done  // 하트비트를 보내지 않는 비정상 고루틴
        log.Println("ward: I am halting.")
    }()
    return nil // 하트비트 채널 없음
}

doWorkWithSteward := newSteward(4*time.Second, doWork)
done := make(chan interface{})
time.AfterFunc(9*time.Second, func() { close(done) })
for range doWorkWithSteward(done, 4*time.Second) {}
```

**출력:**
```
18:28:07 ward: Hello, I'm irresponsible!
18:28:11 steward: ward unhealthy; restarting   ← 4초 후 감지, 재시작
18:28:11 ward: Hello, I'm irresponsible!
18:28:11 ward: I am halting.
18:28:15 steward: ward unhealthy; restarting   ← 4초 후 다시 감지
18:28:15 ward: Hello, I'm irresponsible!
18:28:15 ward: I am halting.
18:28:16 main: halting steward and ward.
```

### 복잡한 워드: bridge-channel과 조합

워드가 재시작되어 새 채널을 생성할 때, **bridge-channel**로 소비자에게 끊김 없는 스트림 제공:

```go
doWorkFn := func(done <-chan interface{}, intList ...int) (startGoroutineFn, <-chan interface{}) {
    intChanStream := make(chan (<-chan interface{}))
    intStream := bridge(done, intChanStream) // 외부에 노출되는 단일 스트림

    doWork := func(done <-chan interface{}, pulseInterval time.Duration) <-chan interface{} {
        intStream := make(chan interface{}) // 각 워드 인스턴스마다 새 채널
        heartbeat := make(chan interface{})
        go func() {
            defer close(intStream)
            intChanStream <- intStream // bridge에 새 채널 등록
            pulse := time.Tick(pulseInterval)
            for _, intVal := range intList {
                if intVal < 0 {
                    log.Printf("negative value: %v\n", intVal) // 비정상 상태 시뮬레이션
                    return
                }
                // 하트비트와 값 전송 (for-select 패턴)
            }
        }()
        return heartbeat
    }
    return doWork, intStream
}

// 사용: 음수에서 충돌하는 워드를 스튜어드가 자동 재시작
doWork, intStream := doWorkFn(done, 1, 2, -1, 3, 4, 5)
doWorkWithSteward := newSteward(1*time.Millisecond, doWork)
doWorkWithSteward(done, 1*time.Hour)
for intVal := range take(done, intStream, 6) {
    fmt.Printf("Received: %v\n", intVal)
}
```

**출력:**
```
Received: 1
negative value: -1        ← 워드 충돌
Received: 2
steward: ward unhealthy; restarting
Received: 1               ← 재시작 후 처음부터 다시
negative value: -1
Received: 2
steward: ward unhealthy; restarting
Received: 1
...
```

> 재시작 시 처음부터 다시 시작되므로 중복 값이 발생할 수 있다. 상태를 클로저에서 업데이트(`intList = intList[1:]`)하거나 재시작 횟수 제한 등으로 보완 가능하다.

---

## 패턴 종합

```
에러 전파      — 모듈 경계마다 에러 타입 래핑, 버그/엣지케이스 구분
타임아웃/취소  — 모든 동시성 연산에 취소 가능성 내재화, 공유 상태 변경 최소화
하트비트       — 고루틴 생존 신호, 결정론적 테스트 구현
복제 요청      — 여러 핸들러 병렬 실행 후 첫 응답 채택 (속도 vs 리소스 트레이드오프)
레이트 리미팅  — 토큰 버킷 + MultiLimiter로 다계층·다차원 제한 구성
고루틴 힐링    — 스튜어드/워드 패턴으로 비정상 고루틴 자동 재시작
```

이 패턴들은 Chapter 4의 패턴 위에 쌓여 대규모·장기 실행 시스템의 안정성과 운영 가능성을 확보한다.
