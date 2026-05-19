# Item 9: 박싱과 언박싱을 최소화하라 (Minimize Boxing and Unboxing)

> **Chapter 1: C# Language Idioms**

---

## 핵심 요약

박싱(boxing)은 값 타입을 참조 타입으로 변환하는 과정으로, 힙 할당과 복사가 발생한다. 박싱은 성능 비용이 크고 타입 안전성을 해친다. 제네릭(Generic)을 활용하면 박싱 없이 값 타입을 다룰 수 있다.

---

## 박싱과 언박싱이란

```
박싱 (Boxing):   값 타입 → object (힙에 복사본 생성)
언박싱 (Unboxing): object → 값 타입 (힙에서 스택으로 복사)
```

```csharp
int i = 42;
object obj = i;    // 박싱: 힙에 int 복사본 생성, obj는 그것을 가리킴
int j = (int)obj;  // 언박싱: 힙의 복사본을 스택으로 다시 복사
```

### 박싱의 메모리 구조

```
스택                 힙
┌──────┐            ┌──────────────┐
│  i   │ = 42       │ Object Header │
│      │            │ Type Pointer  │
│ obj  │ ────────→  │    42 (int)   │
└──────┘            └──────────────┘
```

박싱은 힙에 새 객체를 할당하고 값을 복사한다. 이 객체는 GC의 관리를 받는다.

---

## 박싱이 발생하는 흔한 상황

### 1. 제네릭 이전의 컬렉션

```csharp
// 나쁜 예: ArrayList는 object를 다룸 → 모든 값 타입이 박싱됨
ArrayList list = new ArrayList();
list.Add(42);       // 박싱!
list.Add(3.14);     // 박싱!
int n = (int)list[0]; // 언박싱!

// 좋은 예: 제네릭 컬렉션은 박싱 없음
List<int> list = new List<int>();
list.Add(42);       // 박싱 없음
int n = list[0];    // 언박싱 없음
```

### 2. 인터페이스를 통한 값 타입 접근

```csharp
interface IDisplayable { void Display(); }

struct Point : IDisplayable
{
    public int X, Y;
    public void Display() => Console.WriteLine($"({X}, {Y})");
}

Point p = new Point { X = 1, Y = 2 };
IDisplayable d = p;  // 박싱! 인터페이스 변수에 값 타입 할당
d.Display();
```

### 3. string.Format() / 문자열 연산

```csharp
int x = 42;
// Format의 인수는 object[]로 전달 → 박싱
string s = string.Format("값: {0}", x); // 박싱 발생

// 문자열 보간은 박싱을 줄여준다 (컴파일러 최적화)
string s = $"값: {x}"; // 박싱 없음 (ToString() 직접 호출)
```

### 4. object 매개변수를 받는 메서드

```csharp
void Log(object message) { ... } // 값 타입 전달 시 박싱

Log(42);     // 박싱
Log("text"); // 박싱 없음 (string은 참조 타입)
```

---

## 해결책: 제네릭(Generic) 사용

```csharp
// 박싱 없는 제네릭 메서드
void Log<T>(T message) { ... }

Log(42);     // 박싱 없음! T = int로 특수화
Log("text"); // 박싱 없음! T = string으로 특수화
```

제네릭은 값 타입에 대해 **타입별로 전용 IL 코드를 생성**하므로 박싱이 발생하지 않는다.

---

## 박싱의 성능 비용

박싱은 세 가지 비용을 수반한다:

1. **힙 메모리 할당**: GC 부담 증가
2. **값 복사**: 스택 → 힙 복사
3. **GC 수거**: 박싱된 객체는 GC가 처리해야 함

루프 안에서 박싱이 발생하면 심각한 성능 저하가 일어난다.

```csharp
// 박싱이 100만 번 발생하는 나쁜 예
var list = new ArrayList();
for (int i = 0; i < 1_000_000; i++)
    list.Add(i); // 100만 번 박싱 → 힙 객체 100만 개

// 박싱 없는 좋은 예
var list = new List<int>();
for (int i = 0; i < 1_000_000; i++)
    list.Add(i); // 박싱 없음
```

---

## 언박싱의 타입 안전성 문제

```csharp
object obj = 42; // int 박싱

// 잘못된 타입으로 언박싱 → 런타임 예외
long l = (long)obj; // InvalidCastException! int → long 직접 언박싱 불가

// 올바른 언박싱 후 변환
long l = (long)(int)obj; // int로 먼저 언박싱, 그 다음 long으로 변환
```

---

## ToString() 재정의로 박싱 방지

값 타입을 문자열로 변환할 때 `object.ToString()`이 호출되면 박싱이 발생한다. 값 타입에서 `ToString()`을 재정의하면 박싱 없이 처리된다.

```csharp
struct Temperature
{
    private double celsius;

    public Temperature(double celsius) => this.celsius = celsius;

    // ToString 재정의로 박싱 방지
    public override string ToString() => $"{celsius:F1}°C";
}

Temperature t = new Temperature(36.5);
Console.WriteLine(t); // ToString() 직접 호출, 박싱 없음
```

---

## 결론

| 상황 | 권장 |
|------|------|
| 컬렉션에 값 타입 저장 | `List<T>`, `Dictionary<K,V>` 등 제네릭 사용 |
| 값 타입을 받는 메서드 | 제네릭 메서드로 변경 |
| 문자열 포맷 | 문자열 보간 사용 (박싱 최소화) |
| 값 타입의 인터페이스 사용 | 최소화, 필요 시 명시적 관리 |

> 박싱은 성능 비용이 크고 런타임 타입 안전성을 해친다. 제네릭을 활용해 박싱을 제거하고, 박싱이 불가피한 경우 명시적으로 인지하고 최소화하라.
