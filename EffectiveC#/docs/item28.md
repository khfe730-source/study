# Item 28: 확장 메서드로 구체화된 타입을 강화하는 것을 고려하라 (Consider Enhancing Constructed Types with Extension Methods)

> **Chapter 3: Working with Generics**

---

## 핵심 요약

`List<int>`, `Dictionary<string, User>` 같이 이미 구체화된(constructed) 제네릭 타입에 도메인 특화 기능이 필요할 때, 상속 대신 **확장 메서드**를 활용하라. 특정 타입 인수에만 적용되는 확장 메서드를 정의해 타입을 강화할 수 있다.

---

## 문제: 구체화된 제네릭 타입을 상속으로 확장

```csharp
// 나쁜 예: List<int>를 상속해서 수학 연산 추가
public class IntList : List<int>
{
    public double Average() => this.Average();
    public int Sum() => this.Sum();
    public IntList Filter(Func<int, bool> predicate)
        => new IntList(this.Where(predicate));
}

// 문제점:
// 1. List<int>를 반환하는 코드에서 IntList를 사용 불가
// 2. 상속 계층이 불필요하게 복잡해짐
// 3. 기존 List<int> 인스턴스를 IntList로 변환 불가
```

---

## 해결책: 구체화된 타입에 대한 확장 메서드

```csharp
// 좋은 예: List<int>에 직접 확장 메서드 추가
public static class IntListExtensions
{
    // List<int>에만 적용되는 확장 메서드
    public static double StandardDeviation(this IEnumerable<int> sequence)
    {
        var items = sequence.ToList();
        if (!items.Any()) return 0;

        double avg = items.Average();
        double sumSquares = items.Sum(x => (x - avg) * (x - avg));
        return Math.Sqrt(sumSquares / items.Count);
    }

    public static List<int> Normalize(this List<int> list)
    {
        int min = list.Min();
        int max = list.Max();
        int range = max - min;
        if (range == 0) return list.Select(_ => 0).ToList();
        return list.Select(x => (x - min) * 100 / range).ToList();
    }
}

// 사용: 기존 List<int> 인스턴스에서 바로 사용 가능
List<int> scores = new List<int> { 85, 92, 78, 95, 88 };
double stdDev = scores.StandardDeviation();
List<int> normalized = scores.Normalize();
```

---

## 특정 타입 인수 조합에만 적용

```csharp
// Dictionary<string, List<T>>에만 적용되는 확장 메서드
public static class DictionaryListExtensions
{
    public static void AddToList<TKey, TValue>(
        this Dictionary<TKey, List<TValue>> dict,
        TKey key, TValue value)
    {
        if (!dict.TryGetValue(key, out var list))
        {
            list = new List<TValue>();
            dict[key] = list;
        }
        list.Add(value);
    }

    public static IEnumerable<TValue> GetOrEmpty<TKey, TValue>(
        this Dictionary<TKey, List<TValue>> dict, TKey key)
        => dict.TryGetValue(key, out var list) ? list : Enumerable.Empty<TValue>();
}

// 사용
var groupedUsers = new Dictionary<string, List<User>>();
groupedUsers.AddToList("Admin", adminUser);
groupedUsers.AddToList("Admin", anotherAdmin);
var admins = groupedUsers.GetOrEmpty("Admin");
```

---

## 도메인 특화 확장

비즈니스 로직에 특화된 확장 메서드로 코드를 더 표현적으로 만들 수 있다.

```csharp
// IEnumerable<Order>에 도메인 특화 기능 추가
public static class OrderQueryExtensions
{
    public static IEnumerable<Order> Pending(this IEnumerable<Order> orders)
        => orders.Where(o => o.Status == OrderStatus.Pending);

    public static IEnumerable<Order> OverThreshold(
        this IEnumerable<Order> orders, decimal threshold)
        => orders.Where(o => o.TotalAmount > threshold);

    public static decimal TotalRevenue(this IEnumerable<Order> orders)
        => orders.Sum(o => o.TotalAmount);

    public static IEnumerable<Order> PlacedBetween(
        this IEnumerable<Order> orders, DateTime start, DateTime end)
        => orders.Where(o => o.PlacedAt >= start && o.PlacedAt <= end);
}

// 사용: 도메인 언어로 읽히는 코드
decimal revenue = allOrders
    .Pending()
    .OverThreshold(1000m)
    .PlacedBetween(DateTime.Today.AddDays(-30), DateTime.Today)
    .TotalRevenue();
```

---

## 확장 메서드와 인스턴스 메서드의 우선순위

```csharp
public class MyList<T> : List<T>
{
    // 인스턴스 메서드: 확장 메서드보다 항상 우선
    public new int Sum() => this.Count * 100; // 의미 없는 예시
}

// 같은 이름의 확장 메서드
public static int Sum(this MyList<int> list) => list.Aggregate(0, (a, b) => a + b);

MyList<int> myList = new();
myList.Sum(); // 인스턴스 메서드 호출 (확장 메서드가 아님)
```

---

## 결론

> 구체화된 제네릭 타입(`List<int>`, `Dictionary<string, T>` 등)에 도메인 특화 기능을 추가할 때 상속 대신 확장 메서드를 사용하라. 기존 타입의 인스턴스에서 직접 사용 가능하고, 불필요한 상속 계층을 만들지 않으면서도 표현력 있는 API를 제공할 수 있다.
