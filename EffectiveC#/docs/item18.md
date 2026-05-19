# Item 18: 필요하고 충분한 제약 조건을 정의하라 (Always Define Constraints That Are Necessary and Sufficient)

> **Chapter 3: Working with Generics**

---

## 핵심 요약

제네릭 타입 매개변수의 제약 조건(constraint)은 컴파일러에게 타입 매개변수가 어떤 기능을 지원하는지 알려준다. 제약 조건이 너무 적으면 컴파일 오류가 발생하고, 너무 많으면 사용자가 해당 제네릭 타입을 사용하기 어려워진다. 필요한 만큼만, 정확하게 정의하라.

---

## 제약 조건이 없을 때의 문제

```csharp
// 제약 조건 없는 제네릭: T에 대해 아무 것도 가정할 수 없음
public T Max<T>(T left, T right)
{
    // 컴파일 오류: T에 비교 연산자가 없을 수 있음
    return left > right ? left : right; // ❌
}
```

제약 조건 없이는 `T`를 `object`처럼 취급해야 하므로, 타입이 실제로 지원하는 기능을 사용할 수 없다.

---

## 제약 조건의 종류

```csharp
// 1. 인터페이스 제약
where T : IComparable<T>

// 2. 기반 클래스 제약
where T : MyBaseClass

// 3. 참조 타입 제약
where T : class

// 4. 값 타입 제약
where T : struct

// 5. 기본 생성자 제약
where T : new()

// 6. 다중 제약 (콤마로 구분)
where T : class, IDisposable, new()

// 7. 복수 타입 매개변수 제약
where TKey : IComparable<TKey>
where TValue : class
```

---

## 올바른 제약 조건 설계

```csharp
// 비교가 필요한 메서드 → IComparable<T> 제약
public T Max<T>(T left, T right) where T : IComparable<T>
{
    return left.CompareTo(right) > 0 ? left : right;
}

// 새 인스턴스 생성이 필요한 팩토리 → new() 제약
public T Create<T>() where T : new()
{
    return new T();
}

// 리소스 해제가 필요한 경우 → IDisposable 제약
public void UseAndDispose<T>(T resource) where T : IDisposable
{
    try { /* 사용 */ }
    finally { resource.Dispose(); }
}
```

---

## 제약 조건이 너무 많을 때의 문제

```csharp
// 과도한 제약: 사용할 수 있는 타입이 매우 제한됨
public void Process<T>(T item)
    where T : class, IComparable<T>, IDisposable, ISerializable, new()
{
    // 이 모든 인터페이스를 구현하는 타입이 얼마나 될까?
}
```

모든 제약을 만족하는 타입이 거의 없어 제네릭의 재사용성이 사라진다.

---

## 제약 조건 vs 런타임 타입 검사

제약 조건으로 표현하기 어려운 경우, 런타임 타입 검사를 사용할 수 있다. 단 이는 컴파일 타임 안전성을 희생한다 (→ Item 19 참고).

```csharp
// 제약 조건으로 표현
public void Sort<T>(List<T> list) where T : IComparable<T>
{
    list.Sort(); // IComparable<T> 보장
}

// 런타임 타입 검사 (차선책)
public void Sort<T>(List<T> list)
{
    if (list is List<IComparable> comparableList)
        comparableList.Sort();
    else
        throw new ArgumentException("T must implement IComparable");
}
```

---

## 제약 조건의 상속

파생 클래스나 인터페이스 구현에서 제약 조건은 상속되지 않는다. 재정의 시 동일한 제약 조건을 명시해야 한다.

```csharp
public abstract class Repository<T> where T : class, new()
{
    public abstract T FindById(int id);
}

// 파생 클래스에서도 동일한 제약 필요
public class UserRepository<T> : Repository<T> where T : class, new()
{
    public override T FindById(int id) { ... }
}
```

---

## 결론

> 타입 매개변수에 실제로 필요한 기능만 제약 조건으로 명시하라. 부족한 제약은 컴파일 오류를 유발하고, 과도한 제약은 재사용성을 떨어뜨린다. 제약 조건은 제네릭 메서드/타입의 "계약서"다.
