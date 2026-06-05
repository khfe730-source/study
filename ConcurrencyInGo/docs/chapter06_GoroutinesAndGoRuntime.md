# Chapter 6. Goroutines and the Go Runtime

## Work Stealing

Go 런타임이 고루틴을 OS 스레드에 다중화하는 방식: **Work Stealing 전략**.

### 단순 분배 전략의 문제

**공평 스케줄링(Fair Scheduling):** n개 프로세서에 x개 태스크를 x/n씩 균등 분배.

```
문제 1: 태스크 시간 불균등

Time │  P1   │  P2
─────┼───────┼──────
 T1  │  T1   │  T2    ← T2가 T1+T3보다 오래 걸림
  +a │  T3   │  T2
  +b │ (idle)│  T4    ← P1 유휴 낭비

문제 2: 태스크 간 의존성

Time    │   P1       │  P2
────────┼────────────┼──────────
  T1    │  T1        │  T2
 +a     │ (blocked)  │  T2     ← T1이 T4 결과에 의존
 +b     │ (blocked)  │  T4     ← P1은 T4를 실행할 수 있었음!
 +c     │  T1        │ (idle)
 +d     │  T3        │ (idle)
```

**중앙 집중식 큐(FIFO Queue):** 프로세서가 큐에서 태스크를 가져가는 방식. 유휴 문제는 해결하지만 **중앙 자료구조에 대한 경합**과 **캐시 지역성 악화** 문제가 남는다.

### 분산 큐와 Work Stealing

각 스레드가 자신만의 **양쪽 끝 큐(deque, double-ended queue)**를 보유한다.

**Work Stealing 알고리즘 규칙:**

```
실행 스레드 기준:
1. 포크 지점 → 새 태스크를 자신의 deque 꼬리(tail)에 추가
2. 스레드가 유휴 → 다른 임의 스레드의 deque 머리(head)에서 태스크 도둑질
3. 미실현 조인 지점(join point) → 자신의 deque 꼬리에서 태스크 꺼내기
4. 자신의 deque가 비었으면:
   a. 조인 지점에서 대기(stall), 또는
   b. 다른 임의 스레드의 deque 머리에서 도둑질
```

```
분산 deque 구조:

T1  [tail← ... →head]    T2  [tail← ... →head]
      ↑ push/pop              ↑ steal
    (자신의 작업)            (남의 작업)
```

**꼬리에서 push/pop하는 이유:**
- 꼬리의 태스크는 부모 조인을 완료하는 데 가장 필요한 작업일 가능성이 높다
- 꼬리의 태스크는 현재 프로세서 캐시에 아직 남아 있을 가능성이 높다 → **캐시 미스 감소**

---

## Stealing Tasks or Continuations? (태스크 vs 연속 도둑질)

포크-조인 모델에서 무엇을 큐에 올리고 도둑질하는가: **태스크(goroutine)** vs **연속(continuation)**.

```go
fib = func(n int) <-chan int {
    result := make(chan int)
    go func() { // ← 이 goroutine이 태스크(task)
        // ...
        result <- <-fib(n-1) + <-fib(n-2)
    }()
    return result // ← goroutine 호출 이후의 모든 것이 연속(continuation)
}
fmt.Printf("fib(4) = %d", <-fib(4))
```

### 태스크 도둑질 vs 연속 도둑질 비교

`fib(4)` 실행 결과 (2 스레드, 2 프로세서):

| 통계 | 연속 도둑질 | 태스크 도둑질 |
|------|:-----------:|:------------:|
| 실행 스텝 수 | 14 | 15 |
| 최대 deque 길이 | 2 | 2 |
| 스탈링 조인 수 | 2 (유휴 스레드에서) | 3 (바쁜 스레드에서) |
| 콜 스택 크기 | 2 | 3 |

**연속 도둑질이 우수한 이유:**

연속을 deque 꼬리에 넣으면, 다른 스레드가 머리에서 훔쳐가기 어렵다. 따라서 현재 스레드는 goroutine 실행 후 연속을 그냥 꺼내 이어서 실행할 가능성이 높다 → **함수 호출처럼 동작**, 스탈링 없음.

