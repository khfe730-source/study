# Item 42: IEnumerable<T>와 IQueryable<T>를 구분하라 (Distinguish Between IEnumerable and IQueryable)

> **Chapter 4: Working with LINQ**

---

## 핵심 요약

`IEnumerable<T>`와 `IQueryable<T>`는 같은 쿼리 구문을 지원하지만 내부 동작이 완전히 다르다. `IEnumerable<T>`는 C# 델리게이트로 실행되고, `IQueryable<T>`는 Expression Tree로 변환되어 쿼리 공급자(Entity Framework 등)가 SQL이나 다른 쿼리 언어로 번역한다.

---

## 핵심 차이

```
IEnumerable<T>                 IQueryable<T>
─────────────────────────────  ───────────────────────────────
Where(Func<T, bool>)           Where(Expression<Func<T, bool>>)
메모리에서 실행                 LINQ 공급자가 번역 (SQL 등)
C# 코드로 직접 실행            쿼리 트리를 분석·변환 후 실행
모든 C# 기능 사용 가능         공급자가 이해하는 연산만 지원
```

---

## Expression Tree란

```csharp
// 람다를 Func<>로 컴파일: 실행 가능한 코드
Func<User, bool> func = u => u.Age > 18;
// u.Age > 18이 IL 코드로 직접 컴파일됨

// 람다를 Expression<Func<>>로 컴파일: 분석 가능한 트리
Expression<Func<User, bool>> expr = u => u.Age > 18;
// 트리 구조:
//   BinaryExpression (>)
//   ├── MemberExpression (u.Age)
//   └── ConstantExpression (18)
// EF가 이 트리를 읽어 "WHERE Age > 18" SQL 생성
```

---

## 실수 패턴 1: AsEnumerable 없이 클라이언트 측 필터링

```csharp
// 나쁜 예: C# 메서드를 IQueryable 쿼리에서 사용
bool IsVip(User u) => u.PurchaseCount > 10 && GetVipStatus(u); // DB가 모름

// 이 코드는 전체 테이블을 메모리에 올린 뒤 C#에서 필터링!
var vipUsers = dbContext.Users
    .Where(u => IsVip(u))  // 컴파일은 되지만 EF가 번역 불가
    .ToList();              // 실제로는 AsEnumerable()이 암묵적으로 발생

// 좋은 예: DB에서 할 수 있는 부분을 먼저 필터링
var candidates = dbContext.Users
    .Where(u => u.PurchaseCount > 10)  // SQL: WHERE PurchaseCount > 10
    .AsEnumerable()                     // 이 시점부터 C# 실행
    .Where(u => GetVipStatus(u));      // C# 메서드 사용 가능
```

---

## 실수 패턴 2: IQueryable을 IEnumerable로 조기 전환

```csharp
// 나쁜 예: AsEnumerable() 후에 필터링 — 전체를 메모리에 올림
var result = dbContext.Orders
    .AsEnumerable()            // 여기서 전체 Orders를 메모리에 로드!
    .Where(o => o.Total > 100)
    .ToList();

// 좋은 예: 가능한 한 IQueryable 상태를 유지
var result = dbContext.Orders
    .Where(o => o.Total > 100) // SQL WHERE 절로 변환
    .ToList();                  // 필터링된 결과만 전송
```

---

## 실수 패턴 3: IEnumerable 반환 타입으로 쿼리 능력 손실

```csharp
// 나쁜 예: IEnumerable<T>로 반환하면 호출자가 추가 필터를 DB에서 실행 못 함
public IEnumerable<Order> GetOrders()
    => dbContext.Orders; // IQueryable → IEnumerable로 암묵적 변환

// 호출측
var expensiveOrders = GetOrders()
    .Where(o => o.Total > 1000); // 메모리에서 필터링 (전체 Orders 로드됨)

// 좋은 예: IQueryable<T>로 반환
public IQueryable<Order> GetOrders()
    => dbContext.Orders;

// 호출측
var expensiveOrders = GetOrders()
    .Where(o => o.Total > 1000) // SQL WHERE 절로 변환
    .ToList();
```

---

## IQueryable의 한계

```csharp
// EF가 번역하지 못하는 C# 기능들
var result = dbContext.Users
    .Where(u => CustomHelper.IsEligible(u))       // 일반 메서드: 번역 불가
    .Where(u => new Regex(@"\d+").IsMatch(u.Name)) // Regex: 번역 불가
    .Where(u => u.Tags.Contains("vip",            // StringComparer: 번역 불가
        StringComparer.OrdinalIgnoreCase))
    .ToList();
// → InvalidOperationException 또는 전체 클라이언트 평가로 폴백

// 해결: DB에서 할 수 있는 것과 없는 것을 명시적으로 분리
var candidates = dbContext.Users
    .Where(u => u.Tags.Contains("vip"))  // SQL이 처리
    .AsEnumerable()
    .Where(u => CustomHelper.IsEligible(u)); // C#이 처리
```

---

## 타입 판별 및 전환 흐름

```
데이터 소스
    │
    ├─ DB (EF Core)  → IQueryable<T> → SQL 생성 → DB 실행
    │                                        ↓
    │                               AsEnumerable()
    │                                        ↓
    └─ 메모리 컬렉션 → IEnumerable<T> → C# 람다로 실행

IQueryable<T>
    .AsEnumerable()  → IEnumerable<T>  (DB → 메모리 경계)
    .ToList()        → List<T>         (실체화, IQueryable 종료)

IEnumerable<T>
    .AsQueryable()   → IQueryable<T>   (메모리 컬렉션에 Expression Tree 씌우기)
                                       (단, DB로 내려가지는 않음)
```

---

## 결론

> `IQueryable<T>`는 Expression Tree를 통해 LINQ 쿼리를 SQL 등으로 변환한다. DB 쿼리는 가능한 한 `IQueryable<T>` 상태를 유지하여 서버에서 실행하고, `AsEnumerable()`로 경계를 명시적으로 표시한 뒤 C# 전용 로직을 적용하라. 반환 타입도 `IQueryable<T>`로 노출하여 호출자가 추가 필터를 DB 쪽에서 처리할 수 있게 하라.
