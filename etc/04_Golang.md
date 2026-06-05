# Golang 핵심 정리

---

## 1. Go 언어 특징

- **컴파일 언어**: 정적 타입, 빠른 컴파일
- **가비지 컬렉션**: 메모리 자동 관리
- **고루틴(Goroutine)**: 경량 스레드, 수천~수만 개 동시 실행 가능
- **채널(Channel)**: 고루틴 간 안전한 데이터 전달
- **인터페이스**: 덕 타이핑 (implements 키워드 없음)
- **에러 처리**: 예외(Exception) 없음, 명시적 error 반환

---

## 2. 고루틴 (Goroutine)

```go
// 고루틴 시작
go func() {
    fmt.Println("concurrent task")
}()

// WaitGroup으로 완료 대기
var wg sync.WaitGroup
for i := 0; i < 10; i++ {
    wg.Add(1)
    go func(n int) {
        defer wg.Done()
        process(n)
    }(i)
}
wg.Wait()
```

**고루틴 vs OS 스레드**
- 고루틴: 수 KB 스택, Go 런타임이 M:N 스케줄링 (M 고루틴 : N OS 스레드)
- 스레드: 수 MB 스택, OS 스케줄링

**GOMAXPROCS**: 동시에 실행할 OS 스레드 수. 기본값은 CPU 코어 수.

---

## 3. 채널 (Channel)

```go
// 채널 생성
ch := make(chan int)        // 비버퍼드 채널 (동기)
bch := make(chan int, 100)  // 버퍼드 채널 (비동기, 버퍼 가득 차면 블록)

// 송수신
ch <- 42        // 전송 (수신자 없으면 블록)
val := <-ch     // 수신 (전송자 없으면 블록)

// 단방향 채널
func send(ch chan<- int) { ch <- 1 }   // 송신 전용
func recv(ch <-chan int) { <-ch }      // 수신 전용

// select로 다중 채널 처리
select {
case msg := <-ch1:
    fmt.Println(msg)
case ch2 <- data:
    fmt.Println("sent")
case <-time.After(time.Second):
    fmt.Println("timeout")
default:
    fmt.Println("non-blocking")
}

// 채널 닫기
close(ch)
val, ok := <-ch  // ok=false 이면 닫힌 채널
```

---

## 4. 동기화 (sync 패키지)

```go
// Mutex
var mu sync.Mutex
mu.Lock()
defer mu.Unlock()

// RWMutex
var rw sync.RWMutex
rw.RLock()   // 읽기 락
rw.RUnlock()
rw.Lock()    // 쓰기 락
rw.Unlock()

// Once - 한 번만 실행
var once sync.Once
once.Do(func() { initialize() })

// atomic
var counter int64
atomic.AddInt64(&counter, 1)
val := atomic.LoadInt64(&counter)

// Map (동시성 안전한 맵)
var sm sync.Map
sm.Store("key", "value")
val, ok := sm.Load("key")
sm.Range(func(k, v interface{}) bool { return true })
```

---

## 5. 메모리 관리 / GC

**Go GC 특징**
- 삼색 마킹(Tri-color Mark-and-Sweep) 알고리즘
- STW(Stop-the-World) 시간 최소화 (대부분 concurrent)
- Go 1.14+: 비선점 스케줄링 제거 → STW 크게 감소

**메모리 최적화 팁**
```go
// 슬라이스 사전 할당
s := make([]int, 0, 1000)  // len=0, cap=1000

// 구조체 재사용 (sync.Pool)
var pool = sync.Pool{
    New: func() interface{} { return &MyStruct{} },
}
obj := pool.Get().(*MyStruct)
defer pool.Put(obj)

// 탈출 분석: 힙 할당 최소화
// go build -gcflags="-m" 으로 확인
```

---

## 6. 인터페이스와 덕 타이핑