```
연속 도둑질의 단일 스레드 실행:

T1 call stack │ T1 work deque
──────────────┼──────────────
main          │
fib(4)        │ cont. of main
fib(3)        │ cont. of main, cont. of fib(4)
fib(2)        │ cont. of main, cont. of fib(4), cont. of fib(3)
(returns 1)   │ cont. of main, cont. of fib(4)
fib(1)        │ cont. of main, cont. of fib(4)
(returns 1)   │ cont. of main, cont. of fib(4)
(returns 2)   │ cont. of main
fib(2)        │ cont. of main
(return 1)    │ cont. of main
(return 3)    │ cont. of main
main(print 3) │

→ 단일 스레드에서도 일반 함수 호출과 동일한 동작!
```

**연속 도둑질 vs 태스크(child) 도둑질 요약:**

| 항목 | 연속 도둑질 | 태스크 도둑질 |
|------|:-----------:|:------------:|
| deque 크기 | 제한됨(Bounded) | 무제한(Unbounded) |
| 실행 순서 | 직렬(Serial) | 비순서(Out of Order) |
| 조인 지점 | 비스탈링(Nonstalling) | 스탈링(Stalling) |

연속 도둑질은 컴파일러 지원이 필요하다. Go는 자체 컴파일러를 보유하고 있으므로 연속 도둑질을 구현한다. 컴파일러 없이 라이브러리로만 구현하는 언어들은 주로 태스크(child) 도둑질을 사용한다.

---

## Go 스케줄러 내부 구조

Go 스케줄러의 세 가지 핵심 개념 (소스 코드 표기):

| 기호 | 전체 이름 | 역할 |
|------|-----------|------|
| **G** | Goroutine | 고루틴. PC(프로그램 카운터) 포함 → 연속 표현 가능 |
| **M** | Machine (OS Thread) | OS 스레드 |
| **P** | Processor (Context) | 논리 프로세서. work deque에 해당. `GOMAXPROCS`가 수 결정 |

```
G (수천~수백만)
  │ 스케줄링
  ▼
P (GOMAXPROCS 수, 기본 = 논리 CPU 수)
  │ 호스팅
  ▼
M (P 수 이상 존재. 스레드 풀 보유)
  │
  ▼
OS / CPU
```

**핵심 보장:** 런타임은 항상 모든 P를 호스팅하기에 충분한 M을 유지한다.

### I/O 블록 시 컨텍스트 핸드오프

```
정상 상태:            I/O 블록 발생:

M1 ── P1 ── G(running)    M1 ── G(blocked)   ← M1은 G와 함께 블록
                           │
                           P1 ─→ M2          ← P는 새 M으로 이동
                                 └── G2       ← P는 계속 다른 G 실행
```

블록이 해제되면:
1. M1이 다른 P를 훔쳐오려 시도 → 성공 시 G 재개
2. 실패 시 G를 **전역 컨텍스트(global context)**에 넣고, M1은 스레드 풀로 복귀

전역 컨텍스트의 G가 방치되지 않도록:
- P들이 주기적으로 전역 컨텍스트 확인
- P의 로컬 deque가 비었을 때 전역 컨텍스트를 먼저 확인 (다른 P의 deque 훔치기 전에)

### 선점 (Preemption)

Go 런타임은 함수 호출 지점에서 고루틴을 선점할 수 있다. 이는 세밀한 동시성 태스크를 효율적으로 스케줄하기 위한 것이다.

**예외 (현재 해결 중인 문제):**
I/O, 시스템 콜, 함수 호출이 없는 고루틴(예: 순수 계산 루프)은 선점 불가능. 이런 경우 GC 지연이나 교착 상태를 유발할 수 있다. 실제로는 매우 드문 경우다.

---

## Presenting All of This to the Developer

```go
go someFunction() // 이게 전부다
```

개발자는 `go` 키워드 하나로 위의 모든 복잡성(work stealing, continuation 도둑질, M:N 스케줄링, I/O 블록 핸드오프)을 투명하게 활용한다. 새로운 자료구조나 스케줄링 알고리즘을 이해할 필요 없이, 기존의 함수 추상화 위에서 동시성을 표현한다.

**확장성 + 효율 + 단순함** — 이것이 고루틴의 본질이다.

---

## Appendix: 도구와 디버깅

### 고루틴 에러 해부 (Anatomy of a Goroutine Error)

Go 1.6 이후: 패닉 시 **패닉을 일으킨 고루틴의 스택 트레이스만** 출력 (이전에는 모든 고루틴 출력).

```go
func main() {
    waitForever := make(chan interface{})
    go func() {
        panic("test panic")
    }()
    <-waitForever
}
```

