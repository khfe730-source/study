# Item 16: 생성자에서 가상 함수를 호출하지 마라 (Never Call Virtual Functions in Constructors)

> **Chapter 2: .NET Resource Management**

---

## 핵심 요약

생성자에서 가상 함수를 호출하면, 파생 클래스의 생성자 본문이 아직 실행되지 않은 상태에서 파생 클래스의 메서드가 호출된다. 멤버 변수가 의도하지 않은 값을 가질 수 있어 예측 불가능한 버그가 발생한다.

---

## 문제 상황

```csharp
class B
{
    protected B()
    {
        VFunc(); // 가상 함수 호출!
    }

    protected virtual void VFunc()
    {
        Console.WriteLine("VFunc in B");
    }
}

class Derived : B
{
    private readonly string msg = "Set by initializer";

    public Derived(string msg)
    {
        this.msg = msg; // 생성자 본문에서 값 설정
    }

    protected override void VFunc()
    {
        Console.WriteLine(msg);
    }

    public static void Main()
    {
        var d = new Derived("Constructed in main");
        // 출력: "Set by initializer"  ← 예상: "Constructed in main"
    }
}
```

### 생성 순서 추적

| 단계 | 실행 내용 | msg 값 |
|------|-----------|--------|
| 1 | `Derived` 이니셜라이저 실행 | `"Set by initializer"` |
| 2 | `B()` 생성자 실행 → `VFunc()` 호출 | `"Set by initializer"` |
| 3 | C# 런타임: 실제 타입은 `Derived` → `Derived.VFunc()` 호출됨 | `"Set by initializer"` 출력 |
| 4 | `Derived("Constructed in main")` 생성자 본문 실행 | `"Constructed in main"` |

`B()` 생성자에서 `VFunc()`가 호출될 시점에 `Derived`의 생성자 본문은 아직 실행되지 않았다. 이니셜라이저 값만 설정된 상태다.

---

## C++와 C#의 차이

| 언어 | 생성자 실행 중 가상 함수 호출 시 동작 |
|------|---------------------------------------|
| C++ | 현재 실행 중인 생성자의 타입 메서드 호출 (단계적 타입 변화) |
| C# | 항상 실제 객체의 최종 런타임 타입 메서드 호출 |

C++에서는 `B()` 실행 중 `B::VFunc()`가 호출되고, `Derived()` 실행 중 `Derived::VFunc()`가 호출된다. C#에서는 항상 `Derived.VFunc()`가 호출된다.

추상 기반 클래스의 경우 C++에서는 런타임 오류가 발생하지만, C#에서는 항상 구체 구현체가 호출되어 컴파일은 된다. 그러나 미초기화 상태의 파생 클래스 메서드가 실행되는 문제는 동일하다.

---

## 왜 위험한가

```csharp
abstract class B
{
    protected B()
    {
        VFunc(); // 추상 메서드 호출
    }

    protected abstract void VFunc();
}

class Derived : B
{
    private readonly string msg = "Set by initializer";

    public Derived(string msg)
    {
        this.msg = msg;
    }

    protected override void VFunc()
    {
        Console.WriteLine(msg); // msg는 이니셜라이저 값, 생성자 값 아님
    }
}
```

- 파생 클래스의 생성자 본문이 실행되기 전에 파생 클래스 메서드가 호출됨
- `readonly` 필드가 두 가지 값을 가지는 시점이 존재
- 일반 필드는 이니셜라이저 초기값(또는 기본값 0/null)만 설정된 상태
- 파생 클래스 개발자가 이 제약을 알지 못하면 반드시 버그가 발생

---

## 도구 지원

FxCop과 Visual Studio 정적 코드 분석기(Static Code Analyzer)가 이 패턴을 경고로 감지한다.

---

## 결론

> 생성자에서 가상 함수를 절대 호출하지 마라. 파생 클래스의 초기화 상태를 보장할 수 없으며, 이는 모든 파생 클래스 개발자에게 숨겨진 함정이 된다.
