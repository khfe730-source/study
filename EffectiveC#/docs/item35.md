# Item 35: 확장 메서드를 절대 오버로드하지 마라 (Never Overload Extension Methods)

> **Chapter 4: Working with LINQ**

---

## 핵심 요약

확장 메서드는 오버로드 해석(overload resolution) 규칙이 인스턴스 메서드와 다르게 동작한다. 같은 이름의 확장 메서드를 여러 네임스페이스에 정의하면 `using` 지시문에 따라 어떤 메서드가 호출될지 예측하기 어렵고, 혼란스러운 버그가 발생한다.

---

## 오버로드 해석 규칙

C#의 오버로드 해석 우선순위:

```
1. 인스턴스 메서드 (항상 최우선)
2. 현재 네임스페이스의 확장 메서드
3. using으로 가져온 네임스페이스의 확장 메서드
```

확장 메서드끼리의 오버로드는 `using` 순서와 네임스페이스 위치에 따라 달라진다.

---

## 문제: 같은 이름의 확장 메서드가 여러 곳에 존재

```csharp
// 네임스페이스 A
namespace MyApp.StringUtils
{
    public static class Extensions
    {
        public static string Format(this string s) => s.Trim().ToUpper();
    }
}

// 네임스페이스 B
namespace MyApp.Logging
{
    public static class Extensions
    {
        public static string Format(this string s) => $"[LOG] {s}";
    }
}

// 사용측: 어떤 Format이 호출될지 using 순서에 따라 달라짐
using MyApp.StringUtils;
using MyApp.Logging;

string result = "hello".Format(); // 컴파일 오류: 모호한 호출!
```

---

## 더 교묘한 문제: 조용히 다른 메서드가 호출됨

```csharp
// 기존 코드
using MyApp.StringUtils;
string result = "hello".Format(); // StringUtils.Format 호출

// 나중에 다른 개발자가 추가
using MyApp.Logging; // 이 줄 추가 후 컴파일은 되지만...
string result = "hello".Format(); // Logging.Format이 호출될 수 있음!
```

컴파일 오류도 없이 동작이 바뀔 수 있다.

---

## 올바른 접근: 이름을 명확하게 구분

```csharp
// 이름 충돌을 피하기 위해 의도를 이름에 명확히 담는다
namespace MyApp.StringUtils
{
    public static class Extensions
    {
        public static string NormalizeFormat(this string s) => s.Trim().ToUpper();
    }
}

namespace MyApp.Logging
{
    public static class Extensions
    {
        public static string ToLogFormat(this string s) => $"[LOG] {s}";
    }
}
```

---

## 확장 메서드 설계 지침

```csharp
// 1. 확장 메서드 이름은 전역적으로 유일하도록 설계
// 2. 도메인 특화 접두사/접미사 활용
public static class OrderExtensions
{
    public static bool IsOrderEligibleForDiscount(this Order o) => ...;
    public static decimal CalculateOrderTotal(this Order o) => ...;
}

// 3. 동일 타입에 대한 확장 메서드는 하나의 정적 클래스에 모은다
// 4. 라이브러리 확장 메서드는 별도 네임스페이스에 격리
namespace MyLib.Extensions // 사용자가 명시적으로 using해야 함
{
    public static class StringExtensions { ... }
}
```

---

## 결론

> 확장 메서드는 오버로드하지 마라. 같은 타입에 대해 같은 이름의 확장 메서드가 여러 네임스페이스에 존재하면 `using` 지시문에 따라 동작이 달라지는 예측 불가능한 버그가 발생한다. 이름을 의도가 명확하게 드러나도록 설계해 충돌을 방지하라.
