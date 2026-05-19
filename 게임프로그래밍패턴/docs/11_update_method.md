# Chapter 10: 업데이트 메서드 패턴 (Update Method Pattern)

> **Game Programming Patterns** — Robert Nystrom 저  
> Part III: Sequencing Patterns / Chapter 10

---

## 의도 (Intent)

> 각 객체에게 한 번에 한 프레임씩 행동하도록 지시함으로써, 독립적인 객체들의 컬렉션을 시뮬레이션한다.

---

## 동기 (Motivation)

### 문제 상황: 게임 루프가 오염된다

용사가 지하 납골당에 들어서자 해골 경비병이 등장한다. 가장 단순한 구현 방법은:

```cpp
// 등장: 단순한 구현
while (true)
{
    // 오른쪽으로 순찰
    for (double x = 0; x < 100; x++)
        skeleton.setX(x);

    // 왼쪽으로 순찰
    for (double x = 100; x > 0; x--)
        skeleton.setX(x);
}
```

**문제**: 이 루프는 영원히 실행되어 게임이 응답하지 않는다. 플레이어는 해골이 움직이는 것을 볼 수 없다.

---

### 1단계: 게임 루프에 통합

게임 루프에서 프레임마다 한 스텝씩 실행하도록 변경:

```cpp
Entity skeleton;
bool patrollingLeft = false;
double x = 0;

// 메인 게임 루프
while (true)
{
    if (patrollingLeft)
    {
        x--;
        if (x == 0) patrollingLeft = false;
    }
    else
    {
        x++;
        if (x == 100) patrollingLeft = true;
    }

    skeleton.setX(x);

    // 사용자 입력 처리 및 렌더링...
}
```

루프 구조에서 한 번에 한 스텝으로 변경됨으로써 코드가 더 복잡해졌다. `patrollingLeft` 변수를 명시적으로 관리해야 한다.

---

### 2단계: 여러 개체 추가 시 게임 루프 오염

여기에 마법 조각상 둘을 추가하면:

```cpp
// 해골 관련 변수들...
Entity leftStatue;
Entity rightStatue;
int leftStatueFrames = 0;
int rightStatueFrames = 0;

// 메인 게임 루프
while (true)
{
    // 해골 코드...

    if (++leftStatueFrames == 90)
    {
        leftStatueFrames = 0;
        leftStatue.shootLightning();
    }

    if (++rightStatueFrames == 80)
    {
        rightStatueFrames = 0;
        rightStatue.shootLightning();
    }

    // 사용자 입력 처리 및 렌더링...
}
```

게임 루프가 점점 커지는 변수 더미와 명령형 코드로 채워진다. 개체가 늘어날수록 이 문제는 심화된다.

> **"무질서"가 아키텍처를 정확히 묘사한다면, 뭔가 문제가 있는 것이다.**

---

### 해결책: 업데이트 메서드 캡슐화

각 개체가 **자신의 행동을 스스로 캡슐화**하도록 한다. 추상 `update()` 메서드를 정의하고, 게임 루프는 컬렉션을 순회하며 각 객체의 `update()`를 호출한다.

```
게임 루프 (Game Loop)
    │
    ▼
for each entity:
    entity.update()     ← 각 개체의 1프레임 행동
    │
    ├── Skeleton.update()    → 순찰 한 스텝 이동
    ├── Statue.update()      → 번개 발사 타이머 감소
    └── Hero.update()        → 플레이어 입력 반응
```

---

## 패턴 구조 (The Pattern)

게임 세계(Game World)는 객체들의 컬렉션을 유지한다. 각 객체는 **한 프레임분의 행동을 시뮬레이션하는 `update()` 메서드**를 구현한다. 매 프레임마다 게임은 컬렉션의 모든 객체를 업데이트한다.

```
┌─────────────────────────────────────┐
│            World                    │
│                                     │
│  Entity* entities_[MAX]             │
│                                     │
│  gameLoop():                        │
│    while (true) {                   │
│      for each entity:               │
│        entity->update();  ──────┐   │
│    }                            │   │
└─────────────────────────────────┼───┘
                                  │
          ┌───────────────────────┼──────────────┐
          │                       │              │
          ▼                       ▼              ▼
   ┌─────────────┐      ┌──────────────┐  ┌──────────────┐
   │  Entity     │      │  Skeleton    │  │  Statue      │
   │ (abstract)  │      │ (concrete)   │  │ (concrete)   │
   │             │      │              │  │              │
   │ update()=0  │◄─────│ update()     │  │ update()     │
   │             │      │ patrolLeft_  │  │ frames_      │
   └─────────────┘      └──────────────┘  │ delay_       │
                                          └──────────────┘
```

