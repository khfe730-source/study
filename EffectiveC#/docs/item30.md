# Item 30: 루프보다 쿼리 구문을 선호하라 (Prefer Query Syntax to Loops)

> **Chapter 4: Working with LINQ**

---

## 핵심 요약

명령형 루프(`for`, `foreach`, `while`) 대신 LINQ 쿼리 구문(query syntax) 또는 메서드 구문(method syntax)을 사용하면 코드가 **무엇을 하는지(what)**를 표현하고, **어떻게 하는지(how)**를 숨길 수 있다. 코드가 짧아지고 의도가 명확해진다.

---

## 루프 vs 쿼리

```csharp
// 명령형 루프: 어떻게 하는지에 집중
var result = new List<string>();
foreach (var person in people)
{
    if (person.Age >= 18)
    {
        result.Add(person.Name.ToUpper());
    }
}
result.Sort();

// LINQ 쿼리 구문: 무엇을 원하는지에 집중
var result =
    (from p in people
     where p.Age >= 18
     orderby p.Name
     select p.Name.ToUpper())
    .ToList();

// LINQ 메서드 구문
var result = people
    .Where(p => p.Age >= 18)
    .OrderBy(p => p.Name)
    .Select(p => p.Name.ToUpper())
    .ToList();
```

---

## 쿼리 구문이 더 읽기 좋은 경우: Join과 Group

```csharp
// 루프로 조인: 복잡하고 길다
var result = new List<string>();
foreach (var order in orders)
    foreach (var customer in customers)
        if (order.CustomerId == customer.Id)
            result.Add($"{customer.Name}: {order.Total}");

// 쿼리 구문: SQL처럼 직관적
var result =
    from o in orders
    join c in customers on o.CustomerId equals c.Id
    select $"{c.Name}: {o.Total}";

// GroupBy 예시
var grouped =
    from p in people
    group p by p.Department into g
    select new { Department = g.Key, Count = g.Count(), AvgAge = g.Average(x => x.Age) };
```

---

## 메서드 구문이 더 적합한 경우

쿼리 구문으로 표현할 수 없는 연산은 메서드 구문을 사용한다.

```csharp
// 쿼리 구문으로 표현 불가한 연산들
var count   = people.Count(p => p.Age >= 18);
var first   = people.FirstOrDefault(p => p.Name == "Alice");
var any     = people.Any(p => p.Age < 0);
var max     = people.Max(p => p.Age);
var sum     = people.Sum(p => p.Salary);
var flat    = people.SelectMany(p => p.Addresses);
var zipped  = list1.Zip(list2, (a, b) => a + b);
```

---

## 중첩 쿼리와 let 절

```csharp
// let 절로 중간 결과를 변수에 저장
var result =
    from p in people
    let fullName = $"{p.FirstName} {p.LastName}"
    let age = DateTime.Today.Year - p.BirthYear
    where age >= 18
    orderby fullName
    select new { FullName = fullName, Age = age };
```

---

## 결론

| 상황 | 권장 |
|------|------|
| 필터, 정렬, 변환, 조인, 그룹 | 쿼리 구문 또는 메서드 구문 ✅ |
| Count, Sum, Any, First 등 집계 | 메서드 구문 |
| 복잡한 조인/그룹 | 쿼리 구문 (SQL과 유사해 가독성 높음) |
| 부수 효과가 있는 작업 | 명령형 루프 |

> 데이터 변환과 조회에는 루프 대신 LINQ 쿼리를 사용하라. 코드가 짧아지고 의도가 명확해진다. 단, 부수 효과(side effect)가 필요한 작업은 `foreach` 루프가 더 적합하다.
