# Item 22: 제네릭 공변성과 반공변성을 지원하라 (Support Generic Covariance and Contravariance)

> **Chapter 3: Working with Generics**

---

## 핵심 요약

`out` 키워드로 공변성(covariance)을, `in` 키워드로 반공변성(contravariance)을 선언해 제네릭 인터페이스/델리게이트의 유연성을 높여라. 인터페이스를 설계할 때 타입 매개변수가 출력(반환)에만 쓰이면 `out`, 입력(매개변수)에만 쓰이면 `in`을 적용할 수 있다.

---

## 불변성(Invariance)의 문제

```csharp
// 배열은 공변성 지원 (하지만 런타임 오류 가능)
string[] strings = new string[3];
object[] objects = strings; // 컴파일 OK (배열 공변성)
objects[0] = 42; // 런타임 ArrayTypeMismatchException!

// 제네릭은 기본적으로 불변(invariant)
List<string> strList = new List<string>();
List<object> objList = strList; // 컴파일 오류! ❌
```

`List<string>`은 `List<object>`로 대입할 수 없다. 이를 불변성이라 한다.

---

## 공변성 (Covariance): `out`

타입 매개변수가 **반환값(출력)에만** 사용될 때 공변성을 선언할 수 있다.

```
공변성: Derived → Base 방향으로 변환 가능
IEnumerable<string> → IEnumerable<object>
```

```csharp
// IEnumerable<T>는 공변 (out T)
public interface IEnumerable<out T>
{
    IEnumerator<T> GetEnumerator();
}

// 공변성 덕분에 가능한 코드
IEnumerable<string> strings = new List<string> { "a", "b" };
IEnumerable<object> objects = strings; // OK! (out T 공변성)

// 직접 공변 인터페이스 선언
public interface IProducer<out T>
{
    T Produce(); // T는 반환에만 사용 → out 가능
    // void Consume(T item); // ❌ 매개변수에 사용 → out 불가
}

IProducer<string> stringProducer = GetStringProducer();
IProducer<object> objectProducer = stringProducer; // OK!
```

---

## 반공변성 (Contravariance): `in`

타입 매개변수가 **매개변수(입력)에만** 사용될 때 반공변성을 선언할 수 있다.

```
반공변성: Base → Derived 방향으로 변환 가능
IComparer<object> → IComparer<string>
```

```csharp
// IComparer<T>는 반공변 (in T)
public interface IComparer<in T>
{
    int Compare(T x, T y); // T는 입력에만 사용 → in 가능
}

// 반공변성 덕분에 가능한 코드
IComparer<object> objComparer = Comparer<object>.Default;
IComparer<string> strComparer = objComparer; // OK! (in T 반공변성)

// 직접 반공변 인터페이스 선언
public interface IConsumer<in T>
{
    void Consume(T item); // T는 입력에만 사용 → in 가능
    // T Produce(); // ❌ 반환에 사용 → in 불가
}

IConsumer<object> objConsumer = GetObjectConsumer();
IConsumer<string> strConsumer = objConsumer; // OK!
```

---

## 델리게이트의 공변성과 반공변성

`Func<>`, `Action<>` 같은 표준 델리게이트도 공변/반공변을 지원한다.

```csharp
// Func<out TResult>: 반환 타입에 대해 공변
Func<string> strFunc = () => "hello";
Func<object> objFunc = strFunc; // OK! string → object (공변)

// Func<in T, out TResult>: 매개변수에 반공변, 반환에 공변
Func<object, string> f1 = obj => obj.ToString();
Func<string, object> f2 = f1; // OK! (반공변 + 공변)
```

---

## 공변성/반공변성 결정 기준

```
타입 매개변수 T가...
├── 반환값에만 사용됨 → out (공변)
├── 매개변수에만 사용됨 → in (반공변)
└── 둘 다 사용됨 → 불변 (변성 선언 불가)
```

```csharp
// 공변 가능
public interface IReader<out T>
{
    T Read();           // 반환에만 사용 ✅
}

// 반공변 가능
public interface IWriter<in T>
{
    void Write(T value); // 매개변수에만 사용 ✅
}

// 불변 (공변/반공변 불가)
public interface IReadWrite<T>
{
    T Read();            // 반환에 사용
    void Write(T value); // 매개변수에도 사용 → 불변
}
```

---

## 결론

> 인터페이스나 델리게이트를 설계할 때 타입 매개변수의 사용 방향을 분석하라. 출력(반환)에만 쓰이면 `out`으로 공변성을, 입력(매개변수)에만 쓰이면 `in`으로 반공변성을 선언해 API의 유연성을 높여라.
