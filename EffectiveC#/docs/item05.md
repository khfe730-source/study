# Item 5: 문화권별 문자열에는 FormattableString을 사용하라 (Prefer FormattableString for Culture-Specific Strings)

> **Chapter 1: C# Language Idioms**

---

## 핵심 요약

보간 문자열(`$"..."`)은 현재 스레드의 Culture를 사용해 즉시 `string`으로 변환된다. 문화권에 따라 다르게 표현되어야 하는 숫자, 날짜, 통화 등은 `FormattableString`을 사용해 Culture를 명시적으로 제어해야 한다.

---

## 보간 문자열의 Culture 문제

```csharp
double price = 1234.56;

// 현재 스레드 Culture가 ko-KR이면: "1,234.56"
// 현재 스레드 Culture가 de-DE이면: "1.234,56" (독일식 소수점)
string s = $"가격: {price}";
```

보간 문자열은 **현재 스레드의 `CultureInfo.CurrentCulture`** 를 사용한다. 서버 환경에서 다국어 요청을 처리할 때 예상치 못한 형식으로 출력될 수 있다.

---

## FormattableString이란

C# 6.0에서 보간 문자열의 컴파일 결과는 대상 타입에 따라 달라진다.

```csharp
// string으로 할당 → 즉시 현재 Culture로 변환
string s = $"가격: {price:C}";

// FormattableString으로 할당 → 서식 문자열과 인수를 보존
FormattableString fs = $"가격: {price:C}";
```

`FormattableString`은 서식 문자열(`Format`)과 인수 배열(`GetArguments()`)을 보존하며, 원하는 Culture를 지정해 최종 문자열을 생성할 수 있다.

---

## FormattableString 활용

```csharp
using System.Globalization;

FormattableString fs = $"가격: {price:C}, 날짜: {DateTime.Now:d}";

// 특정 Culture 지정
string korean  = fs.ToString(new CultureInfo("ko-KR"));
string german  = fs.ToString(new CultureInfo("de-DE"));
string english = fs.ToString(new CultureInfo("en-US"));

Console.WriteLine(korean);  // 가격: ₩1,235, 날짜: 2024-01-15
Console.WriteLine(german);  // 가격: 1.235,00 €, 날짜: 15.01.2024
Console.WriteLine(english); // 가격: $1,234.56, 날짜: 1/15/2024
```

---

## InvariantCulture 사용

데이터베이스 저장, 직렬화, 로그 등 **사람이 아닌 시스템이 읽는 문자열**은 InvariantCulture를 사용해야 한다.

```csharp
// 항상 동일한 형식 (영어권 기본 서식, 소수점은 '.')
string invariant = FormattableString.Invariant($"가격: {price}, 날짜: {DateTime.Now:o}");
```

`FormattableString.Invariant()`는 정적 헬퍼 메서드로 `CultureInfo.InvariantCulture`를 사용한다.

---

## 실전 패턴: 다국어 지원 API

```csharp
public class PriceFormatter
{
    // UI 표시용: 사용자 Culture 사용
    public string FormatForDisplay(double price, CultureInfo culture)
    {
        FormattableString fs = $"{price:C}";
        return fs.ToString(culture);
    }

    // DB 저장용: 항상 고정 형식
    public string FormatForStorage(double price)
    {
        return FormattableString.Invariant($"{price:F4}");
    }
}
```

---

## 보간 문자열 vs FormattableString 선택 기준

| 상황 | 권장 |
|------|------|
| 로그, 디버그 메시지 | `$"..."` (현재 Culture 무방) |
| UI에 표시할 문자열 | `FormattableString` + 사용자 Culture |
| DB/파일 저장, 직렬화 | `FormattableString.Invariant()` |
| 네트워크 전송 (숫자/날짜 포함) | `FormattableString.Invariant()` |
| SQL 쿼리 문자열 | `FormattableString.Invariant()` |

---

## 결론

> 보간 문자열은 편리하지만 Culture를 암묵적으로 사용한다. 숫자, 날짜, 통화처럼 문화권에 따라 표현이 달라지는 데이터를 포함하는 문자열은 반드시 `FormattableString`으로 받아 Culture를 명시적으로 제어하라.
