# Chapter 14: 컴포넌트 패턴 (Component Pattern)

> **Game Programming Patterns** — Robert Nystrom 저  
> Part V: Decoupling Patterns / Chapter 14

---

## 의도 (Intent)

> 단일 엔티티가 여러 도메인에 걸쳐 있을 때, 도메인들을 서로 커플링하지 않고 독립적으로 유지할 수 있도록 한다.

---

## 동기 (Motivation)

### 문제: 모놀리식 게임 오브젝트

플랫포머 게임의 주인공 클래스를 만든다고 하자. 주인공은:

- **입력(Input)**: 조이스틱 읽기 → 속도 조절
- **물리(Physics)**: 충돌 감지, 위치 갱신
- **렌더링(Rendering)**: 스프라이트 선택 및 화면 출력
- **사운드(Sound)**: 효과음 재생
- **AI**: (데모 모드 등)

이 모든 것을 하나의 클래스에 넣으면:

```cpp
// 5,000줄짜리 괴물 클래스
class Bjorn
{
    void update()
    {
        // 입력 처리 코드 (물리를 알아야 함)
        // 물리 코드 (그래픽을 알아야 함)
        // 렌더링 코드 (물리 상태를 알아야 함)
        // 사운드 코드 (물리, 렌더링 모두 알아야 함)
        // ...
        if (collidingWithFloor() && (getRenderState() != INVISIBLE))
            playSound(HIT_FLOOR);  // 물리 + 그래픽 + 오디오 동시 커플링!
    }
};
```

**문제 1 — 규모**: 누구나 이 파일을 건드려야 하는데, 너무 거대해서 무서워진다.

**문제 2 — 커플링의 고르디우스 매듭**: 물리, 그래픽, 사운드가 하나로 얽혀 어느 하나도 독립적으로 수정할 수 없다.

> 멀티코어 환경에서는 더 심각하다. AI는 코어 1, 사운드는 코어 2, 렌더링은 코어 3에서 돌리려면 이들이 서로 디커플링되어야 한다. 하나의 클래스에 `UpdateSounds()`와 `RenderGraphics()`가 함께 있으면 교착(deadlock)과 레이스 컨디션을 초래한다.

---

### 해결책: 알렉산더 대왕처럼 칼로 끊어라

모놀리식 클래스를 **도메인 경계를 따라** 분리한다:

```
[Bjorn 모놀리식]
      ↓  도메인별로 분리
┌─────────────┬────────────────┬──────────────────┐
│ Input       │ Physics        │ Graphics         │
│ Component   │ Component      │ Component        │
└─────────────┴────────────────┴──────────────────┘
      ↑               ↑                ↑
  조이스틱만         월드/충돌만       스프라이트만
  알면 됨           알면 됨          알면 됨
```

---

### 상속의 한계와 컴포넌트의 장점

게임 오브젝트 유형들:

| 오브젝트 | 렌더링 | 물리 |
|---|---|---|
| Decoration (나무, 잔해) | ✅ | ❌ |
| Zone (트리거 영역) | ❌ | ✅ |
| Prop (상자, 바위) | ✅ | ✅ |

**상속으로 해결 시**:

```
      [GameObject]
      /          \
  [Zone]      [Decoration]
     ↑              ↑
  [Prop]?  → Deadly Diamond 문제!
  (Prop이 Zone과 Decoration을 동시에 상속할 수 없음)
```

**컴포넌트로 해결 시**:

```cpp
// 장식물: 그래픽만
new GameObject(nullptr, nullptr, new GraphicsComponent());

// 존: 물리만
new GameObject(nullptr, new PhysicsComponent(), nullptr);

// 소품: 둘 다
new GameObject(nullptr, new PhysicsComponent(), new GraphicsComponent());
```

클래스 4개 → **3개**(GameObject + 2 컴포넌트). 중복 코드 없음, 다중 상속 불필요.

> 컴포넌트 = 플러그 앤 플레이. 레스토랑 비유로는 **à la carte(단품 주문)** — 콤보 메뉴(상속) 대신 원하는 것만 골라 조합.

---

## 패턴 구조 (The Pattern)

단일 엔티티가 여러 도메인에 걸쳐있을 때, 각 도메인의 코드를 별도의 컴포넌트 클래스에 둔다. 엔티티는 컴포넌트들의 단순한 컨테이너로 축소된다.

