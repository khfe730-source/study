# Item 36: 쿼리 표현식이 메서드 호출로 어떻게 매핑되는지 이해하라 (Understand How Query Expressions Map to Method Calls)

> **Chapter 4: Working with LINQ**

---

## 핵심 요약

C#의 쿼리 구문(query syntax)은 컴파일러가 특정 메서드 호출로 변환하는 **문법 설탕(syntactic sugar)**이다. 이 매핑을 이해하면 쿼리의 동작을 정확히 예측하고, 필요한 경우 커스텀 타입에도 LINQ 쿼리를 적용할 수 있다.

---

## 기본 매핑표

| 쿼리 구문 | 메서드 구문 |
|-----------|------------|
| `from x in source` | (시작점) |
| `where 조건` | `.Where(x => 조건)` |
| `select 표현식` | `.Select(x => 표현식)` |
| `orderby 키` | `.OrderBy(x => 키)` |
| `orderby 키 descending` | `.OrderByDescending(x => 키)` |
| `orderby 키1, 키2` | `.OrderBy(x => 키1).ThenBy(x => 키2)` |
| `group x by 키` | `.GroupBy(x => 키)` |
| `join y in ... on ... equals ...` | `.Join(...)` |
| `let 변수 = 표현식` | `.Select(x => new { x, 변수 = 표현식 })` |
| `into 변수` | (연속 쿼리) |

---

## 상세 매핑 예시

```csharp
// 쿼리 구문
var result =
    from p in people
    where p.Age >= 18
    orderby p.Name
    select p.Name.ToUpper();

// 컴파일러가 변환하는 메서드 호출
var result = people
    .Where(p => p.Age >= 18)
    .OrderBy(p => p.Name)
    .Select(p => p.Name.ToUpper());
```

---

## let 절의 매핑

```csharp
// 쿼리 구문
var result =
    from p in people
    let fullName = p.FirstName + " " + p.LastName
    where fullName.Length > 10
    select fullName;

// 컴파일러 변환 (익명 타입으로 중간 상태 보존)
var result = people
    .Select(p => new { p, fullName = p.FirstName + " " + p.LastName })
    .Where(t => t.fullName.Length > 10)
    .Select(t => t.fullName);
```

---

## join의 매핑

```csharp
// 쿼리 구문
var result =
    from o in orders
    join c in customers on o.CustomerId equals c.Id
    select new { o.Total, c.Name };

// 컴파일러 변환
var result = orders.Join(
    inner: customers,
    outerKeySelector: o => o.CustomerId,
    innerKeySelector: c => c.Id,
    resultSelector: (o, c) => new { o.Total, c.Name });
```

---

## 커스텀 타입에 LINQ 적용

컴파일러는 **덕 타이핑(duck typing)**으로 메서드를 찾는다. 인터페이스 구현 없이 메서드 이름과 시그니처만 맞으면 쿼리 구문을 사용할 수 있다.

```csharp
public class NumberRange
{
    private readonly int start, end;
    public NumberRange(int start, int end) { this.start = start; this.end = end; }

    // Where 메서드만 있으면 where 절 사용 가능
    public NumberRange Where(Func<int, bool> predicate)
        => new NumberRange(
            Enumerable.Range(start, end - start).Where(predicate).First(),
            Enumerable.Range(start, end - start).Where(predicate).Last() + 1);

    // Select 메서드가 있으면 select 절 사용 가능
    public IEnumerable<T> Select<T>(Func<int, T> selector)
        => Enumerable.Range(start, end - start).Select(selector);
}

// 쿼리 구문 사용 가능
var squares = from n in new NumberRange(1, 11)
              select n * n;
```

---

## IEnumerable vs IQueryable

같은 쿼리 구문이지만 타입에 따라 다른 메서드가 호출된다.

```csharp
IEnumerable<User> memoryList = users.ToList();
// Where(Func<User, bool>) 호출 → 메모리에서 필터링

IQueryable<User> dbQuery = dbContext.Users;
// Where(Expression<Func<User, bool>>) 호출 → SQL WHERE 절로 변환
```

→ [Item 42](./item42.md)에서 자세히 다룸

---

## 결론

> 쿼리 구문은 컴파일러가 메서드 호출로 변환하는 문법 설탕이다. 매핑을 이해하면 쿼리의 실행 방식을 정확히 예측할 수 있고, 커스텀 타입에도 LINQ 쿼리를 적용할 수 있다. 쿼리 구문과 메서드 구문은 동등하므로 상황에 따라 가독성이 높은 것을 선택하라.
