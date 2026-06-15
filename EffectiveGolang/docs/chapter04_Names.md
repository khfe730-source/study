# Chapter 04. Names

> 원문: [Effective Go — Names](https://go.dev/doc/effective_go#names)

---

## 0. 이름은 의미를 가진다 (가시성 규칙)

Go에서 이름(name)은 다른 언어만큼 중요하며, **심지어 의미론적(semantic) 효과**까지 가진다.

> **패키지 밖에서의 가시성(visibility)은 이름의 첫 글자가 대문자인지로 결정된다.**

- 첫 글자 **대문자** → 익스포트(exported), 패키지 외부에서 접근 가능
- 첫 글자 **소문자** → 언익스포트(unexported), 패키지 내부 전용

이 단순한 규칙이 `public`/`private` 키워드를 대체한다. 따라서 네이밍은 단순한 스타일이 아니라 **API 설계 그 자체**다. 이 챕터는 네 가지 관례를 다룬다: 패키지 이름, getter, 인터페이스 이름, MixedCaps.

---

## 1. Package names — 짧고 간결하고 환기적으로

패키지를 임포트하면 **패키지 이름이 그 내용에 접근하는 식별자(accessor)**가 된다.

```go
import "bytes"
// 이후 bytes.Buffer 로 참조
```

### 규칙

- **짧고(short), 간결하고(concise), 환기적인(evocative)** 이름. 모든 사용자가 매번 타이핑하므로 **간결함 쪽으로 기울여라.**
- **소문자 한 단어(lower case, single-word).** 언더스코어나 mixedCaps 불필요.
- **충돌(collision)을 미리 걱정하지 말 것.** 패키지 이름은 임포트의 *기본* 이름일 뿐 전역적으로 유일할 필요가 없다. 드물게 충돌하면 임포트 측에서 로컬 별칭을 정하면 된다.
- 패키지 이름 = **소스 디렉토리의 base name.** `src/encoding/base64`는 `"encoding/base64"`로 임포트하지만 이름은 `base64`다 (`encoding_base64`나 `encodingBase64`가 아님).

### 핵심: 패키지 이름과 익스포트 이름은 함께 읽힌다

사용자는 항상 `패키지명.식별자`로 접근하므로, **익스포트 이름에서 패키지 이름을 반복하지 마라.**

```go
bufio.Reader   // NOT bufio.BufReader — 사용자는 "bufio.Reader"로 보므로 이미 명확
ring.New()     // NOT ring.NewRing — ring이 유일한 타입이므로 생성자는 그냥 New
once.Do(setup) // NOT once.DoOrWaitUntilDone(setup) — 길다고 더 읽기 쉬운 게 아니다
```

- `bufio.Reader`는 `io.Reader`와 충돌하지 않는다 — 항상 패키지 이름과 함께 쓰이기 때문.
- Go에서 **생성자(constructor)의 관용은 `NewXxx`**, 단 타입이 하나뿐이면 그냥 `New`.

> 교훈: **긴 이름이 자동으로 가독성을 높이지 않는다.** 좋은 doc comment 한 줄이 긴 이름보다 가치 있을 때가 많다. 패키지 구조를 이용해 이름을 짧게 유지하라.
>
> 참고: `import .` 표기(점 임포트)는 패키지 외부에서 도는 테스트를 단순화할 때만 예외적으로 쓰고, 그 외엔 피한다.

---

## 2. Getters — 이름에 `Get`을 넣지 마라

Go는 getter/setter를 자동 지원하지 않는다. 직접 제공해도 되고 종종 적절하지만, **이름에 `Get`을 붙이는 것은 관용적이지도 필요하지도 않다.**

- 언익스포트 필드 `owner` → getter는 `Owner()` (`GetOwner` 아님)
- setter가 필요하면 `SetOwner()`

대문자 익스포트 규칙이 **필드와 메서드를 구분하는 훅(hook)** 역할을 하기 때문에 `Get` 접두사가 불필요하다.

```go
owner := obj.Owner()
if owner != user {
    obj.SetOwner(user)
}
```

두 이름 모두 실무에서 자연스럽게 읽힌다.

---

## 3. Interface names — `-er` 접미사

관례상 **메서드가 하나인 인터페이스는 "메서드 이름 + `-er` 접미사"**로 행위 주체(agent noun)를 만든다.

```go
Reader, Writer, Formatter, CloseNotifier // Read/Write/Format/... + er
```

### 정준 시그니처(canonical signature)를 존중하라

`Read`, `Write`, `Close`, `Flush`, `String` 등은 **정해진 시그니처와 의미**를 가진다.

- **혼란 방지:** 같은 시그니처·의미가 아니라면 이 이름들을 메서드에 쓰지 마라.
- **반대로:** 잘 알려진 타입의 메서드와 같은 의미라면 **같은 이름·시그니처를 줘라.** 문자열 변환 메서드는 `ToString`이 아니라 **`String`**으로 (이것이 `fmt.Stringer` 인터페이스를 만족시킨다).

```go
type Stringer interface {
    String() string
}
// 따라서 직접 만든 타입의 문자열 변환도 String() string 로 — fmt 패키지가 자동 인식
```

---

## 4. MixedCaps — 언더스코어 대신 카멜케이스

여러 단어로 된 이름은 **언더스코어가 아니라 `MixedCaps` 또는 `mixedCaps`**로 쓴다.

```go
maxLength    // O  (언익스포트)
MaxLength    // O  (익스포트)
max_length   // X
MAX_LENGTH   // X (상수도 마찬가지 — Go는 상수에 대문자+언더스코어를 쓰지 않는다)
```

첫 글자의 대소문자가 가시성을 결정하므로, 단어 구분은 대소문자 전환으로 처리한다.

---

## 5. 정리

| 항목 | 관례 |
|---|---|
| 가시성 | 첫 글자 **대문자=익스포트**, 소문자=언익스포트 |
| 패키지 이름 | 짧은 **소문자 한 단어**, 디렉토리 base name, 익스포트 이름과 함께 읽히게 |
| 생성자 | `NewXxx`, 타입이 하나면 `New` |
| Getter/Setter | `Owner()` / `SetOwner()` — **`Get` 금지** |
| 인터페이스 | 단일 메서드는 **`-er`** (`Reader`), 정준 이름 시그니처 존중 |
| 문자열 변환 | **`String()`** (`ToString` 아님) — `fmt.Stringer` 만족 |
| 다단어 이름 | **MixedCaps**, 언더스코어 금지 |

**한 줄 요약:** Go에서 이름은 가시성을 결정하는 의미론적 장치다. `패키지명.식별자`가 함께 읽히도록 짧게 짓고, `Get` 접두사·언더스코어를 버리고, `-er`·`String()` 같은 정준 관례를 따르면 다른 Go 코드와 자연스럽게 맞물린다.