```
┌──────────────────────────────────────────────────────────┐
│                      GameObject                          │
│                                                          │
│  int x, y, velocity  ← 공유 상태 (pan-domain state)      │
│                                                          │
│  update():                                               │
│    input_->update(*this)                                 │
│    physics_->update(*this, world)                        │
│    graphics_->update(*this, graphics)                    │
│                                                          │
│  InputComponent*    input_                               │
│  PhysicsComponent*  physics_                             │
│  GraphicsComponent* graphics_                            │
└──────────────────────────────────────────────────────────┘
         ▲                  ▲                   ▲
┌────────────────┐  ┌───────────────┐  ┌────────────────┐
│ InputComponent │  │PhysicsComp.   │  │GraphicsComp.   │
│ (인터페이스)   │  │ (인터페이스)  │  │ (인터페이스)   │
└────────┬───────┘  └───────┬───────┘  └───────┬────────┘
         ↓                  ↓                   ↓
┌────────────────┐  ┌───────────────┐  ┌────────────────┐
│PlayerInput     │  │BjornPhysics   │  │BjornGraphics   │
│DemoInput       │  │               │  │                │
└────────────────┘  └───────────────┘  └────────────────┘
```

---

## 예제 코드 (Sample Code)

### Step 1: 모놀리식 클래스

```cpp
class Bjorn
{
public:
    Bjorn() : velocity_(0), x_(0), y_(0) {}

    void update(World& world, Graphics& graphics)
    {
        // 입력 처리
        switch (Controller::getJoystickDirection())
        {
            case DIR_LEFT:  velocity_ -= WALK_ACCELERATION; break;
            case DIR_RIGHT: velocity_ += WALK_ACCELERATION; break;
        }

        // 물리: 이동 및 충돌
        x_ += velocity_;
        world.resolveCollision(volume_, x_, y_, velocity_);

        // 렌더링: 스프라이트 선택 및 출력
        Sprite* sprite = &spriteStand_;
        if (velocity_ < 0)       sprite = &spriteWalkLeft_;
        else if (velocity_ > 0)  sprite = &spriteWalkRight_;
        graphics.draw(*sprite, x_, y_);
    }

private:
    static const int WALK_ACCELERATION = 1;
    int velocity_, x_, y_;
    Volume volume_;
    Sprite spriteStand_, spriteWalkLeft_, spriteWalkRight_;
};
```

---

### Step 2: 컴포넌트로 분리

**InputComponent**:

```cpp
class InputComponent
{
public:
    virtual ~InputComponent() {}
    virtual void update(Bjorn& bjorn) = 0;
};

class PlayerInputComponent : public InputComponent
{
public:
    virtual void update(Bjorn& bjorn)
    {
        switch (Controller::getJoystickDirection())
        {
            case DIR_LEFT:  bjorn.velocity -= WALK_ACCELERATION; break;
            case DIR_RIGHT: bjorn.velocity += WALK_ACCELERATION; break;
        }
    }
private:
    static const int WALK_ACCELERATION = 1;
};

// 데모 모드용: AI로 자동 조작
class DemoInputComponent : public InputComponent
{
public:
    virtual void update(Bjorn& bjorn)
    {
        // AI 로직으로 bjorn 조종
    }
};
```

**PhysicsComponent**:

```cpp
class PhysicsComponent
{
public:
    virtual ~PhysicsComponent() {}
    virtual void update(GameObject& obj, World& world) = 0;
};

class BjornPhysicsComponent : public PhysicsComponent
{
public:
    virtual void update(GameObject& obj, World& world)
    {
        obj.x += obj.velocity;
        world.resolveCollision(volume_, obj.x, obj.y, obj.velocity);
    }
private:
    Volume volume_;  // 충돌 볼륨 데이터도 컴포넌트로 이동
};
```

**GraphicsComponent**:

```cpp
class GraphicsComponent
{
public:
    virtual ~GraphicsComponent() {}
    virtual void update(GameObject& obj, Graphics& graphics) = 0;
};

class BjornGraphicsComponent : public GraphicsComponent
{
public:
    virtual void update(GameObject& obj, Graphics& graphics)
    {
        Sprite* sprite = &spriteStand_;
        if (obj.velocity < 0)       sprite = &spriteWalkLeft_;
        else if (obj.velocity > 0)  sprite = &spriteWalkRight_;
        graphics.draw(*sprite, obj.x, obj.y);
    }
private:
    Sprite spriteStand_, spriteWalkLeft_, spriteWalkRight_;
};
```

