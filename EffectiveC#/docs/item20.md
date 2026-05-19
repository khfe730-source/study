# Item 20: IComparable<T>와 IComparer<T>로 순서 관계를 구현하라 (Implement Ordering Relations with IComparable<T> and IComparer<T>)

> **Chapter 3: Working with Generics**

---

## 핵심 요약

타입의 **자연 순서(natural ordering)** 는 `IComparable<T>`로 정의하고, 대안적 정렬 기준은 `IComparer<T>`로 분리한다. 제네릭 버전을 사용해 박싱을 피하고, 관계 연산자(`<`, `>`, `<=`, `>=`)도 함께 오버로드해 일관성을 유지하라.

---

## IComparable<T> vs IComparable

```csharp
// 비제네릭 (구식): object를 받아 박싱 발생
public interface IComparable
{
    int CompareTo(object obj); // 박싱 + 타입 안전성 없음
}

// 제네릭 (권장): 타입 안전, 박싱 없음
public interface IComparable<T>
{
    int CompareTo(T other); // 타입 안전, 박싱 없음
}
```

`CompareTo()` 반환값 규칙:
- **음수**: this < other
- **0**: this == other
- **양수**: this > other

---

## IComparable<T> 구현 예시

```csharp
public struct Temperature : IComparable<Temperature>, IComparable
{
    private readonly double celsius;

    public Temperature(double celsius) => this.celsius = celsius;

    // 제네릭 구현 (핵심)
    public int CompareTo(Temperature other)
        => celsius.CompareTo(other.celsius);

    // 비제네릭 구현 (하위 호환성)
    public int CompareTo(object obj)
    {
        if (obj is Temperature t) return CompareTo(t);
        throw new ArgumentException("Object is not a Temperature");
    }

    // 관계 연산자 오버로드 (IComparable<T>와 일관성 유지)
    public static bool operator <(Temperature left, Temperature right)
        => left.CompareTo(right) < 0;

    public static bool operator >(Temperature left, Temperature right)
        => left.CompareTo(right) > 0;

    public static bool operator <=(Temperature left, Temperature right)
        => left.CompareTo(right) <= 0;

    public static bool operator >=(Temperature left, Temperature right)
        => left.CompareTo(right) >= 0;

    public static bool operator ==(Temperature left, Temperature right)
        => left.CompareTo(right) == 0;

    public static bool operator !=(Temperature left, Temperature right)
        => left.CompareTo(right) != 0;

    public override bool Equals(object obj)
        => obj is Temperature t && CompareTo(t) == 0;

    public override int GetHashCode() => celsius.GetHashCode();

    public override string ToString() => $"{celsius:F1}°C";
}

// 사용
var temps = new List<Temperature>
{
    new Temperature(100), new Temperature(0), new Temperature(37)
};
temps.Sort(); // IComparable<T>를 사용해 정렬
```

---

## IComparer<T>: 대안적 정렬 기준

자연 순서 외에 다른 기준으로 정렬이 필요할 때 `IComparer<T>`를 사용한다.

```csharp
public class Person
{
    public string Name { get; set; }
    public int Age { get; set; }
    public string Department { get; set; }
}

// 이름순 정렬
public class NameComparer : IComparer<Person>
{
    public int Compare(Person x, Person y)
        => string.Compare(x.Name, y.Name, StringComparison.Ordinal);
}

// 나이순 정렬
public class AgeComparer : IComparer<Person>
{
    public int Compare(Person x, Person y)
        => x.Age.CompareTo(y.Age);
}

// 부서 → 이름 순 복합 정렬
public class DepartmentThenNameComparer : IComparer<Person>
{
    public int Compare(Person x, Person y)
    {
        int deptCompare = string.Compare(x.Department, y.Department,
            StringComparison.Ordinal);
        return deptCompare != 0
            ? deptCompare
            : string.Compare(x.Name, y.Name, StringComparison.Ordinal);
    }
}

// 사용
var people = new List<Person> { ... };
people.Sort(new NameComparer());
people.Sort(new AgeComparer());
people.Sort(new DepartmentThenNameComparer());
```

---

## Comparison<T>으로 인라인 정렬

`IComparer<T>` 클래스를 별도로 만들지 않고 람다로 정렬 기준을 지정할 수 있다.

```csharp
// 람다로 직접 정렬 기준 정의
people.Sort((x, y) => x.Age.CompareTo(y.Age));

// LINQ OrderBy도 람다 사용 가능
var sorted = people.OrderBy(p => p.Name)
                   .ThenBy(p => p.Age)
                   .ToList();
```

---

## IComparable<T>와 IComparer<T> 선택 기준

| 상황 | 사용 |
|------|------|
| 타입의 자연스러운 기본 순서 정의 | `IComparable<T>` |
| 상황에 따른 대안적 정렬 기준 | `IComparer<T>` |
| 일회성 정렬 (코드 내부) | `Comparison<T>` 람다 |
| 제네릭 알고리즘에서 정렬 기준 주입 | `IComparer<T>` 매개변수 |

---

## Comparer<T>.Default 활용

타입이 `IComparable<T>`를 구현하는지 모를 때 `Comparer<T>.Default`를 사용하면 안전하게 비교할 수 있다.

```csharp
public static T Min<T>(T left, T right)
{
    // T가 IComparable<T>를 구현하지 않으면 런타임 예외
    // (IComparable<T> 제약을 추가하는 것이 더 좋다)
    return Comparer<T>.Default.Compare(left, right) <= 0 ? left : right;
}
```

---

## 결론

> 타입의 자연 순서는 `IComparable<T>`로 정의하고, 관계 연산자도 함께 오버로드해 일관성을 유지하라. 다양한 정렬 기준이 필요하면 `IComparer<T>`로 분리하고, 일회성 정렬은 람다 `Comparison<T>`를 활용하라.
