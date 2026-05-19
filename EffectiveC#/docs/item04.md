# Item 4: string.Format() 대신 보간 문자열을 사용하라 (Replace string.Format() with Interpolated Strings)

> **Chapter 1: C# Language Idioms**

---

## 핵심 요약

C# 6.0에 추가된 문자열 보간(string interpolation)은 `string.Format()`보다 가독성이 높고 오류가 적다. 인덱스 번호 대신 변수명을 직접 사용하므로 순서 오류나 인수 누락 버그를 컴파일 타임에 방지할 수 있다.

---

## string.Format()의 문제점

```csharp
// 인덱스 기반: {0}, {1}, {2} 순서를 사람이 추적해야 함
string s = string.Format(
    "안녕하세요, {0}님. 오늘은 {1}이고, 잔액은 {2:C}입니다.",
    name, DateTime.Now, balance);

// 문제 1: 인수 순서 실수
string s = string.Format("{0} + {1} = {2}", a, b, a + b);
// 나중에 인수를 추가하다가 순서를 바꾸면 버그 발생

// 문제 2: 인수 개수 불일치 → 런타임 예외
string s = string.Format("{0} {1} {2}", a, b); // FormatException 발생
```

---

## 문자열 보간 (String Interpolation)

```csharp
// $ 접두사 + {표현식} 형태
string s = $"안녕하세요, {name}님. 오늘은 {DateTime.Now}이고, 잔액은 {balance:C}입니다.";
```

변수명이 직접 보이므로 순서 실수가 불가능하고, 컴파일러가 유효성을 검사한다.

---

## 서식 지정자 (Format Specifiers)

`string.Format()`의 모든 서식 지정자를 동일하게 사용할 수 있다.

```csharp
double price = 1234.5;
DateTime now = DateTime.Now;
int count = 42;

// 숫자 서식
Console.WriteLine($"가격: {price:C}");        // 통화: ₩1,235
Console.WriteLine($"가격: {price:F2}");       // 소수점 2자리: 1234.50
Console.WriteLine($"개수: {count:D5}");       // 5자리 정수: 00042
Console.WriteLine($"16진수: {count:X}");      // 16진수: 2A

// 날짜 서식
Console.WriteLine($"날짜: {now:yyyy-MM-dd}"); // 날짜: 2024-01-15
Console.WriteLine($"시간: {now:HH:mm:ss}");   // 시간: 14:30:00

// 정렬 (너비 지정)
Console.WriteLine($"|{name,10}|");  // 오른쪽 정렬, 너비 10
Console.WriteLine($"|{name,-10}|"); // 왼쪽 정렬, 너비 10
```

---

## 표현식 사용

보간 문자열 안에는 단순 변수뿐 아니라 **임의의 C# 표현식**을 사용할 수 있다.

```csharp
var items = new List<string> { "a", "b", "c" };

// 메서드 호출
Console.WriteLine($"개수: {items.Count}");

// 조건 연산자
Console.WriteLine($"상태: {(items.Count > 0 ? "있음" : "없음")}");

// 산술 표현식
Console.WriteLine($"합계: {a + b}");

// null 병합 연산자
Console.WriteLine($"이름: {name ?? "미지정"}");
```

---

## 다중 줄 보간 문자열

```csharp
string report = $@"
=== 보고서 ===
이름:   {name}
날짜:   {DateTime.Now:yyyy-MM-dd}
총액:   {total:C}
";
```

`$@` 또는 `@$`를 조합하면 축자 문자열(verbatim string)과 함께 사용할 수 있다.

---

## string.Format()을 써야 하는 경우

문자열 보간이 항상 우월한 것은 아니다. **문화권(Culture)에 민감한 문자열**을 만들 때는 Item 5에서 다루는 `FormattableString`을 사용해야 한다.

```csharp
// 보간 문자열은 현재 스레드의 Culture를 사용
string s = $"가격: {price:C}"; // 현재 Culture로 고정됨

// 특정 Culture를 명시하려면 string.Format 또는 FormattableString 사용
string s = string.Format(CultureInfo.InvariantCulture, "가격: {0:C}", price);
```

---

## 결론

| 상황 | 권장 |
|------|------|
| 일반적인 문자열 조합 | `$"..."` 보간 문자열 ✅ |
| 서식 지정자 필요 | `$"{value:F2}"` ✅ |
| 문화권 지정 필요 | `FormattableString` (Item 5) |
| 동적 서식 문자열 필요 | `string.Format()` |

> `string.Format()`의 인덱스 기반 접근은 인수 순서 실수와 런타임 예외의 원인이 된다. C# 6.0 이상에서는 항상 문자열 보간을 먼저 고려하라.
