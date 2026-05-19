# Item 19: 런타임 타입 검사로 제네릭 알고리즘을 특수화하라 (Specialize Generic Algorithms Using Runtime Type Checking)

> **Chapter 3: Working with Generics**

---

## 핵심 요약

제네릭 알고리즘은 일반화된 동작을 제공하지만, 특정 타입에 대해 더 효율적인 구현이 존재할 수 있다. 런타임 타입 검사(`is`, `as`)를 사용해 특정 타입에 대한 최적화된 경로를 추가하면, 범용성을 유지하면서도 성능을 개선할 수 있다.

---

## 문제: 일반화된 알고리즘의 한계

```csharp
// IEnumerable<T>를 순회하는 일반적인 방법
public static T Last<T>(IEnumerable<T> sequence)
{
    T result = default;
    foreach (var item in sequence)
        result = item;
    return result; // 전체를 순회해야 마지막 요소를 알 수 있음
}
```

`IEnumerable<T>`만 사용하면 항상 전체를 순회해야 한다. 하지만 실제로 `IList<T>`가 전달되면 인덱스로 마지막 요소에 바로 접근할 수 있다.

---

## 런타임 타입 검사로 특수화

```csharp
public static T Last<T>(IEnumerable<T> sequence)
{
    // IList<T>인 경우: 인덱스로 O(1) 접근
    if (sequence is IList<T> list)
    {
        if (list.Count == 0)
            throw new InvalidOperationException("Sequence is empty");
        return list[list.Count - 1];
    }

    // IReadOnlyList<T>인 경우
    if (sequence is IReadOnlyList<T> readOnlyList)
    {
        if (readOnlyList.Count == 0)
            throw new InvalidOperationException("Sequence is empty");
        return readOnlyList[readOnlyList.Count - 1];
    }

    // 일반 IEnumerable<T>: 전체 순회 O(n)
    T result = default;
    bool hasValue = false;
    foreach (var item in sequence)
    {
        result = item;
        hasValue = true;
    }

    if (!hasValue)
        throw new InvalidOperationException("Sequence is empty");

    return result;
}
```

---

## LINQ의 실제 활용 사례

.NET의 `Enumerable.Count()`, `Enumerable.Last()` 등이 이 패턴을 내부적으로 사용한다.

```csharp
// Enumerable.Count() 내부 구현 (개념적)
public static int Count<T>(this IEnumerable<T> source)
{
    // ICollection<T>는 Count 속성을 직접 제공
    if (source is ICollection<T> collection)
        return collection.Count;  // O(1)

    // ICollection도 동일
    if (source is ICollection col)
        return col.Count;  // O(1)

    // 순회 필요
    int count = 0;
    foreach (var _ in source)
        count++;
    return count; // O(n)
}
```

---

## 타입별 특수화된 직렬화 예시

```csharp
public static string Serialize<T>(T value)
{
    // 특수 타입에 대한 최적화된 처리
    if (value is string s)
        return $"\"{s}\"";

    if (value is DateTime dt)
        return dt.ToString("o"); // ISO 8601

    if (value is IEnumerable<object> enumerable)
        return $"[{string.Join(",", enumerable.Select(Serialize))}]";

    // 일반 처리
    return value?.ToString() ?? "null";
}
```

---

## 주의: 과도한 런타임 타입 검사는 설계 냄새

런타임 타입 검사를 과하게 사용하면 제네릭의 의미가 퇴색된다.

```csharp
// 나쁜 예: 사실상 타입별 switch문과 다를 바 없음
public static void Process<T>(T value)
{
    if (value is int i) { ProcessInt(i); return; }
    if (value is string s) { ProcessString(s); return; }
    if (value is DateTime dt) { ProcessDateTime(dt); return; }
    // T의 일반화된 처리가 없음 → 이게 제네릭이 맞는가?
}
```

런타임 타입 검사는 **성능 최적화를 위한 추가 경로**로만 사용해야 하며, 일반 경로는 반드시 존재해야 한다.

---

## 결론

> 제네릭 알고리즘의 일반 경로를 유지하면서, 특정 타입에 대해 더 효율적인 구현이 있다면 런타임 타입 검사로 최적화 경로를 추가하라. 이 패턴은 범용성과 성능을 함께 얻는 실용적인 접근이다.