---

## 예제 코드 (Sample Code)

### Entity 기반 클래스

```cpp
class Entity
{
public:
    Entity()
    : x_(0), y_(0)
    {}

    virtual ~Entity() {}
    virtual void update() = 0;  // 파생 클래스가 반드시 구현

    double x() const { return x_; }
    double y() const { return y_; }

    void setX(double x) { x_ = x; }
    void setY(double y) { y_ = y; }

private:
    double x_;
    double y_;
};
```

### World: 게임 세계와 게임 루프

```cpp
class World
{
public:
    World()
    : numEntities_(0)
    {}

    void gameLoop();

private:
    static const int MAX_ENTITIES = 100;
    Entity* entities_[MAX_ENTITIES];
    int numEntities_;
};

void World::gameLoop()
{
    while (true)
    {
        // 사용자 입력 처리...

        // 모든 개체 업데이트
        for (int i = 0; i < numEntities_; i++)
        {
            entities_[i]->update();
        }

        // 물리 및 렌더링...
    }
}
```

### Skeleton: 순찰 행동 구현

```cpp
class Skeleton : public Entity
{
public:
    Skeleton()
    : patrollingLeft_(false)
    {}

    virtual void update()
    {
        if (patrollingLeft_)
        {
            setX(x() - 1);
            if (x() == 0) patrollingLeft_ = false;
        }
        else
        {
            setX(x() + 1);
            if (x() == 100) patrollingLeft_ = true;
        }
    }

private:
    bool patrollingLeft_;  // 상태를 필드로 보존
};
```

`patrollingLeft_`가 게임 루프의 지역 변수가 아닌 **멤버 필드**가 된 것이 핵심이다. `update()` 호출 사이에 상태가 유지된다.

### Statue: 번개 발사 타이머

```cpp
class Statue : public Entity
{
public:
    Statue(int delay)
    : frames_(0),
      delay_(delay)
    {}

    virtual void update()
    {
        if (++frames_ == delay_)
        {
            shootLightning();
            frames_ = 0;  // 타이머 리셋
        }
    }

private:
    int frames_;  // 프레임 카운터
    int delay_;   // 발사 주기

    void shootLightning()
    {
        // 번개 발사 처리...
    }
};
```

이제 **조각상을 원하는 만큼** 생성할 수 있고, 각 인스턴스는 고유한 타이머를 유지한다.

```cpp
// 사용 예시: 서로 다른 발사 주기를 가진 조각상들
World world;
world.add(new Statue(90));  // 90프레임마다 번개 발사
world.add(new Statue(80));  // 80프레임마다 번개 발사
world.add(new Statue(60));  // 60프레임마다 번개 발사
world.add(new Skeleton());
```

레벨 데이터 파일에서 이런 설정을 읽어들여 월드를 구성하는 것도 자연스럽게 가능해진다.

---

## 가변 타임스텝 적용 (Variable Time Step)

게임 루프가 가변 타임스텝을 사용한다면 `elapsed` 시간을 함께 전달한다:

```cpp
void Skeleton::update(double elapsed)
{
    if (patrollingLeft_)
    {
        x -= elapsed;
        if (x <= 0)
        {
            patrollingLeft_ = false;
            x = -x;  // 경계 초과분 반전
        }
    }
    else
    {
        x += elapsed;
        if (x >= 100)
        {
            patrollingLeft_ = true;
            x = 100 - (x - 100);  // 경계 초과분 반전
        }
    }
}
```

> 가변 타임스텝은 경계를 오버슈트(overshoot)할 수 있어 경계 처리가 더 복잡해진다.  
> 고정 타임스텝 방식을 선호하는 이유 중 하나다.

---

## 주의사항 (Keep in Mind)

### 1. 코드를 프레임 단위로 쪼개면 복잡도가 증가한다

단일 루프를 프레임 단위 슬라이스로 분해하면, 암묵적인 실행 흐름으로 관리되던 상태를 **명시적 필드로 보존**해야 한다.

```
단순한 루프 (Sequential)        vs     업데이트 방식 (Frame-sliced)
─────────────────────────────────────────────────────────────────
for (x = 0; x < 100; x++)              bool patrollingLeft_;
    skeleton.setX(x);                   void update() {
                                            if (patrollingLeft_) x--;
for (x = 100; x > 0; x--)                  else                x++;
    skeleton.setX(x);                   }
```