**GameObject (컨테이너)**:

```cpp
class GameObject
{
public:
    int velocity;
    int x, y;

    GameObject(InputComponent* input,
               PhysicsComponent* physics,
               GraphicsComponent* graphics)
    : input_(input),
      physics_(physics),
      graphics_(graphics)
    {}

    void update(World& world, Graphics& graphics)
    {
        input_->update(*this);
        physics_->update(*this, world);
        graphics_->update(*this, graphics);
    }

private:
    InputComponent*   input_;
    PhysicsComponent* physics_;
    GraphicsComponent* graphics_;
};
```

**팩토리 함수로 조립**:

```cpp
// 플레이어 모드 Bjørn
GameObject* createBjorn()
{
    return new GameObject(
        new PlayerInputComponent(),
        new BjornPhysicsComponent(),
        new BjornGraphicsComponent()
    );
}

// 데모 모드 Bjørn: 컴포넌트 하나만 교체
GameObject* createDemoBjorn()
{
    return new GameObject(
        new DemoInputComponent(),   // AI 조작으로 교체
        new BjornPhysicsComponent(),
        new BjornGraphicsComponent()
    );
}

// 장식물: 물리 없음
GameObject* createDecoration()
{
    return new GameObject(
        new NullInputComponent(),
        nullptr,
        new DecorationGraphicsComponent()
    );
}
```

---

## 컴포넌트 간 통신 방식

컴포넌트들이 완전히 독립적이면 이상적이지만, 실제로는 서로 정보를 주고받아야 한다.

### 방법 1: 컨테이너 객체의 공유 상태

```
[InputComponent]  →  obj.velocity 수정
[PhysicsComponent] ← obj.velocity 읽기, obj.x 수정
[GraphicsComponent] ← obj.x, obj.velocity 읽기
```

- **장점**: 컴포넌트들이 서로를 모름. 느슨한 커플링 유지
- **단점**: 공유할 필요 없는 상태도 컨테이너로 올라가 오염됨. 처리 순서에 암묵적으로 의존

---

### 방법 2: 컴포넌트 간 직접 참조

```cpp
class BjornGraphicsComponent : public GraphicsComponent
{
public:
    // 물리 컴포넌트를 직접 참조
    BjornGraphicsComponent(BjornPhysicsComponent* physics)
    : physics_(physics)
    {}

    virtual void update(GameObject& obj, Graphics& graphics)
    {
        Sprite* sprite;
        if (!physics_->isOnGround())  // 물리 컴포넌트에 직접 질의
            sprite = &spriteJump_;
        else
            sprite = /* ... */;
        graphics.draw(*sprite, obj.x, obj.y);
    }

private:
    BjornPhysicsComponent* physics_;  // 강한 커플링
    Sprite spriteJump_;
    // ...
};
```

- **장점**: 단순하고 빠름. 어떤 메서드든 직접 호출 가능
- **단점**: 두 컴포넌트 간 강한 커플링 → 모놀리식 방향으로 후퇴

적합한 경우: 논리적으로 밀접한 컴포넌트 쌍 (애니메이션-렌더링, 입력-AI 등)

---

### 방법 3: 메시지 시스템

```cpp
class Component
{
public:
    virtual ~Component() {}
    virtual void receive(int message) = 0;  // 메시지 수신
};

class ContainerObject
{
public:
    void send(int message)
    {
        for (int i = 0; i < MAX_COMPONENTS; i++)
        {
            if (components_[i] != NULL)
                components_[i]->receive(message);  // 모든 컴포넌트에 브로드캐스트
        }
    }

private:
    static const int MAX_COMPONENTS = 10;
    Component* components_[MAX_COMPONENTS];
};
```

```cpp
// 사용 예: 물리 컴포넌트가 충돌 메시지 발송
void BjornPhysicsComponent::update(GameObject& obj, World& world)
{
    if (collidedWithSomething)
        obj.send(MSG_COLLISION);  // 오디오 컴포넌트가 수신해 효과음 재생
}
```

- **장점**: 컴포넌트들이 서로를 완전히 모름. "fire-and-forget" 방식. GoF Mediator 패턴
- **단점**: 구현 복잡. 메시지 타입 관리 필요
- Event Queue 패턴과 결합하면 지연 처리도 가능

---

### 통신 방식 선택 가이드