```go
// 인터페이스 정의
type Animal interface {
    Sound() string
    Move()
}

// 구현 (implements 키워드 불필요)
type Dog struct{}
func (d Dog) Sound() string { return "woof" }
func (d Dog) Move() { fmt.Println("run") }

// 빈 인터페이스
var any interface{} = "anything"

// 타입 단언
str, ok := any.(string)

// 타입 스위치
switch v := any.(type) {
case string:
    fmt.Println("string:", v)
case int:
    fmt.Println("int:", v)
}
```

---

## 7. 에러 처리

```go
// 에러 반환 패턴
func divide(a, b float64) (float64, error) {
    if b == 0 {
        return 0, fmt.Errorf("division by zero")
    }
    return a / b, nil
}

// 에러 래핑 (Go 1.13+)
return fmt.Errorf("context: %w", err)

// 에러 언래핑
var targetErr *MyError
if errors.As(err, &targetErr) { ... }
if errors.Is(err, ErrNotFound) { ... }

// 커스텀 에러
type GameError struct {
    Code    int
    Message string
}
func (e *GameError) Error() string {
    return fmt.Sprintf("[%d] %s", e.Code, e.Message)
}

// panic/recover (예외적 상황에만)
defer func() {
    if r := recover(); r != nil {
        log.Printf("recovered: %v", r)
    }
}()
```

---

## 8. context 패키지

```go
// 타임아웃
ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
defer cancel()

// 취소
ctx, cancel := context.WithCancel(context.Background())
go func() {
    select {
    case <-ctx.Done():
        return  // 취소됨
    case result := <-work():
        // 처리
    }
}()
cancel()  // 취소 신호 전송

// 값 전달 (요청 스코프 데이터만)
ctx = context.WithValue(ctx, "userID", 123)
userID := ctx.Value("userID").(int)
```

---

## 9. 성능 프로파일링

```bash
# pprof 프로파일링
go tool pprof http://localhost:6060/debug/pprof/profile  # CPU
go tool pprof http://localhost:6060/debug/pprof/heap     # 메모리

# 벤치마크
go test -bench=. -benchmem ./...

# 레이스 컨디션 감지
go test -race ./...
go run -race main.go
```

---

## 10. 게임 서버에서 Go 활용 패턴

```go
// 게임 룸 관리 (고루틴 기반)
type Room struct {
    players map[int]*Player
    msgCh   chan Message
    mu      sync.RWMutex
}

func (r *Room) Run() {
    for msg := range r.msgCh {
        r.handleMessage(msg)
    }
}

// 채널로 브로드캐스트
func (r *Room) Broadcast(msg Message) {
    r.mu.RLock()
    defer r.mu.RUnlock()
    for _, p := range r.players {
        select {
        case p.ch <- msg:
        default:
            // 버퍼 가득 찼을 때 드롭
        }
    }
}

// 타이머 틱 (게임 루프)
ticker := time.NewTicker(100 * time.Millisecond)  // 10fps
defer ticker.Stop()
for {
    select {
    case <-ticker.C:
        r.Update()
    case msg := <-r.msgCh:
        r.handleMessage(msg)
    case <-ctx.Done():
        return
    }
}
```

---

## 11. 자주 나오는 면접 질문

**Q. 고루틴 리크(leak)란?**
시작된 고루틴이 종료되지 않고 계속 메모리를 점유하는 상태. 채널이 닫히지 않거나 context 취소가 누락될 때 발생. `goleak` 패키지로 감지.

**Q. 슬라이스와 배열의 차이?**
배열은 고정 크기, 값 타입. 슬라이스는 가변 크기, 참조 타입(내부적으로 포인터+길이+용량).

**Q. defer 실행 순서?**
LIFO(Last In, First Out). 함수 반환 직전에 역순으로 실행.

**Q. Go에서 상속은?**
Go는 상속 없음. 임베딩(Embedding)으로 구성(Composition) 구현. 인터페이스로 다형성 구현.
