# Chapter 07. Functions

> 원문: [Effective Go — Functions](https://go.dev/doc/effective_go#functions)

---

## 1. Multiple return values — 다중 반환값

Go의 독특한 기능 중 하나는 **함수와 메서드가 여러 값을 반환**할 수 있다는 점이다. 이는 C 프로그램의 어색한 두 가지 관용을 개선한다.

- **in-band 에러 반환** — 예: EOF를 `-1`로 표현하는 식
- **주소로 넘긴 인자를 수정** — 참조 매개변수(reference parameter) 흉내

C에서는 쓰기 에러를 음수 카운트로 알리고 에러 코드는 별도의 휘발성 위치에 숨긴다. Go에서는 `Write`가 **카운트와 에러를 함께** 반환한다 — "바이트를 쓰긴 했지만 장치가 가득 차서 전부는 못 썼다"를 그대로 표현한다.

```go
func (file *File) Write(b []byte) (n int, err error)
// 쓴 바이트 수 n과, n != len(b)일 때 non-nil error를 반환
```

같은 접근으로 **반환값 포인터를 넘겨 참조 매개변수를 흉내 낼 필요**도 사라진다. 바이트 슬라이스의 특정 위치에서 숫자를 읽어 **숫자와 다음 위치를 함께 반환**하는 예:

```go
func nextInt(b []byte, i int) (int, int) {
    for ; i < len(b) && !isDigit(b[i]); i++ {
    }
    x := 0
    for ; i < len(b) && isDigit(b[i]); i++ {
        x = x*10 + int(b[i]) - '0'
    }
    return x, i
}

// 사용:
for i := 0; i < len(b); {
    x, i = nextInt(b, i)
    fmt.Println(x)
}
```

---

## 2. Named result parameters — 명명된 반환값

반환("결과") 매개변수에 **이름을 붙여 일반 변수처럼** 사용할 수 있다(들어오는 매개변수와 동일하게).

- 이름이 붙으면 함수 시작 시 해당 타입의 **제로 값(zero value)으로 초기화**된다.
- 함수가 **인자 없는 `return`(naked return)**을 실행하면, **결과 매개변수의 현재 값**이 반환된다.
- 이름은 필수가 아니지만 코드를 **짧고 명확하게** 하고, **그 자체로 문서** 역할을 한다.

```go
// 어떤 int가 무엇인지 이름으로 분명해진다
func nextInt(b []byte, pos int) (value, nextPos int)
```

명명된 결과는 초기화되고 naked `return`과 묶이므로, 코드를 **단순화하면서 명확하게** 한다. `io.ReadFull`의 잘 쓴 버전:

```go
func ReadFull(r Reader, buf []byte) (n int, err error) {
    for len(buf) > 0 && err == nil {
        var nr int
        nr, err = r.Read(buf)
        n += nr
        buf = buf[nr:]
    }
    return  // n, err를 그대로 반환
}
```

> 실무 주의: naked return은 짧은 함수에서만 권장된다. 긴 함수에서 남발하면 어떤 값이 반환되는지 추적이 어려워 가독성을 해친다.

---

## 3. Defer — 지연 호출

`defer` 문은 **지연된 함수(deferred function) 호출을, defer를 실행한 함수가 반환되기 직전에** 실행하도록 예약한다. 함수가 **어떤 경로로 반환되든 반드시 해제해야 하는 자원**을 다루는 데 효과적이다. 정준 예시는 뮤텍스 잠금 해제, 파일 닫기다.

```go
// Contents returns the file's contents as a string.
func Contents(filename string) (string, error) {
    f, err := os.Open(filename)
    if err != nil {
        return "", err
    }
    defer f.Close() // 작업이 끝나면 f.Close가 실행됨

    var result []byte
    buf := make([]byte, 100)
    for {
        n, err := f.Read(buf[0:])
        result = append(result, buf[0:n]...) // append는 뒤에서 다룸
        if err != nil {
            if err == io.EOF {
                break
            }
            return "", err // 여기서 반환해도 f는 닫힌다
        }
    }
    return string(result), nil // 여기서 반환해도 f는 닫힌다
}
```

`Close` 같은 호출을 defer하면 두 가지 이점이 있다.

1. **파일 닫기를 절대 잊지 않는다.** 나중에 새 return 경로를 추가해도 안전하다.
2. **close가 open 근처에 위치**해, 함수 끝에 두는 것보다 훨씬 명확하다.

### 인자 평가 시점 — defer 실행 시점에 평가된다

지연 함수의 인자(메서드라면 리시버 포함)는 **호출 시점이 아니라 `defer`가 실행되는 시점에 평가**된다. 덕분에 함수 실행 중 변수 값이 바뀌어도 걱정할 필요가 없고, **하나의 defer 지점이 여러 호출을 예약**할 수 있다.

```go
for i := 0; i < 5; i++ {
    defer fmt.Printf("%d ", i)
}
```

지연 함수는 **LIFO(후입선출) 순서**로 실행되므로, 함수 반환 시 `4 3 2 1 0`이 출력된다.

### 응용: 함수 실행 추적(tracing)

인자가 defer 실행 시점에 평가된다는 사실을 활용하면, **trace 루틴이 untrace 루틴의 인자를 설정**할 수 있다.

```go
func trace(s string) string {
    fmt.Println("entering:", s)
    return s
}
func un(s string) {
    fmt.Println("leaving:", s)
}
func a() {
    defer un(trace("a"))  // trace("a")는 즉시 실행되고, un("a")는 반환 직전 실행
    fmt.Println("in a")
}
func b() {
    defer un(trace("b"))
    fmt.Println("in b")
    a()
}
func main() {
    b()
}
```

출력:

```
entering: b
in b
entering: a
in a
leaving: a
leaving: b
```

> 다른 언어의 **블록 단위(block-level) 자원 관리**에 익숙한 프로그래머에게 defer는 낯설 수 있지만, 가장 흥미롭고 강력한 응용은 정확히 **defer가 블록 기반이 아니라 함수 기반(function-based)**이라는 점에서 나온다. (panic/recover 절에서 또 다른 활용을 본다.)

---

## 4. 정리

| 항목 | 핵심 |
|---|---|
| 다중 반환 | `(n int, err error)`로 결과+에러 동시 반환 — in-band 에러·포인터 인자 불필요 |
| 명명 결과 | 제로 값 초기화 + naked `return`, 그 자체가 문서 (긴 함수에선 자제) |
| defer | 반환 직전 실행, 자원 해제 누락 방지, close를 open 근처에 |
| defer 인자 평가 | **defer 실행 시점**에 평가 (호출 시점 아님) |
| defer 순서 | **LIFO** |
| defer 본질 | 블록 기반이 아닌 **함수 기반** |

**한 줄 요약:** 다중 반환값과 명명된 결과로 에러를 명시적으로 다루고, `defer`로 자원 해제를 open 근처에 묶어 어떤 반환 경로에서도 누락 없이 정리한다. defer 인자는 defer 시점에 평가되고 LIFO로 실행된다.
