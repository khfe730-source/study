# Chapter 4: 옵저버 패턴 (Observer Pattern)

> **Game Programming Patterns** — Robert Nystrom 저  
> Part II: Design Patterns Revisited / Chapter 4

---

## 의도 (Intent)

> 객체 사이에 일대다(one-to-many) 의존 관계를 정의해, 한 객체의 상태가 바뀌면 그 객체에 의존하는 다른 객체들에게 자동으로 알림이 가고 갱신되도록 한다.

옵저버 패턴은 GoF 패턴 중 가장 널리 사용되는 패턴 중 하나다.  
MVC(Model-View-Controller) 아키텍처의 근간이며, Java(`java.util.Observer`), C#(`event` 키워드) 등 언어 수준에서 지원할 만큼 보편적이다.

---

## 동기 (Motivation): 업적 시스템 (Achievement System)

게임에 업적(achievement) 시스템을 추가한다고 가정하자. "다리에서 떨어지기", "죽은 족제비만 들고 레벨 클리어" 같은 수십 가지 업적이 있다.

### 나쁜 구현

업적 코드를 물리 엔진 안에 직접 삽입:

```cpp
// 충돌 해결 알고리즘 한가운데에…
void Physics::updateEntity(Entity& entity)
{
    // ... 선형대수 코드 ...
    unlockFallOffBridge(); // 물리 코드에 업적 코드가 침투!
}
```

물리 엔진이 업적 시스템에 **강하게 커플링(coupling)** 된다. 업적을 추가할 때마다 물리 코드를 수정해야 한다.

### 좋은 구현: Observer 패턴 적용

물리 엔진은 단지 "무언가 흥미로운 일이 일어났다"고 **알림(notify)** 만 보낸다:

```cpp
void Physics::updateEntity(Entity& entity)
{
    bool wasOnSurface = entity.isOnSurface();
    entity.accelerate(GRAVITY);
    entity.update();
    if (wasOnSurface && !entity.isOnSurface())
    {
        notify(entity, EVENT_START_FALL); // 누가 듣는지 알 필요 없다
    }
}
```

물리 엔진은 **누가 이 알림을 받는지 전혀 알지 못한다.** 업적 시스템을 통째로 제거해도 물리 코드는 수정할 필요가 없다.

---

## 구현 (How it Works)

### Observer (관찰자) 인터페이스

알림을 받고 싶은 객체가 구현하는 인터페이스:

```cpp
class Observer
{
public:
    virtual ~Observer() {}
    virtual void onNotify(const Entity& entity, Event event) = 0;
};
```

업적 시스템은 이 인터페이스를 구현한다:

```cpp
class Achievements : public Observer
{
public:
    virtual void onNotify(const Entity& entity, Event event)
    {
        switch (event)
        {
        case EVENT_ENTITY_FELL:
            if (entity.isHero() && heroIsOnBridge_)
            {
                unlock(ACHIEVEMENT_FELL_OFF_BRIDGE);
            }
            break;
        // 다른 이벤트 처리...
        }
    }

private:
    void unlock(Achievement achievement) { /* ... */ }
    bool heroIsOnBridge_;
};
```

---

### Subject (주체) 클래스

관찰 대상 객체. 옵저버 목록을 유지하고 알림을 발송한다.

```cpp
class Subject
{
public:
    void addObserver(Observer* observer)   { /* 배열에 추가 */ }
    void removeObserver(Observer* observer){ /* 배열에서 제거 */ }

protected:
    void notify(const Entity& entity, Event event)
    {
        for (int i = 0; i < numObservers_; i++)
        {
            observers_[i]->onNotify(entity, event);
        }
    }

private:
    Observer* observers_[MAX_OBSERVERS];
    int numObservers_;
};
```

**Subject가 옵저버 목록을 단일 포인터가 아닌 리스트로 유지하는 이유:**  
옵저버끼리의 암묵적 커플링을 막기 위해서다. 예를 들어 오디오 엔진도 낙하 이벤트를 구독한다면, 단일 옵저버 구조에서는 후등록자가 선등록자를 덮어쓴다.

---

### Observable Physics 연결

```cpp
class Physics : public Subject
{
public:
    void updateEntity(Entity& entity);
};
```

`Subject`를 상속함으로써:
- `notify()`는 `protected` → Physics 내부에서만 호출 가능
- `addObserver()` / `removeObserver()`는 `public` → 외부에서 구독 등록/해제 가능

```
[Physics] ──notify()──▶ [Observer 리스트] ──onNotify()──▶ [Achievements]
                                                    └──────────▶ [AudioEngine]
                                                    └──────────▶ [...]
```

> **Observer vs Event 시스템의 차이**  
> - Observer: "흥미로운 일을 한 객체"를 직접 관찰  
> - Event: "흥미로운 일 자체"를 나타내는 객체를 관찰 (`physics.entityFell().addObserver(this)`)

