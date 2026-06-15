# Chapter 12. The blank identifier

> 원문: [Effective Go — The blank identifier](https://go.dev/doc/effective_go#blank)

---

블랭크 식별자(blank identifier) `_`는 **어떤 타입의 어떤 값으로도 할당·선언할 수 있고, 그 값은 무해하게 버려진다.** Unix의 `/dev/null`에 쓰는 것과 비슷하다 — 변수가 필요하지만 실제 값은 무의미한 자리에 쓰는 **쓰기 전용(write-only) 자리표시자**다.

---

## 1. The blank identifier in multiple assignment

`for range`에서의 사용은 일반적 상황인 **다중 할당(multiple assignment)**의 특수 케이스다. 왼쪽에 여러 값이 필요한데 그중 하나를 안 쓸 때, 더미 변수를 만들 필요 없이 `_`로 **버릴 의도를 분명히** 한다.

```go
if _, err := os.Stat(path); os.IsNotExist(err) {
    fmt.Printf("%s does not exist\n", path)
}
```

> **에러를 버리는 것은 끔찍한 관행이다.** 에러 반환은 이유가 있어 제공되니 **항상 검사하라.**
> ```go
> // 나쁨! path가 없으면 크래시한다.
> fi, _ := os.Stat(path)
> if fi.IsDir() { ... }
> ```

---

## 2. Unused imports and variables

Go에서는 **사용하지 않는 임포트나 변수가 컴파일 에러**다. 미사용 임포트는 프로그램을 부풀리고 컴파일을 늦추며, 초기화만 되고 안 쓰이는 변수는 낭비이자 버그 징후일 수 있다.

하지만 개발 중에는 이런 것들이 자주 생기고, 컴파일을 위해 지웠다가 나중에 다시 넣는 게 번거롭다. 블랭크 식별자가 **임시 우회책**을 준다.

```go
package main

import (
    "fmt"
    "io"
    "log"
    "os"
)

var _ = fmt.Printf // 디버깅용; 끝나면 삭제
var _ io.Reader    // 디버깅용; 끝나면 삭제

func main() {
    fd, err := os.Open("test.go")
    if err != nil {
        log.Fatal(err)
    }
    // TODO: use fd.
    _ = fd
}
```

- 미사용 임포트는 그 패키지의 심볼을 `_`에 참조시켜 침묵시킨다.
- 미사용 변수는 `_ = fd`로 할당해 침묵시킨다.

> 관례: 임포트 에러를 막는 전역 선언은 **임포트 바로 뒤에 두고 주석**을 달아, 찾기 쉽고 나중에 정리하라는 알림이 되게 한다.

---

## 3. Import for side effect

때로는 **부수 효과(side effect)만을 위해** 명시적 사용 없이 패키지를 임포트하는 게 유용하다. 예를 들어 `net/http/pprof`는 `init` 함수에서 디버깅 정보를 제공하는 HTTP 핸들러를 등록한다. 대부분의 클라이언트는 핸들러 등록만 필요하고 데이터는 웹 페이지로 접근한다.

부수 효과만을 위해 임포트하려면 패키지를 **블랭크 식별자로 리네임**한다.

```go
import _ "net/http/pprof"
```

이 형태는 패키지를 부수 효과 때문에 임포트함을 분명히 한다 — 이 파일에서 이름이 없으니 다른 사용 가능성이 없기 때문이다. (이름이 있는데 안 쓰면 컴파일러가 거부한다.)

---

## 4. Interface checks

타입은 인터페이스를 구현한다고 **명시적으로 선언할 필요가 없다** — 메서드를 구현하기만 하면 된다. 대부분의 인터페이스 변환은 **정적(static)이라 컴파일 타임에 검사**된다. 예를 들어 `*os.File`을 `io.Reader`를 기대하는 함수에 넘기면, `*os.File`이 `io.Reader`를 구현하지 않는 한 컴파일되지 않는다.

일부 검사는 **런타임**에 일어난다. `encoding/json`의 `Marshaler` 인터페이스가 그렇다 — JSON 인코더는 타입 단언으로 런타임에 확인한다.

```go
m, ok := val.(json.Marshaler)
```

타입이 인터페이스를 구현하는지 **묻기만** 하면 될 때(인터페이스 자체는 안 쓰고)는 단언 값을 블랭크로 무시한다.

```go
if _, ok := val.(json.Marshaler); ok {
    fmt.Printf("value %v of type %T implements json.Marshaler\n", val, val)
}
```

### 컴파일 타임 인터페이스 보증

패키지 안에서 타입이 인터페이스를 **실제로 만족함을 보증**해야 할 때가 있다. 예를 들어 `json.RawMessage`가 커스텀 JSON 표현을 위해 `json.Marshaler`를 구현해야 하는데, 그것을 컴파일러가 자동 검증할 **정적 변환이 없을** 수 있다. 우연히 인터페이스를 만족하지 못하면 인코더는 동작하되 커스텀 구현을 안 쓴다.

구현이 올바름을 보증하려면 **블랭크 식별자를 쓴 전역 선언**:

```go
var _ json.Marshaler = (*RawMessage)(nil)
```

`*RawMessage`를 `Marshaler`로 변환하는 이 할당은 `*RawMessage`가 `Marshaler`를 구현할 것을 요구하며, 이 성질이 **컴파일 타임에 검사**된다. 인터페이스가 바뀌면 이 패키지가 더는 컴파일되지 않아 갱신 필요를 알게 된다.

여기서 블랭크 식별자는 이 선언이 **변수 생성이 아니라 타입 검사만**을 위한 것임을 나타낸다. 단, 인터페이스를 만족하는 **모든** 타입에 이러지는 말 것 — 관례상 **이미 정적 변환이 코드에 없을 때만**(드문 경우) 쓴다.

---

## 5. 정리

| 용도 | 핵심 |
|---|---|
| 다중 할당 | 안 쓰는 반환값을 `_`로 버림 (단 **에러는 항상 검사**) |
| 미사용 임포트/변수 | `var _ = pkg.X`, `_ = v`로 임시 침묵 (주석 달고 나중에 정리) |
| 부수 효과 임포트 | `import _ "net/http/pprof"` — `init`만 실행 |
| 인터페이스 검사 | `if _, ok := v.(I); ok`, 보증은 `var _ I = (*T)(nil)`로 컴파일 타임 검증 |

**한 줄 요약:** `_`는 버릴 값을 나타내는 자리표시자다. 안 쓰는 반환값·임포트·변수를 침묵시키고, `import _`로 부수 효과만 취하며, `var _ I = (*T)(nil)`로 인터페이스 구현을 컴파일 타임에 보증한다. 단 에러는 절대 `_`로 버리지 마라.
