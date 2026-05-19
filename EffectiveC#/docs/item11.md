# Item 11: .NET 리소스 관리를 이해하라 (Understand .NET Resource Management)

> **Chapter 2: .NET Resource Management**

---

## 핵심 요약

.NET의 GC(가비지 컬렉터)는 관리 메모리를 자동으로 처리하지만, 비관리 리소스는 개발자가 직접 해제해야 한다. 파이널라이저는 최후의 방어 수단이며 성능 비용이 크므로 `IDisposable` 패턴으로 대체해야 한다.

---

## GC의 작동 방식

GC는 **Mark and Compact** 알고리즘으로 동작한다.

1. 애플리케이션 루트 객체에서 시작해 객체 트리를 탐색 **(Mark)**
2. 도달 불가능한 객체를 가비지로 판단
3. 살아있는 객체를 힙 앞쪽으로 모아 연속된 여유 공간 확보 **(Compact)**

```
[GC 전]                         [GC 후]
Main Form (C, E)                Main Form (C, E)
B  C  D  E(F)  F                C  E(F)  F
↑                               ↑
B, D는 도달 불가 → 수거         힙 압축 완료 → 연속 여유 공간
```

이 방식 덕분에 **순환 참조(circular references)** 도 자동으로 처리된다. COM처럼 각 객체가 참조 카운트를 직접 관리할 필요가 없다.

---

## GC가 관리하지 않는 것들

GC는 **관리 힙(managed heap)** 의 메모리만 책임진다. 아래 리소스는 개발자가 직접 해제해야 한다.

- 데이터베이스 연결 (DB connections)
- GDI+ 객체
- COM 객체
- 기타 OS 시스템 객체

또한 **이벤트 핸들러/델리게이트**로 연결된 객체는 GC가 수거하지 못해 의도보다 오래 메모리에 남을 수 있다.

---

## 파이널라이저 (Finalizer)의 문제점

C++의 RAII 패턴(소멸자에서 자원 해제)은 C#에서 그대로 적용되지 않는다.

```csharp
// Good C++, bad C#:
class CriticalSection
{
    public CriticalSection()  { EnterCriticalSection(); }
    ~CriticalSection()        { ExitCriticalSection(); }  // 호출 시점 보장 불가!
}

void Func()
{
    CriticalSection s = new CriticalSection();
    // C++: 함수 종료 시 소멸자 즉시 호출 → 크리티컬 섹션 해제 보장
    // C#:  GC가 실행될 때 파이널라이저 호출 → 언제인지 알 수 없음
}
```

C#의 파이널라이저는 **비결정론적(nondeterministic)** 으로 실행된다. 파이널라이저가 있는 객체는 아래와 같은 성능 비용이 발생한다.

1. GC 실행 시 즉시 수거 불가 → **Finalization Queue**에 추가
2. 별도 파이널라이저 스레드가 실행
3. 파이널라이저 완료 후 다음 GC 사이클에서야 수거

```
[GC 사이클 1]                [GC 사이클 2]              [GC 사이클 3]
Main Form (C, E)             Main Form (C, E)           Main Form (C, E)
B(★)  C  D  E(F)  F         C  E(F)  F  B(★)          C  E(F)  F
★ = 파이널라이저 대기         B 파이널라이저 실행 중       B 수거 완료
```

---

## 세대별 GC (Generational GC)

GC는 성능 최적화를 위해 세대(Generation) 개념을 사용한다.

| 세대 | 설명 | 검사 빈도 |
|------|------|-----------|
| Generation 0 | 최근 생성된 객체 (단명 객체) | 매 GC 사이클 |
| Generation 1 | GC 1회 이상 생존 | 약 10회 중 1회 |
| Generation 2 | GC 2회 이상 생존 (장수 객체) | 약 100회 중 1회 |

**파이널라이저가 있는 객체의 실제 비용**: 파이널라이저가 있으면 Generation이 올라가며 생존하고, Generation 2에 도달하면 추가로 약 100 GC 사이클 동안 메모리에 잔류할 수 있다.

---

## 올바른 접근법

> 비관리 리소스 해제는 파이널라이저 대신 `IDisposable` 인터페이스와 표준 Dispose 패턴을 사용하라 (→ [Item 17](./item17.md) 참고).

파이널라이저는 클라이언트가 `Dispose()`를 호출하지 않을 경우를 대비한 **최후의 방어 수단**으로만 존재해야 한다.

---

## 결론

- GC는 관리 메모리(managed memory)를 책임진다
- 비관리 리소스(DB 연결, GDI+, COM 등)는 개발자가 직접 해제해야 한다
- 파이널라이저는 비결정론적이며 GC 성능에 상당한 부담을 준다
- `IDisposable` + 표준 Dispose 패턴으로 파이널라이저 실행을 억제하는 것이 권장 방식이다