**출력:**
```
panic: test panic

goroutine 4 [running]:
main.main.func1()           ← 패닉 발생 위치 (익명 함수는 자동 ID 부여)
    /tmp/.../go-src.go:6 +0x65
created by main.main        ← 고루틴이 시작된 위치
    /tmp/.../go-src.go:7 +0x4e
exit status 2
```

모든 고루틴 스택 트레이스를 보려면:
```bash
GOTRACEBACK=all go run main.go
```

---

### Race Detection (레이스 감지)

Go 1.1에서 추가된 `-race` 플래그. 대부분의 go 커맨드에 사용 가능:

```bash
go test -race mypkg       # 패키지 테스트
go run -race mysrc.go     # 컴파일 후 실행
go build -race mycmd      # 빌드
go install -race mypkg    # 설치
```

**주의:** 레이스 감지기는 실제로 **실행된 코드 경로**에서만 레이스를 감지한다. Go 팀의 권장사항: `-race`로 빌드된 바이너리를 **실제 운영 부하 하에서** 지속적으로 실행해야 한다.

**레이스 감지 출력 예시:**

```
==================
WARNING: DATA RACE
Write by goroutine 6:         ← 비동기화 쓰기 시도
  main.main.func1()
      /tmp/.../go-src.go:6 +0x44
Previous read by main goroutine:  ← 동일 메모리를 읽으려 한 고루틴
  main.main()
      /tmp/.../go-src.go:7 +0x8e
==================
Found 1 data race(s)
exit status 66
```

**환경 변수 옵션:**

| 변수 | 기본값 | 설명 |
|------|--------|------|
| `LOG_PATH` | `stderr` | 레포트 출력 경로 (`stdout`, `stderr`, 파일 경로) |
| `STRIP_PATH_PREFIX` | - | 파일 경로 앞부분 제거 (출력 간결화) |
| `HISTORY_SIZE` | 기본값 | 고루틴당 이전 메모리 접근 기억 수. 범위 `[0, 7]`, 32KB~4MB |

> **"failed to restore the stack"** 메시지가 보이면 `HISTORY_SIZE`를 높여라 (메모리 사용량 증가 감수).

**CI 통합 권장:** 레이스 감지를 CI 파이프라인에 포함하고, 실제 운영 시나리오에서 지속 실행.

---

### pprof

Go 표준 라이브러리의 런타임 프로파일러. 실행 중이거나 저장된 런타임 통계를 분석한다.

**내장 프로파일 종류:**

| 프로파일 | 내용 |
|----------|------|
| `goroutine` | 현재 실행 중인 모든 고루틴의 스택 트레이스 |
| `heap` | 힙 할당 샘플링 |
| `threadcreate` | 새 OS 스레드 생성을 유발한 스택 트레이스 |
| `block` | 동기화 프리미티브에서 블록된 스택 트레이스 |
| `mutex` | 경합 중인 뮤텍스 보유자의 스택 트레이스 |

**고루틴 누수 감지 예시:**

```go
go func() {
    goroutines := pprof.Lookup("goroutine")
    for range time.Tick(1 * time.Second) {
        log.Printf("goroutine count: %d\n", goroutines.Count())
    }
}()

// 영원히 블록되는 고루틴들 생성
var blockForever chan struct{}
for i := 0; i < 10; i++ {
    go func() { <-blockForever }()
    time.Sleep(500 * time.Millisecond)
}
// goroutine count: 1 → 2 → 3 → ... 로 증가하면 누수
```

**커스텀 프로파일:**

```go
func newProfIfNotDef(name string) *pprof.Profile {
    prof := pprof.Lookup(name)
    if prof == nil {
        prof = pprof.NewProfile(name)
    }
    return prof
}
prof := newProfIfNotDef("my_package_namespace")
```

---

## 전체 아키텍처 요약

```
개발자
  │ go keyword
  ▼
Goroutine (G)
  │ 스케줄링 (continuation stealing)
  ▼
Processor / Context (P) — GOMAXPROCS 개
  │ 호스팅 + deque 관리
  ▼
OS Thread / Machine (M) — P 수 이상 존재
  │ I/O 블록 시 P를 다른 M으로 핸드오프
  ▼
OS / CPU
```

Go의 런타임은 이 모든 계층을 투명하게 관리한다. 개발자는 `go` 키워드 하나로 이 전체 메커니즘을 활용하며, 스레드 풀, 캐시 지역성, 스케줄링 알고리즘을 직접 다룰 필요가 없다.
