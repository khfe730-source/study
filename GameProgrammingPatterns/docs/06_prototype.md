# Chapter 5: 프로토타입 패턴 (Prototype Pattern)

> **Game Programming Patterns** — Robert Nystrom 저  
> Part II: Design Patterns Revisited / Chapter 5

---

## 의도 (Intent)

> 원형(prototype)이 되는 인스턴스를 사용해 생성할 객체의 종류를 명시하고, 이 원형을 복사해서 새로운 객체를 생성한다.

이 챕터는 단순히 GoF 디자인 패턴의 Prototype을 설명하는 것을 넘어, **언어 패러다임으로서의 프로토타입**과 **데이터 모델링에서의 프로토타입** 활용까지 폭넓게 다룬다.

---

## 1부: 프로토타입 디자인 패턴

### 문제 — 몬스터 스포너 클래스 폭발

Gauntlet 스타일의 게임을 만든다고 가정하자. 각 몬스터 종류마다 별도 클래스가 있다.

```cpp
class Monster  { /* ... */ };
class Ghost    : public Monster {};
class Demon    : public Monster {};
class Sorcerer : public Monster {};
```

몬스터를 생성하는 스포너(spawner)도 각 타입마다 별도 클래스가 필요하다:

```cpp
class Spawner      { public: virtual Monster* spawnMonster() = 0; };
class GhostSpawner  : public Spawner { Monster* spawnMonster() { return new Ghost(); } };
class DemonSpawner  : public Spawner { Monster* spawnMonster() { return new Demon(); } };
class SorcererSpawner : public Spawner { Monster* spawnMonster() { return new Sorcerer(); } };
```

몬스터 종류가 늘수록 **클래스 계층이 평행하게 폭발**한다. 중복 코드, 보일러플레이트 범람.

```
Monster ──▶ Ghost, Demon, Sorcerer, ...
Spawner ──▶ GhostSpawner, DemonSpawner, SorcererSpawner, ...  ← 완전 중복!
```

---

### 해결: Prototype 패턴

**핵심 아이디어**: 객체가 자기 자신과 유사한 객체를 스폰할 수 있게 한다.

**1단계 — Monster에 `clone()` 추가**

```cpp
class Monster
{
public:
    virtual ~Monster() {}
    virtual Monster* clone() = 0;
};

class Ghost : public Monster
{
public:
    Ghost(int health, int speed)
        : health_(health), speed_(speed)
    {}

    virtual Monster* clone()
    {
        return new Ghost(health_, speed_); // 상태까지 복사
    }

private:
    int health_;
    int speed_;
};
```

**2단계 — Spawner는 하나만 남긴다**

```cpp
class Spawner
{
public:
    Spawner(Monster* prototype)
        : prototype_(prototype)
    {}

    Monster* spawnMonster()
    {
        return prototype_->clone();
    }

private:
    Monster* prototype_; // 템플릿 역할의 숨겨진 몬스터 인스턴스
};
```

**3단계 — 사용**

```cpp
// 원형 몬스터 생성
Monster* ghostPrototype = new Ghost(15, 3);

// 스포너는 프로토타입을 보관하고 복제
Spawner* ghostSpawner = new Spawner(ghostPrototype);

// 빠른 유령, 약한 유령 스포너도 원형만 바꾸면 된다
Monster* fastGhostPrototype = new Ghost(15, 10);
Spawner* fastGhostSpawner   = new Spawner(fastGhostPrototype);
```

> **프로토타입의 핵심**: 클래스만 복제하는 것이 아니라 **상태(state)까지 복제**한다.  
> 빠른 유령, 약한 유령, 느린 유령을 위한 별도 클래스 없이, 원형 인스턴스 하나만 바꾸면 된다.

---

### 패턴의 한계

| 문제 | 설명 |
|---|---|
| **`clone()` 구현 부담** | Spawner 클래스마다 작성하던 코드를 Monster 클래스마다 작성하게 됨. 코드 절감이 크지 않음 |
| **깊은/얕은 복사 모호성** | Demon이 Pitchfork를 참조한다면, 복제 시 Pitchfork도 복제해야 하는가? |
| **클래스 계층 자체가 문제** | 현대 게임 엔진은 몬스터마다 클래스를 만들지 않는다. **Component 패턴**, **Type Object 패턴**이 더 적합한 해법 |

저자의 솔직한 평가: *"프로토타입 디자인 패턴이 최선인 경우를 실제로는 본 적이 없다."*

---

### 대안 1: 스폰 함수 (Spawn Functions)

클래스 대신 단순 함수 포인터 사용:

```cpp
Monster* spawnGhost() { return new Ghost(); }

typedef Monster* (*SpawnCallback)();

class Spawner
{
public:
    Spawner(SpawnCallback spawn) : spawn_(spawn) {}
    Monster* spawnMonster() { return spawn_(); }
private:
    SpawnCallback spawn_;
};

Spawner* ghostSpawner = new Spawner(spawnGhost);
```

---

### 대안 2: 템플릿 (Templates)

C++ 템플릿으로 타입 파라미터화:

```cpp
class Spawner
{
public:
    virtual ~Spawner() {}
    virtual Monster* spawnMonster() = 0;
};

template <class T>
class SpawnerFor : public Spawner
{
public:
    virtual Monster* spawnMonster() { return new T(); }
};

Spawner* ghostSpawner = new SpawnerFor<Ghost>();
```

> `Spawner` 기반 클래스를 두는 이유: 몬스터 타입에 관계없이 스포너를 `Spawner*`로 다룰 수 있게 하기 위해서다.

---

### 대안 3: 일급 타입 (First-Class Types)

JavaScript, Python, Ruby처럼 **클래스 자체가 일급 객체(first-class object)** 인 언어라면, 타입을 그냥 변수에 담아 넘길 수 있다:

```python
class Spawner:
    def __init__(self, monster_class):
        self.monster_class = monster_class

    def spawn_monster(self):
        return self.monster_class()

ghost_spawner = Spawner(Ghost)
```

---

## 2부: 언어 패러다임으로서의 프로토타입

### Self 언어

1980년대 Dave Ungar와 Randall Smith가 만든 **Self** 언어는 클래스 없이 순수 프로토타입으로만 동작한다.

**클래스 기반 언어와의 차이:**

| 항목 | 클래스 기반 언어 | Self (프로토타입 기반) |
|---|---|---|
| **상태 접근** | 인스턴스 메모리에서 직접 접근 | 객체 자체에서 접근 |
| **메서드 접근** | 클래스를 거쳐 간접 접근 (vtable) | 객체 자체에서 접근, 없으면 부모에 위임 |
| **상속 구조** | 클래스 계층 | 부모(parent) 객체에 위임(delegation) |
| **새 객체 생성** | `new ClassName()` | 기존 객체 복제(clone) |

**위임(Delegation)의 원리:**

```
object.someMethod() 탐색 순서:
1. object 자체에서 탐색
2. 없으면 object.parent에서 탐색
3. 없으면 parent.parent에서 탐색 (재귀)
```

Self의 모든 객체는 자동으로 프로토타입 패턴을 지원한다. `clone()`을 직접 구현할 필요가 없다.

**저자의 평가**: 우아하고 영리하지만, **실제로 사용하기는 불편하다.** 클래스가 제공하는 구조가 없으면 코드가 '이상하게 무르(mushy)'게 된다.

> Self의 진정한 유산: Self를 빠르게 실행하기 위해 개발된 **JIT 컴파일**, **GC**, **메서드 디스패치 최적화** 기법들이 현대 동적 언어(JavaScript, Python 등)의 성능을 가능하게 했다.

---

### JavaScript의 프로토타입

JavaScript는 Self에서 영감을 받았지만, 실제로는 **클래스 기반 언어에 가깝다**.

**생성자 함수(Constructor Function) 방식:**

```javascript
// 생성자 함수 (상태 정의)
function Weapon(range, damage) {
    this.range  = range;
    this.damage = damage;
}

// 프로토타입에 메서드 추가 (행동 정의)
Weapon.prototype.attack = function(target) {
    if (distanceTo(target) > this.range) {
        console.log("Out of range!");
    } else {
        target.health -= this.damage;
    }
};

var sword = new Weapon(10, 16);
sword.attack(enemy); // Weapon.prototype.attack으로 위임
```

```
sword 인스턴스              Weapon.prototype
┌──────────────────┐       ┌──────────────────┐
│ range:  10       │──▶    │ attack: function  │
│ damage: 16       │       │                  │
└──────────────────┘       └──────────────────┘
```

JavaScript의 패턴을 분석하면:

- 타입을 나타내는 **생성자 함수** (클래스 역할)
- **인스턴스**에 상태 저장
- **프로토타입 객체**에 공유 메서드 저장 (클래스 메서드 역할)

→ 결국 클래스 기반 언어와 구조적으로 동일하다.

> JavaScript의 핵심 프로토타입 연산인 **복제(cloning)는 사실상 없다**.  
> `Object.create()`가 ES5에서야 추가되었고, 일반적인 사용 패턴은 클래스 기반이다.

---

## 3부: 데이터 모델링에서의 프로토타입

저자가 실제로 유용하다고 생각하는 프로토타입 활용 영역이다.

### 문제 — 데이터의 중복

게임 데이터를 JSON으로 정의할 때 유사한 엔티티 사이의 중복이 발생한다:

```json
{
    "name": "goblin grunt",
    "minHealth": 20,
    "maxHealth": 30,
    "resists": ["cold", "poison"],
    "weaknesses": ["fire", "light"]
}

{
    "name": "goblin wizard",
    "minHealth": 20,
    "maxHealth": 30,
    "resists": ["cold", "poison"],
    "weaknesses": ["fire", "light"],
    "spells": ["fire ball", "lightning bolt"]
}

{
    "name": "goblin archer",
    "minHealth": 20,
    "maxHealth": 30,
    "resists": ["cold", "poison"],
    "weaknesses": ["fire", "light"],
    "attacks": ["short bow"]
}
```

