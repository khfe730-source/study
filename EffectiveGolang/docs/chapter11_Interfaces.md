# Chapter 11. Interfaces and other types

> 원문: [Effective Go — Interfaces and other types](https://go.dev/doc/effective_go#interfaces_and_types)

---

## 1. Interfaces — 행위의 명세

인터페이스는 객체의 **행위(behavior)**를 명세한다: "이걸 할 수 있다면, 여기서 쓸 수 있다." 커스텀 프린터는 `String` 메서드로, `Fprintf`는 `Write` 메서드를 가진 무엇에든 출력한다. **메서드 1~2개짜리 인터페이스가 흔하며**, 보통 메서드에서 딴 이름(`io.Writer` 등)을 가진다.

한 타입은 **여러 인터페이스를 구현**할 수 있다. 예를 들어 `sort.Interface`(`Len()`, `Less(i, j int) bool`, `Swap(i, j int)`)를 구현하면 `sort` 패키지로 정렬되고, 커스텀 포매터도 가질 수 있다.

```go
type Sequence []int

// sort.Interface가 요구하는 메서드
func (s Sequence) Len() int           { return len(s) }
func (s Sequence) Less(i, j int) bool { return s[i] < s[j] }
func (s Sequence) Swap(i, j int)      { s[i], s[j] = s[j], s[i] }

func (s Sequence) Copy() Sequence {
    copy := make(Sequence, 0, len(s))
    return append(copy, s...)
}

// 출력 메서드 - 출력 전 정렬
func (s Sequence) String() string {
    s = s.Copy() // 복사: 인자를 덮어쓰지 않음
    sort.Sort(s)
    str := "["
    for i, elem := range s { // O(N²) — 다음 예제에서 개선
        if i > 0 {
            str += " "
        }
        str += fmt.Sprint(elem)
    }
    return str + "]"
}
```

---

## 2. Conversions — 메서드 집합 접근을 위한 변환

위 `String`은 `Sprint`가 슬라이스에 대해 이미 하는 일을 재구현하며 O(N²)로 느리다. **`Sequence`를 평범한 `[]int`로 변환**한 뒤 `Sprint`를 부르면 작업을 공유하고 빨라진다.

```go
func (s Sequence) String() string {
    s = s.Copy()
    sort.Sort(s)
    return fmt.Sprint([]int(s))
}
```

이는 `String`에서 `Sprintf`를 안전하게 부르는 변환 기법의 또 다른 예다. 두 타입(`Sequence`와 `[]int`)은 **타입 이름을 무시하면 같으므로** 변환이 적법하다. 이 변환은 **새 값을 만들지 않고**, 기존 값이 새 타입인 것처럼 잠시 행세할 뿐이다. (정수→실수처럼 새 값을 만드는 변환도 있다.)

**다른 메서드 집합에 접근하려고 표현식의 타입을 변환하는 것은 Go의 관용**이다. 기존 `sort.IntSlice`를 쓰면:

```go
func (s Sequence) String() string {
    s = s.Copy()
    sort.IntSlice(s).Sort()
    return fmt.Sprint([]int(s))
}
```

이제 `Sequence`가 여러 인터페이스를 구현하는 대신, **한 데이터가 여러 타입(`Sequence`, `sort.IntSlice`, `[]int`)으로 변환**되며 각 타입이 일부 일을 맡는다.

---

## 3. Interface conversions and type assertions

타입 스위치는 변환의 한 형태다 — 인터페이스를 받아 각 케이스에서 그 케이스 타입으로 변환하는 셈이다. `fmt.Printf` 아래 코드가 값을 문자열로 바꾸는 단순화 버전:

```go
type Stringer interface {
    String() string
}

var value interface{} // 호출자가 제공한 값
switch str := value.(type) {
case string:
    return str
case Stringer:
    return str.String()
}
```

첫 케이스는 구체 값(concrete value)을, 둘째는 인터페이스를 **다른 인터페이스로** 변환한다.

### 타입 단언(type assertion)

관심 타입이 하나뿐이라면 **타입 단언**으로 충분하다. 인터페이스 값에서 명시한 타입의 값을 추출한다.

```go
value.(typeName)  // 결과는 정적 타입 typeName의 새 값
```

`typeName`은 인터페이스가 담은 구체 타입이거나, 값이 변환될 수 있는 두 번째 인터페이스 타입이어야 한다.

```go
str := value.(string)  // value가 string이 아니면 런타임 에러로 크래시
```

크래시를 막으려면 **"comma, ok" 관용**:

```go
str, ok := value.(string)
if ok {
    fmt.Printf("string value is: %q\n", str)
} else {
    fmt.Printf("value is not a string\n")
}
```

단언이 실패하면 `str`은 존재하되 `string`의 제로 값(빈 문자열)이 된다. 위 타입 스위치와 동등한 if-else:

```go
if str, ok := value.(string); ok {
    return str
} else if str, ok := value.(Stringer); ok {
    return str.String()
}
```

---

## 4. Generality — 인터페이스만 익스포트하기

어떤 타입이 **오직 인터페이스를 구현하기 위해서만 존재**하고 그 인터페이스 외에 익스포트할 메서드가 없다면, **타입 자체를 익스포트할 필요가 없다.** 인터페이스만 익스포트하면 "이 값엔 인터페이스에 기술된 것 외의 흥미로운 행위가 없다"가 분명해지고, 공통 메서드 문서를 매 구현마다 반복할 필요도 없다.

이런 경우 **생성자는 구현 타입이 아니라 인터페이스 값을 반환**해야 한다. 해시 라이브러리에서 `crc32.NewIEEE`와 `adler32.New`는 모두 `hash.Hash32` 인터페이스를 반환한다. → CRC-32를 Adler-32로 바꾸려면 **생성자 호출만 바꾸면** 되고 나머지 코드는 영향받지 않는다.

`crypto/cipher`도 같은 접근으로 블록 암호와 스트리밍 암호를 분리한다.

```go
type Block interface {
    BlockSize() int
    Encrypt(dst, src []byte)
    Decrypt(dst, src []byte)
}
type Stream interface {
    XORKeyStream(dst, src []byte)
}

// 블록 암호를 카운터 모드(CTR) 스트리밍 암호로 변환 — 블록 세부는 추상화됨
func NewCTR(block Block, iv []byte) Stream
```

`NewCTR`은 **`Block` 인터페이스의 어떤 구현에도** 적용되며, 인터페이스 값을 반환하므로 암호 모드 교체가 국소적 변경으로 끝난다.

---

## 5. Interfaces and methods — (거의) 무엇이든 인터페이스를 만족한다

거의 모든 것에 메서드를 붙일 수 있으니, 거의 모든 것이 인터페이스를 만족할 수 있다. `http` 패키지의 `Handler`가 대표 예다.

```go
type Handler interface {
    ServeHTTP(ResponseWriter, *Request)
}
```

`ResponseWriter` 자체도 인터페이스이며 표준 `Write` 메서드를 포함하므로 `io.Writer`가 쓰이는 곳에서 쓸 수 있다.

### 구조체로 만든 핸들러

```go
type Counter struct {
    n int
}
func (ctr *Counter) ServeHTTP(w http.ResponseWriter, req *http.Request) {
    ctr.n++
    fmt.Fprintf(w, "counter = %d\n", ctr.n)
}

ctr := new(Counter)
http.Handle("/counter", ctr)
```

### 정수로 만든 핸들러

구조체일 필요도 없다 — 정수면 충분하다(증가가 호출자에게 보이도록 리시버는 포인터).

```go
type Counter int
func (ctr *Counter) ServeHTTP(w http.ResponseWriter, req *http.Request) {
    *ctr++
    fmt.Fprintf(w, "counter = %d\n", *ctr)
}
```

### 채널로 만든 핸들러

```go
type Chan chan *http.Request
func (ch Chan) ServeHTTP(w http.ResponseWriter, req *http.Request) {
    ch <- req
    fmt.Fprint(w, "notification sent")
}
```

### 함수로 만든 핸들러 — `http.HandlerFunc`

포인터·인터페이스를 뺀 어떤 타입에도 메서드를 정의할 수 있으니 **함수 타입**에도 가능하다.

```go
// HandlerFunc는 평범한 함수를 HTTP 핸들러로 쓰게 해주는 어댑터다.
type HandlerFunc func(ResponseWriter, *Request)

func (f HandlerFunc) ServeHTTP(w ResponseWriter, req *Request) {
    f(w, req)  // 리시버가 함수이고, 메서드가 그 함수를 호출
}
```

따라서 적절한 시그니처의 함수를 `HandlerFunc`로 변환하면 핸들러가 된다.

```go
func ArgServer(w http.ResponseWriter, req *http.Request) {
    fmt.Fprintln(w, os.Args)
}
http.Handle("/args", http.HandlerFunc(ArgServer))
```

> 이 절에서 **구조체·정수·채널·함수**로 HTTP 서버를 만들었다 — 인터페이스는 단지 메서드의 집합이고, 그것을 (거의) 어떤 타입에든 정의할 수 있기 때문이다.

---

## 6. 정리

| 항목 | 핵심 |
|---|---|
| 인터페이스 | 행위의 명세, 메서드만 구현하면 **암묵적으로 만족**(implements 선언 불필요) |
| 다중 구현 | 한 타입이 여러 인터페이스 구현 가능 |
| 변환 | 타입 이름만 다르고 동일하면 변환 적법, 다른 메서드 집합 접근에 활용 |
| 타입 단언 | `v.(T)` — 실패 시 크래시, **comma-ok로 안전 검사** |
| 타입 스위치 | `v.(type)`으로 동적 타입 분기 (구체 타입↔인터페이스 혼용 가능) |
| Generality | 구현 타입 대신 **인터페이스 반환** → 알고리즘 교체가 국소적 |
| 메서드 대상 | 구조체·정수·채널·함수 등 거의 모든 타입 → 모두 핸들러가 될 수 있음 |

**한 줄 요약:** Go 인터페이스는 메서드 집합이고 암묵적으로 만족된다. 타입 변환으로 메서드 집합을 갈아끼우고, comma-ok 타입 단언으로 안전하게 동적 타입을 다루며, 생성자가 인터페이스를 반환하게 하면 구현 교체가 국소적 변경으로 끝난다.
