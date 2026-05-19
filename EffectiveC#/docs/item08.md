# Item 8: 이벤트 호출에는 null 조건 연산자를 사용하라 (Use the Null Conditional Operator for Event Invocations)

> **Chapter 1: C# Language Idioms**

---

## 핵심 요약

이벤트를 안전하게 호출하려면 null 검사와 호출 사이에 구독자가 해제되는 **레이스 컨디션(race condition)** 을 방지해야 한다. C# 6.0의 null 조건 연산자(`?.`)를 사용하면 스레드 안전하게 이벤트를 호출할 수 있다.

---

## 문제: null 검사 후 호출 사이의 레이스 컨디션

```csharp
public event EventHandler<EventArgs> OnChanged;

// 나쁜 예 1: null 검사 없음 → NullReferenceException 가능
OnChanged(this, EventArgs.Empty);

// 나쁜 예 2: null 검사 후 호출 → 멀티스레드 환경에서 레이스 컨디션
if (OnChanged != null)
{
    // 이 시점에 다른 스레드가 마지막 구독자를 해제하면?
    OnChanged(this, EventArgs.Empty); // NullReferenceException 발생 가능!
}
```

### 레이스 컨디션 발생 시나리오

```
스레드 A                          스레드 B
if (OnChanged != null)  ←  null 아님 확인
                              OnChanged -= handler; ← 마지막 구독자 해제
OnChanged(...)          ←  NullReferenceException!
```

---

## 구식 해결책: 임시 변수에 복사

```csharp
// C# 6.0 이전의 안전한 패턴
var handler = OnChanged; // 현재 참조를 로컬에 복사 (스냅샷)
if (handler != null)
{
    handler(this, EventArgs.Empty); // 복사본으로 호출
}
```

임시 변수에 복사하면 스레드 B가 구독자를 해제해도 로컬 변수 `handler`는 원래 참조를 유지한다. 하지만 매번 이 패턴을 작성해야 해서 번거롭고 실수하기 쉽다.

---

## null 조건 연산자로 간결하게

```csharp
// C# 6.0 이후: null 조건 연산자 활용
OnChanged?.Invoke(this, EventArgs.Empty);
```

`?.`는 내부적으로 임시 변수 복사 패턴을 컴파일러가 생성한다. 스레드 안전하고 간결하다.

```
// 컴파일러가 생성하는 실제 코드와 동일한 의미
var temp = OnChanged;
if (temp != null) temp.Invoke(this, EventArgs.Empty);
```

---

## null 조건 연산자의 다양한 활용

```csharp
// 속성 접근
string name = person?.Name;          // person이 null이면 null 반환
int? length = person?.Name?.Length;  // 체이닝 가능

// 메서드 호출
person?.Speak("Hello");              // person이 null이면 아무것도 안 함

// 인덱서
var first = list?[0];                // list가 null이면 null 반환

// null 병합 연산자와 조합
string name = person?.Name ?? "미지정";

// 이벤트 호출 (핵심 사용 사례)
PropertyChanged?.Invoke(this, new PropertyChangedEventArgs(nameof(Name)));
```

---

## 완성된 INotifyPropertyChanged 구현

```csharp
public class Person : INotifyPropertyChanged
{
    public event PropertyChangedEventHandler PropertyChanged;

    private string name;
    public string Name
    {
        get => name;
        set
        {
            if (name == value) return;
            name = value;
            // 스레드 안전한 이벤트 호출
            PropertyChanged?.Invoke(this,
                new PropertyChangedEventArgs(nameof(Name)));
        }
    }
}
```

---

## null 조건 연산자와 일반 null 검사의 차이

```csharp
// null 조건 연산자: 좌항이 null이면 전체 표현식이 null
int? count = list?.Count;  // list가 null이면 count = null

// 일반 null 검사: 명시적
int count = list != null ? list.Count : 0;

// null 병합으로 기본값 설정
int count = list?.Count ?? 0;
```

---

## 결론

| 상황 | 권장 |
|------|------|
| 이벤트 호출 | `event?.Invoke(...)` ✅ |
| null일 수 있는 객체의 멤버 접근 | `obj?.Member` ✅ |
| 체이닝된 null 체크 | `a?.b?.c` ✅ |
| 기본값이 필요한 경우 | `obj?.Value ?? default` ✅ |

> 이벤트 호출에는 반드시 `?.Invoke()` 패턴을 사용하라. 임시 변수 복사 패턴보다 간결하고 동일하게 스레드 안전하다. null 조건 연산자는 이벤트 호출을 넘어 null일 수 있는 모든 참조 접근에 활용할 수 있다.
