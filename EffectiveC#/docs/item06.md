# Item 6: 문자열 기반 API를 피하라 (Avoid String-ly Typed APIs)

> **Chapter 1: C# Language Idioms**

---

## 핵심 요약

속성 이름, 의존성 속성 키, 이벤트 이름 등을 문자열 리터럴로 전달하는 패턴은 컴파일러가 오류를 잡지 못하고 리팩터링 시 깨지기 쉽다. `nameof` 연산자, 강타입 표현식, CallerMemberName 특성 등을 사용해 타입 안전한 API를 만들어야 한다.

---

## 문자열 기반 API의 문제

```csharp
// INotifyPropertyChanged 구현 예시
public class Person : INotifyPropertyChanged
{
    private string name;

    public string Name
    {
        get => name;
        set
        {
            name = value;
            // 문제: 오타가 있어도 컴파일 오류 없음
            OnPropertyChanged("Name");  // "Nmae" 로 잘못 써도 모름
        }
    }

    // 리팩터링으로 속성 이름이 바뀌면 문자열은 자동으로 안 바뀜
    protected void OnPropertyChanged(string propertyName)
        => PropertyChanged?.Invoke(this, new PropertyChangedEventArgs(propertyName));

    public event PropertyChangedEventHandler PropertyChanged;
}
```

---

## 해결책 1: nameof 연산자 (C# 6.0)

`nameof`는 식별자(변수, 속성, 메서드, 타입)의 이름을 **컴파일 타임 문자열 상수**로 반환한다.

```csharp
public string Name
{
    get => name;
    set
    {
        name = value;
        OnPropertyChanged(nameof(Name)); // 컴파일러가 유효성 검사
    }
}
```

리팩터링으로 `Name`이 다른 이름으로 변경되면 `nameof(Name)`도 자동으로 업데이트된다.

```csharp
// nameof 사용 사례들
Console.WriteLine(nameof(Person));         // "Person"
Console.WriteLine(nameof(Person.Name));    // "Name"
Console.WriteLine(nameof(list.Count));     // "Count"

// 예외 메시지
void Process(string input)
{
    if (input == null)
        throw new ArgumentNullException(nameof(input)); // "input"
}
```

---

## 해결책 2: CallerMemberName 특성

`INotifyPropertyChanged`처럼 속성 세터에서 속성 이름을 전달하는 패턴에서는 `[CallerMemberName]`을 사용해 자동으로 호출자 이름을 주입받을 수 있다.

```csharp
using System.Runtime.CompilerServices;

public class Person : INotifyPropertyChanged
{
    private string name;

    public string Name
    {
        get => name;
        set
        {
            if (SetField(ref name, value))  // 속성 이름 전달 불필요
                ; // 변경 처리
        }
    }

    protected bool SetField<T>(ref T field, T value,
        [CallerMemberName] string propertyName = null)
    {
        if (EqualityComparer<T>.Default.Equals(field, value))
            return false;

        field = value;
        OnPropertyChanged(propertyName); // "Name"이 자동으로 주입됨
        return true;
    }

    protected void OnPropertyChanged([CallerMemberName] string propertyName = null)
        => PropertyChanged?.Invoke(this, new PropertyChangedEventArgs(propertyName));

    public event PropertyChangedEventHandler PropertyChanged;
}
```

---

## 해결책 3: 강타입 표현식 (Expression<Func<T>>)

WPF 바인딩, 검증 라이브러리 등에서 람다 표현식으로 속성을 참조하는 패턴이 있다.

```csharp
// 문자열 기반 (취약)
Bind("UserName");

// 표현식 기반 (타입 안전)
Bind(() => model.UserName); // 리팩터링에도 안전

// 표현식에서 속성 이름 추출 유틸리티
public static string GetPropertyName<T>(Expression<Func<T>> expression)
{
    var memberExpr = expression.Body as MemberExpression;
    return memberExpr?.Member.Name;
}
```

---

## 문자열 기반 API가 나타나는 흔한 패턴들

```csharp
// 1. INotifyPropertyChanged
OnPropertyChanged("PropertyName"); // ❌ → nameof 또는 CallerMemberName

// 2. 의존성 속성 (DependencyProperty)
DependencyProperty.Register("Name", ...); // ❌ → nameof(Name)

// 3. 예외 파라미터 이름
throw new ArgumentNullException("param"); // ❌ → nameof(param)

// 4. 리플렉션 기반 접근
typeof(MyClass).GetProperty("Name"); // ❌ → nameof(Name)
```

---

## 결론

| 패턴 | 권장 대안 |
|------|-----------|
| 속성/변수 이름 문자열 | `nameof` |
| 속성 세터에서 자신의 이름 | `[CallerMemberName]` |
| 표현식으로 속성 참조 | `Expression<Func<T>>` |
| 예외 파라미터 이름 | `nameof` |

> 타입 이름, 속성 이름, 이벤트 이름을 문자열 리터럴로 전달하는 코드를 발견하면 `nameof`로 대체하라. 이는 컴파일 타임 안전성을 확보하고 리팩터링 내성을 높이는 가장 간단한 방법이다.