HP, 저항, 약점이 세 번 반복된다. 코드라면 추상화(abstraction)로 해결할 텐데, JSON은 그런 기능이 없다.

---

### 해결: 데이터의 프로토타입 위임

`"prototype"` 필드를 메타데이터로 도입해, 해당 필드가 없으면 원형 객체에서 탐색한다:

```json
{
    "name": "goblin grunt",
    "minHealth": 20,
    "maxHealth": 30,
    "resists": ["cold", "poison"],
    "weaknesses": ["fire", "light"]
}

{
    "name": "goblin wizard",
    "prototype": "goblin grunt",
    "spells": ["fire ball", "lightning bolt"]
}

{
    "name": "goblin archer",
    "prototype": "goblin grunt",
    "attacks": ["short bow"]
}
```

**위임 탐색 로직 (의사코드):**

```
getProperty(entity, key):
    if entity has key → return entity[key]
    if entity has "prototype" → return getProperty(prototype_of(entity), key)
    return null
```

**특이점**: 추상 기반 엔티티("base goblin")를 별도로 만들지 않고, **가장 단순한 구체 객체**(goblin grunt)를 프로토타입으로 사용한다. 프로토타입 기반 시스템의 자연스러운 특성이다.

---

### 활용 예: 고유 아이템 정의

보스, 전설 아이템처럼 기본 객체의 **정제(refinement)** 인 경우에 특히 유용하다:

```json
{
    "name": "Sword of Head-Detaching",
    "prototype": "longsword",
    "damageBonus": 20
}
```

`longsword`의 모든 속성을 상속받으면서 `damageBonus`만 추가한다. 디자이너가 클래스 정의 없이 쉽게 변형을 만들 수 있다.

---

## 패턴 구조 요약

```
    ┌──────────────────────┐
    │       Monster        │  ← Prototype 기반 클래스
    │──────────────────────│
    │ + clone(): Monster*  │  ← 핵심 메서드
    └──────────────────────┘
              ▲
    ┌─────────┴────────┐
    │                  │
┌───────┐          ┌───────┐
│ Ghost │          │ Demon │  ← Concrete Prototype
│clone()│          │clone()│
└───────┘          └───────┘

    ┌──────────────────────┐
    │       Spawner        │  ← Client
    │──────────────────────│
    │ - prototype_: Monster│
    │ + spawnMonster()     │──▶ prototype_->clone()
    └──────────────────────┘
```

---

## 언제 사용할까? (When to Use It)

**GoF 디자인 패턴으로서:**
- 생성할 객체의 타입이 런타임에 결정될 때
- 초기화 비용이 크고, 기존 인스턴스 상태를 재활용하고 싶을 때
- 클래스 계층을 피하고 싶을 때 (단, 현대 게임에서는 더 나은 대안이 많다)

**데이터 모델링으로서:**
- JSON 같은 데이터 정의 파일에서 **엔티티 간 중복**이 많을 때
- 보스, 특수 아이템처럼 기본 객체의 정제(refinement)가 필요할 때
- 디자이너가 코드 없이 변형을 정의해야 할 때

---

## 주의사항 (Keep in Mind)

- **깊은 복사 vs 얕은 복사(deep copy vs shallow copy)**: `clone()`이 참조하는 객체까지 복제해야 하는가? 명확히 정의해야 한다.
- **`clone()` 유지 관리**: 필드가 추가될 때마다 `clone()`도 업데이트해야 한다.
- **현대 게임에서의 적합성**: 몬스터마다 클래스를 만드는 설계 자체가 이미 문제일 수 있다. Component, Type Object 패턴을 먼저 고려하라.

---

## 관련 패턴 (See Also)

| 패턴 | 관계 |
|---|---|
| **Component** | 몬스터마다 클래스를 만드는 대신, 컴포넌트 조합으로 다양한 몬스터를 표현하는 현대적 대안 |
| **Type Object** | 클래스 계층 없이 "타입" 개념을 데이터로 모델링하는 대안 |
| **Factory Method** (GoF) | 생성 로직을 캡슐화하는 다른 접근법. 프로토타입의 대안 |
| **Abstract Factory** (GoF) | 관련 객체 군을 함께 생성해야 할 때의 대안 |
| **Object Pool** | 빈번하게 생성/소멸되는 객체를 프로토타입으로 찍어낼 때 풀과 함께 활용 |

---

## 핵심 정리

| 개념 | 설명 |
|---|---|
| **clone()** | 객체가 자기 자신을 복제해 새 인스턴스를 반환하는 핵심 메서드 |
| **상태 복제** | 클래스뿐 아니라 인스턴스 상태까지 복사. 다양한 변형 생성 가능 |
| **위임(Delegation)** | 프로퍼티 탐색 실패 시 부모/프로토타입 객체에 위임하는 메커니즘 |
| **Self 언어** | 순수 프로토타입 기반 OOP의 원형. JIT/GC 혁신의 산실 |
| **데이터 프로토타입** | JSON 데이터 모델에서 `"prototype"` 필드로 상속/재사용 구현 |

---

*© 2009-2015 Robert Nystrom*
