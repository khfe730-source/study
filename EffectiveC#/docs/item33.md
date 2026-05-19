# Item 33: 요청에 따라 시퀀스 항목을 생성하라 (Generate Sequence Items as Requested)

> **Chapter 4: Working with LINQ**

---

## 핵심 요약

데이터를 미리 전부 생성해 메모리에 올리지 말고, 소비자가 요청할 때 그 시점에 생성하라. `yield return`을 활용한 지연 생성(lazy generation)은 메모리 효율을 높이고 불필요한 연산을 제거한다.

---

## Enumerable.Range / Repeat / Empty 활용

.NET이 제공하는 시퀀스 생성 유틸리티를 먼저 활용한다.

```csharp
// 범위 시퀀스: 0~99
var range = Enumerable.Range(0, 100);

// 반복 시퀀스: "Hello" 5번
var repeated = Enumerable.Repeat("Hello", 5);

// 빈 시퀀스 (null 대신 사용)
var empty = Enumerable.Empty<int>();

// 실용 예시: 100개의 랜덤 숫자
var rng = new Random();
var randoms = Enumerable.Range(0, 100)
    .Select(_ => rng.Next(1, 101));
```

---

## 커스텀 시퀀스 생성기

```csharp
// 날짜 범위 생성기
public static IEnumerable<DateTime> DateRange(DateTime start, DateTime end)
{
    for (var date = start; date <= end; date = date.AddDays(1))
        yield return date;
}

// 소수 생성기 (무한)
public static IEnumerable<int> Primes()
{
    var primes = new List<int>();
    for (int candidate = 2; ; candidate++)
    {
        bool isPrime = primes.All(p => candidate % p != 0);
        if (isPrime)
        {
            primes.Add(candidate);
            yield return candidate;
        }
    }
}

// 사용: 처음 10개 소수만 필요
var first10Primes = Primes().Take(10).ToList();
// 2, 3, 5, 7, 11, 13, 17, 19, 23, 29
```

---

## 재귀적 시퀀스 생성

```csharp
// 트리 구조 평탄화 (깊이 우선)
public static IEnumerable<T> Flatten<T>(this TreeNode<T> node)
{
    yield return node.Value;
    foreach (var child in node.Children)
        foreach (var item in child.Flatten())
            yield return item;
}

// 파일 시스템 재귀 탐색
public static IEnumerable<string> GetAllFiles(string directory)
{
    foreach (var file in Directory.GetFiles(directory))
        yield return file;

    foreach (var subDir in Directory.GetDirectories(directory))
        foreach (var file in GetAllFiles(subDir))
            yield return file;
}
```

---

## 캐싱이 필요한 경우

지연 생성은 반복 순회 시 매번 재실행된다. 결과를 재사용해야 한다면 한 번만 실체화한다.

```csharp
// 나쁜 예: 두 번 순회하면 두 번 생성
var expensive = GenerateExpensiveSequence();
int count = expensive.Count(); // 전체 실행
var first = expensive.First(); // 또 전체 실행

// 좋은 예: 한 번 실체화 후 재사용
var materialized = GenerateExpensiveSequence().ToList();
int count = materialized.Count;
var first = materialized[0];
```

---

## 결론

> 시퀀스 데이터는 미리 전부 생성하지 말고, 소비자가 요청할 때 `yield return`으로 하나씩 생성하라. 무한 시퀀스, 대용량 데이터, 비용이 큰 연산에서 특히 효과적이다. 단, 결과를 여러 번 순회해야 한다면 `ToList()`로 한 번만 실체화하라.
