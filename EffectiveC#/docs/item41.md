# Item 41: 클로저에서 비용이 큰 리소스 캡처를 피하라 (Avoid Capturing Expensive Resources in Closures)

> **Chapter 4: Working with LINQ**

---

## 핵심 요약

람다가 외부 변수를 캡처하면 해당 변수의 수명이 람다의 수명만큼 연장된다. 지연 실행되는 LINQ 쿼리가 DB 컨텍스트, 파일 핸들, 네트워크 연결 같은 비용이 크거나 수명이 제한된 리소스를 캡처하면 메모리 누수, `ObjectDisposedException`, 예상치 못한 동작이 발생한다.

---

## 클로저와 캡처의 원리

```csharp
// 컴파일러가 생성하는 클로저 클래스 (개념적)
// var query = numbers.Where(n => n > threshold);
// →
private class Closure
{
    public int threshold; // 캡처된 변수는 힙에 올라감
}

// threshold가 스택이 아닌 힙의 Closure 객체에 저장됨
// query가 살아 있는 한 Closure도 살아 있고, threshold도 살아 있음
```

---

## 문제 1: Dispose된 리소스 참조

```csharp
// 나쁜 예
IEnumerable<User> GetActiveUsers()
{
    using var context = new AppDbContext(); // 메서드 종료 시 Dispose
    return context.Users.Where(u => u.IsActive); // 지연 쿼리 반환
}                                                 // ← 여기서 context.Dispose()

// 호출측
var users = GetActiveUsers(); // Dispose된 context를 참조하는 쿼리
foreach (var u in users)      // ObjectDisposedException!
    Console.WriteLine(u.Name);
```

```csharp
// 좋은 예: 메서드 내에서 실체화
IReadOnlyList<User> GetActiveUsers()
{
    using var context = new AppDbContext();
    return context.Users.Where(u => u.IsActive).ToList(); // 실체화 후 반환
}
// context는 메서드 종료 전에 Dispose, 데이터는 List로 분리
```

---

## 문제 2: 대용량 컬렉션 캡처

```csharp
// 나쁜 예: 수십만 행의 참조 테이블 전체를 캡처
var allProducts = LoadAllProducts(); // 10만 개, 50MB

var query = orders
    .Select(o => new {
        o.OrderId,
        // allProducts 전체가 캡처됨 — query가 살아 있는 동안 50MB 유지
        ProductName = allProducts.First(p => p.Id == o.ProductId).Name
    });

// 좋은 예: 필요한 부분만 미리 추출
var productNameMap = LoadAllProducts()
    .ToDictionary(p => p.Id, p => p.Name); // 필요한 데이터만 Dict로

var query = orders
    .Select(o => new {
        o.OrderId,
        ProductName = productNameMap.GetValueOrDefault(o.ProductId)
    });
```

---

## 문제 3: 스레드 공유 리소스 캡처

```csharp
// 나쁜 예: 병렬 처리에서 공유 DbContext
var context = new AppDbContext(); // DbContext는 스레드 안전하지 않음

var results = orders.AsParallel()
    .Select(o => context.Products.Find(o.ProductId)) // 경쟁 조건(race condition)!
    .ToList();

// 좋은 예: 병렬 처리에서 각 스레드가 독립적인 컨텍스트 생성
var results = orders.AsParallel()
    .Select(o => {
        using var ctx = new AppDbContext();
        return ctx.Products.Find(o.ProductId);
    })
    .ToList();
```

---

## 안전한 캡처 패턴

```csharp
// 값 타입 및 불변 객체 — 안전하게 캡처 가능
int threshold = 18;
string countryCode = "KR";
var query = people.Where(p => p.Age > threshold && p.Country == countryCode);

// 캡처 전에 실체화
var localData = expensiveResource.LoadData(); // 실체화
expensiveResource.Dispose();                  // 즉시 해제
var query = localData.Where(x => x.IsValid); // 안전: 이미 메모리에 있음

// 팩토리 패턴으로 사용 시점에 생성
Func<AppDbContext> contextFactory = () => new AppDbContext();
var query = orders.Select(o => {
    using var ctx = contextFactory();
    return ctx.Products.Find(o.ProductId);
});
```

---

## 캡처 수명 다이어그램

```
람다를 캡처하는 변수의 수명:

정상 케이스:
┌─────────────────────────────────────────┐
│ 변수 수명 = 람다 수명 이후 종료         │
│ 람다: [생성──사용──종료]                │
│ 변수: [생성────────────종료]            │
└─────────────────────────────────────────┘

문제 케이스 (지연 평가):
┌─────────────────────────────────────────┐
│ Dispose된 자원이 아직 캡처됨            │
│ using 블록: [생성──사용──Dispose]       │
│ 람다: [생성────────────────사용]        │
│                              ↑ 여기서 ObjectDisposedException
└─────────────────────────────────────────┘
```

---

## 결론

> 람다가 캡처한 변수는 람다가 실행되기 전까지 GC 대상이 되지 않는다. 지연 실행 쿼리가 DB 컨텍스트·파일 핸들·대용량 컬렉션을 캡처하면 리소스 누수와 `ObjectDisposedException`으로 이어진다. 리소스는 사용 시점에 생성·소멸하거나, 데이터를 미리 실체화하여 람다에서 경량화된 값만 캡처하도록 설계하라.
