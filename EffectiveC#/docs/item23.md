# Item 23: 타입 매개변수에 메서드 제약을 정의하려면 델리게이트를 사용하라 (Use Delegates to Define Method Constraints on Type Parameters)

> **Chapter 3: Working with Generics**

---

## 핵심 요약

C#의 제약 조건은 인터페이스, 기반 클래스, `new()`, `class`, `struct`만 지원하며 특정 메서드 시그니처(예: "덧셈 연산자가 있는 타입")를 제약으로 표현할 수 없다. 이 경우 **델리게이트를 매개변수로 받아** 타입이 지원해야 할 연산을 외부에서 주입하면 된다.

---

## 문제: 연산자 제약을 표현할 수 없음

```csharp
// 덧셈이 가능한 타입에 대한 합계를 구하고 싶다
public static T Sum<T>(IEnumerable<T> sequence)
{
    T total = default;
    foreach (var item in sequence)
        total += item; // ❌ 컴파일 오류: T에 + 연산자가 없을 수 있음
    return total;
}
```

`T`에 `+` 연산자가 있다는 것을 제약 조건으로 표현할 수 없다.

---

## 해결책: 델리게이트로 연산 주입

```csharp
// 덧셈 연산을 델리게이트로 주입
public static T Sum<T>(IEnumerable<T> sequence, Func<T, T, T> add)
{
    T total = default;
    foreach (var item in sequence)
        total = add(total, item);
    return total;
}

// 사용: 타입별 연산을 람다로 전달
int intSum = Sum(new[] { 1, 2, 3, 4, 5 }, (a, b) => a + b);
double dblSum = Sum(new[] { 1.1, 2.2, 3.3 }, (a, b) => a + b);
string concat = Sum(new[] { "a", "b", "c" }, (a, b) => a + b);
```

---

## 팩토리 패턴: new() 제약의 한계 극복

`new()` 제약은 매개변수 없는 생성자만 호출할 수 있다. 매개변수가 있는 생성자 호출이 필요하면 `Func<T>` 팩토리 델리게이트를 사용한다.

```csharp
// new() 제약: 매개변수 없는 생성자만 가능
public static List<T> CreateList<T>(int count) where T : new()
{
    var list = new List<T>(count);
    for (int i = 0; i < count; i++)
        list.Add(new T()); // 매개변수 없는 생성자만 호출 가능
    return list;
}

// Func<T> 팩토리: 유연한 생성
public static List<T> CreateList<T>(int count, Func<T> factory)
{
    var list = new List<T>(count);
    for (int i = 0; i < count; i++)
        list.Add(factory()); // 팩토리가 생성 방법을 결정
    return list;
}

// 사용: 인덱스를 받는 생성자도 활용 가능
int index = 0;
var connections = CreateList(5,
    () => new SqlConnection($"Server=db{index++}"));
```

---

## 비교 연산 주입

```csharp
// Comparison<T>으로 비교 로직 주입
public static T FindMax<T>(IEnumerable<T> sequence,
    Comparison<T> comparison)
{
    T max = default;
    bool first = true;

    foreach (var item in sequence)
    {
        if (first || comparison(item, max) > 0)
        {
            max = item;
            first = false;
        }
    }

    return max;
}

// 사용
var people = GetPeople();
var oldest = FindMax(people, (x, y) => x.Age.CompareTo(y.Age));
var tallest = FindMax(people, (x, y) => x.Height.CompareTo(y.Height));
```

---

## 실전 예시: 제네릭 수학 연산 라이브러리

```csharp
public class MathOps<T>
{
    private readonly Func<T, T, T> add;
    private readonly Func<T, T, T> multiply;
    private readonly T zero;

    public MathOps(Func<T, T, T> add, Func<T, T, T> multiply, T zero)
    {
        this.add = add;
        this.multiply = multiply;
        this.zero = zero;
    }

    public T Sum(IEnumerable<T> values)
        => values.Aggregate(zero, add);

    public T Product(IEnumerable<T> values)
        => values.Aggregate(zero, multiply);
}

// int 버전
var intMath = new MathOps<int>(
    add: (a, b) => a + b,
    multiply: (a, b) => a * b,
    zero: 0
);

// double 버전
var dblMath = new MathOps<double>(
    add: (a, b) => a + b,
    multiply: (a, b) => a * b,
    zero: 0.0
);
```

---

## 결론

> C#의 제약 조건은 연산자나 특정 메서드 시그니처를 직접 표현할 수 없다. 이런 경우 필요한 연산을 `Func<>`, `Action<>`, `Comparison<T>` 등의 델리게이트로 매개변수화하여 주입하라. 타입 안전성과 유연성을 동시에 얻을 수 있다.
