# Item 13: 정적 클래스 멤버는 적절히 초기화하라 (Use Proper Initialization for Static Class Members)

> **Chapter 2: .NET Resource Management**

---

## 핵심 요약

정적 멤버 초기화에는 **정적 이니셜라이저** 또는 **정적 생성자**를 사용하라. 단순한 경우는 이니셜라이저로, 복잡한 로직이나 예외 처리가 필요한 경우는 정적 생성자로 처리한다.

---

## 정적 이니셜라이저 vs 정적 생성자

### 정적 이니셜라이저 — 단순한 초기화

```csharp
public class MySingleton
{
    private static readonly MySingleton theOneAndOnly = new MySingleton();

    public static MySingleton TheOnly => theOneAndOnly;
    private MySingleton() { }
}
```

선언과 동시에 초기화. 로직이 단순할 때 가장 간결하다.

### 정적 생성자(static constructor) — 복잡한 로직 또는 예외 처리

```csharp
public class MySingleton2
{
    private static readonly MySingleton2 theOneAndOnly;

    static MySingleton2()
    {
        theOneAndOnly = new MySingleton2();
    }

    public static MySingleton2 TheOnly => theOneAndOnly;
    private MySingleton2() { }
}
```

정적 생성자는 인자를 받지 않으며 클래스당 하나만 정의 가능하다.

---

## 정적 생성자의 실행 규칙

- CLR이 해당 타입이 **처음 접근되기 직전에 자동 호출**
- **정적 이니셜라이저는 정적 생성자보다 먼저 실행**된다
- 기반 클래스의 정적 생성자보다 먼저 실행될 수 있다

---

## ⚠️ 정적 생성자에서 예외가 탈출하면 치명적

정적 이니셜라이저를 사용하면 예외를 직접 잡을 수 없다. 예외 가능성이 있다면 반드시 정적 생성자 안에서 처리해야 한다.

```csharp
static MySingleton2()
{
    try
    {
        theOneAndOnly = new MySingleton2();
    }
    catch
    {
        // 반드시 예외를 처리해야 한다!
    }
}
```

예외가 정적 생성자 밖으로 탈출하면:

1. CLR이 `TypeInitializationException`을 던진다
2. 해당 `AppDomain`이 언로드될 때까지 그 타입의 인스턴스를 생성할 수 없다
3. CLR은 정적 생성자를 재실행하지 않는다 → 타입이 영구적으로 초기화 실패 상태

---

## 결론

| 상황 | 권장 방식 |
|------|-----------|
| 단순한 정적 멤버 할당 | 정적 이니셜라이저 ✅ |
| 복잡한 초기화 로직 필요 | 정적 생성자 |
| 초기화 중 예외 처리 필요 | 정적 생성자 (필수) |

> 정적 멤버는 정적 이니셜라이저 또는 정적 생성자로만 초기화하라. 인스턴스 생성자나 별도의 private 메서드를 사용하지 마라.
