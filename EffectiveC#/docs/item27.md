# Item 27: 최소한의 인터페이스 계약을 확장 메서드로 보강하라 (Augment Minimal Interface Contracts with Extension Methods)

> **Chapter 3: Working with Generics**

---

## 핵심 요약

인터페이스는 구현자에게 최소한의 메서드만 요구하고, 그 위에 편의 기능은 **확장 메서드(extension method)** 로 제공하라. 인터페이스를 비대하게 만들지 않으면서도 풍부한 API를 제공할 수 있다.

---

## 인터페이스를 비대하게 만드는 문제

```csharp
// 나쁜 예: 모든 편의 기능을 인터페이스에 포함
public interface IRepository<T>
{
    T FindById(int id);
    IEnumerable<T> FindAll();
    void Add(T entity);
    void Remove(T entity);
    // 아래는 편의 메서드인데 인터페이스에 넣으면 모든 구현자가 작성해야 함
    bool Exists(int id);
    int Count();
    IEnumerable<T> FindWhere(Func<T, bool> predicate);
    T FindFirst(Func<T, bool> predicate);
}
```

인터페이스 구현자는 모든 메서드를 구현해야 하므로 부담이 크다.

---

## 최소 인터페이스 + 확장 메서드

```csharp
// 핵심 계약: 최소한의 메서드만 정의
public interface IRepository<T>
{
    T FindById(int id);
    IEnumerable<T> FindAll();
    void Add(T entity);
    void Remove(T entity);
}

// 편의 기능: 확장 메서드로 제공 (구현자 부담 없음)
public static class RepositoryExtensions
{
    public static bool Exists<T>(this IRepository<T> repo, int id)
        => repo.FindById(id) != null;

    public static int Count<T>(this IRepository<T> repo)
        => repo.FindAll().Count();

    public static IEnumerable<T> FindWhere<T>(
        this IRepository<T> repo, Func<T, bool> predicate)
        => repo.FindAll().Where(predicate);

    public static T FindFirst<T>(
        this IRepository<T> repo, Func<T, bool> predicate)
        => repo.FindAll().FirstOrDefault(predicate);
}

// 구현자: 핵심 메서드 4개만 구현
public class UserRepository : IRepository<User>
{
    public User FindById(int id) { ... }
    public IEnumerable<User> FindAll() { ... }
    public void Add(User entity) { ... }
    public void Remove(User entity) { ... }
    // Exists, Count, FindWhere, FindFirst는 확장 메서드로 자동 제공
}

// 사용측: 확장 메서드도 마치 인터페이스 메서드처럼 사용
IRepository<User> repo = new UserRepository();
bool exists = repo.Exists(42);
int count = repo.Count();
var adults = repo.FindWhere(u => u.Age >= 18);
```

---

## 확장 메서드의 기본 구현 vs 오버라이드

특정 구현체가 더 효율적인 방법을 알고 있다면, 같은 이름의 인스턴스 메서드로 재정의할 수 있다. C#은 항상 인스턴스 메서드를 확장 메서드보다 우선한다.

```csharp
public class OptimizedUserRepository : IRepository<User>
{
    public User FindById(int id) { ... }
    public IEnumerable<User> FindAll() { ... }
    public void Add(User entity) { ... }
    public void Remove(User entity) { ... }

    // Count()를 직접 구현: DB에서 COUNT(*) 쿼리로 효율적으로 처리
    public int Count() => db.Users.Count(); // SELECT COUNT(*), 전체 로드 불필요
}

// 사용측 코드 동일, 하지만 최적화된 구현 호출
IRepository<User> repo = new OptimizedUserRepository();
int count = repo.Count(); // 인스턴스 메서드 우선 → 최적화된 버전 호출
```

---

## LINQ의 설계 철학

LINQ가 바로 이 패턴을 채택했다. `IEnumerable<T>`는 `GetEnumerator()`만 요구하고, `Where`, `Select`, `OrderBy` 등 수십 개의 메서드는 모두 확장 메서드로 제공된다.

```csharp
// IEnumerable<T>의 핵심 계약
public interface IEnumerable<out T>
{
    IEnumerator<T> GetEnumerator(); // 이것만!
}

// Enumerable 클래스의 확장 메서드들 (일부)
public static class Enumerable
{
    public static IEnumerable<T> Where<T>(
        this IEnumerable<T> source, Func<T, bool> predicate) { ... }

    public static IEnumerable<TResult> Select<T, TResult>(
        this IEnumerable<T> source, Func<T, TResult> selector) { ... }

    // 수십 개의 확장 메서드...
}
```

---

## 결론

> 인터페이스는 핵심 계약만 담고 가볍게 유지하라. 편의 기능은 확장 메서드로 제공하면 구현자의 부담을 줄이면서 풍부한 API를 제공할 수 있다. 특정 구현체가 더 효율적인 방법을 알고 있다면 같은 이름의 인스턴스 메서드로 확장 메서드를 대체할 수 있다.
