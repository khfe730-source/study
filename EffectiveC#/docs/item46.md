# Item 46: try 블록 내 코드를 최소화하라 (Minimize the Code in try Blocks)

> **Chapter 5: Exception Practices**

---

## 핵심 요약

`try` 블록 안에 코드를 많이 넣을수록 어떤 줄에서 예외가 발생했는지 추적하기 어렵고, 예외 발생 경로가 복잡해진다. `try` 블록은 **예외가 발생할 수 있는 최소한의 코드**만 포함하도록 좁혀야 한다.

---

## 거대한 try 블록의 문제

```csharp
// 나쁜 예: try 블록이 지나치게 넓음
public ProcessResult ProcessOrder(Order order)
{
    try
    {
        ValidateOrder(order);           // ArgumentException 가능
        var customer = LoadCustomer(order.CustomerId); // DbException 가능
        var inventory = CheckInventory(order);          // InventoryException 가능
        ApplyDiscount(order, customer); // 로직 오류 가능
        var payment = ChargePayment(order.Total);       // PaymentException 가능
        UpdateInventory(inventory);     // DbException 가능
        SendConfirmationEmail(customer); // SmtpException 가능
        return new ProcessResult(payment.TransactionId);
    }
    catch (Exception ex)
    {
        Logger.Log(ex);
        return ProcessResult.Failed;
        // 어느 단계에서 실패했는지, 어디까지 완료됐는지 알 수 없음
    }
}
```

---

## 단계별로 try 블록 분리

```csharp
public ProcessResult ProcessOrder(Order order)
{
    // 각 단계를 분리하여 실패 지점을 명확히
    ValidateOrder(order); // 여기서 예외 → 바로 전파 (호출자의 잘못)

    Customer customer;
    try { customer = LoadCustomer(order.CustomerId); }
    catch (DbException ex)
    {
        throw new OrderProcessingException("고객 정보 로드 실패", ex); // 감싸서 전파
    }

    Inventory inventory;
    try { inventory = CheckInventory(order); }
    catch (InventoryException ex)
    {
        throw new OrderProcessingException("재고 확인 실패", ex);
    }

    ApplyDiscount(order, customer); // 예외 없다고 확신하면 try 불필요

    PaymentResult payment;
    try { payment = ChargePayment(order.Total); }
    catch (PaymentException ex)
    {
        throw new OrderProcessingException("결제 실패", ex);
    }

    // 이 이후 실패는 보상 트랜잭션이 필요한 심각한 오류
    UpdateInventory(inventory);
    SendConfirmationEmail(customer);

    return new ProcessResult(payment.TransactionId);
}
```

---

## 예외 필터로 예외 유형별 처리

```csharp
// try 블록을 좁히는 대신 when으로 예외를 선별
try
{
    result = LoadData(id);
}
catch (SqlException ex) when (ex.Number == 1205) // 교착 상태(deadlock)
{
    return RetryWithBackoff(() => LoadData(id));
}
catch (SqlException ex) when (ex.Number == -2)   // 타임아웃
{
    throw new ServiceUnavailableException("DB 타임아웃", ex);
}
// 다른 SqlException은 전파
```

---

## finally 블록의 올바른 위치

```csharp
// try 범위가 좁을수록 finally의 의미가 명확해짐
FileStream file = null;
try
{
    file = File.OpenRead(path);   // 이 한 줄만 try
}
catch (FileNotFoundException)
{
    return null;
}

try
{
    return ParseContent(file);    // 파싱은 별도 try
}
finally
{
    file?.Dispose();              // 파일은 항상 닫음
}
```

---

## 제어 흐름에 예외를 사용하지 마라

```csharp
// 나쁜 예: 예외를 정상적인 흐름 제어에 사용
public bool TryGetUser(int id, out User user)
{
    try
    {
        user = _repo.GetUser(id);
        return true;
    }
    catch (UserNotFoundException)
    {
        user = null;
        return false; // 예외가 정상 경로 — 성능 저하, 의미 왜곡
    }
}

// 좋은 예: 예외 없이 처리
public bool TryGetUser(int id, out User user)
{
    user = _repo.FindUser(id); // null 반환하는 버전
    return user != null;
}
```

---

## 결론

> `try` 블록에는 예외가 실제로 발생할 수 있는 최소한의 코드만 넣어라. 블록이 작을수록 예외 발생 지점이 명확하고 진단이 쉬우며, 예외 처리 의도도 분명해진다. 예외를 제어 흐름에 사용하는 것은 성능과 가독성 모두 해치므로 피하라.
