# Item 2: const보다 readonly를 선호하라 (Prefer readonly to const)

> **Chapter 1: C# Language Idioms**

---

## 핵심 요약

`const`는 컴파일 타임 상수로 값이 소비자 어셈블리에 **인라인**된다. `readonly`는 런타임 상수로 참조를 통해 접근한다. 라이브러리/API 경계를 넘는 값에는 `const` 대신 `readonly`를 사용해야 버전 관리 문제를 피할 수 있다.

---

## const의 동작 방식

`const`는 **컴파일 타임 상수(compile-time constant)** 다. 컴파일러가 `const` 값을 참조하는 모든 코드에 값을 직접 인라인(inline)한다.

```csharp
// 라이브러리 A
public class Constants
{
    public const int MaxValue = 100;
}

// 라이브러리 B (A를 참조)
int limit = Constants.MaxValue; // 컴파일 후: int limit = 100;
```

컴파일 결과 IL에는 `Constants.MaxValue` 참조가 남지 않고 리터럴 `100`이 직접 삽입된다.

---

## const의 버전 관리 문제

```csharp
// v1.0: MaxValue = 100
public const int MaxValue = 100;

// v2.0: MaxValue = 200으로 변경
public const int MaxValue = 200;
```

라이브러리를 v2.0으로 교체해도 **소비자 코드를 재컴파일하지 않으면** 이전 값 `100`을 계속 사용한다. 배포된 바이너리에 이미 `100`이 박혀 있기 때문이다.

```
[소비자 어셈블리 컴파일 시점]   [런타임]
int limit = 100;               limit은 여전히 100
                               (라이브러리 v2.0의 200이 아님)
```

---

## readonly의 동작 방식

`readonly`는 **런타임 상수(runtime constant)** 다. 값은 라이브러리 어셈블리에 저장되고 소비자는 참조를 통해 접근한다.

```csharp
public class Constants
{
    public static readonly int MaxValue = 100;
}
```

라이브러리를 교체하면 소비자 코드 재컴파일 없이도 새 값이 반영된다.

---

## const를 사용해도 되는 경우

`const`가 적합한 경우는 **값이 절대 바뀌지 않는 진정한 상수**일 때다.

```csharp
// 수학/물리 상수 — 절대 변하지 않음
public const double Pi = 3.14159265358979;
public const int DaysInWeek = 7;

// 컴파일 타임에 값이 필요한 경우
// (특성(Attribute) 매개변수, switch case, 배열 크기 등)
[SomeAttribute(MaxRetry)]        // 특성 매개변수는 const 필수
const int MaxRetry = 3;

switch (value)
{
    case MaxRetry: break;        // switch case는 const 필수
}
```

---

## const와 readonly 비교

| 항목 | `const` | `readonly` |
|------|---------|------------|
| 결정 시점 | 컴파일 타임 | 런타임 (생성자에서 설정 가능) |
| 값 저장 위치 | 소비자 어셈블리에 인라인 | 정의 어셈블리 |
| 라이브러리 업데이트 | 소비자 재컴파일 필요 | 불필요 |
| 허용 타입 | 기본 타입, string, enum | 모든 타입 |
| 인스턴스 멤버 가능 | ❌ (static만 가능) | ✅ |
| 성능 | 약간 빠름 (인라인) | 참조 접근 |

---

## readonly의 유연성

`readonly` 필드는 **생성자에서 설정**할 수 있어 더 유연하다.

```csharp
public class Config
{
    public readonly int MaxConnections;

    public Config(int max)
    {
        MaxConnections = max; // 생성자에서 설정 가능
    }
}
```

인스턴스마다 다른 값을 가질 수 있으며, `const`로는 불가능한 패턴이다.

---

## 결론

> 퍼블릭 API나 라이브러리 경계를 넘는 상수는 `readonly`를 사용하라. `const`는 값이 절대 변하지 않는 수학/물리 상수나 컴파일 타임 상수가 반드시 필요한 경우(특성 매개변수, switch case)에만 사용한다.
