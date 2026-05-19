# Item 47: 애플리케이션 전용 예외 클래스를 완전하게 작성하라 (Create Complete Application-Specific Exception Classes)

> **Chapter 5: Exception Practices**

---

## 핵심 요약

커스텀 예외 클래스를 만들 때 직렬화 지원, inner exception 전달, 충분한 진단 정보를 모두 포함하는 **완전한 예외**를 작성해야 한다. 불완전한 커스텀 예외는 표준 예외보다 오히려 진단 능력을 떨어뜨린다.

---

## 커스텀 예외가 필요한 경우

```
커스텀 예외를 만들어야 할 때:
- 호출자가 이 특정 오류를 다른 오류와 구분하여 처리해야 할 때
- 오류를 진단하는 데 필요한 도메인 특화 데이터가 있을 때
- 라이브러리/레이어 경계를 넘을 때 예외를 감쌀 때

커스텀 예외가 불필요한 경우:
- 단순히 메시지만 다를 때 (ArgumentException 사용)
- 상위 레이어에서 구분하여 처리할 필요가 없을 때
```

---

## 완전한 커스텀 예외 클래스 템플릿

```csharp
[Serializable]
public class OrderProcessingException : Exception
{
    // 도메인 특화 데이터 — 진단에 필요한 정보 포함
    public int OrderId { get; }
    public OrderStatus CurrentStatus { get; }

    // 1. 기본 생성자 (역직렬화 외에는 사용 자제)
    public OrderProcessingException()
        : base("주문 처리 중 오류가 발생했습니다.") { }

    // 2. 메시지 생성자
    public OrderProcessingException(string message)
        : base(message) { }

    // 3. 메시지 + inner exception (가장 많이 사용)
    public OrderProcessingException(string message, Exception innerException)
        : base(message, innerException) { }

    // 4. 도메인 데이터 포함 생성자
    public OrderProcessingException(int orderId, OrderStatus status, string message)
        : base(message)
    {
        OrderId = orderId;
        CurrentStatus = status;
    }

    public OrderProcessingException(
        int orderId, OrderStatus status, string message, Exception innerException)
        : base(message, innerException)
    {
        OrderId = orderId;
        CurrentStatus = status;
    }

    // 5. 직렬화 생성자 (분산 시스템, AppDomain 경계 통과에 필요)
    protected OrderProcessingException(
        System.Runtime.Serialization.SerializationInfo info,
        System.Runtime.Serialization.StreamingContext context)
        : base(info, context)
    {
        OrderId = info.GetInt32(nameof(OrderId));
        CurrentStatus = (OrderStatus)info.GetInt32(nameof(CurrentStatus));
    }

    // 6. GetObjectData 오버라이드 — 커스텀 필드 직렬화
    public override void GetObjectData(
        System.Runtime.Serialization.SerializationInfo info,
        System.Runtime.Serialization.StreamingContext context)
    {
        base.GetObjectData(info, context);
        info.AddValue(nameof(OrderId), OrderId);
        info.AddValue(nameof(CurrentStatus), (int)CurrentStatus);
    }

    // 7. 의미 있는 ToString
    public override string ToString()
        => $"{base.ToString()}{Environment.NewLine}" +
           $"OrderId: {OrderId}, Status: {CurrentStatus}";
}
```

---

## Inner Exception의 중요성

```csharp
// 나쁜 예: 원인 정보 소실
try
{
    dbContext.SaveChanges();
}
catch (DbUpdateException ex)
{
    // inner exception 없이 새 예외 생성 → 원래 DB 오류 정보 소실
    throw new OrderProcessingException("저장 실패");
}

// 좋은 예: inner exception으로 원인 연결
try
{
    dbContext.SaveChanges();
}
catch (DbUpdateException ex)
{
    throw new OrderProcessingException(
        orderId: order.Id,
        status: order.Status,
        message: $"주문 {order.Id} 저장 실패: {ex.Message}",
        innerException: ex); // 원래 예외를 보존
}
// 로그에서: OrderProcessingException → caused by → DbUpdateException → caused by → SqlException
```

---

## 예외 계층 설계

```
ApplicationException (사용 자제, 과거 관행)

권장 계층:
Exception
├── OrderException                   (주문 도메인 기반 예외)
│   ├── OrderNotFoundException       (주문을 찾을 수 없음)
│   ├── OrderProcessingException     (처리 중 오류)
│   └── OrderValidationException     (유효성 검사 실패)
├── PaymentException                 (결제 도메인 기반 예외)
│   ├── InsufficientFundsException
│   └── PaymentGatewayException
└── InventoryException

// 호출자는 계층 구조로 선택적 처리 가능
catch (OrderNotFoundException)   { /* 특정 처리 */ }
catch (OrderException)           { /* 주문 관련 전체 */ }
catch (Exception)                { /* 모든 오류 */ }
```

---

## 결론

> 커스텀 예외는 기본 생성자 4종, 직렬화 생성자, `GetObjectData`, 도메인 특화 데이터 프로퍼티를 모두 갖춰 작성하라. inner exception을 항상 전달하여 예외 연쇄(exception chaining)를 보존하라. 계층 구조로 설계하면 호출자가 처리 범위를 세밀하게 조정할 수 있다.
