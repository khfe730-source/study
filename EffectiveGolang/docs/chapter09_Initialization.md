# Chapter 09. Initialization

> 원문: [Effective Go — Initialization](https://go.dev/doc/effective_go#initialization)

---

겉보기엔 C/C++의 초기화와 크게 달라 보이지 않지만, Go의 초기화는 더 강력하다. 초기화 중 **복잡한 구조를 만들 수 있고**, 초기화되는 객체들 사이의 **순서 문제(ordering)는 서로 다른 패키지 간에도 올바르게** 처리된다.

---

## 1. Constants

Go의 상수(constant)는 말 그대로 상수다. **컴파일 타임에 생성**되며(함수 내 지역으로 정의해도), **숫자·문자(rune)·문자열·불리언**만 가능하다. 컴파일 타임 제약 때문에 상수를 정의하는 표현식은 **컴파일러가 평가할 수 있는 상수 표현식**이어야 한다.

- `1 << 3` → 상수 표현식 (OK)
- `math.Sin(math.Pi/4)` → 함수 호출이 런타임에 일어나야 하므로 상수 아님

### iota 열거자

열거 상수(enumerated constants)는 **`iota` 열거자**로 만든다. `iota`는 표현식의 일부가 될 수 있고 표현식은 암묵적으로 반복되므로, 정교한 값 집합을 쉽게 만든다.

```go
type ByteSize float64

const (
    _           = iota             // 첫 값(0)은 blank identifier로 무시
    KB ByteSize = 1 << (10 * iota) // 1 << 10
    MB                             // 1 << 20 (표현식 암묵 반복)
    GB                             // 1 << 30
    TB
    PB
    EB
    ZB
    YB
)
```

`String` 메서드를 임의의 사용자 정의 타입에 붙일 수 있어, 스칼라 타입(예: `ByteSize` 같은 실수 타입)도 출력 시 스스로 포맷할 수 있다.

```go
func (b ByteSize) String() string {
    switch {
    case b >= YB:
        return fmt.Sprintf("%.2fYB", b/YB)
    case b >= ZB:
        return fmt.Sprintf("%.2fZB", b/ZB)
    // ... EB, PB, TB, GB, MB
    case b >= KB:
        return fmt.Sprintf("%.2fKB", b/KB)
    }
    return fmt.Sprintf("%.2fB", b)
}
```

`YB`는 `1.00YB`, `ByteSize(1e13)`은 `9.09TB`로 출력된다.

> **재귀 회피 포인트:** 여기서 `Sprintf`가 안전한 이유는 변환 때문이 아니라 **`%f`(문자열 포맷이 아님)를 쓰기 때문**이다. `Sprintf`는 문자열이 필요할 때만 `String` 메서드를 호출하는데, `%f`는 실수 값을 원하므로 재귀가 발생하지 않는다.

---

## 2. Variables

변수는 상수처럼 초기화할 수 있지만, **초기화 식이 런타임에 계산되는 일반 표현식**이어도 된다.

```go
var (
    home   = os.Getenv("HOME")
    user   = os.Getenv("USER")
    gopath = os.Getenv("GOPATH")
)
```

---

## 3. The init function

각 소스 파일은 필요한 상태를 설정하는 **무인자(niladic) `init` 함수**를 정의할 수 있다(한 파일에 여러 개도 가능).

**`init`의 호출 순서**가 핵심이다 — "finally는 정말로 finally":

1. 임포트된 **모든 패키지가 먼저 초기화**되고,
2. 그 다음 패키지의 **모든 변수 선언의 초기화 식이 평가**되고,
3. **그 후에야 `init`이 호출**된다.

선언으로 표현할 수 없는 초기화 외에, `init`의 흔한 용도는 **실제 실행 전에 프로그램 상태의 정합성을 검증·교정**하는 것이다.

```go
func init() {
    if user == "" {
        log.Fatal("$USER not set")
    }
    if home == "" {
        home = "/home/" + user
    }
    if gopath == "" {
        gopath = home + "/go"
    }
    // gopath는 커맨드라인 --gopath 플래그로 재정의 가능
    flag.StringVar(&gopath, "gopath", gopath, "override default GOPATH")
}
```

---

## 4. 정리

| 항목 | 핵심 |
|---|---|
| 상수 | **컴파일 타임** 생성, 숫자/rune/문자열/불리언만, 상수 표현식만 |
| `iota` | 열거 상수 생성기, 표현식 암묵 반복으로 집합 구성 |
| 변수 | 런타임 표현식으로 초기화 가능 |
| `init` | **임포트 패키지 → 변수 초기화 → init** 순서, 상태 검증·교정에 사용 |

**한 줄 요약:** 상수는 컴파일 타임에 평가되고 `iota`로 열거를 만든다. 초기화 순서는 "임포트 패키지 → 변수 초기화 식 → `init`"로 보장되며, `init`은 실행 전 상태 검증에 적합하다.
