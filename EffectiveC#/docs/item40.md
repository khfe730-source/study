# Item 40: 즉시 실행과 지연 실행을 구분하라 (Distinguish Early from Deferred Execution)

> **Chapter 4: Working with LINQ**

---

## 핵심 요약

LINQ 연산자는 **즉시 실행(immediate execution)**과 **지연 실행(deferred execution)** 두 종류로 나뉜다. 어떤 연산자가 언제 실행되는지 정확히 알아야 성능 버그와 논리 오류를 피할 수 있다.

---

## 실행 시점 분류표

| 구분 | 즉시 실행 | 지연 실행 |
|------|-----------|-----------|
| 특징 | 호출 즉시 전체 순회 | 소비(foreach, ToList 등) 시 실행 |
| 반환 타입 | 스칼라, 컬렉션(`List<T>` 등) | `IEnumerable<T>`, `IQueryable<T>` |
| 예시 | `Count()`, `Sum()`, `ToList()`, `First()`, `Any()`, `Max()` | `Where()`, `Select()`, `OrderBy()`, `GroupBy()`, `Join()` |

---

## 즉시 실행 연산자

```csharp
var numbers = new[] { 1, 2, 3, 4, 5 };

// 아래 모두 호출 즉시 전체 순회
int count    = numbers.Count();          // 5
int sum      = numbers.Sum();            // 15
int max      = numbers.Max();            // 5
bool hasEven = numbers.Any(n => n % 2 == 0); // true
int first    = numbers.First(n => n > 3);    // 4

List<int>    list  = numbers.ToList();   // 복사본 생성
int[]        arr   = numbers.ToArray();  // 배열 복사
Dictionary<int,int> dict = numbers.ToDictionary(n => n, n => n * n);
```

---

## 지연 실행 연산자

```csharp
// 아래 모두 호출 시점에는 아무것도 실행되지 않음
var filtered = numbers.Where(n => n > 2);         // 실행 안 됨
var doubled  = numbers.Select(n => n * 2);         // 실행 안 됨
var sorted   = numbers.OrderBy(n => n);            // 실행 안 됨
var grouped  = numbers.GroupBy(n => n % 2 == 0);   // 실행 안 됨

// 이 시점에 실행
foreach (var n in filtered) { ... }
```

---

## 스트리밍 vs 비스트리밍 지연 실행

지연 실행 연산자 중에도 동작 방식에 차이가 있다.

```
스트리밍(streaming):    원소를 하나씩 요청할 때마다 처리
                       Where, Select, Take, Skip, SelectMany

비스트리밍(buffering):  첫 원소를 반환하기 전에 전체 입력을 처리해야 함
                       OrderBy, GroupBy, Reverse
```

```csharp
// Where는 스트리밍 — 소스에서 하나씩 가져와 즉시 필터링
var stream = Enumerable.Range(1, 1_000_000)
    .Where(n => n % 2 == 0)
    .Take(5);
// 처음 짝수 5개만 처리하고 중단

// OrderBy는 비스트리밍 — 첫 원소를 반환하려면 전체를 봐야 함
var buffered = Enumerable.Range(1, 1_000_000)
    .OrderBy(n => n)
    .Take(5);
// 백만 개를 전부 정렬한 후 앞 5개 반환
```

---

## 지연 실행이 만드는 미묘한 버그

```csharp
// 버그 1: 캡처된 변수 변경
var list = new List<int> { 1, 2, 3 };
var query = list.Where(n => n > 1); // 지연 — 아직 실행 안 됨
list.Clear();                        // 소스 변경!
var result = query.ToList();         // 빈 리스트 반환 (list가 비어 있으므로)

// 버그 2: 루프 변수 캡처 (C# 5 이전 문제, 현재는 수정됨)
var actions = new List<Action>();
foreach (var i in Enumerable.Range(0, 5))
    actions.Add(() => Console.WriteLine(i));
// C# 5부터: 각 람다가 독립된 i를 캡처 → 0,1,2,3,4 출력
// C# 5 이전: 모든 람다가 같은 i를 캡처 → 5,5,5,5,5 출력

// 버그 3: 예외의 늦은 발생
var query = data.Select(s => int.Parse(s)); // 정의 시점: 오류 없음
DoOtherWork();
var result = query.ToList(); // 이 시점에 FormatException 발생
// → DoOtherWork()가 범인처럼 보임
```

---

## 실행 시점 확인 방법

```csharp
// 디버깅용: 실행 시점을 로깅
var traced = numbers
    .Where(n => {
        Console.WriteLine($"Where 평가: {n}");
        return n > 2;
    })
    .Select(n => {
        Console.WriteLine($"Select 변환: {n}");
        return n * 10;
    });

Console.WriteLine("쿼리 정의 완료"); // 먼저 출력됨
var result = traced.ToList();        // 여기서 모든 로그 출력
```

---

## 결론

> `Where`, `Select` 같은 대부분의 LINQ 연산자는 지연 실행되고, `Count`, `ToList`, `First` 같은 집계·실체화 연산자는 즉시 실행된다. `OrderBy`처럼 비스트리밍인 지연 연산자는 내부적으로 전체 버퍼링이 필요하다. 이 차이를 모르면 소스 변경, 예외 발생 시점, 성능 모두 예측할 수 없게 된다.
