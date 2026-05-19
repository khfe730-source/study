# Item 7: 콜백은 델리게이트로 표현하라 (Express Callbacks with Delegates)

> **Chapter 1: C# Language Idioms**

---

## 핵심 요약

콜백(callback)을 표현하는 방법으로 단일 메서드 인터페이스보다 **델리게이트(delegate)** 를 우선하라. 델리게이트는 멀티캐스트, 람다 표현식, 메서드 그룹 변환을 지원하며 단일 메서드 콜백에 더 적합하다. `Func<>`, `Action<>`, `Predicate<T>`를 활용하면 델리게이트 타입 선언도 줄일 수 있다.

---

## 콜백의 두 가지 표현 방법

### 인터페이스 기반 (단일 메서드)

```csharp
// 인터페이스 정의
public interface ICallback
{
    void Execute(string message);
}

// 사용
public void Process(ICallback callback)
{
    callback.Execute("처리 완료");
}

// 호출 측에서 클래스를 별도로 만들어야 함
public class MyCallback : ICallback
{
    public void Execute(string message) => Console.WriteLine(message);
}

Process(new MyCallback()); // 번거로움
```

### 델리게이트 기반

```csharp
// 델리게이트로 직접 표현
public void Process(Action<string> callback)
{
    callback("처리 완료");
}

// 람다로 간결하게 전달 가능
Process(msg => Console.WriteLine(msg));

// 메서드 그룹 변환
Process(Console.WriteLine);
```

---

## 표준 델리게이트 타입 활용

직접 델리게이트 타입을 정의하는 대신, .NET이 제공하는 표준 제네릭 델리게이트를 활용한다.

```csharp
// Action<T...>: 반환값 없는 콜백 (void)
Action                    // 매개변수 없음
Action<string>            // string 하나
Action<string, int>       // string, int 두 개
Action<T1, T2, T3>        // 최대 16개 매개변수

// Func<T..., TResult>: 반환값 있는 콜백
Func<int>                 // int 반환, 매개변수 없음
Func<string, int>         // string → int
Func<string, bool>        // string → bool

// Predicate<T>: bool 반환 (Func<T, bool>과 동일)
Predicate<string>         // string → bool
```

---

## 멀티캐스트 (Multicast)

델리게이트는 **여러 메서드를 연결**할 수 있다. 인터페이스로는 불가능한 기능이다.

```csharp
Action<string> callbacks = null;

callbacks += msg => Console.WriteLine($"[로그] {msg}");
callbacks += msg => Console.WriteLine($"[알림] {msg}");
callbacks += msg => SaveToDb(msg);

callbacks?.Invoke("이벤트 발생"); // 세 가지 동작 모두 실행
```

---

## 클로저 (Closure)

람다는 외부 변수를 캡처할 수 있어 상태를 포함한 콜백을 인터페이스 없이 표현할 수 있다.

```csharp
int count = 0;

Action increment = () => count++;
Action<int> add = n => count += n;

increment();
increment();
add(5);

Console.WriteLine(count); // 7
```

---

## 인터페이스가 더 적합한 경우

모든 상황에서 델리게이트가 우월한 것은 아니다. 아래의 경우에는 인터페이스를 사용한다.

```csharp
// 1. 여러 메서드를 묶어야 할 때
public interface IRepository<T>
{
    T GetById(int id);
    void Save(T entity);
    void Delete(int id);
}

// 2. 콜백이 상태를 가져야 하고 명시적 계약이 필요할 때
// 3. COM Interop이나 기존 API와의 호환성이 필요할 때
```

---

## 실전 예제: 이벤트 처리 파이프라인

```csharp
public class DataProcessor
{
    private readonly List<Action<DataItem>> _preprocessors = new();
    private readonly Func<DataItem, bool> _filter;
    private readonly Action<DataItem> _onComplete;

    public DataProcessor(
        Func<DataItem, bool> filter,
        Action<DataItem> onComplete)
    {
        _filter = filter;
        _onComplete = onComplete;
    }

    public void AddPreprocessor(Action<DataItem> preprocessor)
        => _preprocessors.Add(preprocessor);

    public void Process(IEnumerable<DataItem> items)
    {
        foreach (var item in items.Where(_filter))
        {
            foreach (var preprocessor in _preprocessors)
                preprocessor(item);

            _onComplete(item);
        }
    }
}

// 사용
var processor = new DataProcessor(
    filter: item => item.IsValid,
    onComplete: item => Console.WriteLine($"완료: {item.Id}")
);

processor.AddPreprocessor(item => item.Normalize());
processor.AddPreprocessor(item => item.Validate());
```

---

## 결론

| 상황 | 권장 |
|------|------|
| 단일 메서드 콜백 | 델리게이트 (`Action`, `Func`) ✅ |
| 멀티캐스트 콜백 필요 | 델리게이트 ✅ |
| 람다/익명 함수로 전달 | 델리게이트 ✅ |
| 관련 메서드 묶음 계약 | 인터페이스 |
| 여러 메서드를 포함하는 API | 인터페이스 |

> 단일 메서드 콜백에는 인터페이스보다 델리게이트를 사용하라. `Action<>`, `Func<>` 등 표준 제네릭 델리게이트를 활용하면 별도의 타입 선언 없이도 타입 안전한 콜백을 표현할 수 있다.
