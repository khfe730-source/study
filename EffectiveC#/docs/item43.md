# Item 43: 예외를 메서드 계약 위반 보고에 사용하라 (Use Exceptions to Report Method Contract Failures)

> **Chapter 5: Exception Practices**

---

## 핵심 요약

예외는 메서드의 **사전 조건(precondition)** 또는 **사후 조건(postcondition)** 위반처럼 계약이 깨졌을 때 사용하는 메커니즘이다. 예외가 없다면 호출자가 실패를 무시할 수 있다. 반환 코드(return code) 방식보다 예외가 더 명확하고 안전한 실패 보고 수단이다.

---

## 계약(Contract) 개념

메서드는 계약을 갖는다:
- **사전 조건(precondition)**: 메서드 호출 전 호출자가 보장해야 할 것
- **사후 조건(postcondition)**: 메서드 종료 후 구현자가 보장하는 것
- **불변 조건(invariant)**: 항상 참이어야 하는 클래스 상태

계약이 깨지면 예외를 던진다.

---

## 반환 코드 vs 예외

```csharp
// 나쁜 예: 반환 코드 — 호출자가 무시할 수 있음
public int Divide(int a, int b)
{
    if (b == 0) return -1; // 오류 코드? 유효한 음수값? 모호함
    return a / b;
}

var result = Divide(10, 0); // 반환값 무시 가능, 오류 전파 안 됨

// 좋은 예: 예외 — 호출자가 반드시 처리해야 함
public int Divide(int a, int b)
{
    if (b == 0) throw new ArgumentException("나누는 수는 0이 될 수 없습니다.", nameof(b));
    return a / b;
}
```

---

## 예외 유형별 사용 지침

```csharp
public class BankAccount
{
    private decimal balance;

    public void Deposit(decimal amount)
    {
        // 사전 조건 위반 → ArgumentException 계열
        if (amount <= 0)
            throw new ArgumentOutOfRangeException(nameof(amount),
                "입금액은 0보다 커야 합니다.");

        balance += amount;

        // 사후 조건 위반 → InvalidOperationException (내부 버그)
        if (balance < 0)
            throw new InvalidOperationException(
                "입금 후 잔액이 음수가 됐습니다. 내부 오류입니다.");
    }

    public void Withdraw(decimal amount)
    {
        // 상태 전제 조건 위반 → InvalidOperationException
        if (amount > balance)
            throw new InvalidOperationException(
                $"잔액({balance})이 출금액({amount})보다 적습니다.");

        balance -= amount;
    }
}
```

---

## 표준 예외 클래스 선택 기준

```
인자 관련 오류:
  ArgumentNullException       null이 허용되지 않는 인자
  ArgumentOutOfRangeException 값이 허용 범위를 벗어남
  ArgumentException           기타 인자 계약 위반

상태 관련 오류:
  InvalidOperationException   현재 객체 상태에서 작업이 불가
  ObjectDisposedException     Dispose된 객체에 접근

컬렉션 관련:
  IndexOutOfRangeException    배열/컬렉션 인덱스 범위 초과
  KeyNotFoundException        딕셔너리 키 없음

지원하지 않는 작업:
  NotSupportedException       이 구현에서 해당 기능 미지원
  NotImplementedException     아직 구현되지 않음 (임시)
```

---

## Guard 패턴으로 사전 조건 집중화

```csharp
// 재사용 가능한 Guard 클래스
public static class Guard
{
    public static void NotNull<T>(T value, string paramName) where T : class
    {
        if (value is null)
            throw new ArgumentNullException(paramName);
    }

    public static void NotEmpty(string value, string paramName)
    {
        if (string.IsNullOrWhiteSpace(value))
            throw new ArgumentException("빈 문자열은 허용되지 않습니다.", paramName);
    }

    public static void InRange(int value, int min, int max, string paramName)
    {
        if (value < min || value > max)
            throw new ArgumentOutOfRangeException(paramName,
                $"값은 [{min}, {max}] 범위여야 합니다. 실제 값: {value}");
    }
}

// 사용
public void RegisterUser(string username, int age)
{
    Guard.NotEmpty(username, nameof(username));
    Guard.InRange(age, 0, 150, nameof(age));

    // 이 아래부터는 사전 조건이 만족됨이 보장됨
    ...
}
```

---

## 결론

> 예외는 계약 위반을 호출자에게 강제로 알리는 메커니즘이다. 사전 조건은 `Argument` 계열로, 상태 위반은 `InvalidOperationException`으로 표현하라. 반환 코드는 무시될 수 있지만 예외는 처리하지 않으면 프로세스가 종료된다. 이 특성을 이용해 명확하고 안전한 API를 설계하라.
