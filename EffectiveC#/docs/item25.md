# Item 25: 타입 매개변수가 인스턴스 필드가 아닌 경우 제네릭 메서드를 사용하라 (Prefer Generic Methods Unless Type Parameters Are Instance Fields)

> **Chapter 3: Working with Generics**

---

## 핵심 요약

제네릭 타입 매개변수가 클래스의 인스턴스 필드로 저장되지 않고 특정 메서드에서만 사용된다면, 클래스 전체를 제네릭으로 만들지 말고 **해당 메서드만 제네릭으로** 선언하라. 제네릭 메서드는 타입 인수를 호출 시점에 추론하므로 사용하기 더 편리하고 클래스 설계도 단순해진다.

---

## 제네릭 클래스 vs 제네릭 메서드

```csharp
// 나쁜 예: 타입 매개변수가 메서드에서만 쓰임에도 클래스 전체를 제네릭으로
public class Utilities<T>
{
    public T Max(T left, T right) where T : IComparable<T>
        => left.CompareTo(right) > 0 ? left : right;
}

// 사용 시 타입 명시 필요 (불편)
var util = new Utilities<int>();
int max = util.Max(3, 5);

// 좋은 예: 메서드만 제네릭으로
public class Utilities
{
    public T Max<T>(T left, T right) where T : IComparable<T>
        => left.CompareTo(right) > 0 ? left : right;
}

// 타입 인수 추론: 타입 명시 불필요
var util = new Utilities();
int max = util.Max(3, 5);       // T = int 추론
double dMax = util.Max(3.0, 5.0); // T = double 추론
```

---

## 제네릭 클래스가 필요한 경우: 인스턴스 필드

타입 매개변수가 **인스턴스 필드에 저장**될 때는 클래스 전체를 제네릭으로 만들어야 한다.

```csharp
// T가 인스턴스 필드로 저장됨 → 클래스 제네릭 필요
public class Repository<T> where T : class
{
    private readonly List<T> items = new(); // T가 필드 타입
    private readonly Func<T, int> keySelector;

    public Repository(Func<T, int> keySelector)
    {
        this.keySelector = keySelector;
    }

    public void Add(T item) => items.Add(item);
    public T FindById(int id) => items.FirstOrDefault(i => keySelector(i) == id);
}
```

---

## 정적 유틸리티 메서드는 항상 제네릭 메서드로

```csharp
// 정적 유틸리티 클래스의 제네릭 메서드
public static class CollectionUtils
{
    // 각 메서드가 독립적으로 타입 매개변수를 가짐
    public static T Max<T>(T left, T right) where T : IComparable<T>
        => left.CompareTo(right) > 0 ? left : right;

    public static void Swap<T>(ref T left, ref T right)
        => (left, right) = (right, left);

    public static IEnumerable<T> Flatten<T>(IEnumerable<IEnumerable<T>> sequences)
        => sequences.SelectMany(s => s);

    public static bool IsNullOrEmpty<T>(IEnumerable<T> sequence)
        => sequence == null || !sequence.Any();
}

// 사용: 타입 인수 추론으로 깔끔
int max = CollectionUtils.Max(10, 20);
string sMax = CollectionUtils.Max("apple", "banana");

int a = 1, b = 2;
CollectionUtils.Swap(ref a, ref b); // a=2, b=1
```

---

## 타입 추론의 한계

제네릭 메서드의 타입 추론은 매개변수 타입에서만 작동한다. 반환 타입만으로는 추론되지 않는다.

```csharp
public static T Default<T>() => default;

// 반환 타입으로만 T를 사용 → 타입 명시 필요
int i = Default<int>();    // T = int 명시 필요
string s = Default<string>(); // T = string 명시 필요
```

---

## 결론

| 상황 | 권장 |
|------|------|
| T가 인스턴스 필드 타입으로 저장됨 | 제네릭 클래스 `class Foo<T>` |
| T가 특정 메서드에서만 사용됨 | 제네릭 메서드 `void Bar<T>()` |
| 정적 유틸리티 함수 | 제네릭 메서드 |

> 타입 매개변수가 메서드 경계 안에서만 사용된다면 클래스 전체가 아닌 해당 메서드만 제네릭으로 선언하라. 사용 측에서 타입 인수를 추론할 수 있어 더 간결한 코드를 작성할 수 있다.
