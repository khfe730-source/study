# Item 39: 함수와 액션에서 예외를 던지지 마라 (Avoid Throwing Exceptions in Functions and Actions)

> **Chapter 4: Working with LINQ**

---

## 핵심 요약

LINQ 파이프라인에 전달되는 람다(predicate, selector, action)에서 예외를 던지면 예외의 발생 위치가 불명확해지고, 지연 평가 특성 때문에 디버깅이 어려워진다. 예외 대신 **안전한 변환 패턴**이나 **`Try-` 패턴**을 사용하라.

---

## 문제: 파이프라인 중간의 예외

```csharp
// 예외가 어느 원소에서 발생했는지 스택 트레이스만으로 파악하기 어렵다
var results = rawStrings
    .Select(s => int.Parse(s))   // 파싱 실패 시 FormatException
    .Where(n => n > 0)
    .ToList();

// 지연 평가로 인해 예외 발생 지점이 ToList()로 표시됨
// 어떤 문자열이 문제였는지 알기 어려움
```

---

## 해결 1: Try 패턴으로 안전한 변환

```csharp
// int.TryParse를 활용한 안전한 처리
var results = rawStrings
    .Select(s => int.TryParse(s, out int n) ? (int?)n : null)
    .Where(n => n.HasValue)
    .Select(n => n!.Value)
    .ToList();

// 또는 커스텀 TryParse 래퍼
static int? TryParseInt(string s) =>
    int.TryParse(s, out int n) ? n : null;

var results = rawStrings
    .Select(TryParseInt)
    .Where(n => n.HasValue)
    .Select(n => n!.Value)
    .ToList();
```

---

## 해결 2: 결과 래퍼 타입 사용

```csharp
// 성공/실패를 값으로 표현
public readonly struct ParseResult<T>
{
    public bool IsSuccess { get; }
    public T Value { get; }
    public string Error { get; }

    private ParseResult(T value) { IsSuccess = true; Value = value; Error = null; }
    private ParseResult(string error) { IsSuccess = false; Error = error; }

    public static ParseResult<T> Success(T value) => new(value);
    public static ParseResult<T> Failure(string error) => new(error);
}

// 파이프라인에서 사용
var results = rawData
    .Select(s => {
        if (int.TryParse(s, out int n))
            return ParseResult<int>.Success(n);
        return ParseResult<int>.Failure($"파싱 실패: '{s}'");
    })
    .ToList();

var successes = results.Where(r => r.IsSuccess).Select(r => r.Value);
var failures  = results.Where(r => !r.IsSuccess).Select(r => r.Error);
```

---

## 해결 3: 필터로 사전 제거

```csharp
// 예외를 던지기 전에 유효하지 않은 원소를 제거
var results = rawStrings
    .Where(s => int.TryParse(s, out _)) // 파싱 불가능한 문자열 제거
    .Select(s => int.Parse(s))          // 이제 안전하게 파싱
    .ToList();
```

---

## Action에서의 예외

```csharp
// 나쁜 예: ForEach 액션에서 예외 발생 시 나머지 원소 처리 중단
orders.ForEach(o => ProcessOrder(o)); // ProcessOrder가 던지면 이후 주문 처리 안 됨

// 좋은 예: 개별 오류를 수집하고 계속 진행
var errors = new List<(Order order, Exception ex)>();

foreach (var order in orders)
{
    try { ProcessOrder(order); }
    catch (Exception ex) { errors.Add((order, ex)); }
}

// 또는 LINQ로 안전한 처리
var processed = orders
    .Select(o => {
        try { return (Success: true, Result: ProcessOrder(o), Error: (Exception)null); }
        catch (Exception ex) { return (Success: false, Result: default, Error: ex); }
    })
    .ToList();
```

---

## 예외가 허용되는 경우

```csharp
// 프로그래밍 오류 (버그) — 예외가 맞음
var query = source.Select(x => {
    if (x == null) throw new ArgumentNullException(nameof(x)); // null은 버그
    return Transform(x);
});

// 복구 불가능한 상황 — 예외가 맞음
// 단순 데이터 변환 오류 — Try 패턴을 선호
```

---

## 결론

> LINQ 파이프라인의 람다에서 예외를 던지면 지연 평가 특성으로 인해 실제 오류 원인을 추적하기 어렵다. 데이터 변환이나 파싱처럼 실패 가능한 작업은 `Try-` 패턴이나 결과 래퍼 타입으로 실패를 값으로 표현하라. 예외는 복구 불가능한 상황이나 명백한 프로그래밍 오류에만 사용하라.
