# Item 50: 약한 참조로 대용량 캐시를 구현하라 (Use Weak References for Large Object Caches)

> **Chapter 5: Exception Practices**

---

## 핵심 요약

캐시가 보유한 강한 참조(strong reference)는 객체의 GC를 영구히 막는다. **약한 참조(WeakReference<T>)**를 사용하면 메모리가 부족할 때 GC가 캐시된 객체를 수거할 수 있어, 메모리 압력 없이 캐시 이점을 누릴 수 있다.

---

## 강한 참조 캐시의 문제

```csharp
// 나쁜 예: 강한 참조 캐시 — 사용하지 않아도 메모리에서 제거 불가
private readonly Dictionary<int, LargeReport> _cache = new();

public LargeReport GetReport(int id)
{
    if (_cache.TryGetValue(id, out var cached))
        return cached;

    var report = GenerateReport(id); // 수십 MB
    _cache[id] = report; // 강한 참조 — GC 수거 불가
    return report;
}
// _cache가 살아 있는 한 모든 LargeReport가 힙에 존재 → OutOfMemoryException 가능
```

---

## WeakReference<T>로 캐시 구현

```csharp
private readonly Dictionary<int, WeakReference<LargeReport>> _cache = new();

public LargeReport GetReport(int id)
{
    // 캐시에 약한 참조가 있고, GC가 아직 수거하지 않은 경우
    if (_cache.TryGetValue(id, out var weakRef)
        && weakRef.TryGetTarget(out var cached))
    {
        return cached; // 캐시 히트
    }

    // 캐시 미스 또는 GC가 수거한 경우 — 재생성
    var report = GenerateReport(id);
    _cache[id] = new WeakReference<LargeReport>(report); // 약한 참조로 저장
    return report;
}
// GC는 메모리가 부족하면 report를 수거 가능 — 다음 요청 시 재생성됨
```

---

## WeakReference vs WeakReference<T>

```csharp
// WeakReference (비제네릭, .NET 1.x 스타일)
var weak = new WeakReference(largeObject);
if (weak.IsAlive)
{
    var obj = (LargeReport)weak.Target; // null 체크 필요, 박싱 가능
}

// WeakReference<T> (제네릭, 권장)
var weak = new WeakReference<LargeReport>(largeObject);
if (weak.TryGetTarget(out var obj))
{
    // obj는 이미 null이 아닌 것이 보장된 LargeReport
    // TryGetTarget은 원자적이므로 IsAlive 이후 null 경쟁 조건 없음
}
```

---

## 캐시 항목 제거 (ConditionalWeakTable)

약한 참조 캐시에서 키는 여전히 강한 참조다. `ConditionalWeakTable<TKey, TValue>`를 사용하면 키와 값 모두 GC에 맡길 수 있다.

```csharp
// 키 객체가 GC되면 항목이 자동으로 제거됨
private readonly ConditionalWeakTable<Customer, LargeReport> _reportCache = new();

public LargeReport GetReportFor(Customer customer)
{
    return _reportCache.GetValue(customer, c => GenerateReport(c));
    // customer 객체가 GC되면 연결된 LargeReport도 자동으로 테이블에서 제거
}
```

---

## 약한 참조 캐시의 특성 및 트레이드오프

```
장점:
  - 메모리 압력 시 GC가 자동으로 캐시 항목 수거
  - OutOfMemoryException 방지
  - 짧은 시간 내 재사용 시 재생성 비용 절약

단점:
  - GC 수거 시점은 예측 불가 — 히트율이 보장되지 않음
  - 재생성 비용이 캐시 이점보다 클 경우 효과 없음
  - 딕셔너리의 키(int)는 여전히 강한 참조 → 주기적 정리 필요

적합한 사용 사례:
  - 재생성 가능하지만 비용이 큰 객체 (리포트, 이미지, 파싱 결과)
  - 크기를 제한하기 어려운 캐시
  - 메모리 압력에 민감한 환경 (모바일, 서버 고부하)

부적합한 사용 사례:
  - 재생성 불가능한 데이터 (캐시가 유일한 복사본)
  - 고성능이 필요한 빈번한 조회 (TryGetTarget 오버헤드)
```

---

## 약한 참조 캐시의 키 정리

```csharp
// 오래된 약한 참조(GC가 수거한 항목)를 주기적으로 딕셔너리에서 제거
public void Cleanup()
{
    var deadKeys = _cache
        .Where(kvp => !kvp.Value.TryGetTarget(out _))
        .Select(kvp => kvp.Key)
        .ToList();

    foreach (var key in deadKeys)
        _cache.Remove(key);
}

// 또는 ConcurrentDictionary + 주기적 타이머로 자동화
```

---

## 결론

> 대용량 객체를 캐시할 때 강한 참조는 메모리를 영구히 점유하여 `OutOfMemoryException`으로 이어질 수 있다. `WeakReference<T>`를 사용하면 GC가 메모리 압력에 따라 캐시 항목을 수거할 수 있고, 키와 값 모두 GC에 맡기려면 `ConditionalWeakTable<TKey, TValue>`를 활용하라. 재생성 가능한 크고 비용이 큰 객체에 적합한 패턴이다.
