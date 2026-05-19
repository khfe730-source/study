# Item 3: 캐스트보다 is 또는 as 연산자를 사용하라 (Prefer the is or as Operators to Casts)

> **Chapter 1: C# Language Idioms**

---

## 핵심 요약

강제 형변환(cast)은 변환 실패 시 예외를 던지고 사용자 정의 변환 연산자까지 실행할 수 있어 예측하기 어렵다. `as`와 `is` 연산자는 변환 실패 시 `null`이나 `false`를 반환해 더 안전하고 의도가 명확하다.

---

## 강제 형변환(cast)의 문제점

```csharp
object obj = GetObject();

// 강제 형변환: 실패 시 InvalidCastException 발생
MyType t = (MyType)obj;

// 문제 1: 예외 처리를 강제함
try
{
    MyType t = (MyType)obj;
}
catch (InvalidCastException)
{
    // 변환 실패 처리
}

// 문제 2: 사용자 정의 변환 연산자도 실행됨 (의도치 않은 변환 가능)
```

---

## as 연산자

`as`는 참조 타입 및 nullable 타입에 대한 변환을 시도하고, 실패하면 `null`을 반환한다. 예외를 던지지 않는다.

```csharp
object obj = GetObject();

MyType t = obj as MyType;

if (t != null)
{
    // 변환 성공 시 처리
    t.DoSomething();
}
```

### as vs cast 성능 비교

`as`는 내부적으로 `isinst` IL 명령 하나로 처리된다. 캐스트 + `try/catch`보다 훨씬 효율적이다.

```
캐스트 실패: isinst → 실패 → 예외 객체 생성 → 스택 언와인드 → catch
as 실패:     isinst → null 반환 (끝)
```

---

## is 연산자 (C# 7.0 이후 패턴 매칭)

C# 7.0부터 `is`에 패턴 매칭이 추가되어 타입 확인과 변환을 한 번에 처리할 수 있다.

```csharp
object obj = GetObject();

// C# 7.0 이전
if (obj is MyType)
{
    MyType t = (MyType)obj; // 두 번 확인
}

// C# 7.0 이후 (타입 패턴)
if (obj is MyType t)
{
    t.DoSomething(); // 바로 사용 가능
}
```

### switch 문에서의 패턴 매칭

```csharp
switch (shape)
{
    case Circle c:
        Console.WriteLine($"원 반지름: {c.Radius}");
        break;
    case Rectangle r:
        Console.WriteLine($"직사각형: {r.Width} x {r.Height}");
        break;
    case null:
        throw new ArgumentNullException(nameof(shape));
    default:
        throw new ArgumentException("알 수 없는 도형");
}
```

---

## as를 사용할 수 없는 경우

`as`는 **값 타입(value type)** 에 직접 사용할 수 없다. 값 타입은 `null`이 될 수 없기 때문이다.

```csharp
object obj = 42;

int n = obj as int;         // 컴파일 오류!
int? n = obj as int?;       // nullable로 감싸면 가능
```

값 타입 변환 시에는 `is` 패턴 매칭을 사용한다.

```csharp
if (obj is int n)
{
    Console.WriteLine(n);
}
```

---

## 언박싱(Unboxing)과의 관계

언박싱은 박싱된 값 타입을 꺼낼 때 반드시 정확한 타입을 지정해야 한다. `as`를 사용할 수 없으므로 `is` 패턴을 활용한다.

```csharp
object boxed = 42; // int를 object에 박싱

// 나쁜 예: 잘못된 타입으로 언박싱 시 InvalidCastException
long l = (long)boxed; // 오류! int는 long으로 직접 언박싱 불가

// 좋은 예
if (boxed is int i)
{
    long l = i; // 언박싱 후 안전하게 변환
}
```

---

## 세 가지 방식 비교

| 방식 | 실패 시 | 값 타입 지원 | 사용자 변환 연산자 |
|------|---------|-------------|------------------|
| `(T)obj` (cast) | `InvalidCastException` | ✅ | ✅ (실행됨) |
| `obj as T` | `null` 반환 | ❌ (nullable 필요) | ❌ (무시) |
| `obj is T t` | `false` / `t`는 기본값 | ✅ | ❌ (무시) |

---

## 결론

> 타입 변환이 실패할 가능성이 있는 경우 항상 `as` 또는 `is` 패턴 매칭을 사용하라. 강제 형변환은 변환이 반드시 성공함을 확신할 때만 사용한다. C# 7.0 이후라면 `is` 패턴 매칭이 `as` + null 확인보다 간결하고 안전하다.