이 복잡도 비용은 피할 수 없다. 단, **코루틴(coroutine)** 이나 **파이버(fiber)** 같은 경량 동시성 구조를 지원하는 언어에서는 순차적으로 보이는 코드를 프레임마다 일시 중단/재개할 수 있다.

> **Bytecode 패턴** 역시 애플리케이션 수준에서 실행 스레드를 만드는 또 다른 옵션이다.

---

### 2. 업데이트 순서가 결과에 영향을 미친다

객체들이 진정으로 동시에 실행되는 것이 아니다. A가 B보다 앞서 업데이트되면:

```
프레임 N 업데이트 순서:
[A] → [B] → [C] → [D]
 ↑
 A가 업데이트될 때 B는 N-1 프레임 상태
 B가 업데이트될 때 A는 이미 N 프레임 상태
```

**체스** 비유: 흑과 백이 동시에 움직인다면, 두 말이 같은 칸을 점령하려 할 때 충돌 해결이 어렵다. 순차적 업데이트는 세계를 항상 유효한 상태에서 다음 유효한 상태로 전이시킨다.

> 진정한 동시 상태가 필요하다면 **Double Buffer 패턴**을 사용한다.  
> 업데이트 순서에 무관하게 모든 객체가 이전 프레임의 상태를 본다.

---

### 3. 업데이트 중 컬렉션 수정 시 주의

**객체 추가**: 루프 중 끝에 추가된 객체는 **같은 프레임에 업데이트**될 수 있다.

```cpp
// 해결책: 루프 시작 시 개수를 캐시
int numObjectsThisTurn = numObjects_;
for (int i = 0; i < numObjectsThisTurn; i++)
{
    objects_[i]->update();
}
// 이번 프레임에 추가된 객체는 다음 프레임부터 업데이트됨
```

**객체 제거**: 현재 인덱스 앞의 객체가 제거되면 **건너뜀(skip)** 발생.

```
삭제 전:  [Hero(0)] [Beast(1)] [Peasant(2)]
                       ↑ i=1, Hero가 Beast를 처치
삭제 후:  [Hero(0)] [Peasant(1)]
i 증가: i=2 → Peasant(1)을 건너뜀!
```

**해결책**:

```
① 역방향 순회: 객체 제거 시 이미 처리된 부분만 shift됨
② 지연 제거 (Deferred Removal):
   - 삭제 대신 "dead" 플래그 설정
   - update() 시 dead 객체 건너뜀
   - 루프 완료 후 dead 객체 일괄 제거
```

---

## 설계 결정 (Design Decisions)

### 1. update()를 어느 클래스에 두는가?

| 위치 | 특징 | 적합한 경우 |
|---|---|---|
| **Entity 클래스** | 가장 단순 | 개체 종류가 적을 때. 대규모 프로젝트에서는 계층이 복잡해짐 |
| **Component 클래스** | 컴포넌트별 독립 업데이트 | Component 패턴 사용 시. 렌더링/물리/AI를 각자 처리 |
| **위임 클래스(Delegate)** | State/Type Object 패턴과 연동 | 행동을 런타임에 교체해야 할 때 |

**위임 방식 예시 (State 패턴 연동):**

```cpp
void Entity::update()
{
    state_->update();  // 현재 상태 객체에 위임
}
```

상태 객체를 교체하는 것만으로 개체의 행동 전체를 바꿀 수 있다.

---

### 2. 비활성(dormant) 객체를 어떻게 관리하는가?

화면 밖 객체, 비활성 객체를 매 프레임 순회하는 것은 낭비다.

| 방식 | 장점 | 단점 |
|---|---|---|
| **단일 컬렉션 (비활성 포함)** | 단순 | 비활성 객체 순회 낭비. CPU 캐시 미스 발생 가능 |
| **활성 전용 컬렉션 분리** | 불필요한 순회 없음 | 추가 메모리. 두 컬렉션 동기화 필요 |

> **CPU 캐시 효율성**: 비활성 객체를 건너뛰면 캐시 라인을 낭비해 캐시 미스(cache miss)가 증가한다. 이는 CPU가 더 많은 메인 메모리 접근을 유발한다.  
> **Data Locality 패턴**에서 이 문제를 심층적으로 다룬다.

**활성 전용 컬렉션 예시:**