```
공유 상태        → 위치, 크기처럼 모든 컴포넌트가 필요한 기본 정보
직접 참조        → 논리적으로 밀접한 컴포넌트 쌍 (애니메이션↔렌더링)
메시지 시스템    → "덜 중요한" 이벤트성 통신 (충돌 시 효과음 등)

원칙: 단순하게 시작하고, 필요해지면 통신 경로를 추가하라.
```

---

## 설계 결정 (Design Decisions)

### 1. 컴포넌트를 누가 생성하는가?

| 방식 | 장점 | 단점 |
|---|---|---|
| **객체가 직접 생성** | 필요한 컴포넌트를 항상 보장 | 재구성 어려움. 하드코딩 |
| **외부 코드가 주입** | 유연한 재조합 가능. 인터페이스만 노출 | 외부에서 올바르게 조립해야 함 |

---

### 2. Entity-Component System (ECS)

컴포넌트 패턴을 극단적으로 발전시키면 **ECS**가 된다:

```
전통적 방식:
  GameObject → {InputComponent, PhysicsComponent, GraphicsComponent}

ECS 방식:
  엔티티 = 정수 ID (예: entity #42)
  컴포넌트들은 별도 컬렉션에서 엔티티 ID로 조회

  InputComponents[42]    → InputComponent 데이터
  PhysicsComponents[42]  → PhysicsComponent 데이터
  GraphicsComponents[42] → GraphicsComponent 데이터
```

ECS의 장점: 컴포넌트 배열이 연속 메모리에 저장 → **Data Locality 패턴**과 결합해 캐시 최적화 극대화

---

## 주의사항 (Keep in Mind)

- **복잡도 증가**: 개념상 하나인 오브젝트가 여러 객체 클러스터가 됨. 인스턴스화, 초기화, 연결 모두 복잡해짐
- **간접 참조 비용**: 컴포넌트에 접근할 때 포인터를 한 번 더 따라가야 함 (성능에 민감한 내부 루프에서 주의)
- **역설적 성능 향상 가능**: 컴포넌트 배열을 연속 메모리로 정렬하면 Data Locality 패턴과 시너지로 캐시 효율 향상

---

## 핵심 정리

| 개념 | 설명 |
|---|---|
| **컴포넌트 (Component)** | 특정 도메인의 로직과 데이터를 캡슐화한 독립적인 클래스 |
| **컨테이너 객체 (Container Object)** | 컴포넌트들의 집합체. 공유 상태(위치 등)와 update() 진입점만 보유 |
| **Deadly Diamond** | 다중 상속에서 동일한 기반 클래스에 두 경로로 접근하는 문제. 컴포넌트로 우회 |
| **pan-domain state** | 여러 컴포넌트가 공통으로 사용하는 상태(x, y, velocity). 컨테이너 객체에 위치 |
| **ECS (Entity-Component System)** | 엔티티를 ID로만 표현하고 컴포넌트를 별도 배열로 관리하는 극단적 컴포넌트 시스템 |

---

## 관련 패턴 (See Also)

| 패턴 | 관계 |
|---|---|
| **Update Method** | 컴포넌트 시스템을 구동하는 외부 루프. 각 컴포넌트의 `update()`를 호출 |
| **Strategy (GoF)** | 유사한 구조. 차이점: Strategy는 무상태 알고리즘, Component는 상태를 가지며 객체의 정체성 일부를 정의 |
| **Data Locality** | 컴포넌트 배열을 연속 메모리로 배치해 캐시 성능 극대화. ECS와 결합 시 시너지 |
| **Event Queue** | 컴포넌트 간 메시지 시스템을 비동기 큐로 구현할 때 사용 |
| **Mediator (GoF)** | 컴포넌트 간 메시지 라우팅에서 컨테이너 객체가 Mediator 역할 수행 |
| **Factory Method (GoF)** | 팩토리 함수로 컴포넌트를 조립해 게임 오브젝트를 생성하는 방식 |

---

## 실제 사례

| 엔진/프레임워크 | 컴포넌트 구현 |
|---|---|
| **Unity** | `GameObject` + `MonoBehaviour` 컴포넌트 시스템 |
| **Unreal Engine** | `AActor` + `UActorComponent` |
| **Delta3D** | `GameActor` + `ActorComponent` |
| **Microsoft XNA** | `Game` + `GameComponent` (게임 레벨에서 적용) |

---

*© 2009-2015 Robert Nystrom*
