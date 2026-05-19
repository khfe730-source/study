# Item 17: 표준 Dispose 패턴을 구현하라 (Implement the Standard Dispose Pattern)

> **Chapter 2: .NET Resource Management**

---

## 핵심 요약

비관리 리소스를 보유하는 타입은 `IDisposable` + `virtual Dispose(bool)` + (필요 시) 파이널라이저로 구성된 **표준 Dispose 패턴**을 따라야 한다. 클라이언트가 `Dispose()`를 호출한 경우와 잊은 경우 모두를 안전하게 처리한다.

---

## 패턴이 해결하는 두 가지 상황

| 상황 | 처리 방식 |
|------|-----------|
| 클라이언트가 `Dispose()` 호출 | `IDisposable.Dispose()` → 즉시 리소스 해제 + 파이널라이저 억제 |
| 클라이언트가 `Dispose()` 잊음 | 파이널라이저 → 비관리 리소스만 해제 (최후 방어) |

---

## 기반 클래스 구현

```csharp
public class MyResourceHog : IDisposable
{
    private bool alreadyDisposed = false;

    // IDisposable 구현: 관리 + 비관리 리소스 해제 후 파이널라이저 억제
    public void Dispose()
    {
        Dispose(true);
        GC.SuppressFinalize(this); // 파이널라이저 실행 억제 → 성능 향상
    }

    // 실제 정리 작업을 담당하는 virtual 메서드
    protected virtual void Dispose(bool isDisposing)
    {
        if (alreadyDisposed)
            return;

        if (isDisposing)
        {
            // 관리 리소스 해제 (다른 IDisposable 객체, 이벤트 핸들러 해제 등)
        }

        // 비관리 리소스 해제 (핸들, 네이티브 메모리 등)

        alreadyDisposed = true;
    }

    // Dispose 후 메서드 호출 시 예외 발생
    public void ExampleMethod()
    {
        if (alreadyDisposed)
            throw new ObjectDisposedException("MyResourceHog",
                "Called ExampleMethod on disposed object");
        // ...
    }
}
```

---

## 파생 클래스 구현

```csharp
public class DerivedResourceHog : MyResourceHog
{
    private bool disposed = false;

    protected override void Dispose(bool isDisposing)
    {
        if (disposed)
            return;

        if (isDisposing)
        {
            // 파생 클래스의 관리 리소스 해제
        }

        // 파생 클래스의 비관리 리소스 해제

        base.Dispose(isDisposing); // 반드시 기반 클래스 호출 (GC.SuppressFinalize 포함)

        disposed = true;
    }
}
```

기반 클래스와 파생 클래스 각각 독립적인 `disposed` 플래그를 갖는다. 한 클래스의 실수가 다른 클래스에 영향을 미치지 않도록 캡슐화한다.

---

## `Dispose(bool isDisposing)` 파라미터의 의미

```
Dispose(true)  ← IDisposable.Dispose() 경유
    → 관리 리소스 + 비관리 리소스 모두 해제 가능

Dispose(false) ← 파이널라이저 경유
    → 비관리 리소스만 해제
    → 관리 리소스는 이미 GC에 의해 처리되었을 수 있어 접근 위험
```

---

## 전체 흐름 다이어그램

```
클라이언트가 Dispose() 호출
    ↓
IDisposable.Dispose()
    → Dispose(true)            ← 관리 + 비관리 리소스 해제
    → GC.SuppressFinalize()    ← 파이널라이저 억제 (성능 향상)

클라이언트가 Dispose() 미호출
    ↓
GC가 파이널라이저 실행
    → Dispose(false)           ← 비관리 리소스만 해제
```

---

## 파이널라이저 추가 규칙

> **비관리 리소스를 직접 보유하는 클래스에만 파이널라이저를 추가하라.**

`IDisposable`만 필요하고 파이널라이저가 없는 경우에도 `virtual Dispose(bool)` 패턴 전체를 구현해야 한다. 그래야 파생 클래스에서 비관리 리소스를 추가할 때 패턴을 올바르게 확장할 수 있다.

파이널라이저 자체는 없더라도 그 존재만으로 GC 성능에 상당한 부담을 준다. 필요하지 않으면 절대 추가하지 마라.

---

## ⚠️ 객체 부활(Object Resurrection) 금지

```csharp
// 절대 하지 말 것!
public class BadClass
{
    private static readonly List<BadClass> finalizedList = new List<BadClass>();

    ~BadClass()
    {
        finalizedList.Add(this); // 자신을 살아있는 참조로 등록 → 객체 부활!
    }
}
```

부활된 객체의 문제점:
- GC는 파이널라이저가 이미 호출됨을 기록 → 재실행하지 않음
- 참조하는 다른 객체가 이미 파이널라이즈되어 사용 불가능할 수 있음
- 파이널라이저에서 자신의 레퍼런스를 저장하는 어떤 코드도 동일한 문제를 유발

> **Dispose/Finalizer에서는 리소스 해제 외에 다른 작업을 절대 하지 마라.**

---

## Dispose 구현 시 반드시 지킬 것

1. **멱등성(idempotent)**: `Dispose()`는 여러 번 호출해도 안전해야 한다
2. **`Dispose()`는 ObjectDisposedException의 예외**: Dispose된 객체에서 다른 public 메서드는 예외를 던지지만, `Dispose()` 자체는 예외 없이 무시해야 한다
3. **파이널라이저에서 null 확인 불필요**: 참조 객체는 여전히 메모리에 있지만, 이미 Dispose/Finalize되었을 수 있음
4. **리소스 해제만 수행**: 다른 작업 수행 금지

---

## 결론

> 비관리 리소스를 보유하는 모든 타입에 표준 Dispose 패턴을 구현하라. `IDisposable`만 필요한 경우에도 `virtual Dispose(bool)` 전체 구조를 갖춰 파생 클래스 확장을 지원하라. Dispose/Finalizer에서는 오직 리소스 해제만 수행하라.
