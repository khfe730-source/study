# Chapter 01. Introduction

> 원문: [Effective Go — Introduction](https://go.dev/doc/effective_go#introduction)

> **주의 (원문 명시):** 이 문서는 2009년 Go 릴리스 시점에 작성되었고 이후 적극적으로 갱신되지 않았다. 핵심 언어를 익히는 가이드로는 여전히 유효하지만, 제네릭(generics), 모듈(modules), 그 이후 추가된 라이브러리 등 언어·생태계의 중요한 변화는 다루지 않는다. 이 점을 전제로 읽되, 실무에서는 최신 [release notes](https://go.dev/doc/devel/release)와 함께 보는 것을 권한다.

---

## 1. 이 문서의 위치

`Effective Go`는 **명세(spec)나 튜토리얼이 아니라 "관용구(idiom) 가이드"**다. 원문은 다음 세 문서를 먼저 읽었다고 전제한다.

- [Language Specification](https://go.dev/ref/spec) — 언어의 정확한 규칙
- [Tour of Go](https://go.dev/tour/) — 인터랙티브 입문
- [How to Write Go Code](https://go.dev/doc/code) — 패키지/모듈 구성과 빌드 워크플로

즉 *Effective Go*는 "문법적으로 맞는 코드"를 넘어 **"Go 답게(idiomatic) 명료한 코드"를 쓰는 법**을 다룬다. 이 구분이 책 전체의 톤을 결정한다.

---

## 2. 핵심 주장: "Go로 번역하지 말고 Go로 사고하라"

원문에서 가장 자주 인용되는 문장:

> A straightforward translation of a C++ or Java program into Go is unlikely to produce a satisfactory result — **Java programs are written in Java, not Go.**

Go는 단순성(simplicity), 신뢰성(reliability), 효율성(efficiency)에 초점을 두고 **대규모 소프트웨어를 쉽게 만들기 위해** 설계되었다. 기존 언어에서 아이디어를 빌려왔지만, 그 결과 효과적인 Go 프로그램은 친척 언어(C++, Java 등)로 작성한 프로그램과 **성격 자체가 다르다.**

C++/Java 프로그램을 1:1로 옮기면 동작은 하더라도 비-관용적(non-idiomatic)이고 어색한 결과가 나온다. 같은 문제라도 **Go의 관점에서 다시 생각하면** 성공적이지만 꽤 다른 구조의 프로그램이 나온다.

전문가 관점에서 이 명제가 구체적으로 의미하는 바:

| 다른 언어의 습관 | Go의 관용 | 이유 |
|---|---|---|
| 깊은 클래스 상속 계층 | 작은 인터페이스 + 구성(composition), 임베딩(embedding) | Go에는 상속이 없다. 동작은 인터페이스로, 재사용은 임베딩으로 |
| try/catch 예외 전파 | 다중 반환값으로 `error`를 명시적으로 반환·검사 | 제어 흐름이 눈에 보이게 강제됨 |
| 스레드 + 락(lock) 중심 동시성 | 고루틴(goroutine) + 채널(channel)로 "통신으로 공유" | *Share memory by communicating, don't communicate by sharing memory* |
| getter/setter 보일러플레이트 | 필드 직접 노출, 필요할 때만 메서드 | `GetX()`가 아니라 `X()` 네이밍 (뒤 챕터 Getters 참고) |
| 포매팅 스타일 논쟁 | `gofmt`가 강제하는 단일 스타일 | 스타일은 기계가 처리, 사람은 로직에 집중 |

핵심은 단순히 "문법을 바꾸는" 것이 아니라 **설계 단위(상속 vs 구성, 예외 vs 명시적 에러, 스레드 vs 고루틴)를 Go의 모델로 바꾸는 것**이다.

---

## 3. "Go를 잘 쓴다"는 것의 두 축

원문은 좋은 Go 코드의 조건을 두 가지로 정리한다.

1. **언어의 속성과 관용구(properties and idioms)를 이해하기** — 무엇이 가능하고 무엇이 자연스러운지.
2. **확립된 관례(established conventions)를 따르기** — 네이밍(naming), 포매팅(formatting), 프로그램 구성(program construction) 등. 이를 지켜야 **다른 Go 프로그래머가 내 코드를 쉽게 이해**한다.

두 번째 축이 중요하다. Go 커뮤니티는 "개인 취향"보다 **일관성(consistency)**을 강하게 선호한다. `gofmt`, `go vet`, 표준 네이밍 규칙 등이 모두 이 철학의 산물이며, *Effective Go*의 이후 챕터(Formatting, Commentary, Names, Semicolons …)는 대부분 이 "공유된 관례"를 설명하는 데 할애된다.

---

## 4. Examples — 표준 라이브러리를 교재로 삼아라

> 원문: [Examples](https://go.dev/doc/effective_go#examples)

Go 패키지의 소스 코드는 **핵심 라이브러리인 동시에 "언어 사용 예제집"으로 의도되었다.** 표준 라이브러리는 의도적으로 읽기 쉽고 관용적으로 작성되어 있어, "이 문제를 어떻게 접근하지?", "이건 보통 어떻게 구현하지?"라는 질문에 대한 답·아이디어·배경을 직접 제공한다.

또한 많은 패키지는 **실행 가능한 자체 완결형 예제(executable examples)**를 포함한다. 이것은 단순 주석이 아니라 도구 체인에 통합된 기능이다.

```go
// example_test.go
package strings_test

import (
    "fmt"
    "strings"
)

// 함수 이름이 Example... 로 시작하면 go test가 예제로 인식하고,
// 함께 적힌 "// Output:" 주석과 실제 출력을 비교해 검증까지 한다.
func ExampleToUpper() {
    fmt.Println(strings.ToUpper("hello"))
    // Output: HELLO
}
```

전문가 관점의 포인트:

- `ExampleXxx` 함수는 **godoc 문서에 그대로 노출**되고, **`go test`가 `// Output:` 주석과 대조해 회귀 테스트로도 동작**한다. 즉 문서·예제·테스트가 한 몸이다.
- 따라서 표준 라이브러리(`net/http`, `io`, `strings`, `sync` 등)를 읽는 것은 *Effective Go*가 권하는 가장 실전적인 학습법이다. 관용구는 설명으로 배우는 것보다 **잘 쓰인 코드를 읽으며** 체득된다.

---

## 5. 요약

- *Effective Go*는 명세가 아니라 **관용구(idiom) 가이드**다. spec·Tour·How to Write Go Code를 먼저 읽은 전제 위에 선다.
- 핵심 메시지: **다른 언어를 번역하지 말고 Go의 모델(인터페이스·구성·임베딩, 명시적 에러, 고루틴·채널)로 다시 설계하라.**
- 좋은 Go의 두 축 = **언어의 속성·관용구 이해** + **공유된 관례 준수**(일관성).
- **표준 라이브러리 소스와 실행 가능한 예제(`ExampleXxx`)**가 최고의 교재이며, 문서·예제·테스트가 통합되어 있다.

> 다음 챕터(Formatting)부터는 이 "공유된 관례"를 `gofmt`를 시작으로 하나씩 구체화한다.