```cpp
class World
{
public:
    void activate(Entity* entity)
    {
        activeEntities_.push_back(entity);
    }

    void deactivate(Entity* entity)
    {
        activeEntities_.erase(
            std::find(activeEntities_.begin(), activeEntities_.end(), entity));
    }

    void gameLoop()
    {
        while (true)
        {
            // 활성 객체만 업데이트
            for (auto* entity : activeEntities_)
                entity->update();
        }
    }

private:
    std::vector<Entity*> allEntities_;     // 전체 목록
    std::vector<Entity*> activeEntities_;  // 활성 목록만
};
```

---

## 전체 구조 다이어그램

```
게임 루프 1 프레임
┌────────────────────────────────────────────────────────────┐
│                                                            │
│  1. processInput()                                         │
│                                                            │
│  2. for (int i = 0; i < numEntities_; i++)                 │
│         entities_[i]->update()                             │
│                                                            │
│     ┌─────────────┬────────────────┬──────────────┐        │
│     │ Skeleton    │ Statue ×N      │ Hero         │        │
│     │ update()    │ update()       │ update()     │        │
│     │ 한 스텝 이동 │ 타이머 감소    │ 입력 반응    │        │
│     └─────────────┴────────────────┴──────────────┘        │
│                                                            │
│  3. render()                                               │
│                                                            │
└────────────────────────────────────────────────────────────┘
```

---

## 업데이트 메서드 패턴의 가치

이 패턴은 **게임 루프 오염 문제를 해결**하고, **레벨 구성과 구현을 분리**한다.

```
패턴 적용 전:                         패턴 적용 후:
──────────────────────────            ──────────────────────────
while (true) {                        while (true) {
    // 해골 코드 (30줄)                   for each entity:
    // 조각상1 코드 (20줄)                    entity->update(); ← 1줄
    // 조각상2 코드 (20줄)              }
    // 히어로 코드 (40줄)
    // ...                            // 행동은 각 클래스에 캡슐화됨
}
```

새 적 유형을 추가할 때: **새 클래스를 작성하고 컬렉션에 추가**하면 된다. 게임 루프를 건드릴 필요가 없다.

---

## 핵심 정리

| 개념 | 설명 |
|---|---|
| **업데이트 메서드** | 각 개체가 한 프레임분의 행동을 실행하는 가상 메서드 |
| **프레임 슬라이싱 (Frame Slicing)** | 순차적 행동 루프를 프레임 단위로 잘라 상태를 필드로 보존 |
| **업데이트 순서의 영향** | 순차적 업데이트는 실제로 동시가 아님. 앞선 객체의 변경이 뒤따르는 객체에 영향 |
| **지연 제거 (Deferred Removal)** | 업데이트 루프 중 객체를 직접 제거하지 않고, 완료 후 일괄 처리 |
| **활성 컬렉션 분리** | 비활성 객체 순회 낭비 방지. 캐시 효율성 향상 |
| **결정론적 시뮬레이션** | 업데이트 순서를 일관되게 유지하면 동일 입력 → 동일 결과 보장 |

---

## 관련 패턴 (See Also)

| 패턴 | 관계 |
|---|---|
| **Game Loop** | `update()`를 호출하는 외부 루프. 업데이트 메서드와 함께 게임 엔진의 핵심을 구성 |
| **Component** | `update()`를 Entity가 아닌 Component에 두면 행동 재조합이 유연해짐 |
| **State** | 프레임 사이 상태 보존 문제를 자연스럽게 해결. 위임 방식의 `update()`와 결합 |
| **Double Buffer** | 업데이트 순서 의존성을 없애고 싶을 때. 모든 객체가 이전 프레임 상태를 봄 |
| **Data Locality** | 업데이트 루프의 캐시 성능 최적화. 컴포넌트 배열을 연속 메모리에 배치 |

---

## 실제 사용 사례

| 엔진/프레임워크 | 업데이트 메서드 |
|---|---|
| **Unity** | `MonoBehaviour.Update()` — 매 프레임 호출 |
| **Unity** | `MonoBehaviour.FixedUpdate()` — 고정 타임스텝 물리 업데이트 |
| **Microsoft XNA** | `Game.Update()`, `GameComponent.Update()` |
| **Quintus (JS)** | `Sprite` 클래스의 `update` 메서드 |
| **Unreal Engine** | `AActor::Tick(float DeltaTime)` |

---

*© 2009-2015 Robert Nystrom*
