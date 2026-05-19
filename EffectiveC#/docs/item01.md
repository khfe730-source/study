# Item 1: 암묵적으로 타입된 지역 변수를 선호하라 (Prefer Implicitly Typed Local Variables)

> **Chapter 1: C# Language Idioms**

---

## 핵심 요약

`var`를 사용한 암묵적 타입 선언은 단순한 편의 기능이 아니다. 올바르게 사용하면 코드의 의도를 더 명확히 드러내고, 잘못된 타입 변환으로 인한 미묘한 버그를 방지한다. 단, 타입이 코드 가독성에 중요한 경우에는 명시적 타입을 사용한다.

---

## var는 동적 타입이 아니다

`var`는 컴파일 타임에 타입이 결정되는 **정적 타입(statically typed)** 이다. `dynamic`과 혼동하지 않아야 한다.

```csharp
var i = 5;            // int로 컴파일 타임 결정
var s = "hello";      // string으로 결정
var list = new List<int>(); // List<int>로 결정

// dynamic과의 차이
dynamic d = 5;
d = "now a string";   // 런타임에 타입 변경 가능 → var는 불가능
```

---

## var가 버그를 방지하는 사례

### IEnumerable vs IQueryable 문제

```csharp
// 명시적 타입: 잘못된 타입으로 받아 의도치 않게 다운캐스팅
IEnumerable<string> q =
    from c in db.Customers
    select c.Name; // 실제로는 IQueryable<string>

var q2 =
    from c in db.Customers
    select c.Name; // 올바르게 IQueryable<string>으로 유지
```

`IEnumerable<string>`으로 받으면 LINQ to SQL의 지연 실행 최적화가 깨진다. `var`를 쓰면 실제 반환 타입인 `IQueryable<string>`이 그대로 유지되어 데이터베이스 측 필터링이 적용된다.

### 숫자 타입 정밀도 손실

```csharp
var f = GetMagicNumber(); // 반환 타입이 float라면 float으로 유지

// 명시적 타입 선언 시 의도치 않은 변환 발생 가능
double d = GetMagicNumber(); // float → double 묵시적 변환, 정밀도 손실 없지만 의미가 달라짐
```

---

## var를 사용해야 할 때

### 1. 타입이 우변에서 명확히 보일 때

```csharp
var customers = new List<Customer>(); // List<Customer>임이 명확
var conn = new SqlConnection(connStr); // SqlConnection임이 명확
```

### 2. 익명 타입(Anonymous Type)

`var` 없이는 익명 타입을 지역 변수로 받을 방법이 없다.

```csharp
var result = new { Name = "Alice", Age = 30 }; // 익명 타입은 var 필수
```

### 3. LINQ 쿼리 결과

```csharp
var query = from c in customers
            where c.Age > 18
            select c; // IEnumerable<Customer> 또는 IQueryable<Customer>
```

---

## var를 피해야 할 때

타입이 코드 이해에 중요한 경우에는 명시적으로 선언한다.

```csharp
// 나쁜 예: 반환 타입이 무엇인지 알 수 없음
var result = ProcessData(input);

// 좋은 예: 의도가 명확
ProcessResult result = ProcessData(input);
```

숫자 리터럴도 마찬가지다.

```csharp
var x = 5;    // int? long? 명확하지 않음
var y = 5L;   // long임이 명확 (리터럴 접미사 활용)
var z = 5.0;  // double임이 명확
```

---

## 결론

| 상황 | 권장 |
|------|------|
| 우변에서 타입이 명확히 보임 | `var` ✅ |
| 익명 타입 | `var` 필수 |
| LINQ 쿼리 결과 | `var` ✅ |
| 타입 정보가 가독성에 중요 | 명시적 타입 |
| 숫자 리터럴 (접미사 없이) | 명시적 타입 권장 |

> `var`는 타입을 숨기는 것이 아니라, 컴파일러가 이미 알고 있는 타입을 반복해서 쓰지 않는 것이다. 올바른 상황에서 `var`를 쓰면 코드가 더 명확해지고 미묘한 타입 변환 버그를 방지할 수 있다.
