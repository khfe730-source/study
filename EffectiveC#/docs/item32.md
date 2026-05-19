# Item 32: 반복과 동작, 조건, 함수를 분리하라 (Decouple Iterations from Actions, Predicates, and Functions)

> **Chapter 4: Working with LINQ**

---

## 핵심 요약

반복 로직(iteration)과 각 요소에 수행할 동작(action), 조건(predicate), 변환(function)을 하나의 메서드에 혼합하지 마라. 반복은 LINQ나 `foreach`에 맡기고, 동작·조건·변환은 **델리게이트로 분리**해 주입하면 코드 재사용성과 테스트 가능성이 높아진다.

---

## 문제: 반복과 동작이 혼합된 코드

```csharp
// 반복 + 조건 + 출력이 한 메서드에 뒤섞임
public static void PrintAdultNames(List<Person> people)
{
    foreach (var p in people)
        if (p.Age >= 18)
            Console.WriteLine(p.Name);
}

// 다른 조건이나 다른 동작이 필요하면 메서드를 복제해야 함
public static void PrintSeniorNames(List<Person> people) { ... } // 중복
public static void LogAdultNames(List<Person> people) { ... }    // 중복
```

---

## 해결: 조건과 동작을 델리게이트로 분리

```csharp
// 반복과 동작 분리
public static void ForEach<T>(
    this IEnumerable<T> source,
    Action<T> action)
{
    foreach (var item in source)
        action(item);
}

// 조합해서 사용
people
    .Where(p => p.Age >= 18)       // 조건 분리
    .ForEach(p => Console.WriteLine(p.Name)); // 동작 분리

people
    .Where(p => p.Age >= 65)
    .ForEach(p => logger.Log(p.Name));
```

---

## 조건, 변환, 동작을 각각 매개변수로

```csharp
public static IEnumerable<TResult> Transform<T, TResult>(
    this IEnumerable<T> source,
    Func<T, bool> predicate,   // 조건
    Func<T, TResult> selector) // 변환
    => source.Where(predicate).Select(selector);

// 다양하게 조합
var adultNames = people.Transform(
    predicate: p => p.Age >= 18,
    selector: p => p.Name);

var seniorEmails = people.Transform(
    predicate: p => p.Age >= 65,
    selector: p => p.Email);
```

---

## 실전: 범용 처리 파이프라인

```csharp
public static void Process<T>(
    IEnumerable<T> source,
    Func<T, bool> filter,
    Func<T, T> transform,
    Action<T> action)
{
    foreach (var item in source.Where(filter).Select(transform))
        action(item);
}

// 완전히 분리된 각 관심사
Process(
    source: orders,
    filter: o => o.Total > 1000,
    transform: o => o with { Total = o.Total * 0.9m }, // 10% 할인
    action: o => SendConfirmationEmail(o)
);
```

---

## 결론

> 반복 로직과 비즈니스 로직(조건, 변환, 동작)을 혼합하지 마라. 반복은 LINQ에 맡기고 조건·변환·동작은 델리게이트로 주입하면, 코드 재사용성이 높아지고 각 부분을 독립적으로 테스트할 수 있다.