---

## 자주 나오는 비판과 반론

### "너무 느리다 (It's Too Slow)"

**사실이 아니다.** 알림 발송은 단순히 리스트를 순회하며 가상 메서드를 호출하는 것뿐이다.

- 메시지 큐잉(queueing) 없음
- 동적 메모리 할당 없음
- 단순한 동기 메서드 호출의 간접 참조

성능이 극도로 중요한 핫 코드 패스(hot code path)가 아니라면 비용은 무시할 수 있다.

---

### "동기(Synchronous)라서 위험하다"

옵저버 패턴은 **동기(synchronous)** 방식이다. Subject가 `notify()`를 호출하면, 모든 옵저버의 `onNotify()`가 반환될 때까지 Subject는 블로킹된다.

| 상황 | 위험 |
|---|---|
| 느린 옵저버 | Subject를 블로킹 |
| 스레드 + 락(lock) 사용 | 데드락(deadlock) 가능 |

해결책: 느린 작업은 별도 스레드나 작업 큐로 넘긴다. 멀티스레드 환경에서는 **Event Queue 패턴** 사용을 고려한다.

---

### "동적 할당이 많다 (Too Much Dynamic Allocation)"

기본 구현에서는 옵저버 리스트가 동적 컬렉션이라 메모리 할당/해제가 발생한다.  
**연결 리스트(linked list) 방식**으로 이를 제거할 수 있다.

#### 연결 옵저버 (Linked Observers)

옵저버 객체 자체를 리스트 노드로 사용한다.

```cpp
class Subject
{
public:
    Subject() : head_(NULL) {}

    void addObserver(Observer* observer)
    {
        observer->next_ = head_;
        head_ = observer;
    }

    void removeObserver(Observer* observer)
    {
        if (head_ == observer)
        {
            head_ = observer->next_;
            observer->next_ = NULL;
            return;
        }
        Observer* current = head_;
        while (current != NULL)
        {
            if (current->next_ == observer)
            {
                current->next_ = observer->next_;
                observer->next_ = NULL;
                return;
            }
            current = current->next_;
        }
    }

protected:
    void notify(const Entity& entity, Event event)
    {
        Observer* observer = head_;
        while (observer != NULL)
        {
            observer->onNotify(entity, event);
            observer = observer->next_;
        }
    }

private:
    Observer* head_;
};

class Observer
{
    friend class Subject;
public:
    Observer() : next_(NULL) {}
    virtual void onNotify(const Entity& entity, Event event) = 0;
private:
    Observer* next_; // 다음 옵저버를 가리키는 포인터
};
```

```
Subject
  head_ ──▶ [ObserverC | next_] ──▶ [ObserverB | next_] ──▶ [ObserverA | next_=NULL]
```

**트레이드오프**: 이 방식(침투적 연결 리스트, intrusive linked list)은 하나의 옵저버가 **하나의 Subject만 구독 가능**하다.  
여러 Subject를 동시에 구독하려면, 옵저버를 직접 노드로 쓰는 대신 별도의 **리스트 노드 풀(Object Pool of list nodes)** 을 사용한다.

---

## 남은 문제들 (Remaining Problems)

### 1. Subject/Observer 소멸 시 댕글링 포인터 (Dangling Pointer)

옵저버를 `delete`했는데 Subject가 여전히 그 포인터를 보유하면 **미정의 동작(undefined behavior)** 이 발생한다.

| 해결책 | 설명 |
|---|---|
| **옵저버가 소멸 시 직접 해제** | 소멸자에서 `removeObserver()` 호출. 기억해야 하는 부담이 있음 |
| **Subject가 소멸 전 알림** | "dying breath" 이벤트를 마지막으로 발송, 옵저버가 이를 받아 처리 |
| **기반 클래스에서 자동 해제** | `Observer` 기반 클래스가 구독 중인 Subject 목록을 유지하고, 소멸 시 자동으로 모든 Subject에서 해제 |

---

### 2. 잊혀진 리스너 문제 (Lapsed Listener Problem)

GC가 있는 언어에서도 발생하는 문제다.

**시나리오:**
```
1. UI 화면이 캐릭터의 HP 변경 이벤트를 구독
2. 플레이어가 UI를 닫음
3. UI 객체를 더 이상 참조하지 않지만, Subject(캐릭터)는 여전히 참조를 보유
4. GC가 수거하지 못함 → 좀비 UI 객체 발생
5. HP가 변할 때마다 화면에 없는 UI가 불필요한 연산을 수행
```

> 이 문제는 너무 흔해서 별도의 이름(**Lapsed Listener Problem**)과 위키피디아 문서가 있다.

**해결책**: 구독 등록만큼 **구독 해제에도 규율**이 필요하다.

---

### 3. 추론의 어려움 (What's Going On?)

옵저버 패턴은 의도적으로 커플링을 느슨하게 만들기 때문에, **버그 추적 시 통신 흐름을 파악하기 어렵다**.

