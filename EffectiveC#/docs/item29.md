# Item 29: 컬렉션 반환보다 이터레이터 메서드를 선호하라 (Prefer Iterator Methods to Returning Collections)

> **Chapter 4: Working with LINQ**

---

## 핵심 요약

시퀀스를 반환하는 메서드는 컬렉션(`List<T>`, 배열 등)으로 미리 실체화(materialize)해서 반환하는 대신, **이터레이터 메서드(`yield return`)**로 지연 생성하라. 호출자가 실제로 필요한 만큼만 연산하므로 메모리와 CPU를 절약할 수 있다.

---

## 컬렉션 반환의 문제

```csharp
// 나쁜 예: 전체를 미리 생성해서 반환
public static IEnumerable<int> GetEvenNumbers(int max)
{
    var result = new List<int>();
    for (int i = 0; i <= max; i++)
        if (i % 2 == 0)
            result.Add(i);
    return result; // max가 10억이면 전부 메모리에 올라감
}
```

---

## 이터레이터 메서드 (yield return)

```csharp
// 좋은 예: 요청할 때마다 하나씩 생성
public static IEnumerable<int> GetEvenNumbers(int max)
{
    for (int i = 0; i <= max; i++)
        if (i % 2 == 0)
            yield return i; // 호출자가 요청할 때만 실행
}

// 첫 5개만 필요하면 5번만 실행됨
var first5 = GetEvenNumbers(int.MaxValue).Take(5).ToList();
```

---

## yield return의 동작 원리

컴파일러는 `yield return`을 상태 머신(state machine)으로 변환한다.

```
GetEvenNumbers() 호출
    → IEnumerator<int> 객체 반환 (아직 아무것도 실행 안 됨)

MoveNext() 호출 (foreach 또는 LINQ가 내부적으로 호출)
    → 다음 yield return 지점까지 실행
    → 값 하나 반환 후 일시 정지 (상태 저장)

다음 MoveNext() 호출
    → 이전 중단 지점부터 재개
    → 다음 yield return 또는 메서드 끝까지 실행
```

---

## yield break: 조기 종료

```csharp
public static IEnumerable<string> ReadLines(string path)
{
    if (!File.Exists(path))
        yield break; // 빈 시퀀스 반환 (예외 아님)

    foreach (var line in File.ReadLines(path))
        yield return line;
}
```

---

## 무한 시퀀스 생성

이터레이터 메서드는 무한 시퀀스를 표현할 수 있다. 컬렉션으로는 불가능하다.

```csharp
// 무한 피보나치 수열
public static IEnumerable<long> Fibonacci()
{
    long a = 0, b = 1;
    while (true)
    {
        yield return a;
        (a, b) = (b, a + b);
    }
}

// 처음 10개만 사용
var first10 = Fibonacci().Take(10).ToList();
// 0, 1, 1, 2, 3, 5, 8, 13, 21, 34
```

---

## 주의: yield return 메서드의 인수 검증

이터레이터 메서드는 호출 즉시 실행되지 않으므로, 인수 검증이 지연된다.

```csharp
// 나쁜 예: 인수 검증이 실제 순회 시점까지 지연됨
public static IEnumerable<int> GetRange(int start, int count)
{
    if (count < 0) throw new ArgumentException(); // 순회 전까지 실행 안 됨!

    for (int i = start; i < start + count; i++)
        yield return i;
}

// 좋은 예: 검증 메서드와 이터레이터 분리
public static IEnumerable<int> GetRange(int start, int count)
{
    if (count < 0) throw new ArgumentException(nameof(count));
    return GetRangeIterator(start, count);
}

private static IEnumerable<int> GetRangeIterator(int start, int count)
{
    for (int i = start; i < start + count; i++)
        yield return i;
}
```

---

## 결론

| 상황 | 권장 |
|------|------|
| 시퀀스 전체가 항상 필요 | 컬렉션 반환 |
| 일부만 사용될 가능성이 있음 | `yield return` ✅ |
| 무한 시퀀스 | `yield return` 필수 |
| 대용량 데이터 스트리밍 | `yield return` ✅ |

> 시퀀스를 반환할 때는 기본적으로 이터레이터 메서드를 사용하라. 호출자가 원하는 만큼만 소비하고, 메모리를 최소로 사용하며, 무한 시퀀스도 표현할 수 있다.
