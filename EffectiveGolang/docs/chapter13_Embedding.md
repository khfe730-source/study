# Chapter 13. Embedding

> 원문: [Effective Go — Embedding](https://go.dev/doc/effective_go#embedding)

---

Go는 전형적인 **타입 기반 서브클래싱(subclassing)을 제공하지 않지만**, 타입을 구조체나 인터페이스 안에 **임베딩(embedding)**해 구현 일부를 "빌려올" 수 있다.

---

## 1. 인터페이스 임베딩

인터페이스 임베딩은 매우 단순하다. `io.Reader`와 `io.Writer`:

```go
type Reader interface {
    Read(p []byte) (n int, err error)
}
type Writer interface {
    Write(p []byte) (n int, err error)
}
```

두 메서드를 명시적으로 나열해 `io.ReadWriter`를 만들 수도 있지만, **두 인터페이스를 임베딩**하는 게 더 쉽고 환기적이다.

```go
// ReadWriter는 Reader와 Writer 인터페이스를 결합한다.
type ReadWriter interface {
    Reader
    Writer
}
```

`ReadWriter`는 `Reader`가 하는 일과 `Writer`가 하는 일을 모두 할 수 있다 — 임베딩된 인터페이스들의 **합집합(union)**이다. **인터페이스 안에는 인터페이스만 임베딩**할 수 있다.

---

## 2. 구조체 임베딩

같은 아이디어가 구조체에도 적용되지만 함의가 더 크다. `bufio`는 reader와 writer를 **필드 이름 없이** 구조체에 나열해 결합한다.

```go
// ReadWriter는 Reader와 Writer에 대한 포인터를 저장한다.
// io.ReadWriter를 구현한다.
type ReadWriter struct {
    *Reader // *bufio.Reader
    *Writer // *bufio.Writer
}
```

임베딩된 요소는 구조체 포인터이므로 사용 전에 유효한 구조체를 가리키도록 초기화돼야 한다.

만약 이름을 붙여 작성했다면(`reader *Reader`), 메서드를 **승격(promote)**시키고 `io` 인터페이스를 만족하려면 **포워딩 메서드(forwarding method)**를 직접 써야 한다.

```go
// 임베딩을 안 쓰면 이런 보일러플레이트가 필요
func (rw *ReadWriter) Read(p []byte) (n int, err error) {
    return rw.reader.Read(p)
}
```

**구조체를 직접 임베딩하면 이 부기(bookkeeping)를 피한다.** 임베딩된 타입의 메서드가 공짜로 따라오므로, `bufio.ReadWriter`는 `bufio.Reader`와 `bufio.Writer`의 메서드를 가질 뿐 아니라 **세 인터페이스(`io.Reader`, `io.Writer`, `io.ReadWriter`)를 모두 만족**한다.

### 임베딩 ≠ 서브클래싱 (중요한 차이)

> 타입을 임베딩하면 그 타입의 메서드가 외부 타입의 메서드가 되지만, **호출될 때 메서드의 리시버는 외부 타입이 아니라 내부 타입(inner type)**이다.

`bufio.ReadWriter`의 `Read`가 호출되면, 위에 손으로 쓴 포워딩 메서드와 **정확히 같은 효과**다 — 리시버는 `ReadWriter` 자신이 아니라 그것의 `reader` 필드다. (서브클래싱이라면 리시버가 외부 객체일 것이다.)

---

## 3. 임베딩은 단순한 편의이기도 하다

임베딩 필드와 일반 이름 필드를 함께 쓴 예:

```go
type Job struct {
    Command string
    *log.Logger
}
```

이제 `Job`은 `*log.Logger`의 `Print`, `Printf`, `Println` 등의 메서드를 가진다. `Logger`에 필드 이름을 줄 수도 있었지만 필요 없다. 초기화 후 바로 로깅할 수 있다.

```go
job.Println("starting now...")
```

`Logger`는 `Job` 구조체의 일반 필드이므로 생성자나 복합 리터럴로 평범하게 초기화한다.

```go
func NewJob(command string, logger *log.Logger) *Job {
    return &Job{command, logger}
}
// 또는
job := &Job{command, log.New(os.Stderr, "Job: ", log.Ldate)}
```

임베딩 필드를 직접 참조해야 하면, **패키지 한정자를 무시한 타입 이름이 필드 이름** 역할을 한다. `Job` 변수 `job`의 `*log.Logger`에 접근하려면 `job.Logger`라고 쓴다 — `Logger`의 메서드를 재정의(refine)할 때 유용하다.

```go
func (job *Job) Printf(format string, args ...interface{}) {
    job.Logger.Printf("%q: %s", job.Command, fmt.Sprintf(format, args...))
}
```

---

## 4. 이름 충돌(name conflict) 규칙

임베딩은 이름 충돌 문제를 일으키지만 해결 규칙은 단순하다.

1. **바깥 필드/메서드 `X`는 더 깊이 중첩된 `X`를 가린다(hide).**
   `log.Logger`에 `Command` 필드/메서드가 있어도, `Job`의 `Command` 필드가 우선한다.

2. **같은 중첩 레벨에 같은 이름이 나타나면 대개 에러다.**
   `Job`이 `Logger`라는 다른 필드/메서드를 가지면 `log.Logger`를 임베딩하는 건 에러. 단, **중복된 이름이 타입 정의 밖에서 한 번도 언급되지 않으면 OK**다. 이 단서는 외부에서 임베딩한 타입의 변경에 대한 보호를 제공한다 — 둘 다 쓰이지 않는다면 충돌 필드가 추가돼도 문제없다.

---

## 5. 정리

| 항목 | 핵심 |
|---|---|
| 목적 | 서브클래싱 없이 구현 일부를 **빌림** (메서드 승격) |
| 인터페이스 임베딩 | 인터페이스만 가능, 메서드 **합집합** (`io.ReadWriter`) |
| 구조체 임베딩 | 필드 이름 없이 타입 나열 → 메서드·인터페이스 만족 **공짜**, 포워딩 불필요 |
| vs 서브클래싱 | 메서드 리시버는 **내부 타입**(외부 아님) |
| 직접 참조 | 타입 이름(패키지 한정자 제외)이 필드 이름 (`job.Logger`) |
| 충돌 규칙 | 얕은 쪽이 깊은 쪽을 가림; 같은 레벨 중복은 (안 쓰이면) 허용 |

**한 줄 요약:** 임베딩은 상속이 아니라 합성으로 메서드를 승격시킨다. 구조체에 타입을 이름 없이 넣으면 포워딩 메서드 없이 인터페이스를 만족하지만, 메서드의 리시버는 어디까지나 내부 타입이라는 점이 서브클래싱과 결정적으로 다르다.
