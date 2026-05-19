# Item 24: 속성 접근자의 반환 타입으로 IEnumerable<T>를 사용하지 마라 (Do Not Use IEnumerable<T> as a Return Type for Property Accessors)

> **Chapter 3: Working with Generics**

---

## 핵심 요약

속성(property)이 `IEnumerable<T>`를 반환하면 접근할 때마다 시퀀스가 재평가(지연 실행)되어 예상치 못한 다중 실행과 성능 문제가 발생할 수 있다. 속성은 `IReadOnlyList<T>`, `IReadOnlyCollection<T>` 또는 구체 컬렉션 타입을 반환해야 한다. 지연 실행이 필요하다면 메서드로 표현하라.

---

## 문제: 속성이 IEnumerable<T>를 반환할 때

```csharp
public class DataSource
{
    private readonly IEnumerable<int> numbers;

    public DataSource(IEnumerable<int> numbers)
    {
        this.numbers = numbers;
    }

    // 나쁜 예: 속성이 IEnumerable<T> 반환
    public IEnumerable<int> Numbers => numbers;
}

// 사용측
var source = new DataSource(Enumerable.Range(1, 1000000));

// 두 번 순회 → 실제로 두 번 실행됨
int count = source.Numbers.Count();  // 100만 번 순회
int sum = source.Numbers.Sum();      // 또 100만 번 순회

// 더 심각한 경우: DB 쿼리가 매번 실행
public IEnumerable<User> ActiveUsers =>
    db.Users.Where(u => u.IsActive); // 속성 접근마다 쿼리 실행!
```

---

## 속성의 의미론적 기대

사용자는 속성 접근이 **빠르고 반복 가능**하다고 가정한다. `IEnumerable<T>`는 지연 실행될 수 있어 이 가정을 깨뜨린다.

```csharp
// 사용자의 자연스러운 코딩 패턴 → 위험
if (source.Numbers.Any())               // 1번째 순회
    Console.WriteLine(source.Numbers.First()); // 2번째 순회
```

---

## 해결책 1: IReadOnlyList<T> 또는 IReadOnlyCollection<T> 반환

```csharp
public class DataSource
{
    private readonly List<int> numbers;

    public DataSource(IEnumerable<int> numbers)
    {
        // 생성 시점에 실체화(materialize)
        this.numbers = numbers.ToList();
    }

    // 좋은 예: 실체화된 컬렉션 반환
    public IReadOnlyList<int> Numbers => numbers;
}

// 사용측: 여러 번 접근해도 안전
int count = source.Numbers.Count;  // O(1)
int sum = source.Numbers.Sum();    // 한 번 순회
var first = source.Numbers[0];     // O(1) 인덱스 접근
```

---

## 해결책 2: 지연 실행이 필요하다면 메서드로

지연 실행의 이점이 필요하다면 속성이 아닌 **메서드**로 표현한다. 메서드는 "호출할 때마다 연산이 발생할 수 있다"는 의미를 암묵적으로 전달한다.

```csharp
public class UserRepository
{
    private readonly DbContext db;

    // 지연 실행 쿼리 → 메서드로 표현
    public IEnumerable<User> GetActiveUsers()
        => db.Users.Where(u => u.IsActive);

    // 이미 로드된 컬렉션 → 속성으로 표현
    public IReadOnlyList<User> CachedUsers { get; }
}

// 사용측: GetActiveUsers()가 메서드임을 보고 "매번 실행됨"을 인지
var users = repo.GetActiveUsers().ToList(); // 명시적으로 실체화
```

---

## 빈 컬렉션 vs null

속성이 컬렉션을 반환할 때는 `null`이 아닌 **빈 컬렉션**을 반환하라.

```csharp
public class Order
{
    private List<OrderItem> items = new List<OrderItem>();

    // null 반환 금지: 사용측에서 null 검사 강제
    public IReadOnlyList<OrderItem> Items => items;
    // null이 아닌 빈 리스트를 반환하므로 사용측에서 null 검사 불필요
}
```

---

## 타입 선택 가이드

| 반환 타입 | 용도 |
|-----------|------|
| `IReadOnlyList<T>` | 인덱스 접근, Count가 필요한 컬렉션 속성 |
| `IReadOnlyCollection<T>` | Count만 필요하고 인덱스 접근 불필요 |
| `IEnumerable<T>` | 메서드의 반환값 (지연 실행 허용) |
| `List<T>` / 배열 | 내부에서만 사용, 외부엔 읽기 전용 인터페이스 노출 |

---

## 결론

> 속성은 빠르고 반복 가능해야 한다는 사용자의 기대를 충족시켜라. 속성의 반환 타입으로 `IEnumerable<T>` 대신 `IReadOnlyList<T>` 또는 `IReadOnlyCollection<T>`를 사용하고, 지연 실행이 필요하다면 메서드로 표현하라.
