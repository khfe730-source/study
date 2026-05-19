# Item 10: 기반 클래스 업데이트에 대응하는 경우에만 new 한정자를 사용하라 (Use the new Modifier Only to React to Base Class Updates)

> **Chapter 1: C# Language Idioms**

---

## 핵심 요약

`new` 한정자는 기반 클래스의 멤버를 **의도적으로 숨기는(hide)** 것이며, 다형성(polymorphism)을 깨뜨린다. 새로운 동작을 추가하려면 `override`를 사용하고, `new`는 기반 클래스가 예상치 못하게 동일한 이름의 멤버를 추가했을 때만 한시적으로 사용하는 도구다.

---

## override vs new

```csharp
class Base
{
    public virtual void Print() => Console.WriteLine("Base");
}

// override: 다형성 유지 (권장)
class DerivedWithOverride : Base
{
    public override void Print() => Console.WriteLine("DerivedWithOverride");
}

// new: 기반 클래스 멤버 숨김 (비권장)
class DerivedWithNew : Base
{
    public new void Print() => Console.WriteLine("DerivedWithNew");
}
```

### 다형성 동작 차이

```csharp
Base b1 = new DerivedWithOverride();
b1.Print(); // "DerivedWithOverride" ← 다형성 작동

Base b2 = new DerivedWithNew();
b2.Print(); // "Base" ← 기반 클래스 메서드가 호출됨!
            // DerivedWithNew의 Print는 숨겨져 있음
```

`new`로 선언한 메서드는 **기반 클래스 타입의 변수로 참조할 때 기반 클래스 메서드가 호출**된다. 이는 혼란과 버그의 원인이 된다.

---

## new가 올바른 사용 사례: 기반 클래스 업데이트 대응

기반 클래스가 업데이트되어 파생 클래스의 멤버와 동일한 이름의 메서드가 추가된 상황이다.

```csharp
// 기반 클래스 v1.0: Log() 없음
class Logger { }

// 파생 클래스 v1.0: Log() 직접 정의
class AppLogger : Logger
{
    public void Log(string message) => Console.WriteLine($"[App] {message}");
}

// 기반 클래스 v2.0 업데이트: Log()가 추가됨
class Logger
{
    public virtual void Log(string message) => Console.WriteLine($"[Logger] {message}");
}
```

이 시점에서 `AppLogger.Log()`는 컴파일러 경고를 발생시킨다. 두 가지 선택이 있다:

**선택 1: override로 교체 (권장)**
```csharp
class AppLogger : Logger
{
    // 기반 클래스의 Log()를 재정의 → 다형성 유지
    public override void Log(string message)
    {
        base.Log(message);                          // 기반 동작 유지
        Console.WriteLine($"[App 추가 처리] {message}");
    }
}
```

**선택 2: new로 명시적 숨김 (한시적)**
```csharp
class AppLogger : Logger
{
    // 기반 클래스의 Log()와 독립적인 별도 메서드임을 명시
    public new void Log(string message) => Console.WriteLine($"[App] {message}");
}
```

`new`를 사용하면 "나는 의도적으로 기반 클래스의 `Log()`와 다른 별도 멤버를 정의한다"는 의도를 컴파일러와 개발자에게 명시하는 역할을 한다.

---

## new 없이 동일한 이름의 멤버를 정의하면?

`new` 없이 기반 클래스 멤버와 동일한 이름을 정의하면 컴파일러 경고(CS0108)가 발생한다.

```
warning CS0108: 'AppLogger.Log(string)' hides inherited member 'Logger.Log(string)'.
Use the new keyword if hiding was intended.
```

`new`는 이 숨김이 **의도적**임을 명시해 경고를 없애는 역할도 한다.

---

## new 한정자의 함정: 타입에 따라 다른 메서드가 호출됨

```csharp
class Base
{
    public void Method() => Console.WriteLine("Base.Method");
}

class Derived : Base
{
    public new void Method() => Console.WriteLine("Derived.Method");
}

Derived d = new Derived();
d.Method();       // "Derived.Method"

Base b = d;       // 같은 객체를 Base 타입으로 참조
b.Method();       // "Base.Method" ← 놀라운 결과!
```

이 동작은 라이브러리 사용자에게 매우 혼란스럽다. `override`를 사용했다면 두 경우 모두 `"Derived.Method"`가 출력된다.

---

## 결론

| 상황 | 권장 |
|------|------|
| 기반 클래스 메서드의 동작을 변경하고 싶음 | `override` ✅ |
| 기반 클래스가 나의 메서드와 같은 이름을 새로 추가함 | `override` 검토 후, 불가하면 `new` |
| 의도적으로 기반 클래스와 독립적인 멤버 필요 | `new` (주석으로 이유 명시 권장) |
| 다형성이 필요한 모든 경우 | `override` ✅ |

> `new` 한정자는 다형성을 깨뜨리기 때문에 새로운 기능 추가 목적으로 사용해서는 안 된다. 기반 클래스가 예상치 못하게 동일한 이름의 멤버를 추가했을 때 일시적인 대응 수단으로만 사용하고, 가능하다면 빠르게 `override`로 전환하거나 메서드 이름을 변경하라.
