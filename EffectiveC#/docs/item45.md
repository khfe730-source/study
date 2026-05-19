# Item 45: catch 후 재throw보다 예외 필터를 선호하라 (Prefer Exception Filters to catch and re-throw)

> **Chapter 5: Exception Practices**

---

## 핵심 요약

C# 6에서 도입된 **예외 필터(exception filter)**(`when` 절)는 catch 블록을 진입하기 전에 조건을 평가한다. 조건이 거짓이면 스택이 풀리지 않아 원래 스택 추적(stack trace)이 보존된다. `catch` 후 `throw;`로 재throw하는 방식보다 성능·진단 정보 모두 우수하다.

---

## 스택 추적 보존 문제

```csharp
// 나쁜 예: catch 후 재throw — 스택 추적이 catch 위치로 리셋됨
try
{
    DoWork(); // 원래 예외 발생 위치
}
catch (Exception ex) when (false) { } // 예시용
catch (Exception ex)
{
    if (ShouldHandle(ex))
        Handle(ex);
    else
        throw; // throw; 는 스택 유지, 하지만 catch 블록에 진입한 것 자체가 비용
}

// 더 나쁜 예: throw ex; — 스택 추적을 catch 위치로 완전히 교체
catch (Exception ex)
{
    throw ex; // 원래 예외 발생 위치 정보 소실!
}
```

---

## 예외 필터의 올바른 사용

```csharp
// 좋은 예: when 절 — 조건이 false면 catch 블록 자체에 진입하지 않음
try
{
    DoWork();
}
catch (HttpRequestException ex) when (ex.StatusCode == HttpStatusCode.NotFound)
{
    // 404만 여기서 처리
    HandleNotFound(ex);
}
catch (HttpRequestException ex) when (ex.StatusCode == HttpStatusCode.Unauthorized)
{
    // 401만 여기서 처리
    RefreshTokenAndRetry();
}
// 다른 HttpRequestException은 위로 전파 — 스택 추적 완전 보존
```

---

## 예외 필터로 로깅하기

예외 필터의 흥미로운 활용: `when` 조건에서 부작용(side effect)을 수행하면서 항상 `false`를 반환해 예외가 상위로 전파되도록 만들 수 있다.

```csharp
// 예외를 처리하지 않고 로깅만 하는 필터
static bool LogException(Exception ex, string context)
{
    Logger.LogError(ex, $"[{context}] 예외 발생: {ex.Message}");
    return false; // 항상 false → catch 블록 진입 안 함 → 예외 전파
}

try
{
    RiskyOperation();
}
catch (Exception ex) when (LogException(ex, nameof(RiskyOperation)))
{
    // 이 블록에는 절대 진입하지 않음
    // 예외는 원래 스택 추적을 유지한 채 상위로 전파
}
```

---

## 재시도 패턴

```csharp
public async Task<T> RetryAsync<T>(Func<Task<T>> operation, int maxRetries = 3)
{
    int attempt = 0;
    while (true)
    {
        try
        {
            return await operation();
        }
        catch (TransientException ex) when (++attempt < maxRetries)
        {
            // 일시적 오류이고 재시도 횟수가 남아 있으면 재시도
            await Task.Delay(TimeSpan.FromSeconds(Math.Pow(2, attempt)));
            // catch 블록 진입 → 예외 소비 → 루프 계속
        }
        // maxRetries 도달 시 when 조건 false → 예외 전파
    }
}
```

---

## 예외 필터 vs catch-rethrow 성능 비교

```
catch-rethrow 방식:
  1. 스택 풀기(stack unwinding) 시작
  2. catch 블록 진입
  3. 조건 평가
  4. throw; 로 재throw
  → 스택 풀기가 두 번 발생, 스택 추적 변경 가능성

예외 필터(when) 방식:
  1. when 조건 평가 (스택 풀기 없음)
  2. false → 다음 catch 탐색 또는 상위 전파
  → 조건이 false이면 스택 풀기 자체가 발생하지 않음
  → 원래 스택 추적 100% 보존
  → 성능 우수
```

---

## 안티패턴: throw ex

```csharp
// 절대 하지 말 것
catch (Exception ex)
{
    Logger.Log(ex);
    throw ex; // 스택 추적을 catch 위치로 교체 → 디버깅 시 원인 찾기 어려움
}

// 재throw가 필요하면 반드시
catch (Exception ex)
{
    Logger.Log(ex);
    throw; // 원래 예외 객체와 스택 추적 유지
}

// 또는 예외 필터로
catch (Exception ex) when (LogAndRethrow(ex)) { }
```

---

## 결론

> C# 6의 예외 필터(`when`)는 catch 블록 진입 전에 조건을 평가하여 스택 풀기를 방지한다. 조건부 처리·로깅·재시도 패턴에서 catch-rethrow 대신 `when`을 사용하면 원래 스택 추적이 완전히 보존되고 성능도 우수하다. `throw ex;`는 절대 사용하지 말고 재throw가 필요하면 `throw;`를 사용하라.