- 정적 커플링: IDE에서 메서드 호출 경로를 바로 추적 가능
- 옵저버: 런타임에 어떤 옵저버가 등록되어 있는지 확인해야 함

**지침**: 양쪽을 동시에 이해해야 코드를 파악할 수 있다면, 옵저버 대신 **명시적인 커플링** 을 사용하라.  
옵저버는 **거의 무관한 두 시스템**(예: 물리 ↔ 업적) 사이의 통신에 적합하다.

---

## 현대적 구현 방향

### 클래스 기반 → 함수 기반

OOP 전성기에 설계된 패턴이라 인터페이스 구현이 무겁다. 오늘날은 **함수/클로저**로 대체하는 것이 자연스럽다.

| 언어 | 구현 방식 |
|---|---|
| **C#** | `delegate` + `event` 키워드 |
| **JavaScript** | `EventListener` 프로토콜 또는 단순 함수 |
| **C++** | 멤버 함수 포인터 또는 `std::function` |

```javascript
// 클래스 대신 함수로 등록
subject.on('fall', function(entity) {
    if (entity.isHero()) unlock(ACHIEVEMENT_FELL_OFF_BRIDGE);
});
```

### 미래: 데이터 바인딩 (Data Binding)

옵저버 패턴의 반복적인 보일러플레이트를 줄이려는 시도:

```
// "HP가 바뀌면 healthBar.width를 자동으로 갱신"
healthBar.width <== character.hp * 10
```

**데이터플로 프로그래밍(dataflow programming)**, **함수형 반응형 프로그래밍(functional reactive programming, FRP)** 등이 이 방향의 시도다. 게임 엔진 핵심부에 쓰기엔 아직 무겁지만, UI 레이어에서는 점점 확산 중이다.

---

## 패턴 구조 요약

```
        ┌────────────────────────────────┐
        │           Subject              │
        │────────────────────────────────│
        │ - observers_: Observer*[]      │
        │ + addObserver(Observer*)       │
        │ + removeObserver(Observer*)    │
        │ # notify(Entity&, Event)       │
        └────────────────────────────────┘
                        ▲ 상속
               ┌────────┴────────┐
               │     Physics     │
               │ updateEntity()  │
               └─────────────────┘

        ┌────────────────────────────────┐
        │           Observer             │  ← 인터페이스
        │────────────────────────────────│
        │ + onNotify(Entity&, Event) = 0 │
        └────────────────────────────────┘
                        ▲ 구현
               ┌────────┴────────┐
               │  Achievements   │
               │  onNotify(...)  │
               └─────────────────┘
```

---

## 언제 사용할까? (When to Use It)

- **거의 무관한 두 시스템** 이 서로 통신해야 할 때 (물리 ↔ 업적, 게임 로직 ↔ UI)
- 한 이벤트에 **여러 리스너** 가 반응해야 할 때
- 통신의 수신자를 런타임에 **동적으로 변경** 해야 할 때

---

## 주의사항 (Keep in Mind)

- **동기 방식**: 느린 옵저버는 Subject를 블로킹한다
- **소멸 관리**: 옵저버/Subject 소멸 시 댕글링 포인터 주의
- **Lapsed Listener**: GC 환경에서도 명시적 구독 해제 필요
- **디버깅 난이도**: 런타임 등록 상태를 알아야 흐름 파악 가능
- **양쪽 이해 필요 시**: 옵저버 대신 명시적 커플링을 고려하라

---

## 관련 패턴 (See Also)

| 패턴 | 관계 |
|---|---|
| **Event Queue** | 비동기 알림이 필요하거나 멀티스레드 환경에서 Observer의 동기 방식을 대체 |
| **Chain of Responsibility** (GoF) | 옵저버가 알림 후 처리 여부를 반환해 다음 옵저버로의 전달을 제어하면 이 패턴에 가까워짐 |
| **Object Pool** | 연결 리스트 노드를 풀로 관리해 동적 할당 없이 다수의 Subject 동시 구독 지원 |

---

## 핵심 정리

| 개념 | 설명 |
|---|---|
| **Subject (주체)** | 관찰 대상. 옵저버 목록을 유지하고 알림을 발송 |
| **Observer (관찰자)** | `onNotify()`를 구현해 알림을 수신 |
| **느슨한 커플링** | Subject는 Observer 인터페이스만 알면 되고, 구체적인 옵저버 타입을 알 필요 없음 |
| **동기 알림** | 모든 옵저버의 처리가 끝날 때까지 Subject가 블로킹됨 |
| **Lapsed Listener** | 더 이상 필요 없는 옵저버를 해제하지 않아 발생하는 메모리 누수 |
| **침투적 연결 리스트** | Observer 자체를 노드로 사용해 동적 할당 없이 구현하는 방식 |

---

*© 2009-2015 Robert Nystrom*
