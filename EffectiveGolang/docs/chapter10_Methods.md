# Chapter 10. Methods

> 원문: [Effective Go — Methods](https://go.dev/doc/effective_go#methods)

---

## 1. 메서드는 (거의) 모든 named type에 붙는다

`ByteSize`에서 봤듯, 메서드는 **포인터나 인터페이스를 제외한 모든 named type**에 정의할 수 있다. **리시버(receiver)가 구조체일 필요는 없다.**

앞서 만든 `Append` 함수를 슬라이스의 메서드로 정의해 보자. 먼저 메서드를 묶을 named type을 선언하고, 리시버를 그 타입의 값으로 한다.

```go
type ByteSlice []byte

func (slice ByteSlice) Append(data []byte) []byte {
    // 본문은 앞의 Append 함수와 동일
}
```

이 방식은 여전히 **갱신된 슬라이스를 반환**해야 한다.

---

## 2. Pointers vs. Values — 리시버 선택

리시버를 **`*ByteSlice` 포인터**로 바꾸면 메서드가 호출자의 슬라이스를 덮어쓸 수 있어 반환이 필요 없어진다.

```go
func (p *ByteSlice) Append(data []byte) {
    slice := *p
    // 본문 (return 없음)
    *p = slice
}
```

더 나아가 표준 `Write` 메서드처럼 만들면:

```go
func (p *ByteSlice) Write(data []byte) (n int, err error) {
    slice := *p
    // 위와 동일
    *p = slice
    return len(data), nil
}
```

이제 **`*ByteSlice`는 표준 인터페이스 `io.Writer`를 만족**한다. 그래서 여기에 직접 출력할 수 있다.

```go
var b ByteSlice
fmt.Fprintf(&b, "This hour has %d days\n", 7)
```

`*ByteSlice`만 `io.Writer`를 만족하므로 **`ByteSlice`의 주소를 넘긴다.**

### 핵심 규칙

> **값 메서드(value method)는 포인터와 값 양쪽에서 호출 가능하지만, 포인터 메서드(pointer method)는 포인터에서만 호출 가능하다.**

이 규칙이 생기는 이유: 포인터 메서드는 **리시버를 수정**할 수 있다. 값에 대해 호출하면 메서드가 값의 **복사본**을 받게 되어 수정이 버려지므로, 언어가 이 실수를 금지한다.

### 편리한 예외 — 주소 지정 가능(addressable)하면 자동 변환

값이 **addressable**하면, 값에 대한 포인터 메서드 호출을 컴파일러가 **주소 연산자를 자동 삽입**해 처리한다.

```go
// b는 addressable 변수이므로
b.Write(...)
// 컴파일러가 (&b).Write(...)로 재작성해 준다
```

> 참고: 바이트 슬라이스에 `Write`를 정의하는 이 아이디어가 `bytes.Buffer` 구현의 핵심이다.

---

## 3. 정리

| 항목 | 핵심 |
|---|---|
| 메서드 대상 | 포인터·인터페이스 빼고 **모든 named type** (구조체 아니어도 됨) |
| 값 리시버 | 복사본을 받음 → 수정 불가, 포인터·값 모두에서 호출 가능 |
| 포인터 리시버 | 리시버 수정 가능, **포인터에서만 호출 가능** (인터페이스 만족도 `*T`만) |
| addressable 예외 | 값이 addressable이면 `b.Write` → `(&b).Write` 자동 변환 |

**한 줄 요약:** 리시버를 포인터로 하면 수정이 호출자에게 반영되고 `*T`가 인터페이스를 만족한다. 값 메서드는 값·포인터 모두에서, 포인터 메서드는 포인터에서만(단 addressable 값은 자동 주소화) 호출된다.
