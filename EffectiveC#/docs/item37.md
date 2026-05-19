# Item 37: 쿼리에서 즉시 평가보다 지연 평가를 선호하라 (Prefer Lazy Evaluation to Eager Evaluation in Queries)

> **Chapter 4: Working with LINQ**

---

## 핵심 요약

LINQ 쿼리는 기본적으로 **지연 평가(lazy evaluation)**된다. 쿼리를 정의하는 시점이 아닌, 결과를 실제로 소비하는 시점에 실행된다. 이 특성을 적극 활용하면 불필요한 연산과 메모리 사용을 줄일 수 있다.

---

## 즉시 평가 vs 지연 평가

```csharp
// 즉시 평가(eager): 모든 결과를 지금 당장 메모리에 올린다
List<int> eager = numbers.Where(n => n > 0).ToList(); // 전체 순회 + 메모리 할당

// 지연 평가(lazy): 쿼리 정의만 함, 아직 실행 안 함
IEnumerable<int> lazy = numbers.Where(n => n > 0);   // 아무것도 실행 안 됨

// 소비 시점에 실행
foreach (var n in lazy) { ... } // 이 시점에 필터링 수행
```

---

## 지연 평가의 실질적 이점

### 1. 불필요한 원소 처리 방지

```csharp
// 1,000,000개 중 처음 5개만 필요
var first5 = Enumerable.Range(1, 1_000_000)
    .Where(n => IsPrime(n))   // IsPrime은 비용이 큰 연산
    .Take(5)                  // 5개를 찾는 순간 중단
    .ToList();

// ToList() 없이 쿼리만 정의: IsPrime은 단 한 번도 호출 안 됨
var query = Enumerable.Range(1, 1_000_000).Where(n => IsPrime(n));
```

### 2. 쿼리 조합 후 한 번만 순회

```csharp
// 여러 변환을 파이프라인으로 조합 — 순회는 최종 한 번뿐
var pipeline = source
    .Where(x => x.IsActive)
    .Select(x => Transform(x))
    .OrderBy(x => x.Score)
    .Take(10);

// 위 코드 어디에서도 실행되지 않음
// 아래 한 줄에서 전체 파이프라인이 한 번에 실행
foreach (var item in pipeline) { ... }
```

---

## 지연 평가가 초래하는 함정

### 쿼리 반복 실행

```csharp
var query = numbers.Where(n => {
    Console.WriteLine($"필터링: {n}");
    return n > 0;
});

int count = query.Count();  // 전체 순회 (필터 실행)
int sum   = query.Sum();    // 또 전체 순회 (필터 재실행!)

// 해결: 한 번만 실체화
var materialized = query.ToList();
int count = materialized.Count;
int sum   = materialized.Sum();
```

### 소스 변경의 영향

```csharp
var list = new List<int> { 1, 2, 3 };
var query = list.Where(n => n > 1);

list.Add(4); // 쿼리 정의 후에 추가

foreach (var n in query)
    Console.WriteLine(n); // 2, 3, 4 — 4도 포함됨!
// 지연 평가이므로 순회 시점의 list 상태를 반영
```

---

## 즉시 평가가 필요한 경우

```csharp
// 1. 결과를 여러 번 사용할 때
var results = expensiveQuery.ToList();

// 2. 소스 데이터가 변경될 수 있을 때
var snapshot = dbContext.Orders
    .Where(o => o.Status == "Pending")
    .ToList(); // DB 연결 닫기 전에 실체화

// 3. 예외를 즉시 감지해야 할 때
var validated = ParseData(rawLines).ToList(); // 파싱 오류를 지금 잡아야 함

// 4. 정렬(OrderBy)처럼 전체 데이터가 필요한 연산은 어차피 즉시 평가
var sorted = data.OrderBy(x => x.Key); // 첫 원소를 반환하려면 전체를 봐야 함
```

---

## 지연/즉시 평가 선택 기준

```
┌─────────────────────────────────────┬──────────────────┐
│ 상황                                │ 권장              │
├─────────────────────────────────────┼──────────────────┤
│ 결과의 일부만 사용할 가능성          │ 지연 (lazy)      │
│ 비용이 큰 연산을 조건부로 수행       │ 지연 (lazy)      │
│ 무한 시퀀스                         │ 지연 (lazy)      │
│ 결과를 여러 번 순회                  │ 즉시 (ToList)    │
│ 소스가 변경될 수 있음               │ 즉시 (ToList)    │
│ DB 컨텍스트가 닫힐 수 있음          │ 즉시 (ToList)    │
│ 예외를 즉시 감지해야 함             │ 즉시 (ToList)    │
└─────────────────────────────────────┴──────────────────┘
```

---

## 결론

> LINQ의 지연 평가는 성능 최적화의 강력한 도구다. 결과의 일부만 필요하거나 비용이 큰 연산을 조건부로 수행할 때 지연 평가를 유지하라. 단, 결과를 여러 번 사용하거나 소스가 변경될 수 있는 상황에서는 `ToList()`로 즉시 실체화하라.
