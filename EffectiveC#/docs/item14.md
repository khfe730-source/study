# Item 14: 중복 초기화 로직을 최소화하라 (Minimize Duplicate Initialization Logic)

> **Chapter 2: .NET Resource Management**

---

## 핵심 요약

여러 생성자에 공통 초기화 로직이 있을 때, private 헬퍼 메서드 대신 **생성자 체이닝(`this()`)** 을 사용하라. 컴파일러가 이니셜라이저 및 기반 클래스 생성자 호출의 중복을 제거해 더 효율적인 코드를 생성한다.

---

## 생성자 체이닝 (Constructor Chaining)

```csharp
public class MyClass
{
    private List<ImportantData> coll;
    private string name;

    public MyClass() : this(0, "")                        // → 세 번째 생성자 위임
    { }

    public MyClass(int initialCount) : this(initialCount, string.Empty)  // → 세 번째 생성자 위임
    { }

    public MyClass(int initialCount, string name)         // 실제 초기화 담당
    {
        coll = (initialCount > 0)
            ? new List<ImportantData>(initialCount)
            : new List<ImportantData>();
        this.name = name;
    }
}
```

---

## private 헬퍼 메서드 방식이 나쁜 이유

```csharp
// 나쁜 예: private 헬퍼 메서드 사용
public MyClass()           { commonConstructor(0, ""); }
public MyClass(int n)      { commonConstructor(n, ""); }
public MyClass(int n, string name) { commonConstructor(n, name); }

private void commonConstructor(int count, string name) { ... }
```

컴파일러는 **각 생성자마다** 이니셜라이저 코드와 기반 클래스 생성자 호출을 중복 삽입한다. 컴파일러가 생성하는 실제 코드는 아래와 같다:

```csharp
public MyClass()
{
    // 이니셜라이저 실행 (중복)
    object(); // 기반 클래스 생성자 (중복)
    commonConstructor(0, "");
}

public MyClass(int n)
{
    // 이니셜라이저 실행 (중복)
    object(); // 기반 클래스 생성자 (중복)
    commonConstructor(n, "");
}
```

생성자 체이닝을 사용하면 이니셜라이저와 기반 클래스 생성자 호출이 **최종 생성자에서 단 한 번만** 실행된다.

### readonly 필드 초기화 불가 문제

`readonly` 필드는 생성자에서만 값을 설정할 수 있다. 헬퍼 메서드로는 아예 초기화가 불가능하다.

```csharp
private readonly string name;

private void commonConstructor(string name)
{
    this.name = name; // 컴파일 오류! readonly는 생성자에서만 설정 가능
}
```

생성자 체이닝은 이 문제를 자연스럽게 해결한다.

---

## 기본 매개변수(Default Parameters) 활용

C# 4.0 이후에는 기본 매개변수로 오버로드 수를 줄일 수 있다.

```csharp
public MyClass() : this(0, string.Empty) { }  // new() 제약 충족을 위해 명시적 선언

public MyClass(int initialCount = 0, string name = "")  // "" 사용 (string.Empty는 컴파일 타임 상수 아님)
{
    coll = (initialCount > 0)
        ? new List<ImportantData>(initialCount)
        : new List<ImportantData>();
    this.name = name;
}
```

> **주의 1**: `string.Empty`는 컴파일 타임 상수가 아니므로 기본 매개변수 값으로 사용할 수 없다. `""`를 사용한다.

> **주의 2**: 제네릭 `new()` 제약을 만족하려면 명시적 무인자 생성자가 필요하다. 기본값이 모두 있는 생성자도 `new()` 제약을 충족하지 않는다.

> **주의 3**: 기본 매개변수는 매개변수 이름까지 공개 인터페이스가 된다. 이름이나 기본값 변경은 클라이언트 코드 재컴파일이 필요한 **브레이킹 체인지**다.

---

## 객체 생성 전체 순서

첫 번째 인스턴스 생성 시 전체 순서:

| 순서 | 단계 |
|------|------|
| 1 | 정적 변수 스토리지를 0으로 초기화 |
| 2 | 정적 이니셜라이저 실행 |
| 3 | 기반 클래스의 정적 생성자 실행 |
| 4 | 현재 클래스의 정적 생성자 실행 |
| 5 | 인스턴스 변수 스토리지를 0으로 초기화 |
| 6 | 인스턴스 이니셜라이저 실행 |
| 7 | 기반 클래스 인스턴스 생성자 실행 |
| 8 | 현재 클래스 인스턴스 생성자 실행 |

이후 인스턴스는 5번부터 시작 (정적 초기화는 타입당 한 번만).
생성자 체이닝을 사용하면 6, 7번이 최종 생성자에서 한 번만 실행되도록 컴파일러가 최적화한다.

---

## 결론

| 방법 | 권장 여부 | 이유 |
|------|-----------|------|
| 생성자 체이닝 `this()` | ✅ 권장 | 중복 제거 + readonly 초기화 가능 + 컴파일러 최적화 |
| 기본 매개변수 | ✅ 권장 | 오버로드 수 감소 |
| private 헬퍼 메서드 | ❌ 지양 | 이니셜라이저/기반 생성자 중복, readonly 초기화 불가 |

> 여러 생성자의 공통 로직은 반드시 생성자 체이닝으로 처리하라. 헬퍼 메서드는 덜 효율적이고 readonly 필드를 초기화할 수도 없다.
