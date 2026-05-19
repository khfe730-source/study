# Item 34: 함수 매개변수로 결합도를 낮춰라 (Loosen Coupling by Using Function Parameters)

> **Chapter 4: Working with LINQ**

---

## 핵심 요약

메서드가 특정 타입이나 구현에 직접 의존하는 대신, `Func<>` / `Action<>` 델리게이트를 매개변수로 받아 동작을 외부에서 주입받으면 결합도(coupling)가 낮아지고 재사용성이 높아진다.

---

## 구체 타입 의존의 문제

```csharp
// 나쁜 예: SqlRepository에 직접 의존
public class ReportGenerator
{
    private readonly SqlUserRepository repo;

    public ReportGenerator(SqlUserRepository repo)
    {
        this.repo = repo;
    }

    public string Generate()
    {
        var users = repo.GetActiveUsers();
        return string.Join(", ", users.Select(u => u.Name));
    }
}
```

`SqlUserRepository`가 변경되거나 테스트에서 다른 구현체를 쓰고 싶으면 클래스 전체를 바꿔야 한다.

---

## 함수 매개변수로 주입

```csharp
// 좋은 예: 데이터 공급 방법을 함수로 주입
public class ReportGenerator
{
    private readonly Func<IEnumerable<User>> getUsers;

    public ReportGenerator(Func<IEnumerable<User>> getUsers)
    {
        this.getUsers = getUsers;
    }

    public string Generate()
    {
        var users = getUsers();
        return string.Join(", ", users.Select(u => u.Name));
    }
}

// 프로덕션: 실제 DB
var report = new ReportGenerator(() => sqlRepo.GetActiveUsers());

// 테스트: 인메모리 데이터
var report = new ReportGenerator(() => new[]
{
    new User { Name = "Alice" },
    new User { Name = "Bob" }
});
```

---

## 정렬·필터·변환 주입

```csharp
public static IEnumerable<TResult> ProcessData<T, TResult>(
    IEnumerable<T> source,
    Func<T, bool> filter,       // 어떤 항목을 포함할지
    Func<T, TResult> transform, // 어떻게 변환할지
    IComparer<TResult> comparer = null) // 어떻게 정렬할지
{
    var result = source.Where(filter).Select(transform);
    return comparer != null ? result.OrderBy(x => x, comparer) : result;
}

// 다양한 조합으로 재사용
var adultNames = ProcessData(people,
    filter: p => p.Age >= 18,
    transform: p => p.Name);

var seniorEmails = ProcessData(people,
    filter: p => p.Age >= 65,
    transform: p => p.Email,
    comparer: StringComparer.OrdinalIgnoreCase);
```

---

## 전략 패턴의 함수형 표현

```csharp
// 객체지향: 전략 인터페이스
public interface ISortStrategy<T> { IEnumerable<T> Sort(IEnumerable<T> items); }

// 함수형: Func으로 동일한 효과 (더 간결)
public static IEnumerable<T> ApplySort<T>(
    IEnumerable<T> items,
    Func<IEnumerable<T>, IEnumerable<T>> sortStrategy)
    => sortStrategy(items);

// 다양한 전략
ApplySort(people, items => items.OrderBy(p => p.Name));
ApplySort(people, items => items.OrderByDescending(p => p.Age));
ApplySort(people, items => items); // 정렬 없음
```

---

## 결론

> 특정 구현체에 직접 의존하는 대신 `Func<>` / `Action<>`으로 동작을 매개변수화하라. 결합도가 낮아져 테스트가 쉬워지고, 다양한 시나리오에서 재사용할 수 있다.
