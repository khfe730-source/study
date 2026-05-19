# Item 48: using과 try/finally로 리소스를 반드시 해제하라 (Ensure Resources Are Cleaned Up by Using using and try/finally)

> **Chapter 5: Exception Practices**

---

## 핵심 요약

예외가 발생해도 리소스 해제가 보장되어야 한다. `using` 문(또는 C# 8의 `using` 선언)이 가장 간결한 해법이며, 복잡한 다단계 해제 시나리오에서는 `try/finally`를 직접 사용하라.

---

## using 문의 컴파일러 변환

```csharp
// 개발자가 작성
using (var file = File.OpenRead(path))
{
    Process(file);
}

// 컴파일러 변환 결과 (개념적)
var file = File.OpenRead(path);
try
{
    Process(file);
}
finally
{
    file?.Dispose(); // 예외 여부와 무관하게 항상 실행
}
```

---

## C# 8 using 선언 (중첩 감소)

```csharp
// C# 8 이전: 다중 리소스 시 중첩 깊어짐
using (var conn = new SqlConnection(connStr))
using (var cmd = conn.CreateCommand())
using (var reader = cmd.ExecuteReader())
{
    while (reader.Read()) { ... }
}

// C# 8+: using 선언 — 스코프 종료 시 역순으로 Dispose
using var conn = new SqlConnection(connStr);
using var cmd = conn.CreateCommand();
using var reader = cmd.ExecuteReader();
while (reader.Read()) { ... }
// 스코프 끝에서 reader → cmd → conn 순서로 Dispose
```

---

## 복잡한 시나리오: try/finally 직접 사용

```csharp
// 초기화 도중 실패 시 부분 초기화된 리소스 해제
public static (Stream input, Stream output) OpenStreams(string inPath, string outPath)
{
    Stream input = null;
    Stream output = null;
    try
    {
        input = File.OpenRead(inPath);
        output = File.OpenWrite(outPath); // 여기서 실패하면 input을 해제해야 함
        return (input, output);
    }
    catch
    {
        input?.Dispose();
        output?.Dispose();
        throw; // 원래 예외 재전파
    }
}
```

---

## IAsyncDisposable (C# 8)

비동기 리소스 해제가 필요한 경우:

```csharp
// await using — 비동기 Dispose 지원
await using var conn = new AsyncDbConnection(connStr);
await using var reader = await conn.ExecuteReaderAsync(query);

while (await reader.ReadAsync())
{
    ProcessRow(reader);
}
// 스코프 종료 시 DisposeAsync() 호출
```

---

## 안티패턴: finally에서 예외 던지기

```csharp
// 극히 위험한 패턴
try
{
    DoWork();
}
finally
{
    CleanUp(); // 여기서 예외 발생 시 원래 예외가 완전히 소실됨!
}

// 안전한 패턴: finally에서 예외를 삼킨다
finally
{
    try { CleanUp(); }
    catch (Exception ex) { Logger.LogError(ex, "정리 중 오류 (무시)"); }
}
```

---

## Dispose 패턴과의 관계

```csharp
// IDisposable을 올바르게 구현한 클래스는 using으로 안전하게 사용 가능
public sealed class ManagedResource : IDisposable
{
    private readonly Stream _stream;
    private bool _disposed;

    public ManagedResource(string path)
        => _stream = File.OpenRead(path);

    public void Process() { /* ... */ }

    public void Dispose()
    {
        if (_disposed) return;
        _stream.Dispose();
        _disposed = true;
    }
}

// 사용측: 예외 여부와 무관하게 Dispose 보장
using var resource = new ManagedResource(path);
resource.Process();
```

---

## 결론

> 비관리 리소스나 `IDisposable` 구현체는 반드시 `using` 또는 `try/finally`로 감싸라. C# 8의 `using` 선언으로 중첩을 제거하고, 비동기 리소스는 `await using`을 사용하라. `finally` 블록에서는 예외를 외부로 전파하지 말고 로깅만 하라 — `finally`에서 예외가 나오면 원래 예외 정보가 소실된다.
