# Chapter 08. Data

> 원문: [Effective Go — Data](https://go.dev/doc/effective_go#data)

---

## 1. Allocation with `new`

Go에는 두 개의 할당 프리미티브(allocation primitive), 빌트인 함수 **`new`**와 **`make`**가 있다. 둘은 하는 일과 적용 대상이 다르다.

`new`는 메모리를 할당하지만 **초기화하지 않고 제로화(zero)만** 한다. `new(T)`는 타입 `T`의 새 변수를 위한 제로화된 저장 공간을 할당하고 **그 주소(`*T`)를 반환**한다. = "타입 `T`의 새로 할당된 제로 값에 대한 포인터를 반환".

> Go 1.26부터 `new`는 값 표현식을 인자로 받아 초기값을 지정할 수 있다. 예: `new(int64(300))` → 300으로 초기화된 `int64`의 주소를 반환.

### "제로 값이 곧 쓸모 있는 값"으로 설계하라

`new`가 반환하는 메모리가 제로화되므로, **각 타입의 제로 값이 추가 초기화 없이 바로 쓸 수 있도록** 데이터 구조를 설계하면 좋다. 사용자가 `new`로 만들어 곧장 작업할 수 있다.

- `bytes.Buffer`의 제로 값 = "바로 쓸 수 있는 빈 버퍼"
- `sync.Mutex`는 생성자/`Init`이 없다. **제로 값이 잠금 해제된 뮤텍스**로 정의됨.

이 성질은 **이행적(transitive)**으로 동작한다.

```go
type SyncedBuffer struct {
    lock   sync.Mutex
    buffer bytes.Buffer
}

p := new(SyncedBuffer)  // *SyncedBuffer — 바로 사용 가능
var v SyncedBuffer      // SyncedBuffer  — 바로 사용 가능
```

---

## 2. Constructors and composite literals

제로 값으로 부족해 초기화 생성자가 필요할 때가 있다(`os` 패키지에서 파생한 예).

```go
func NewFile(fd int, name string) *File {
    if fd < 0 {
        return nil
    }
    f := new(File)
    f.fd = fd
    f.name = name
    f.dirinfo = nil
    f.nepipe = 0
    return f
}
```

보일러플레이트가 많다. **복합 리터럴(composite literal)**로 단순화 — 평가될 때마다 새 인스턴스를 만드는 표현식이다.

```go
func NewFile(fd int, name string) *File {
    if fd < 0 {
        return nil
    }
    f := File{fd, name, nil, 0}
    return &f
}
```

C와 달리 **지역 변수의 주소를 반환해도 완전히 OK**다 — 변수에 연결된 저장 공간은 함수 반환 후에도 살아남는다. 사실 복합 리터럴의 주소를 취하면 평가 때마다 새 인스턴스가 할당되므로 두 줄을 합칠 수 있다.

```go
return &File{fd, name, nil, 0}
```

복합 리터럴 규칙:

- 필드는 순서대로 나열되며 **모두 존재해야** 한다.
- `field: value` 쌍으로 **명시적 레이블**을 붙이면 **임의 순서**로 쓸 수 있고, 빠진 필드는 각자의 제로 값이 된다.
  ```go
  return &File{fd: fd, name: name}
  ```
- 필드가 하나도 없으면 그 타입의 제로 값 → `new(File)`과 `&File{}`는 동등하다.

배열·슬라이스·맵에도 복합 리터럴을 쓸 수 있고, 레이블은 인덱스나 맵 키가 된다.

```go
a := [...]string{Enone: "no error", Eio: "Eio", Einval: "invalid argument"}
s := []string{Enone: "no error", Eio: "Eio", Einval: "invalid argument"}
m := map[int]string{Enone: "no error", Eio: "Eio", Einval: "invalid argument"}
```

---

## 3. Allocation with `make`

`make(T, args)`는 `new(T)`와 목적이 다르다.

- **슬라이스, 맵, 채널만** 생성한다.
- **`*T`가 아니라 초기화된(제로화가 아님) `T` 값**을 반환한다.

이유: 이 세 타입은 내부적으로 **사용 전 초기화가 필요한 자료구조에 대한 참조(reference)**를 나타내기 때문. 예를 들어 슬라이스는 (데이터 포인터, 길이, 용량)의 **3-항목 디스크립터**이고, 이것들이 초기화되기 전엔 `nil`이다.

```go
make([]int, 10, 100)
// 100개 int 배열을 할당하고, 그 앞 10개를 가리키는 len=10, cap=100 슬라이스를 만든다
```

`new`와 `make`의 차이:

```go
var p *[]int = new([]int)       // 슬라이스 구조체 할당; *p == nil; 거의 쓸모없음
var v []int = make([]int, 100)  // v는 100개 int의 새 배열을 가리킴

// 불필요하게 복잡:
var p *[]int = new([]int)
*p = make([]int, 100, 100)
// 관용적: var v = make([]int, 100)
```

> `make`는 맵·슬라이스·채널에만 적용되고 **포인터를 반환하지 않는다.** 명시적 포인터가 필요하면 `new`를 쓰거나 변수의 주소를 취한다.

---

## 4. Arrays

배열은 메모리 레이아웃을 세밀하게 계획하거나 할당을 피할 때 유용하지만, 주로 **슬라이스의 빌딩 블록**이다. Go 배열은 C와 크게 다르다.

- **배열은 값(value)이다.** 한 배열을 다른 배열에 할당하면 **모든 원소가 복사**된다.
- 함수에 배열을 넘기면 **포인터가 아니라 복사본**을 받는다.
- **배열 크기는 타입의 일부**다. `[10]int`와 `[20]int`는 다른 타입.

값 성질은 유용하지만 비쌀 수 있다. C 같은 동작·효율을 원하면 배열 포인터를 넘긴다.

```go
func Sum(a *[3]float64) (sum float64) {
    for _, v := range *a {
        sum += v
    }
    return
}
array := [...]float64{7.0, 8.5, 9.1}
x := Sum(&array)  // 명시적 주소 연산자
```

그러나 이것조차 관용적 Go가 아니다. **슬라이스를 써라.**

---

## 5. Slices

슬라이스(slice)는 배열을 감싸 시퀀스에 대한 더 일반적이고 강력하고 편리한 인터페이스를 제공한다. 변환 행렬처럼 차원이 고정된 경우를 빼면 대부분의 배열 프로그래밍은 슬라이스로 한다.

슬라이스는 **기반 배열(underlying array)에 대한 참조**를 가진다. 한 슬라이스를 다른 슬라이스에 할당하면 둘 다 **같은 배열**을 가리킨다. 함수가 슬라이스 인자를 받아 원소를 바꾸면 **호출자에게 보인다**(기반 배열 포인터를 넘긴 것과 유사).

```go
func (f *File) Read(buf []byte) (n int, err error)

n, err := f.Read(buf[0:32])  // 슬라이싱으로 앞 32바이트에 읽기
```

`len`은 길이, `cap`(빌트인)은 슬라이스가 가질 수 있는 최대 길이(용량)를 보고한다. 데이터를 덧붙이는 함수 — 용량 초과 시 재할당:

```go
func Append(slice, data []byte) []byte {
    l := len(slice)
    if l+len(data) > cap(slice) { // 재할당
        // 미래 성장을 위해 필요한 양의 두 배 할당
        newSlice := make([]byte, (l+len(data))*2)
        // copy는 미리 선언된 함수로 어떤 슬라이스 타입에도 동작
        copy(newSlice, slice)
        slice = newSlice
    }
    slice = slice[0 : l+len(data)]
    copy(slice[l:], data)
    return slice
}
```

슬라이스를 **반환해야 하는 이유**: `Append`가 원소는 수정할 수 있지만, **슬라이스 자체(포인터·길이·용량을 담은 런타임 구조체)는 값으로 전달**되기 때문. (`len`/`cap`은 `nil` 슬라이스에도 적법하며 0을 반환.)

---

## 6. Two-dimensional slices

Go의 배열·슬라이스는 1차원이다. 2D는 **배열의 배열** 또는 **슬라이스의 슬라이스**로 만든다.

```go
type Transform [3][3]float64 // 3x3 배열 (배열의 배열)
type LinesOfText [][]byte    // 바이트 슬라이스의 슬라이스 — 각 줄 길이가 독립적
```

2D 슬라이스 할당 두 방법:

```go
// 방법 1: 줄마다 독립 할당 (줄이 늘거나 줄 수 있을 때)
picture := make([][]uint8, YSize)
for i := range picture {
    picture[i] = make([]uint8, XSize)
}

// 방법 2: 단일 배열 한 번 할당 후 잘라 쓰기 (크기 불변일 때 더 효율적)
picture := make([][]uint8, YSize)
pixels := make([]uint8, XSize*YSize)
for i := range picture {
    picture[i], pixels = pixels[:XSize], pixels[XSize:]
}
```

---

## 7. Maps

맵(map)은 키 타입과 값 타입을 연결하는 빌트인 자료구조다. 키는 **동등 연산자(`==`)가 정의된 타입**이면 된다(정수·실수·복소수·문자열·포인터·인터페이스·구조체·배열). **슬라이스는 동등성이 정의되지 않아 키로 못 쓴다.** 맵도 참조를 가지므로 함수에서 내용을 바꾸면 호출자에게 보인다.

```go
var timeZone = map[string]int{
    "UTC": 0 * 60 * 60,
    "EST": -5 * 60 * 60,
    "CST": -6 * 60 * 60,
    "MST": -7 * 60 * 60,
    "PST": -8 * 60 * 60,
}
offset := timeZone["EST"]
```

없는 키를 조회하면 **값 타입의 제로 값**을 반환한다. 집합(set)은 `map[T]bool`로 구현 가능.

```go
attended := map[string]bool{"Ann": true, "Joe": true}
if attended[person] { // person이 없으면 false
    fmt.Println(person, "was at the meeting")
}
```

### "comma ok" 관용 — 부재와 제로 값 구분

```go
var seconds int
var ok bool
seconds, ok = timeZone[tz]
// tz가 있으면 seconds=값, ok=true; 없으면 seconds=0, ok=false
```

값에 관심 없이 존재 여부만 보려면 blank identifier:

```go
_, present := timeZone[tz]
```

삭제는 `delete` 빌트인 — 키가 이미 없어도 안전하다.

```go
delete(timeZone, "PDT")
```

---

## 8. Printing

Go의 포맷 출력은 C의 `printf` 계열과 비슷하지만 더 풍부하다. `fmt` 패키지에 있고 이름이 대문자다(`fmt.Printf`, `fmt.Fprintf`, `fmt.Sprintf`). `Sprintf` 등 문자열 함수는 문자열을 반환한다.

포맷 문자열이 필요 없는 `Print`/`Println`도 있다. `Println`은 인자 사이에 공백을 넣고 개행을 붙이며, `Print`는 양쪽이 모두 문자열이 아닐 때만 공백을 넣는다.

```go
fmt.Printf("Hello %d\n", 23)
fmt.Fprint(os.Stdout, "Hello ", 23, "\n")
fmt.Println("Hello", 23)
fmt.Println(fmt.Sprint("Hello ", 23))
```

C와 갈라지는 지점: `%d` 같은 숫자 포맷에 **부호·크기 플래그가 없다.** 출력 루틴이 **인자의 타입**으로 결정한다.

```go
var x uint64 = 1<<64 - 1
fmt.Printf("%d %x; %d %x\n", x, x, int64(x), int64(x))
// 18446744073709551615 ffffffffffffffff; -1 -1
```

주요 verb:

- `%v` — 기본 변환(catchall). 배열·슬라이스·구조체·맵 등 어떤 값도 출력. 맵은 키로 사전식 정렬.
- `%+v` — 구조체 필드에 **이름 주석**
- `%#v` — **완전한 Go 문법**으로 출력
- `%T` — 값의 **타입** 출력
- `%q` — 문자열/`[]byte`를 따옴표로 (정수·rune엔 작은따옴표 rune 상수), `%#q`는 가능하면 백틱
- `%x` — 문자열/바이트/정수를 16진수로, `% x`는 바이트 사이 공백

```go
type T struct {
    a int
    b float64
    c string
}
t := &T{7, -2.35, "abc\tdef"}
fmt.Printf("%v\n", t)   // &{7 -2.35 abc	def}
fmt.Printf("%+v\n", t)  // &{a:7 b:-2.35 c:abc	def}
fmt.Printf("%#v\n", t)  // &main.T{a:7, b:-2.35, c:"abc\tdef"}
```

### 커스텀 타입의 출력 — `String() string`

타입에 `String() string` 메서드를 정의하면 기본 출력 포맷을 제어할 수 있다(`fmt.Stringer`).

```go
func (t *T) String() string {
    return fmt.Sprintf("%d/%g/%q", t.a, t.b, t.c)
}
```

> **무한 재귀 주의:** `String` 안에서 리시버를 문자열로 직접 출력하면 `String`이 다시 호출되어 무한 재귀에 빠진다.
> ```go
> func (m MyString) String() string {
>     return fmt.Sprintf("MyString=%s", m)         // 에러: 무한 재귀
> }
> func (m MyString) String() string {
>     return fmt.Sprintf("MyString=%s", string(m)) // OK: 기본 타입으로 변환
> }
> ```

### 가변 인자(variadic)와 `...`

`Printf`의 마지막 인자 `...interface{}`는 임의 개수·타입의 인자를 받는다.

```go
func Printf(format string, v ...interface{}) (n int, err error) { ... }

// 다른 가변 함수로 그대로 전달할 땐 v... 로 펼친다
func Println(v ...interface{}) {
    std.Output(2, fmt.Sprintln(v...))
}
```

`...`을 안 붙이면 단일 슬라이스 인자로 넘어가 컴파일되지 않는다. 특정 타입 가변 인자도 가능:

```go
func Min(a ...int) int {
    min := int(^uint(0) >> 1) // 가장 큰 int
    for _, i := range a {
        if i < min {
            min = i
        }
    }
    return min
}
```

---

## 9. Append

이제 `append` 빌트인을 설명할 조각이 갖춰졌다. 시그니처는 개념적으로:

```go
func append(slice []T, elements ...T) []T
```

`T`는 임의의 타입을 위한 자리표시자다. **호출자가 `T`를 결정하는 함수는 Go로 직접 작성할 수 없어** `append`는 컴파일러 지원이 필요한 빌트인이다.

`append`는 원소를 슬라이스 끝에 덧붙이고 결과를 반환한다. (기반 배열이 바뀔 수 있어 반드시 반환값을 받아야 한다.)

```go
x := []int{1, 2, 3}
x = append(x, 4, 5, 6)
fmt.Println(x) // [1 2 3 4 5 6]
```

슬라이스를 슬라이스에 붙이려면 호출 지점에서 `...`을 쓴다.

```go
x := []int{1, 2, 3}
y := []int{4, 5, 6}
x = append(x, y...)
fmt.Println(x) // [1 2 3 4 5 6]
```

`...` 없이는 타입이 맞지 않아(`y`는 `int`가 아님) 컴파일되지 않는다.

---

## 10. 정리

| 항목 | 핵심 |
|---|---|
| `new(T)` | 제로화된 `T`의 **포인터(`*T`)** 반환, 초기화 안 함 |
| 제로 값 설계 | 제로 값이 바로 쓸모 있게 (`bytes.Buffer`, `sync.Mutex`) |
| 복합 리터럴 | `&File{...}`, 지역 변수 주소 반환 OK, `field:value`로 부분 초기화 |
| `make(T)` | **슬라이스/맵/채널만**, 초기화된 `T` 반환(포인터 아님) |
| 배열 | **값** — 복사·크기가 타입의 일부. 보통 슬라이스를 쓴다 |
| 슬라이스 | 기반 배열 참조, 디스크립터(ptr/len/cap)는 값 전달 → 반환 필요 |
| 맵 | `==` 가능 타입만 키, comma-ok로 부재 판별, `delete` |
| 출력 | `%v`/`%+v`/`%#v`/`%T`/`%q`, `String()`로 커스터마이즈(재귀 주의) |
| append | 빌트인, 슬라이스 합치기는 `slice...` |

**한 줄 요약:** `new`는 제로화 포인터, `make`는 슬라이스·맵·채널 초기화. 배열은 값이고 슬라이스는 참조 디스크립터다. 제로 값이 바로 쓸모 있게 설계하고, comma-ok로 맵 부재를 구분하며, `String()`으로 출력을 제어하되 재귀를 조심하라.
