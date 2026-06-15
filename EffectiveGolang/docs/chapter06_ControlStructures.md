# Chapter 06. Control structures

> 원문: [Effective Go — Control structures](https://go.dev/doc/effective_go#control-structures)

---

## 0. C와의 차이 요약

Go의 제어 구조는 C와 닮았지만 중요한 차이가 있다.

- `do`나 `while` 루프가 **없다.** 약간 일반화된 **`for` 하나**로 통합.
- `switch`가 더 **유연(flexible)**하다.
- `if`와 `switch`는 `for`처럼 **선택적 초기화 문(initialization statement)**을 받는다.
- `break`/`continue`는 대상을 지정하는 **선택적 레이블(label)**을 받는다.
- 새 제어 구조: **타입 스위치(type switch)**, 다중 통신 멀티플렉서 **`select`**.
- 문법: **괄호 없음**, 본문은 **항상 중괄호로 감싼다.**

---

## 1. If

```go
if x > 0 {
    return y
}
```

중괄호가 필수(mandatory braces)라 단순한 `if`도 여러 줄로 쓰게 되는데, **그게 좋은 스타일**이다 — 특히 본문에 `return`·`break` 같은 제어문이 있을 때.

### 초기화 문 활용

`if`/`switch`가 초기화 문을 받으므로 **지역 변수를 그 자리에서 설정**하는 관용이 흔하다.

```go
if err := file.Chmod(0664); err != nil {
    log.Print(err)
    return err
}
```

`err`의 스코프가 `if` 블록으로 한정되어 누수되지 않는다.

### 불필요한 else는 생략한다

`if` 본문이 `break`/`continue`/`goto`/`return`으로 끝나 다음 문장으로 이어지지 않으면, **불필요한 `else`를 생략**한다.

```go
f, err := os.Open(name)
if err != nil {
    return err
}
codeUsing(f)
```

에러 케이스를 발생 즉시 제거하며 **성공 흐름이 페이지를 따라 아래로 곧게 흐르도록(happy path down the page)** 쓰는 것이 핵심이다. 에러 케이스는 `return`으로 끝나므로 `else`가 필요 없다.

```go
f, err := os.Open(name)
if err != nil {
    return err
}
d, err := f.Stat()
if err != nil {
    f.Close()
    return err
}
codeUsing(f, d)
```

---

## 2. Redeclaration and reassignment (`:=` 재선언/재할당)

위 예제는 `:=` 단축 선언의 동작 디테일을 보여준다.

```go
f, err := os.Open(name)  // f, err 선언
d, err := f.Stat()       // d는 선언, err는 재할당(re-assign) — 선언처럼 보이지만 아님
```

`err`이 양쪽에 등장하는 이 중복은 **합법**이다. 두 번째 `:=`에서 `err`은 새로 선언되는 게 아니라 **기존 변수에 새 값을 줄 뿐**이다.

### `:=`로 이미 선언된 변수가 다시 나와도 되는 조건

이미 선언된 변수 `v`가 `:=`에 다시 등장하려면 **세 가지를 모두** 만족해야 한다.

1. 이 선언이 기존 `v` 선언과 **같은 스코프(same scope)**일 것 (`v`가 바깥(outer) 스코프에 있으면 **새 변수가 생성**된다).
2. 초기화의 대응 값이 `v`에 **할당 가능(assignable)**할 것.
3. 이 선언으로 **새로 생성되는 변수가 최소 하나 이상** 있을 것.

이 독특한 성질은 순수한 실용주의(pragmatism)로, 긴 if-else 체인에서 **하나의 `err`을 계속 재사용**하기 쉽게 한다.

> §주의: Go에서 **함수 매개변수와 반환값의 스코프는 함수 본문과 동일**하다. 어휘적으로는 중괄호 밖에 있어 보이지만 본문과 같은 스코프다.

---

## 3. For

`for`는 `for`와 `while`을 통합하며 `do-while`은 없다. 세 가지 형태가 있고, 세미콜론이 있는 건 하나뿐이다.

```go
// C의 for
for init; condition; post { }
// C의 while
for condition { }
// C의 for(;;) — 무한 루프
for { }
```

### range 절

배열·슬라이스·문자열·맵을 순회하거나 채널에서 읽을 때 `range`를 쓴다.

```go
for key, value := range oldMap {
    newMap[key] = value
}

for key := range m {          // 키만 필요하면 두 번째 생략
    if key.expired() {
        delete(m, key)
    }
}

sum := 0
for _, value := range array { // 값만 필요하면 blank identifier(_)로 첫 번째 버림
    sum += value
}
```

### 문자열 range는 UTF-8을 디코딩한다

문자열에 `range`를 쓰면 **UTF-8을 파싱해 개별 유니코드 코드 포인트(rune)**로 쪼갠다. 잘못된 인코딩은 1바이트를 소비하고 대체 문자 `U+FFFD`를 만든다. (`rune`은 단일 유니코드 코드 포인트를 가리키는 Go 용어이자 빌트인 타입이다.)

```go
for pos, char := range "日本\x80語" { // \x80은 잘못된 UTF-8
    fmt.Printf("character %#U starts at byte position %d\n", char, pos)
}
```

출력:

```
character U+65E5 '日' starts at byte position 0
character U+672C '本' starts at byte position 3
character U+FFFD '�' starts at byte position 6
character U+8A9E '語' starts at byte position 7
```

> 인덱스 `pos`는 **바이트 위치**다(룬 인덱스가 아님). `本` 다음이 6, `語`가 7인 이유 — 잘못된 바이트 하나(6)가 `U+FFFD`로 처리되고 1바이트만 소비됐기 때문.

### 콤마 연산자 없음, `++`/`--`는 문장

Go에는 **콤마 연산자가 없고** `++`/`--`는 표현식이 아니라 **문장(statement)**이다. 한 `for`에서 여러 변수를 굴리려면 **병렬 할당(parallel assignment)**을 쓴다(단 이러면 `++`/`--`는 못 쓴다).

```go
// a 뒤집기
for i, j := 0, len(a)-1; i < j; i, j = i+1, j-1 {
    a[i], a[j] = a[j], a[i]
}
```

---

## 4. Switch

Go의 `switch`는 C보다 일반적이다.

- 표현식이 **상수나 정수일 필요가 없다.**
- 케이스는 **위에서 아래로 평가**되어 처음 매치되는 곳에서 멈춘다.
- **switch에 표현식이 없으면 `true`를 스위치**한다 → if-else-if 체인을 `switch`로 쓰는 게 관용적.

```go
func unhex(c byte) byte {
    switch {
    case '0' <= c && c <= '9':
        return c - '0'
    case 'a' <= c && c <= 'f':
        return c - 'a' + 10
    case 'A' <= c && c <= 'F':
        return c - 'A' + 10
    }
    return 0
}
```

### 자동 fall through 없음, 콤마 리스트

자동 폴스루(fall through)가 **없다.** 대신 케이스를 **콤마로 나열**할 수 있다.

```go
func shouldEscape(c byte) bool {
    switch c {
    case ' ', '?', '&', '=', '#', '+', '%':
        return true
    }
    return false
}
```

### break와 레이블

`break`로 `switch`를 일찍 끝낼 수 있다(C 계열만큼 흔하진 않다). **둘러싼 루프(surrounding loop)를 빠져나가야** 할 때는 루프에 **레이블(label)**을 붙이고 그 레이블로 break한다.

```go
Loop:
    for n := 0; n < len(src); n += size {
        switch {
        case src[n] < sizeOne:
            if validateOnly {
                break          // switch만 빠져나감
            }
            size = 1
            update(src[n])
        case src[n] < sizeTwo:
            if n+1 >= len(src) {
                err = errShortInput
                break Loop      // 루프 전체를 빠져나감
            }
            if validateOnly {
                break
            }
            size = 2
            update(src[n] + src[n+1]<<shift)
        }
    }
```

`continue`도 선택적 레이블을 받지만 **루프에만 적용**된다.

두 개의 `switch`를 쓴 바이트 슬라이스 비교 루틴:

```go
// Compare returns an integer comparing the two byte slices,
// lexicographically.
// The result will be 0 if a == b, -1 if a < b, and +1 if a > b
func Compare(a, b []byte) int {
    for i := 0; i < len(a) && i < len(b); i++ {
        switch {
        case a[i] > b[i]:
            return 1
        case a[i] < b[i]:
            return -1
        }
    }
    switch {
    case len(a) > len(b):
        return 1
    case len(a) < len(b):
        return -1
    }
    return 0
}
```

---

## 5. Type switch

`switch`로 **인터페이스 변수의 동적 타입(dynamic type)**을 알아낼 수 있다. 타입 스위치는 괄호 안에 `type` 키워드를 쓴 **타입 단언(type assertion)** 문법을 쓴다. switch 표현식에서 변수를 선언하면, 그 변수는 **각 케이스에서 해당 타입**을 가진다. 같은 이름을 재사용하는 것이 관용적이다(케이스마다 같은 이름·다른 타입의 새 변수를 선언하는 셈).

```go
var t interface{}
t = functionOfSomeType()
switch t := t.(type) {
default:
    fmt.Printf("unexpected type %T\n", t) // %T는 t의 타입을 출력
case bool:
    fmt.Printf("boolean %t\n", t)         // t는 bool
case int:
    fmt.Printf("integer %d\n", t)         // t는 int
case *bool:
    fmt.Printf("pointer to boolean %t\n", *t) // t는 *bool
case *int:
    fmt.Printf("pointer to integer %d\n", *t) // t는 *int
}
```

---

## 6. 정리

| 구조 | 핵심 |
|---|---|
| 전반 | 괄호 없음, 본문은 항상 중괄호. `do`/`while` 없음 |
| `if` | 초기화 문 가능, happy path를 아래로 흐르게 — **불필요한 `else` 생략** |
| `:=` 재선언 | 같은 스코프 + 할당 가능 + 새 변수 최소 1개 → 기존 변수는 재할당 |
| `for` | `for`+`while` 통합, `range`로 순회, 문자열 range는 rune 단위(UTF-8) |
| `for` 다변수 | 콤마 연산자 없음, `++`/`--`는 문장 → **병렬 할당** 사용 |
| `switch` | 표현식 비상수 가능, 폴스루 없음(콤마 리스트), 표현식 없으면 `true` 스위치 |
| `break`/레이블 | 레이블로 둘러싼 루프 탈출 |
| type switch | `t.(type)`로 동적 타입 분기, 케이스마다 변수 타입이 달라짐 |

**한 줄 요약:** Go의 제어 구조는 `for` 하나로 루프를 통합하고, 표현식 없는 `switch`와 타입 스위치로 표현력을 확장하며, `if`의 초기화 문과 "불필요한 else 생략"으로 에러 처리를 평평하게 흐르도록 유도한다.
