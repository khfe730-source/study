# Item 15: 불필요한 객체 생성을 피하라 (Avoid Creating Unnecessary Objects)

> **Chapter 2: .NET Resource Management**

---

## 핵심 요약

GC가 효율적이더라도 객체 생성과 수거에는 비용이 따른다. 특히 **자주 호출되는 메서드에서 동일한 참조 타입 객체를 반복 생성**하는 것은 GC 압박을 높이는 주된 원인이다.

---

## 안티패턴: Paint 핸들러에서 GDI 객체 반복 생성

```csharp
// 나쁜 예: 매 Paint 이벤트마다 Font 객체 생성 → 대량의 가비지 발생
protected override void OnPaint(PaintEventArgs e)
{
    using (Font myFont = new Font("Arial", 10.0f))
    {
        e.Graphics.DrawString(DateTime.Now.ToString(),
            myFont, Brushes.Black, new PointF(0, 0));
    }
    base.OnPaint(e);
}
```

`OnPaint()`는 매우 빈번히 호출된다. 호출마다 동일한 `Font` 객체를 생성하고 즉시 가비지로 만든다. 메모리 할당 빈도가 높아지면 GC 실행 빈도도 높아진다.

---

## 해결책 1: 지역 변수 → 멤버 변수로 승격

```csharp
// 좋은 예: 한 번 생성하고 재사용
private readonly Font myFont = new Font("Arial", 10.0f);

protected override void OnPaint(PaintEventArgs e)
{
    e.Graphics.DrawString(DateTime.Now.ToString(),
        myFont, Brushes.Black, new PointF(0, 0));
    base.OnPaint(e);
}
```

**적용 기준**: 참조 타입이고, 자주 호출되는 메서드에서 사용되는 지역 변수.

> `IDisposable`을 구현하는 객체를 멤버 변수로 승격하면 해당 클래스도 `IDisposable`을 구현해야 한다 (→ [Item 17](./item17.md)).

---

## 해결책 2: 정적 멤버로 공통 인스턴스 공유

자주 쓰이는 공통 객체는 정적 멤버로 공유한다. .NET의 `Brushes` 클래스가 좋은 예다.

```csharp
// Brushes 클래스 내부 구현 (지연 초기화 패턴)
private static Brush blackBrush;

public static Brush Black
{
    get
    {
        if (blackBrush == null)
            blackBrush = new SolidBrush(Color.Black);
        return blackBrush;
    }
}
```

처음 요청될 때만 생성하고(Lazy Evaluation), 이후에는 동일한 인스턴스를 반환한다. 사용하지 않는 색상의 Brush는 아예 생성되지 않는다.

> **트레이드오프**: 객체가 메모리에 더 오래 잔류할 수 있고, `Dispose()` 호출 시점을 결정하기 어려워진다.

---

## 해결책 3: 불변 타입은 Builder 패턴 활용

`string`은 불변(immutable)이므로 `+=`로 연결할 때마다 새 객체가 생성된다.

```csharp
// 나쁜 예: 중간 문자열 객체 3개 추가 생성
string msg = "Hello, ";
msg += thisUser.Name;       // 새 string 생성, 이전 것은 가비지
msg += ". Today is ";       // 새 string 생성
msg += DateTime.Now.ToString(); // 새 string 생성

// 좋은 예 1: 문자열 보간 (단순한 경우)
string msg = $"Hello, {thisUser.Name}. Today is {DateTime.Now}";

// 좋은 예 2: StringBuilder (복잡한 로직의 경우)
StringBuilder msg = new StringBuilder("Hello, ");
msg.Append(thisUser.Name);
msg.Append(". Today is ");
msg.Append(DateTime.Now.ToString());
string finalMsg = msg.ToString(); // 최종 불변 string 한 번만 생성
```

`StringBuilder`는 내부적으로 가변(mutable) 버퍼를 사용해 중간 객체 생성 없이 문자열을 조립한다.

---

## 세 가지 기법 정리

| 기법 | 적용 상황 |
|------|-----------|
| 멤버 변수로 승격 | 자주 호출되는 메서드의 참조 타입 지역 변수 |
| 정적 멤버로 공유 | 여러 인스턴스에서 공통으로 쓰이는 객체 |
| Builder 패턴 | 불변 타입(string 등)을 단계적으로 조립하는 경우 |

---

## 결론

> 자주 호출되는 메서드에서 동일한 참조 타입 객체를 반복 생성하지 마라. 멤버 변수로 승격하거나, 정적으로 공유하거나, Builder 패턴을 활용해 GC 압박을 줄여라. 필요하지 않은 객체를 만들지 않는 것이 최선이다.
