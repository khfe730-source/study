# Effective Go — The Go Authors

> [go.dev/doc/effective_go](https://go.dev/doc/effective_go)
> 문법적으로 맞는 코드를 넘어 **"Go 답게(idiomatic)" 명료한 코드**를 쓰는 법을 다루는 관용구 가이드.
> (2009년 작성 기준 문서로, 제네릭·모듈 등 이후 변화는 포함하지 않는다.)

---

## 챕터 목록

| 챕터 | 제목 | 핵심 키워드 |
|------|------|-------------|
| [01](./docs/chapter01_Introduction.md) | Introduction | 관용구 가이드, "번역하지 말고 Go로 사고하라", 표준 라이브러리 예제 |
| [02](./docs/chapter02_Formatting.md) | Formatting | `gofmt`, 탭 들여쓰기, 줄 길이 무제한, 괄호 최소화 |
| [03](./docs/chapter03_Commentary.md) | Commentary | `//`/`/* */`, doc comment, `go doc`, 자동 문서 생성 |
| [04](./docs/chapter04_Names.md) | Names | 대문자=익스포트, 패키지 이름, getter(`Get` 금지), `-er` 인터페이스, MixedCaps |
| [05](./docs/chapter05_Semicolons.md) | Semicolons | 자동 세미콜론 삽입, 여는 중괄호 같은 줄 강제 |
| [06](./docs/chapter06_ControlStructures.md) | Control structures | `if` 초기화·else 생략, `for`/`range`, `switch`(폴스루 없음), 레이블, type switch |
| [07](./docs/chapter07_Functions.md) | Functions | 다중 반환값, 명명된 결과·naked return, `defer`(LIFO·인자 평가 시점) |
| [08](./docs/chapter08_Data.md) | Data | `new`/`make`, 복합 리터럴, 배열·슬라이스·맵, 출력(`%v`/`String()`), `append` |
| [09](./docs/chapter09_Initialization.md) | Initialization | 상수·`iota`, 변수, `init` 순서 |
| [10](./docs/chapter10_Methods.md) | Methods | 리시버 값 vs 포인터, addressable 자동 변환 |
| [11](./docs/chapter11_Interfaces.md) | Interfaces and other types | 암묵 구현, 변환, 타입 단언, generality, `HandlerFunc` |
| [12](./docs/chapter12_BlankIdentifier.md) | The blank identifier | `_` 다중할당·미사용·부수효과 임포트·인터페이스 보증 |
| [13](./docs/chapter13_Embedding.md) | Embedding | 메서드 승격, 인터페이스/구조체 임베딩, vs 서브클래싱 |
| [14](./docs/chapter14_Concurrency.md) | Concurrency | 고루틴, 채널, 세마포어, 채널의 채널, 병렬화, `select` |
| [15](./docs/chapter15_Errors.md) | Errors | `error` 인터페이스, `panic`, `recover` |
| [16](./docs/chapter16_AWebServer.md) | A web server | QR 서버 예제, `HandlerFunc`, `html/template` |

**핵심 요약:** *Effective Go*는 명세가 아니라 관용구 가이드다. C++/Java를 1:1로 옮기지 말고, 인터페이스·구성·임베딩, 명시적 `error`, 고루틴·채널이라는 Go의 모델로 다시 설계하라. 좋은 Go = 언어 속성 이해 + 공유된 관례(일관성) 준수. 표준 라이브러리 소스와 실행 가능한 `ExampleXxx`가 최고의 교재다.
