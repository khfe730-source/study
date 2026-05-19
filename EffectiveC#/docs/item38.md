# Item 38: 메서드보다 람다 표현식을 선호하라 (Prefer Lambda Expressions to Methods)

> **Chapter 4: Working with LINQ**

---

## 핵심 요약

LINQ 쿼리에 전달할 조건·변환 로직은 별도 메서드로 추출하기보다 **람다 표현식(lambda expression)**으로 인라인에 두는 것이 표현력과 유지보수성 면에서 유리하다. 특히 `IQueryable<T>` 기반 쿼리(Entity Framework 등)에서는 메서드가 아닌 람다여야 SQL로 변환된다.

---

## 메서드 추출의 문제

### 가독성 손실

```csharp
// 나쁜 예: 조건 로직이 분산됨
private bool IsEligible(Person p) =>
    p.Age >= 18 && p.Country == "KR" && !p.IsBlocked;

private string FormatName(Person p) =>
    $"{p.LastName}, {p.FirstName}";

var result = people
    .Where(IsEligible)
    .Select(FormatName);

// 좋은 예: 의도가 한눈에 보임
var result = people
    .Where(p => p.Age >= 18 && p.Country == "KR" && !p.IsBlocked)
    .Select(p => $"{p.LastName}, {p.FirstName}");
```

### IQueryable에서의 치명적 문제

```csharp
// 나쁜 예: 메서드 참조는 Expression Tree로 변환 불가
private bool IsActiveUser(User u) => u.IsActive && u.LastLogin > DateTime.Now.AddDays(-30);

// 이 코드는 전체 테이블을 메모리에 올린 뒤 C#에서 필터링!
var users = dbContext.Users.Where(IsActiveUser).ToList();

// 좋은 예: 람다는 Expression Tree로 변환되어 SQL WHERE 절이 됨
var users = dbContext.Users
    .Where(u => u.IsActive && u.LastLogin > DateTime.Now.AddDays(-30))
    .ToList();
// 생성 SQL: WHERE IsActive = 1 AND LastLogin > @cutoff
```

---

## Expression Tree vs Delegate

```csharp
// IEnumerable<T>: Func<T, bool> — C# 코드로 직접 실행
IEnumerable<User> memUsers = allUsers.Where(u => u.Age > 18);

// IQueryable<T>: Expression<Func<T, bool>> — 분석 가능한 트리
IQueryable<User> dbUsers = dbContext.Users.Where(u => u.Age > 18);
//                                         ↑ 컴파일러가 Expression Tree 생성
//                                           LINQ provider가 SQL로 변환
```

메서드 그룹(method group)은 `Func<>`로만 변환되고 `Expression<Func<>>`로는 변환되지 않는다.

---

## 람다가 적합하지 않은 경우

```csharp
// 1. 복잡한 다단계 로직 — 메서드 추출이 맞음
private bool IsComplexEligible(Order order)
{
    if (order.Total <= 0) return false;
    var discountApplied = ApplyBusinessRules(order);
    return discountApplied.Total > MinimumThreshold
           && IsShippingAvailable(order.Destination);
}

// 2. 재사용 빈도가 높고 로직이 독립적일 때
// 3. 단위 테스트가 필요한 조건 로직
```

---

## 캡처 변수와 클로저

람다는 외부 변수를 캡처할 수 있다는 점도 강점이다.

```csharp
int minAge = GetMinimumAge(); // 런타임에 결정되는 값
string country = GetUserCountry();

var filtered = people
    .Where(p => p.Age >= minAge && p.Country == country)
    .ToList();
// 별도 메서드로 추출하면 minAge와 country를 매개변수로 전달해야 함
```

---

## 결론

> LINQ 쿼리의 조건·변환 로직은 기본적으로 람다 표현식으로 인라인에 작성하라. `IQueryable<T>` 기반 쿼리에서는 람다만이 Expression Tree로 변환되어 DB 쪽에서 실행된다. 로직이 복잡하거나 재사용이 필요한 경우에만 메서드로 추출하고, 그 경우에도 `IQueryable`과 조합하려면 Expression 기반으로 설계해야 한다.
