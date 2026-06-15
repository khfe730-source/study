# Chapter 03. Commentary

> 원문: [Effective Go — Commentary](https://go.dev/doc/effective_go#commentary)
> 보충: [Go Doc Comments](https://go.dev/doc/comment)

---

## 1. 두 가지 주석 문법

Go는 C 계열의 두 주석 스타일을 모두 제공한다.

```go
/* C 스타일 블록 주석 (block comment) */
// C++ 스타일 라인 주석 (line comment)
```

관용적 사용 구분:

| 종류 | 관용적 용도 |
|---|---|
| 라인 주석 `//` | **기본(norm).** 대부분의 주석은 라인 주석으로 작성한다. |
| 블록 주석 `/* */` | 주로 **패키지 주석(package comment)**. 그 외에 표현식 중간에 끼우거나, **큰 코드 블록을 일시적으로 비활성화**할 때 유용하다. |

전문가 관점:

- 일상적인 주석은 거의 항상 `//`다. `gofmt`도 라인 주석 정렬을 자연스럽게 처리한다.
- 블록 주석은 **중첩되지 않는다(non-nesting).** 안에 `*/`가 있으면 거기서 닫혀버리므로, 코드 비활성화 용도로 쓸 때 주의한다. 코드를 임시로 끄려면 보통 라인 주석이나 빌드 제약(build constraint)을 쓰는 편이 안전하다.

---

## 2. 핵심: 문서 주석(Doc Comments)

Go 주석의 진짜 무게중심은 **doc comment**다.

> **최상위 선언(top-level declaration) 바로 앞에, 사이에 빈 줄(intervening newline) 없이** 오는 주석은 그 선언 자체를 문서화하는 것으로 간주된다.

이런 "doc comment"는 해당 Go **패키지나 명령(command)의 1차 문서(primary documentation)**가 된다. 즉 주석이 곧 API 문서다 — 별도의 문서 시스템이 아니라 **소스의 주석에서 문서가 자동 생성**된다(`go doc`, pkg.go.dev).

```go
// Package strings implements simple functions to manipulate UTF-8 encoded strings.
package strings

// Reverse returns its argument string reversed rune-wise left to right.
func Reverse(s string) string { ... }
```

위에서 `package strings` 앞 주석은 패키지 문서가 되고, `Reverse` 앞 주석은 함수 문서가 된다.

### 빈 줄이 의미를 바꾼다

"사이에 빈 줄 없이"가 규칙의 핵심이다.

```go
// 이 주석은 Foo의 doc comment 다 (붙어 있음)
func Foo() {}

// 이 주석은 Bar의 문서가 아니다 — 아래 빈 줄 때문에 그냥 일반 주석으로 취급
                                  // (빈 줄이 선언과 주석을 분리)

func Bar() {}
```

---

## 3. Doc Comment 관용 규칙 (보충: Go Doc Comments)

원문 본문은 짧지만, 실무에서 통용되는 핵심 규칙은 다음과 같다.

1. **주석은 문서화 대상의 이름으로 시작한다.**
   `// Reverse returns ...`처럼 함수명/타입명으로 문장을 연다. 그래야 `go doc` 출력과 검색에서 자연스럽게 읽힌다.

2. **완전한 문장(complete sentences)으로 쓴다.**
   대문자로 시작하고 마침표로 끝낸다. 첫 문장은 한 줄 요약(summary)으로, pkg.go.dev의 목록에 그대로 노출된다.

3. **패키지 주석은 패키지를 한 군데에서만** 단다(보통 `doc.go` 또는 대표 파일). `// Package xxx ...`로 시작한다.

4. **포매팅 관용** (Go 1.19+ `gofmt`가 인식):
   - `#`로 시작하는 줄 → 제목(heading)
   - 들여쓴 블록 → 사전 서식 코드(preformatted code)
   - `https://...` → 자동 링크
   - 빈 줄로 구분된 문단(paragraph)

```go
// Package json implements encoding and decoding of JSON.
//
// # Examples
//
// Marshal converts a value to JSON:
//
//	b, err := json.Marshal(v)
//
// See https://pkg.go.dev/encoding/json for details.
package json
```

---

## 4. 정리

| 항목 | 내용 |
|---|---|
| 주석 문법 | `//`(기본), `/* */`(패키지 주석·코드 비활성화) |
| 블록 주석 주의 | 중첩 불가(non-nesting) |
| Doc comment 조건 | 최상위 선언 **바로 앞**, **빈 줄 없이** |
| 역할 | 패키지/명령의 **1차 문서** — `go doc`·pkg.go.dev로 자동 생성 |
| 작성 규칙 | 대상 **이름으로 시작**, 완전한 문장, 첫 문장은 요약 |

**한 줄 요약:** Go에서 주석은 단순 메모가 아니라 **API 문서 그 자체**다. 최상위 선언에 빈 줄 없이 붙이고 대상 이름으로 시작하는 문장으로 작성하면, 그대로 공식 문서가 된다.
