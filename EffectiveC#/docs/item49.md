# Item 49: 예외 전파 시 정보 손실을 방지하라 (Preserve the Original Exception When Wrapping Exceptions)

> **Chapter 5: Exception Practices**

---

## 핵심 요약

레이어 경계를 넘을 때 예외를 래핑하는 것은 좋은 설계지만, 원래 예외를 inner exception으로 반드시 포함시켜야 한다. 그렇지 않으면 근본 원인을 추적할 수 없게 된다. `ExceptionDispatchInfo`를 이용하면 스택 추적까지 완전히 보존할 수 있다.

---

## 예외 연쇄(Exception Chaining)

```csharp
// 레이어 경계: Infrastructure → Application → Domain
// 각 레이어가 예외를 변환하되 원인은 유지

// Infrastructure 레이어
public User GetUser(int id)
{
    try { return _db.Users.Find(id); }
    catch (SqlException ex)
    {
        throw new UserRepositoryException(
            $"사용자 {id} 조회 중 DB 오류",
            innerException: ex); // ← 필수
    }
}

// Application 레이어
public UserDto LoadUserProfile(int userId)
{
    try { return _mapper.Map(_repo.GetUser(userId)); }
    catch (UserRepositoryException ex)
    {
        throw new UserServiceException(
            $"사용자 프로필 로드 실패 (userId={userId})",
            innerException: ex); // ← 연쇄 유지
    }
}

// 로그에서 재현되는 예외 연쇄:
// UserServiceException → UserRepositoryException → SqlException
```

---

## throw ex vs throw: 스택 추적 차이

```csharp
// 나쁜 예: throw ex — 스택 추적을 catch 위치로 교체
catch (Exception ex)
{
    Logger.Log(ex);
    throw ex; // ex.StackTrace가 catch 위치로 리셋됨!
}

// 좋은 예: throw — 원래 스택 추적 유지
catch (Exception ex)
{
    Logger.Log(ex);
    throw; // 원래 스택 추적 보존
}
```

---

## ExceptionDispatchInfo: 완전한 스택 추적 보존

```csharp
// 비동기 코드나 스레드 경계에서 예외를 저장했다가 다른 컨텍스트에서 재throw
using System.Runtime.ExceptionServices;

ExceptionDispatchInfo captured = null;

try
{
    PerformOperation();
}
catch (Exception ex)
{
    // 예외를 캡처 — 스택 추적 포함 전체 상태 저장
    captured = ExceptionDispatchInfo.Capture(ex);
}

// 나중에 다른 스레드/컨텍스트에서
if (captured != null)
{
    captured.Throw(); // 원래 스택 추적을 유지하며 재throw
    // 로그에서: "--- End of stack trace from previous location ---" 구분선 표시
}
```

---

## AggregateException: 여러 예외 수집

```csharp
// 병렬 처리에서 여러 예외를 수집하여 전파
public void ProcessAll(IEnumerable<Order> orders)
{
    var exceptions = new List<Exception>();

    foreach (var order in orders)
    {
        try { Process(order); }
        catch (Exception ex) { exceptions.Add(ex); }
    }

    if (exceptions.Count == 1)
        ExceptionDispatchInfo.Capture(exceptions[0]).Throw(); // 단일 예외는 원본 전파

    if (exceptions.Count > 1)
        throw new AggregateException("일부 주문 처리 실패", exceptions);
}

// 호출자에서 풀기
catch (AggregateException ae)
{
    ae.Handle(ex => {
        if (ex is OrderValidationException) { Handle(ex); return true; }
        return false; // 처리 못한 예외는 다시 포함됨
    });
}
```

---

## 예외 정보를 Data 딕셔너리에 보강

```csharp
// 커스텀 예외 클래스 없이 기존 예외에 컨텍스트 추가
try
{
    ProcessOrder(order);
}
catch (Exception ex) when (AddContext(ex, order))
{
    throw; // when이 항상 false를 반환하므로 여기 도달 안 함
}

static bool AddContext(Exception ex, Order order)
{
    // Exception.Data는 키-값 딕셔너리
    ex.Data["OrderId"] = order.Id;
    ex.Data["CustomerId"] = order.CustomerId;
    ex.Data["OrderTotal"] = order.Total;
    return false; // 예외를 처리하지 않고 전파
}
// 로그: ex.Data에 OrderId, CustomerId, OrderTotal이 포함됨
```

---

## 결론

> 예외를 래핑할 때는 반드시 inner exception을 포함시켜 예외 연쇄를 유지하라. 재throw는 `throw;`로 하고 절대 `throw ex;`를 사용하지 마라. 스레드 경계나 비동기 코드에서 예외를 전파할 때는 `ExceptionDispatchInfo`로 원래 스택 추적까지 보존하라. 진단 가능한 예외가 좋은 예외다.
