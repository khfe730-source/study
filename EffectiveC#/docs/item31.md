# Item 31: 시퀀스에 조합 가능한 API를 만들어라 (Create Composable APIs for Sequences)

> **Chapter 4: Working with LINQ**

---

## 핵심 요약

`IEnumerable<T>`를 입력받고 반환하는 메서드는 LINQ처럼 **체이닝(chaining)**이 가능한 조합형 API를 만든다. 단일 메서드에 여러 로직을 몰아넣는 대신, 작은 변환 단계로 분리해 재사용성과 표현력을 높여라.

---

## 단일 책임 메서드로 분리

```csharp
// 나쁜 예: 하나의 메서드가 너무 많은 일을 함
public static List<string> GetSortedAdultNames(IEnumerable<Person> people)
{
    var result = new List<string>();
    foreach (var p in people)
        if (p.Age >= 18)
            result.Add(p.Name.ToUpper());
    result.Sort();
    return result;
}

// 좋은 예: 작은 변환 단계로 분리 → 조합 가능
public static IEnumerable<Person> Adults(this IEnumerable<Person> people)
    => people.Where(p => p.Age >= 18);

public static IEnumerable<string> Names(this IEnumerable<Person> people)
    => people.Select(p => p.Name);

public static IEnumerable<string> Uppercase(this IEnumerable<string> names)
    => names.Select(n => n.ToUpper());

// 조합해서 사용
var result = people
    .Adults()
    .Names()
    .Uppercase()
    .OrderBy(n => n)
    .ToList();
```

---

## IEnumerable<T> → IEnumerable<T> 파이프라인

```csharp
// 각 단계가 독립적이고 조합 가능
public static IEnumerable<LogEntry> FilterByLevel(
    this IEnumerable<LogEntry> logs, LogLevel level)
    => logs.Where(l => l.Level == level);

public static IEnumerable<LogEntry> FilterByDateRange(
    this IEnumerable<LogEntry> logs, DateTime from, DateTime to)
    => logs.Where(l => l.Timestamp >= from && l.Timestamp <= to);

public static IEnumerable<string> FormatMessages(
    this IEnumerable<LogEntry> logs)
    => logs.Select(l => $"[{l.Level}] {l.Timestamp:HH:mm:ss} {l.Message}");

// 다양하게 조합
var errorMessages = logs
    .FilterByLevel(LogLevel.Error)
    .FilterByDateRange(DateTime.Today, DateTime.Now)
    .FormatMessages();

var recentWarnings = logs
    .FilterByLevel(LogLevel.Warning)
    .FilterByDateRange(DateTime.Now.AddHours(-1), DateTime.Now)
    .FormatMessages();
```

---

## 지연 실행으로 조합의 효율성 확보

조합형 API의 각 단계가 `IEnumerable<T>`를 반환하면 지연 실행이 유지된다. 최종 소비(`ToList()`, `foreach` 등)가 일어날 때 전체 파이프라인이 한 번에 실행된다.

```
logs → FilterByLevel → FilterByDateRange → FormatMessages → ToList()
                                                              ↑
                                              이 시점에 파이프라인 전체 실행
각 항목이 파이프라인을 통과하며 처리됨 (중간 컬렉션 생성 없음)
```

---

## 조합 가능한 API 설계 원칙

```csharp
// 1. IEnumerable<T>를 매개변수로, IEnumerable<T>를 반환
public static IEnumerable<T> Step<T>(this IEnumerable<T> source, ...) { }

// 2. 상태를 가지지 않음 (순수 함수에 가깝게)
// 3. 예외 대신 빈 시퀀스 반환 (null 반환 금지)
public static IEnumerable<T> SafeFilter<T>(
    this IEnumerable<T> source, Func<T, bool> predicate)
    => source?.Where(predicate) ?? Enumerable.Empty<T>();
```

---

## 결론

> `IEnumerable<T>`를 입력받고 반환하는 작은 변환 메서드들로 API를 구성하라. 각 단계가 독립적이고 지연 실행을 유지하므로, 조합했을 때 효율적이고 표현력 있는 데이터 처리 파이프라인을 만들 수 있다.
