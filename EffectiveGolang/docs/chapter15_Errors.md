# Chapter 15. Errors

> 원문: [Effective Go — Errors](https://go.dev/doc/effective_go#errors)

---

## 1. error 인터페이스

라이브러리 루틴은 호출자에게 에러 표시를 반환해야 할 때가 많다. Go의 **다중 반환값**은 정상 반환값과 함께 **상세한 에러 정보**를 반환하기 쉽게 한다 — 이를 활용해 자세한 에러 정보를 제공하는 것이 좋은 스타일이다. 예를 들어 `os.Open`은 실패 시 `nil` 포인터만이 아니라 **무엇이 잘못됐는지 기술하는 error 값**도 반환한다.

관례상 에러는 빌트인 인터페이스 `error` 타입이다.

```go
type error interface {
    Error() string
}
```

라이브러리 작성자는 이 인터페이스를 **더 풍부한 모델로 구현**해 에러뿐 아니라 맥락(context)까지 제공할 수 있다. `os.Open`은 성공 시 에러가 `nil`이지만, 문제 시 `os.PathError`를 담는다.

```go
// PathError는 에러와, 그것을 일으킨 연산·파일 경로를 기록한다.
type PathError struct {
    Op   string // "open", "unlink" 등
    Path string // 관련 파일
    Err  error  // 시스템 콜이 반환한 에러
}

func (e *PathError) Error() string {
    return e.Op + " " + e.Path + ": " + e.Err.Error()
}
```

`PathError`의 `Error`는 다음 같은 문자열을 만든다.

```
open /etc/passwx: no such file or directory
```

문제 파일명·연산·OS 에러를 모두 포함해, 호출 지점에서 멀리 떨어져 출력돼도 유용하다 — 단순한 `"no such file or directory"`보다 훨씬 정보가 많다.

> 가능하면 에러 문자열은 **출처를 식별**해야 한다 — 연산이나 패키지를 가리키는 접두사를 두는 식. 예: `image` 패키지의 알 수 없는 포맷 에러는 `"image: unknown format"`.

### 구체 에러 검사 — 타입 단언/스위치

정확한 에러 세부에 관심 있는 호출자는 **타입 스위치나 타입 단언**으로 특정 에러를 찾아 세부를 추출한다.

```go
for try := 0; try < 2; try++ {
    file, err = os.Create(filename)
    if err == nil {
        return
    }
    if e, ok := err.(*os.PathError); ok && e.Err == syscall.ENOSPC {
        deleteTempFiles() // 공간 확보
        continue
    }
    return
}
```

두 번째 `if`는 타입 단언이다. 실패하면 `ok`는 `false`, `e`는 `nil`. 성공하면 `ok`는 `true`이고 에러가 `*os.PathError` 타입이며, `e`를 들여다봐 더 많은 정보를 얻는다.

---

## 2. Panic

호출자에게 에러를 알리는 통상적 방법은 **error를 추가 반환값으로 반환**하는 것이다(`Read`가 대표 예). 하지만 **에러가 복구 불가능(unrecoverable)**하다면? 때로 프로그램은 그냥 계속할 수 없다.

이를 위해 빌트인 함수 **`panic`**이 있다 — 프로그램을 멈추는 런타임 에러를 만든다(다음 절 참고). 임의 타입 인자(주로 문자열) 하나를 받아 죽으면서 출력한다. **"불가능한 일이 일어났다"**를 나타내는 방법이기도 하다.

```go
// 뉴턴 방법으로 세제곱근을 구하는 장난감 구현
func CubeRoot(x float64) float64 {
    z := x / 3 // 임의 초기값
    for i := 0; i < 1e6; i++ {
        prevz := z
        z -= (z*z*z - x) / (3 * z * z)
        if veryClose(z, prevz) {
            return z
        }
    }
    // 백만 번 반복해도 수렴 못함; 뭔가 잘못됨
    panic(fmt.Sprintf("CubeRoot(%g) did not converge", x))
}
```

> **실제 라이브러리 함수는 `panic`을 피해야 한다.** 문제를 가리거나 우회할 수 있으면 전체 프로그램을 죽이는 것보다 계속 실행하는 게 낫다. 반례 가능성 하나는 **초기화 중** — 라이브러리가 정말로 자신을 설정할 수 없으면 panic이 합당할 수 있다.
> ```go
> var user = os.Getenv("USER")
> func init() {
>     if user == "" {
>         panic("no value for $USER")
>     }
> }
> ```

---

## 3. Recover

`panic`이 호출되면(슬라이스 범위 초과 인덱싱, 타입 단언 실패 같은 런타임 에러로 암묵적 호출 포함), **즉시 현재 함수 실행을 멈추고 고루틴의 스택을 되감기(unwind)** 시작하며 그 과정에서 **지연된 함수(deferred function)들을 실행**한다. 되감기가 고루틴 스택 꼭대기에 도달하면 프로그램이 죽는다. 그러나 빌트인 함수 **`recover`**로 고루틴 제어를 되찾아 정상 실행을 재개할 수 있다.

`recover` 호출은 **되감기를 멈추고 `panic`에 전달된 인자를 반환**한다. 되감기 중 실행되는 코드는 지연 함수 안뿐이므로, **`recover`는 지연 함수 안에서만 유용**하다.

`recover`의 한 응용 — 서버에서 **실패한 고루틴만 종료하고 다른 고루틴은 죽이지 않기**:

```go
func server(workChan <-chan *Work) {
    for work := range workChan {
        go safelyDo(work)
    }
}

func safelyDo(work *Work) {
    defer func() {
        if err := recover(); err != nil {
            log.Println("work failed:", err)
        }
    }()
    do(work)
}
```

`do(work)`가 panic하면 결과가 로깅되고 고루틴은 다른 것들을 방해하지 않고 깔끔히 종료한다. `recover` 호출이 조건을 완전히 처리한다.

> `recover`는 **지연 함수에서 직접 호출되지 않으면 항상 `nil`을 반환**하므로, 지연 코드는 자체적으로 panic/recover를 쓰는 라이브러리 루틴을 실패 없이 호출할 수 있다.

### 복잡한 에러 처리 단순화 — regexp 예제

복구 패턴이 있으면 `do`(와 그것이 부르는 모든 것)는 `panic`으로 어떤 나쁜 상황에서도 깔끔히 빠져나올 수 있다. 파싱 에러를 **지역 에러 타입으로 panic**해 보고하는 이상화된 `regexp` 패키지:

```go
// Error는 파스 에러 타입; error 인터페이스를 만족한다.
type Error string
func (e Error) Error() string {
    return string(e)
}

// error는 *Regexp의 메서드로, Error를 panic해 파싱 에러를 보고한다.
func (regexp *Regexp) error(err string) {
    panic(Error(err))
}

func Compile(str string) (regexp *Regexp, err error) {
    regexp = new(Regexp)
    // doParse는 파스 에러 시 panic한다.
    defer func() {
        if e := recover(); e != nil {
            regexp = nil       // 반환값 클리어
            err = e.(Error)    // 파스 에러가 아니면 re-panic
        }
    }()
    return regexp.doParse(str), nil
}
```

`doParse`가 panic하면 복구 블록이 반환값을 `nil`로 설정한다 — **지연 함수는 명명된 반환값을 수정**할 수 있다. 그 다음 `err` 할당에서 `e.(Error)` 단언으로 **문제가 파스 에러였는지 확인**한다. 아니면 타입 단언이 실패해 런타임 에러가 발생하고, 아무 일도 없던 것처럼 스택 되감기가 계속된다 — 즉 **예상치 못한 일(범위 초과 등)이면 panic/recover를 쓰는데도 코드가 제대로 실패**한다.

이 패턴 덕에 `error` 메서드는 파스 스택을 손으로 되감을 걱정 없이 에러를 보고한다.

```go
if pos == 0 {
    re.error("'*' illegal at start of expression")
}
```

> **이 패턴은 패키지 내부에서만 써야 한다.** `Compile`은 내부 panic을 error 값으로 바꿔 **클라이언트에 panic을 노출하지 않는다** — 좋은 규칙이다.
>
> 참고: 이 re-panic 관용은 실제 에러 발생 시 panic 값을 바꾸지만, 원래·새 실패가 모두 크래시 리포트에 표시되므로 근본 원인은 보인다. 원래 값만 표시하려면 예상치 못한 문제를 걸러 원래 에러로 re-panic하는 코드를 더 쓰면 된다.

---

## 4. 정리

| 항목 | 핵심 |
|---|---|
| `error` | 빌트인 인터페이스 `Error() string`, 다중 반환으로 상세 정보 제공 |
| 풍부한 에러 | `os.PathError`처럼 맥락(연산·경로) 포함, 접두사로 출처 식별 |
| 구체 검사 | 타입 단언/스위치 + comma-ok로 특정 에러 추출 |
| `panic` | 복구 불가 상황·"불가능한 일"에만, **라이브러리는 피하라**(초기화는 예외) |
| `recover` | **지연 함수 안에서만**, 되감기 멈추고 panic 인자 반환 |
| 패턴 | 명명 반환값 수정 + 타입 단언으로 의도된 에러만 복구, **패키지 내부에 한정** |

**한 줄 요약:** Go는 error를 값으로 반환해 명시적으로 다루는 것이 기본이며, `panic`/`recover`는 복구 불가 상황이나 패키지 내부의 깊은 호출 스택을 빠져나올 때만 쓴다. recover는 지연 함수 안에서만 동작하고, 내부 panic을 error로 변환해 클라이언트에 노출하지 않는다.
