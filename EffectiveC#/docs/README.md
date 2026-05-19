# Effective C# (3rd Edition) - Bill Wagner

> C# 6.0 기준, 50가지 구체적인 개선 방법을 다루는 전문가용 C# 심화 가이드.  
> 단순한 문법 소개가 아닌, CLR/GC/언어 설계 원칙에 기반한 실천적 조언을 담는다.

---

## 목차

### Chapter 1: C# 언어 이디엄 (C# Language Idioms)

| Item | 제목 | 핵심 키워드 |
|------|------|-------------|
| [01](./item01.md) | Prefer Implicitly Typed Local Variables | `var`, 암묵적 타입 추론 |
| [02](./item02.md) | Prefer readonly to const | `readonly` vs `const`, 버전 관리 |
| [03](./item03.md) | Prefer the is or as Operators to Casts | `is` / `as`, 패턴 매칭 |
| [04](./item04.md) | Replace string.Format() with Interpolated Strings | 문자열 보간 `$"..."` |
| [05](./item05.md) | Prefer FormattableString for Culture-Specific Strings | `FormattableString`, Culture |
| [06](./item06.md) | Avoid String-ly Typed APIs | `nameof`, `CallerMemberName` |
| [07](./item07.md) | Express Callbacks with Delegates | `Action`, `Func`, 델리게이트 |
| [08](./item08.md) | Use the Null Conditional Operator for Event Invocations | `?.Invoke()`, 레이스 컨디션 |
| [09](./item09.md) | Minimize Boxing and Unboxing | 박싱/언박싱, 제네릭 |
| [10](./item10.md) | Use the new Modifier Only to React to Base Class Updates | `new` vs `override` |

**Chapter 1 핵심 요약:**
- `var`는 동적 타입이 아닌 컴파일 타임 정적 타입이며, LINQ 쿼리처럼 우변에서 타입이 명확할 때 사용한다.
- API 경계를 넘는 상수는 `const` 대신 `readonly`를 사용해 버전 관리 문제를 피한다.
- 타입 변환 실패 가능성이 있으면 캐스트 대신 `as` 또는 `is` 패턴 매칭을 사용한다.
- 이벤트 호출은 `?.Invoke()`로 null 안전성과 스레드 안전성을 동시에 확보한다.
- 값 타입의 박싱은 GC 부담을 키운다 — 제네릭으로 박싱을 제거하라.

---

### Chapter 2: .NET 리소스 관리 (.NET Resource Management)

| Item | 제목 | 핵심 키워드 |
|------|------|-------------|
| [11](./item11.md) | Understand .NET Resource Management | GC, 파이널라이저, IDisposable |
| [12](./item12.md) | Prefer Member Initializers to Assignment Statements | 멤버 이니셜라이저 |
| [13](./item13.md) | Use Proper Initialization for Static Class Members | 정적 생성자 |
| [14](./item14.md) | Minimize Duplicate Initialization Logic | 생성자 체이닝 `this()` |
| [15](./item15.md) | Avoid Creating Unnecessary Objects | 객체 생성 최소화 |
| [16](./item16.md) | Never Call Virtual Functions in Constructors | 생성자 내 가상 함수 호출 금지 |
| [17](./item17.md) | Implement the Standard Dispose Pattern | 표준 Dispose 패턴 |

**Chapter 2 핵심 요약:**
- GC는 관리 메모리를 책임지지만, DB 연결·GDI 객체 등 비관리 리소스는 개발자가 직접 해제해야 한다.
- 파이널라이저는 비결정론적이며 GC 성능에 부담을 준다 → `IDisposable` + 표준 Dispose 패턴으로 대체한다.
- 생성자 초기화는 이니셜라이저 → 생성자 체이닝(`this()`) 순서로 설계하면 중복과 버그를 줄인다.
- 생성자에서 가상 함수를 호출하면 파생 클래스의 미완성 상태에서 메서드가 실행되어 예측 불가한 결과가 발생한다.

---

### Chapter 3: 제네릭 활용 (Working with Generics)

| Item | 제목 | 핵심 키워드 |
|------|------|-------------|
| [18](./item18.md) | Always Define Constraints That Are Necessary and Sufficient | 제약 조건, `where` |
| [19](./item19.md) | Specialize Generic Algorithms Using Runtime Type Checking | 런타임 타입 검사, 최적화 |
| [20](./item20.md) | Implement Ordering Relations with IComparable\<T\> and IComparer\<T\> | 순서 관계, 정렬 |
| [21](./item21.md) | Always Create Generic Classes That Support Disposable Type Parameters | IDisposable, 리소스 관리 |
| [22](./item22.md) | Support Generic Covariance and Contravariance | `out`, `in`, 공변성, 반공변성 |
| [23](./item23.md) | Use Delegates to Define Method Constraints on Type Parameters | 델리게이트 주입, 연산자 제약 |
| [24](./item24.md) | Do Not Use IEnumerable\<T\> as a Return Type for Property Accessors | 속성 반환 타입, 지연 실행 |
| [25](./item25.md) | Prefer Generic Methods Unless Type Parameters Are Instance Fields | 제네릭 메서드, 타입 추론 |
| [26](./item26.md) | Implement Classic Interfaces in Addition to Generic Interfaces | 비제네릭 하위 호환, IComparable |
| [27](./item27.md) | Augment Minimal Interface Contracts with Extension Methods | 확장 메서드, 최소 인터페이스 |
| [28](./item28.md) | Consider Enhancing Constructed Types with Extension Methods | 구체화된 타입 강화 |

**Chapter 3 핵심 요약:**
- 제약 조건은 필요한 것만 정확하게 정의하라. 부족하면 컴파일 오류, 과하면 재사용성 저하.
- 제네릭 알고리즘의 일반 경로를 유지하면서, 특정 타입에 대한 최적화 경로를 런타임 타입 검사로 추가하라.
- `out`(공변)과 `in`(반공변)으로 제네릭 인터페이스의 유연성을 높여라.
- 인터페이스는 최소한의 계약만 담고, 편의 기능은 확장 메서드로 제공하라 (LINQ의 설계 철학).

### Chapter 4: LINQ 활용 (Working with LINQ)
> ⚠️ 현재 PDF에 포함되어 있지 않음 (Item 29~42)

### Chapter 5: 예외 처리 (Exception Practices)
> ⚠️ 현재 PDF에 포함되어 있지 않음 (Item 43~50)

---

## 참고

- **저자**: Bill Wagner
- **버전**: 3rd Edition (C# 6.0 기준, 2016)
- **시리즈**: Scott Meyers의 Effective 시리즈
- **Roslyn 분석기**: https://github.com/BillWagner/EffectiveCSharpAnalyzers
