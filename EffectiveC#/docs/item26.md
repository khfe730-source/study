# Item 26: 제네릭 인터페이스와 함께 고전 인터페이스도 구현하라 (Implement Classic Interfaces in Addition to Generic Interfaces)

> **Chapter 3: Working with Generics**

---

## 핵심 요약

제네릭 인터페이스(`IComparable<T>`, `IEquatable<T>` 등)를 구현할 때는 비제네릭 버전(`IComparable`, `IEquatable`)도 함께 구현해야 한다. 기존 프레임워크 코드, 컬렉션, 정렬 알고리즘이 비제네릭 인터페이스를 기대하는 경우가 많기 때문이다.

---

## 제네릭 인터페이스만 구현했을 때의 문제

```csharp
public struct Temperature : IComparable<Temperature>
{
    private readonly double celsius;

    public Temperature(double celsius) => this.celsius = celsius;

    public int CompareTo(Temperature other) => celsius.CompareTo(other.celsius);
}

// 문제: 비제네릭 IComparable을 기대하는 코드에서 오류
ArrayList list = new ArrayList();
list.Add(new Temperature(100));
list.Add(new Temperature(0));
list.Sort(); // 런타임 오류: Temperature가 IComparable을 구현하지 않음
```

---

## 두 가지 인터페이스를 모두 구현

```csharp
public struct Temperature :
    IComparable<Temperature>,   // 제네릭 (핵심)
    IComparable,                // 비제네릭 (하위 호환성)
    IEquatable<Temperature>,    // 제네릭 동등성
    IEquatable                  // 비제네릭 동등성 (직접 인터페이스는 없음, Equals 재정의)
{
    private readonly double celsius;

    public Temperature(double celsius) => this.celsius = celsius;

    // 제네릭 구현 (핵심 로직)
    public int CompareTo(Temperature other)
        => celsius.CompareTo(other.celsius);

    // 비제네릭 구현 (하위 호환, 제네릭 버전에 위임)
    int IComparable.CompareTo(object obj)
    {
        if (obj is Temperature t) return CompareTo(t);
        if (obj == null) return 1;
        throw new ArgumentException(
            $"Object must be of type {nameof(Temperature)}", nameof(obj));
    }

    // IEquatable<T> 구현
    public bool Equals(Temperature other)
        => celsius.Equals(other.celsius);

    // object.Equals 재정의 (IEquatable<T>와 일관성 유지)
    public override bool Equals(object obj)
        => obj is Temperature t && Equals(t);

    public override int GetHashCode() => celsius.GetHashCode();

    // 관계 연산자
    public static bool operator ==(Temperature left, Temperature right) => left.Equals(right);
    public static bool operator !=(Temperature left, Temperature right) => !left.Equals(right);
    public static bool operator <(Temperature left, Temperature right) => left.CompareTo(right) < 0;
    public static bool operator >(Temperature left, Temperature right) => left.CompareTo(right) > 0;
    public static bool operator <=(Temperature left, Temperature right) => left.CompareTo(right) <= 0;
    public static bool operator >=(Temperature left, Temperature right) => left.CompareTo(right) >= 0;
}
```

---

## 명시적 인터페이스 구현으로 비제네릭 숨기기

비제네릭 버전은 박싱과 타입 안전성 문제가 있으므로 명시적으로 구현해 직접 호출을 막는다.

```csharp
// int IComparable.CompareTo(object obj) → 명시적 구현
// Temperature 타입으로 직접 호출 불가:
Temperature t = new Temperature(100);
// t.CompareTo(someObject); // ❌ 컴파일 오류
// ((IComparable)t).CompareTo(someObject); // ✅ 캐스팅 후 가능

// IComparable<Temperature>는 직접 호출 가능:
t.CompareTo(new Temperature(50)); // ✅
```

---

## IEquatable<T>의 중요성

`IEquatable<T>`를 구현하면 `Dictionary<T, V>`, `HashSet<T>` 등에서 박싱 없이 비교가 이루어진다.

```csharp
// IEquatable<T> 없을 때: object.Equals() 호출 → 박싱 발생
var set = new HashSet<Temperature>(); // Temperature가 IEquatable<Temperature> 없으면 비효율

// IEquatable<Temperature> 있을 때: 직접 Equals(Temperature) 호출 → 박싱 없음
var set = new HashSet<Temperature>(); // 효율적
set.Add(new Temperature(37));
set.Contains(new Temperature(37)); // true, 박싱 없음
```

---

## 구현 체크리스트

값 타입이 순서와 동등성을 지원해야 한다면:

- [ ] `IComparable<T>` 구현
- [ ] `IComparable` 명시적 구현 (하위 호환)
- [ ] `IEquatable<T>` 구현
- [ ] `object.Equals(object)` 재정의
- [ ] `GetHashCode()` 재정의
- [ ] `==`, `!=` 연산자 오버로드
- [ ] `<`, `>`, `<=`, `>=` 연산자 오버로드

---

## 결론

> 제네릭 인터페이스(`IComparable<T>`, `IEquatable<T>`)를 구현할 때는 비제네릭 버전도 함께 구현하라. 비제네릭 버전은 명시적으로 구현해 직접 호출을 억제하고, 프레임워크 호환성만 유지하는 데 사용하라. 핵심 로직은 제네릭 버전에 두고, 비제네릭은 제네릭 버전에 위임한다.
