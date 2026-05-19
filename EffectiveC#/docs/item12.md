# Item 12: 생성자 본문 대신 멤버 이니셜라이저를 사용하라 (Prefer Member Initializers to Assignment Statements)

> **Chapter 2: .NET Resource Management**

---

## 핵심 요약

멤버 변수는 생성자 본문에서 초기화하는 대신, **선언 시점에 이니셜라이저로 초기화**하라. 생성자가 여러 개일 때 초기화 누락을 방지하고 코드를 단순하게 유지할 수 있다.

---

## 문제: 여러 생성자에서 초기화 동기화

생성자가 늘어날수록 각 생성자에서 초기화 코드를 중복 작성하게 되고, 나중에 멤버 변수가 추가될 때 일부 생성자에서 초기화를 빠뜨리기 쉽다.

---

## 해결: 멤버 이니셜라이저

```csharp
public class MyClass
{
    // 선언과 동시에 초기화 → 모든 생성자에서 자동 보장
    private List<string> labels = new List<string>();
}
```

컴파일러는 **각 생성자 코드 앞에** 이니셜라이저 코드를 자동 삽입한다. 새 생성자를 추가해도 초기화를 잊을 위험이 없다.

### 실행 순서

1. 이니셜라이저 실행 (선언 순서대로)
2. 기반 클래스(base class) 생성자 실행
3. 현재 클래스의 생성자 본문 실행

---

## 멤버 이니셜라이저를 쓰지 말아야 하는 3가지 경우

### 1. 0 또는 null로 초기화할 때

```csharp
MyValType myVal1;                   // 시스템이 0으로 설정 (효율적)
MyValType myVal2 = new MyValType(); // initobj 명령 → box/unbox 발생 (비효율)
```

두 문장 모두 0으로 초기화되지만, 두 번째는 불필요한 `initobj` IL 명령이 추가된다. `new MyValType()`을 이니셜라이저로 쓰는 것은 이중 초기화다.

### 2. 생성자에 따라 다르게 초기화할 때

```csharp
public class MyClass2
{
    private List<string> labels = new List<string>(); // 이니셜라이저: 항상 실행

    MyClass2() { }

    MyClass2(int size)
    {
        labels = new List<string>(size); // List가 두 번 생성 → 첫 번째는 즉시 가비지!
    }
}
```

이 경우 이니셜라이저를 제거하고 생성자 체이닝(→ [Item 14](./item14.md))을 사용한다.

### 3. 예외 처리가 필요할 때

이니셜라이저는 `try` 블록으로 감쌀 수 없다. 초기화 중 예외가 발생하면 클래스 내부에서 복구가 불가능하다. 이 경우 생성자 본문에서 초기화하고 예외를 처리해야 한다.

```csharp
public MyClass()
{
    try
    {
        labels = new List<string>(); // 예외 처리 가능
    }
    catch (Exception ex)
    {
        // 복구 로직
    }
}
```

---

## 결론

| 상황 | 권장 방식 |
|------|-----------|
| 모든 생성자에서 동일하게 초기화 | 멤버 이니셜라이저 ✅ |
| 0/null 초기화 | 이니셜라이저 생략 (시스템이 처리) |
| 생성자마다 다른 초기화 | 생성자 체이닝 (Item 14) |
| 초기화 중 예외 처리 필요 | 생성자 본문 |

> 모든 생성자에서 동일한 방식으로 초기화되는 멤버 변수는 이니셜라이저로 선언하라. 코드가 간결해지고 초기화 누락 버그를 방지할 수 있다.
