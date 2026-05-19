# Item 44: 강력한 예외 보장을 제공하라 (Provide Strong Exception Guarantees)

> **Chapter 5: Exception Practices**

---

## 핵심 요약

예외가 발생해도 프로그램이 일관된 상태를 유지하도록 설계하는 것이 **예외 안전성(exception safety)**이다. 예외 보장에는 세 가지 수준이 있으며, 가능하면 **강력한 보장(strong guarantee)**을 제공하라.

---

## 예외 보장의 세 가지 수준

```
nothrow 보장 (nothrow guarantee)
  - 연산이 절대 예외를 던지지 않음
  - 소멸자, swap, move에서 필수
  - 예: Dispose(), ~T(), std::swap

강력한 보장 (strong guarantee)
  - 예외 발생 시 연산 이전 상태로 완전히 롤백됨
  - commit-or-rollback 의미론
  - 예: DB 트랜잭션, 불변 객체 교체

기본 보장 (basic guarantee)
  - 예외 후 객체가 유효한 상태이지만 이전 상태와 다를 수 있음
  - 최소한 보장해야 하는 수준
  - 예: std::vector::push_back (용량 초과 시 재할당 실패)

보장 없음 (no guarantee)
  - 예외 후 객체 상태가 정의되지 않음
  - 피해야 함
```

---

## 강력한 보장 구현: Copy-Swap 이디엄

```csharp
public class DataStore
{
    private List<Record> records;

    // 나쁜 예: 중간에 예외 발생 시 부분 수정된 상태가 됨
    public void UpdateBad(IEnumerable<Record> newRecords)
    {
        records.Clear();                          // 기존 데이터 삭제
        foreach (var r in newRecords)             // 여기서 예외 발생하면
            records.Add(ValidateAndTransform(r)); // records가 일부만 채워짐!
    }

    // 좋은 예: Copy-Swap — 완전히 성공하거나 아무것도 바꾸지 않음
    public void Update(IEnumerable<Record> newRecords)
    {
        // 1. 새 상태를 임시 컬렉션에서 완전히 구성
        var temp = new List<Record>();
        foreach (var r in newRecords)
            temp.Add(ValidateAndTransform(r)); // 여기서 예외 → records는 그대로

        // 2. 완전히 성공했을 때만 교체 (원자적)
        records = temp; // 참조 교체는 예외 없음
    }
}
```

---

## 불변 객체로 강력한 보장 달성

```csharp
// 불변 타입은 기본적으로 예외 안전
public sealed class ImmutablePoint
{
    public int X { get; }
    public int Y { get; }

    public ImmutablePoint(int x, int y) { X = x; Y = y; }

    // 새 인스턴스를 반환 — 기존 인스턴스는 절대 변하지 않음
    public ImmutablePoint WithX(int newX) => new ImmutablePoint(newX, Y);
    public ImmutablePoint WithY(int newY) => new ImmutablePoint(X, newY);
}

// 사용: 실패해도 원본은 안전
ImmutablePoint point = new ImmutablePoint(1, 2);
try
{
    var updated = point.WithX(GetNewX()); // 예외 발생 가능
    point = updated;                      // 성공한 경우에만 교체
}
catch { /* point는 여전히 (1,2) */ }
```

---

## 소멸자와 Dispose: nothrow 보장

```csharp
// 소멸자에서 예외가 나오면 프로세스 종료
// Dispose도 반드시 nothrow여야 함
public class SafeResource : IDisposable
{
    private bool disposed;

    public void Dispose()
    {
        if (disposed) return;
        try
        {
            // 리소스 해제 시도
            ReleaseHandle();
        }
        catch (Exception ex)
        {
            // 로깅만 하고 예외를 밖으로 전파하지 않는다
            Logger.LogError(ex, "리소스 해제 실패 (무시하고 계속)");
        }
        finally
        {
            disposed = true;
        }
    }
}
```

---

## 강력한 보장이 어려울 때: 기본 보장이라도

```csharp
public void Process(List<Order> orders)
{
    // 강력한 보장을 제공할 수 없다면, 최소한 기본 보장은 해야 함:
    // 예외 후 orders가 유효한 상태임을 보장

    foreach (var order in orders)
    {
        try
        {
            ProcessOrder(order); // 개별 주문 처리
        }
        catch (OrderProcessingException ex)
        {
            order.Status = OrderStatus.Failed; // 일관된 실패 상태로 표시
            Logger.LogError(ex, $"주문 {order.Id} 처리 실패");
            // 예외를 삼키고 계속 처리 — orders 전체는 유효한 상태 유지
        }
    }
}
```

---

## 결론

> 예외 안전성 수준은 nothrow > strong > basic > none이다. 소멸자·swap·Dispose는 반드시 nothrow로 구현하고, 상태 변경이 있는 연산은 Copy-Swap 이디엄이나 불변 객체 교체로 강력한 보장을 달성하라. 강력한 보장이 불가능하다면 최소한 기본 보장(예외 후 객체가 유효한 상태)을 제공해야 한다.
