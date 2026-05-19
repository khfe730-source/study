# Item 21: IDisposable을 구현하는 타입 매개변수를 항상 지원하라 (Always Create Generic Classes That Support Disposable Type Parameters)

> **Chapter 3: Working with Generics**

---

## 핵심 요약

제네릭 클래스가 타입 매개변수 `T`의 인스턴스를 생성하거나 소유할 경우, `T`가 `IDisposable`을 구현할 수 있다는 사실을 고려해야 한다. `IDisposable` 제약을 걸거나, 런타임에 `T`가 `IDisposable`인지 검사해 적절히 `Dispose()`를 호출하라.

---

## 문제: IDisposable을 무시한 제네릭 클래스

```csharp
public class ResourceManager<T> where T : new()
{
    private T resource = new T();

    public T Resource => resource;

    ~ResourceManager()
    {
        // T가 IDisposable을 구현해도 Dispose()를 호출하지 않음 → 리소스 누수!
    }
}

// SqlConnection은 IDisposable을 구현하지만 해제되지 않음
var manager = new ResourceManager<SqlConnection>();
```

---

## 해결책 1: IDisposable 제약 추가

`T`가 항상 `IDisposable`이어야 한다면 제약 조건으로 명시한다.

```csharp
public class ResourceManager<T> : IDisposable
    where T : IDisposable, new()
{
    private T resource = new T();
    private bool disposed = false;

    public T Resource => resource;

    public void Dispose()
    {
        Dispose(true);
        GC.SuppressFinalize(this);
    }

    protected virtual void Dispose(bool disposing)
    {
        if (disposed) return;
        if (disposing)
            resource.Dispose(); // T가 IDisposable임이 보장됨
        disposed = true;
    }
}
```

---

## 해결책 2: 런타임 타입 검사 (제약 없이 유연하게)

`T`가 `IDisposable`일 수도 있고 아닐 수도 있을 때, 런타임에 검사한다.

```csharp
public class ResourceHolder<T> : IDisposable where T : new()
{
    private T resource = new T();
    private bool disposed = false;

    public T Resource => resource;

    public void Dispose()
    {
        Dispose(true);
        GC.SuppressFinalize(this);
    }

    protected virtual void Dispose(bool disposing)
    {
        if (disposed) return;

        if (disposing)
        {
            // T가 IDisposable이면 해제
            if (resource is IDisposable disposable)
                disposable.Dispose();
        }

        disposed = true;
    }
}

// T가 IDisposable이든 아니든 안전하게 사용 가능
var intHolder = new ResourceHolder<int>();    // int는 IDisposable 아님 → 문제없음
var connHolder = new ResourceHolder<SqlConnection>(); // SqlConnection은 IDisposable → 해제됨
```

---

## 외부에서 주입받는 경우: 소유권 명확히

```csharp
public class Processor<T>
{
    private readonly T item;
    private readonly bool ownsItem; // 이 클래스가 item을 소유하는가?

    // 외부에서 생성된 item을 받음
    public Processor(T item, bool ownsItem = false)
    {
        this.item = item;
        this.ownsItem = ownsItem;
    }

    public void Dispose()
    {
        // 소유권이 있을 때만 Dispose
        if (ownsItem && item is IDisposable disposable)
            disposable.Dispose();
    }
}
```

---

## using 문과 제네릭

```csharp
// T : IDisposable 제약이 있어야 using 사용 가능
public static void Use<T>(Func<T> factory, Action<T> action)
    where T : IDisposable
{
    using (T item = factory())
    {
        action(item);
    }
}

// 사용
Use(() => new SqlConnection(connStr),
    conn => conn.Open());
```

---

## 결론

| 상황 | 권장 |
|------|------|
| T가 항상 IDisposable이어야 함 | `where T : IDisposable` 제약 |
| T가 IDisposable일 수도 있음 | 런타임 `is IDisposable` 검사 |
| 외부에서 T를 주입받음 | 소유권(ownership) 명시적으로 관리 |

> 제네릭 클래스가 `T`의 인스턴스를 소유한다면, `T`가 `IDisposable`을 구현할 가능성을 항상 고려하라. `Dispose()`를 호출하지 않으면 리소스가 누수된다.
