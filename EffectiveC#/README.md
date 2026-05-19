# Effective C# (3rd Edition) - Bill Wagner

> C# 6.0 기준, 50가지 구체적인 개선 방법을 다루는 전문가용 C# 심화 가이드.  
> 단순한 문법 소개가 아닌, CLR/GC/언어 설계 원칙에 기반한 실천적 조언을 담는다.

---

## Chapter 1: C# 언어 이디엄 (C# Language Idioms)

| Item | 제목 | 핵심 키워드 |
|------|------|-------------|
| [01](./docs/item01.md) | Prefer Implicitly Typed Local Variables | `var`, 암묵적 타입 추론 |
| [02](./docs/item02.md) | Prefer readonly to const | `readonly` vs `const`, 버전 관리 |
| [03](./docs/item03.md) | Prefer the is or as Operators to Casts | `is` / `as`, 패턴 매칭 |
| [04](./docs/item04.md) | Replace string.Format() with Interpolated Strings | 문자열 보간 `$"..."` |
| [05](./docs/item05.md) | Prefer FormattableString for Culture-Specific Strings | `FormattableString`, Culture |
| [06](./docs/item06.md) | Avoid String-ly Typed APIs | `nameof`, `CallerMemberName` |
| [07](./docs/item07.md) | Express Callbacks with Delegates | `Action`, `Func`, 델리게이트 |
| [08](./docs/item08.md) | Use the Null Conditional Operator for Event Invocations | `?.Invoke()`, 레이스 컨디션 |
| [09](./docs/item09.md) | Minimize Boxing and Unboxing | 박싱/언박싱, 제네릭 |
| [10](./docs/item10.md) | Use the new Modifier Only to React to Base Class Updates | `new` vs `override` |

**핵심 요약:** `var`는 동적 타입이 아닌 컴파일 타임 정적 타입이다. API 경계를 넘는 상수는 `readonly`를 쓴다. 이벤트 호출은 `?.Invoke()`로 스레드 안전성을 확보하고, 값 타입의 박싱은 제네릭으로 제거하라.

---

## Chapter 2: .NET 리소스 관리 (.NET Resource Management)

| Item | 제목 | 핵심 키워드 |
|------|------|-------------|
| [11](./docs/item11.md) | Understand .NET Resource Management | GC, 파이널라이저, IDisposable |
| [12](./docs/item12.md) | Prefer Member Initializers to Assignment Statements | 멤버 이니셜라이저 |
| [13](./docs/item13.md) | Use Proper Initialization for Static Class Members | 정적 생성자 |
| [14](./docs/item14.md) | Minimize Duplicate Initialization Logic | 생성자 체이닝 `this()` |
| [15](./docs/item15.md) | Avoid Creating Unnecessary Objects | 객체 생성 최소화 |
| [16](./docs/item16.md) | Never Call Virtual Functions in Constructors | 생성자 내 가상 함수 호출 금지 |
| [17](./docs/item17.md) | Implement the Standard Dispose Pattern | 표준 Dispose 패턴 |

**핵심 요약:** GC는 관리 메모리만 책임진다. 비관리 리소스는 `IDisposable` + 표준 Dispose 패턴으로 해제하라. 생성자 초기화는 이니셜라이저 → `this()` 체이닝 순서로 설계하고, 생성자에서 가상 함수를 절대 호출하지 마라.

---

## Chapter 3: 제네릭 활용 (Working with Generics)

| Item | 제목 | 핵심 키워드 |
|------|------|-------------|
| [18](./docs/item18.md) | Always Define Constraints That Are Necessary and Sufficient | 제약 조건, `where` |
| [19](./docs/item19.md) | Specialize Generic Algorithms Using Runtime Type Checking | 런타임 타입 검사, 최적화 |
| [20](./docs/item20.md) | Implement Ordering Relations with IComparable\<T\> and IComparer\<T\> | 순서 관계, 정렬 |
| [21](./docs/item21.md) | Always Create Generic Classes That Support Disposable Type Parameters | IDisposable, 리소스 관리 |
| [22](./docs/item22.md) | Support Generic Covariance and Contravariance | `out`, `in`, 공변성, 반공변성 |
| [23](./docs/item23.md) | Use Delegates to Define Method Constraints on Type Parameters | 델리게이트 주입, 연산자 제약 |
| [24](./docs/item24.md) | Do Not Use IEnumerable\<T\> as a Return Type for Property Accessors | 속성 반환 타입, 지연 실행 |
| [25](./docs/item25.md) | Prefer Generic Methods Unless Type Parameters Are Instance Fields | 제네릭 메서드, 타입 추론 |
| [26](./docs/item26.md) | Implement Classic Interfaces in Addition to Generic Interfaces | 비제네릭 하위 호환, IComparable |
| [27](./docs/item27.md) | Augment Minimal Interface Contracts with Extension Methods | 확장 메서드, 최소 인터페이스 |
| [28](./docs/item28.md) | Consider Enhancing Constructed Types with Extension Methods | 구체화된 타입 강화 |

**핵심 요약:** 제약 조건은 필요한 것만 정확하게 정의하라. `out`/`in`으로 공변·반공변성을 선언해 API 유연성을 높여라. 인터페이스는 최소 계약만 담고, 편의 기능은 확장 메서드로 제공하라 (LINQ의 철학).

---

## Chapter 4: LINQ 활용 (Working with LINQ)
> ⚠️ 미작성 (Item 29~42)

## Chapter 5: 예외 처리 (Exception Practices)
> ⚠️ 미작성 (Item 43~50)

---

## 참고

- **저자**: Bill Wagner
- **버전**: 3rd Edition (C# 6.0 기준, 2016)
- **시리즈**: Scott Meyers의 Effective 시리즈
- **Roslyn 분석기**: https://github.com/BillWagner/EffectiveCSharpAnalyzers
