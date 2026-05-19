# Chapter 13: 타입 객체 패턴 (Type Object Pattern)

> **Game Programming Patterns** — Robert Nystrom 저  
> Part IV: Behavioral Patterns / Chapter 13

---

## 의도 (Intent)

> 단일 클래스를 만들고, 각 인스턴스가 서로 다른 타입의 객체를 표현하게 함으로써 새로운 "클래스"를 유연하게 생성할 수 있도록 한다.

---

## 동기 (Motivation)

### 문제: 수백 개의 몬스터 종류

RPG 게임에서 몬스터는 종족(breed)에 따라 서로 다른 초기 체력과 공격 방식을 갖는다.

**일반적인 OOP 해결책**:

```cpp
class Monster
{
public:
    virtual ~Monster() {}
    virtual const char* getAttack() = 0;

protected:
    Monster(int startingHealth) : health_(startingHealth) {}

private:
    int health_;
};

class Dragon : public Monster
{
public:
    Dragon() : Monster(230) {}
    virtual const char* getAttack() { return "The dragon breathes fire!"; }
};

class Troll : public Monster
{
public:
    Troll() : Monster(48) {}
    virtual const char* getAttack() { return "The troll clubs you!"; }
};
```

처음에는 잘 동작한다. 그런데 종류가 수십 개를 넘어 수백 개가 되면:

```
문제 ①: 종류마다 7줄짜리 서브클래스 파일 생성 → 재컴파일 반복
문제 ②: 디자이너가 트롤 체력을 48→52로 바꾸려면 프로그래머 개입 필수
         [이메일 수신] → [Troll.h 수정] → [재컴파일] → [체크인] → [이메일 회신] → 반복
문제 ③: 런타임에 새 종족 추가 불가 (다운로드 콘텐츠, 모딩 불가)
```

---

### 핵심 통찰: 클래스를 위한 클래스

상속을 사용하면 클래스 계층이 이렇게 된다:

```
        [Monster]
        /       \
  [Dragon]   [Troll]   [Slime] ... (종류마다 클래스 추가)
```

대신, 몬스터가 **종족 객체에 대한 참조**를 가지게 하면:

```
[Breed]  ←──── [Monster]
  ↑               ↑
  종족 정보 공유   개별 몬스터 인스턴스
 (정적 데이터)    (동적 상태)
```

두 개의 클래스만으로 수백 가지 종족을 표현할 수 있다. `Breed` 인스턴스가 각각의 "타입"을 대표하므로 **타입 객체(Type Object)** 라고 부른다.

---

## 패턴 구조 (The Pattern)

**타입 객체 클래스(Breed)** 와 **타입화된 객체 클래스(Monster)** 를 정의한다. 각 타입 객체 인스턴스는 서로 다른 논리적 타입을 표현한다. 각 타입화된 객체는 자신의 타입을 설명하는 타입 객체에 대한 참조를 저장한다.

```
┌──────────────────────────────────────────────────┐
│                    Breed                         │
│ (타입 객체 — 종족 전체에 걸쳐 공유되는 데이터)    │
│                                                  │
│  health_  : int           // 초기 체력           │
│  attack_  : const char*   // 공격 문자열         │
│                                                  │
│  getHealth()                                     │
│  getAttack()                                     │
│  newMonster()  ← 팩토리 메서드 (선택적)           │
└─────────────────────────┬────────────────────────┘
                          │  참조 (reference)
                          │
┌─────────────────────────▼────────────────────────┐
│                   Monster                        │
│ (타입화된 객체 — 각 몬스터 인스턴스 고유 데이터)   │
│                                                  │
│  health_ : int    // 현재 체력 (인스턴스별)        │
│  breed_  : Breed& // 종족 참조                   │
│                                                  │
│  getAttack()  →  breed_.getAttack() 위임          │
└──────────────────────────────────────────────────┘
```

---

## 예제 코드 (Sample Code)

### 기본 구현

```cpp
class Breed
{
public:
    Breed(int health, const char* attack)
    : health_(health),
      attack_(attack)
    {}

    int         getHealth() { return health_; }
    const char* getAttack() { return attack_; }

private:
    int         health_;  // 종족 초기 체력
    const char* attack_;  // 종족 공격 문자열
};

class Monster
{
public:
    Monster(Breed& breed)
    : health_(breed.getHealth()),  // 종족에서 초기 체력 획득
      breed_(breed)
    {}

    const char* getAttack()
    {
        return breed_.getAttack();  // 종족에 위임
    }

private:
    int    health_;  // 현재 체력 (인스턴스별)
    Breed& breed_;   // 종족 참조
};
```

**사용 예시**:

```cpp
// 종족 정의 (런타임에 파일에서 읽어도 됨)
Breed dragonBreed(230, "The dragon breathes fire!");
Breed trollBreed(48,   "The troll clubs you!");

// 몬스터 생성
Monster dragon(dragonBreed);
Monster troll1(trollBreed);
Monster troll2(trollBreed);  // 같은 종족 공유
Monster troll3(trollBreed);

// troll1, troll2, troll3는 모두 같은 Breed 인스턴스를 참조
```

---

### 타입 객체를 타입처럼: 팩토리 메서드

일반 OOP에서는 클래스 자체에서 인스턴스를 생성한다(`new Dragon()`). 타입 객체에도 같은 패턴을 적용할 수 있다:

```cpp
class Breed
{
public:
    // 팩토리 메서드 — Monster 생성 책임을 Breed가 가짐
    Monster* newMonster() { return new Monster(*this); }

    // 기존 코드...
};

class Monster
{
    friend class Breed;  // Breed만 Monster 생성 가능

public:
    const char* getAttack() { return breed_.getAttack(); }

private:
    Monster(Breed& breed)  // private 생성자
    : health_(breed.getHealth()),
      breed_(breed)
    {}

    int    health_;
    Breed& breed_;
};
```

**사용 방식 변화**:

```cpp
// 이전 방식
Monster* m = new Monster(dragonBreed);

// 팩토리 메서드 방식
Monster* m = dragonBreed.newMonster();
```

**팩토리 메서드의 장점**: 메모리 할당 전략(오브젝트 풀, 커스텀 할당자 등)을 `newMonster()` 안에서 제어할 수 있다. 모든 몬스터가 반드시 이 경로로만 생성되도록 강제할 수도 있다.

---

### 타입 객체에 상속 추가

수백 개의 트롤 변종 체력을 일일이 바꾸는 것은 고통스럽다. 타입 객체 간에도 상속을 지원할 수 있다.

#### 방법 A: 동적 위임 (Dynamic Delegation)

```cpp
class Breed
{
public:
    Breed(Breed* parent, int health, const char* attack)
    : parent_(parent),
      health_(health),
      attack_(attack)
    {}

    int getHealth()
    {
        // 재정의된 경우 자신의 값 반환, 아니면 부모에서 상속
        if (health_ != 0 || parent_ == NULL) return health_;
        return parent_->getHealth();
    }

    const char* getAttack()
    {
        if (attack_ != NULL || parent_ == NULL) return attack_;
        return parent_->getAttack();
    }

private:
    Breed*      parent_;
    int         health_;
    const char* attack_;
};
```

- **장점**: 런타임에 부모 속성이 변경되면 자동 반영
- **단점**: 속성 조회마다 상속 체인 순회 필요 → 느림

#### 방법 B: 복사-다운 위임 (Copy-Down Delegation, 권장)

```cpp
class Breed
{
public:
    Breed(Breed* parent, int health, const char* attack)
    : health_(health),
      attack_(attack)
    {
        // 생성 시점에 부모 속성 복사
        if (parent != NULL)
        {
            if (health_ == 0)    health_ = parent->getHealth();
            if (attack_ == NULL) attack_ = parent->getAttack();
        }
        // parent_ 필드 불필요 — 생성 후 버림
    }

    int         getHealth() { return health_; }  // 직접 반환, 빠름
    const char* getAttack() { return attack_; }

private:
    int         health_;
    const char* attack_;
};
```

- **장점**: 속성 조회가 단순한 필드 읽기로 최적화됨
- **단점**: 런타임에 부모 속성이 변경되어도 자식에 미반영

---

### 데이터 파일에서 종족 정의

```json
{
    "Troll": {
        "health": 25,
        "attack": "The troll hits you!"
    },
    "Troll Archer": {
        "parent": "Troll",
        "health": 0,
        "attack": "The troll archer fires an arrow!"
    },
    "Troll Wizard": {
        "parent": "Troll",
        "health": 0,
        "attack": "The troll wizard casts a spell on you!"
    }
}
```

`health: 0`은 "부모에서 상속"을 의미한다. `Troll`의 체력을 25에서 30으로 바꾸면 `Troll Archer`와 `Troll Wizard`도 자동으로 업데이트된다. **디자이너가 프로그래머 없이 직접 몬스터를 만들 수 있다.**

---

## 상속 방식 비교

```
클래스 기반 상속                     타입 객체 상속
─────────────────────────────        ─────────────────────────────
컴파일 타임에 고정                    런타임에 데이터로 정의
새 종류 → 새 파일 + 재컴파일          새 종류 → JSON 한 줄 추가
디자이너 직접 수정 불가               디자이너가 직접 수정 가능
C++ vtable (자동 관리)               Breed 객체 (수동 관리)
타입별 코드 로직 가능                 타입별 코드는 어렵고 데이터만 쉬움
```

> C++의 vtable은 사실 타입 객체 패턴이 C에 적용된 것을 컴파일러가 자동으로 처리하는 것이다.

---

## 설계 결정 (Design Decisions)

### 1. 타입 객체를 캡슐화할 것인가, 노출할 것인가?

| | 캡슐화 (Monster가 Breed를 숨김) | 노출 (getBreed() 제공) |
|---|---|---|
| **장점** | 구현 세부 사항 은닉. 개별 동작 재정의 가능 | 외부에서 Breed에 직접 접근 가능. 팩토리 메서드 사용 편리 |
| **단점** | 위임 메서드를 일일이 작성해야 함 | Monster의 공개 API가 넓어짐 |

**캡슐화 시 개별 재정의 예시** (체력이 낮을 때 공격 문자열 변경):

```cpp
const char* Monster::getAttack()
{
    if (health_ < LOW_HEALTH)
        return "The monster flails weakly.";  // 인스턴스별 override

    return breed_.getAttack();  // 기본은 종족에 위임
}
```

---

### 2. 타입화된 객체를 어떻게 생성하는가?

| 방식 | 특징 |
|---|---|
| `new Monster(breed)` | 호출 측이 메모리 할당 제어. 유연하지만 메모리 전략 강제 불가 |
| `breed.newMonster()` | 타입 객체가 메모리 할당 제어. Object Pool 등과 연동하기 좋음 |

---

### 3. 타입이 변경될 수 있는가?

| | 타입 고정 | 타입 변경 가능 |
|---|---|---|
| **장점** | 단순하고 추론 쉬움. 디버깅 편의 | 새 객체 생성 비용 절약. 몬스터 사망 → 좀비 변환 등 표현 가능 |
| **단점** | 타입 전환 시 새 객체 생성 필요 | 새 타입의 불변 조건 만족 여부 검증 필요 |

**타입 변경 예시** (사망한 몬스터가 좀비로 변환):

```cpp
// 방법 1: 새 객체 생성 (타입 고정)
Monster* zombie = new Monster(zombieBreed);
// 기존 monster의 속성 복사 후 삭제...

// 방법 2: 타입 변경 (타입 변경 가능)
monster.setBreed(zombieBreed);  // 간단하지만 검증 필요
```

---

### 4. 어떤 종류의 상속을 지원하는가?

| 방식 | 장점 | 단점 |
|---|---|---|
| **상속 없음** | 단순, 구현 쉬움 | 데이터 중복 많음 (50개 엘프 종류 체력을 50번 수정) |
| **단일 상속** | 구현과 이해 모두 쉬움. 강력함과 단순함의 균형 | 체인 순회로 조회 느림 (copy-down으로 해결 가능) |
| **다중 상속** | 데이터 중복 최소화 | 복잡. 다이아몬드 상속 문제. 실제로는 피하는 것이 낫다 |

---

## 주의사항 (Keep in Mind)

### 타입 객체를 수동으로 관리해야 한다

컴파일러는 C++ 클래스 메타데이터를 자동으로 관리한다. 타입 객체 패턴을 쓰면 Breed 인스턴스의 생명주기를 직접 관리해야 한다. 몬스터가 살아있는 동안 해당 Breed 객체도 반드시 살아있어야 한다.

### 타입별 행동 정의가 어렵다

데이터(체력, 공격 문자열)는 쉽게 타입 객체로 이전할 수 있지만, **코드 로직(알고리즘, AI)** 은 이전하기 어렵다.

**해결책**:

```cpp
// 방법 1: 사전 정의된 행동 중 선택
enum AIBehavior { STAND_STILL, CHASE_HERO, FLEE };

class Breed
{
    AIBehavior ai_;  // 행동 종류를 데이터로 저장
    // AI 로직은 코드, 선택만 데이터
};

// 방법 2: 함수 포인터 (실질적으로 vtable 재구현)
class Breed
{
    void (*updateAI_)(Monster&);  // 함수 포인터
};

// 방법 3: Bytecode / Interpreter 패턴으로 행동 자체를 데이터화
```

---

## 전체 구조 요약

```
데이터 파일 (JSON/CSV)
         │
         ▼
┌────────────────────┐
│   Breed Registry   │  ← 모든 Breed 인스턴스 관리
│                    │
│  "Dragon" → Breed  │
│  "Troll"  → Breed  │
│  "Zombie" → Breed  │
└────────────────────┘
         │
         │  참조
         ▼
┌────────────────────────────────────────────┐
│  Monster instances (게임 월드)             │
│                                            │
│  [dragon1] → Breed("Dragon", 230, "...")  │
│  [troll1]  → Breed("Troll",   48, "...")  │
│  [troll2]  → Breed("Troll",   48, "...")  │
│  [troll3]  → Breed("Troll",   48, "...")  │
└────────────────────────────────────────────┘
```

---

## 핵심 정리

| 개념 | 설명 |
|---|---|
| **타입 객체 (Type Object)** | 논리적 "타입"을 표현하는 객체. 같은 종류의 인스턴스들이 공유하는 데이터/행동 포함 |
| **타입화된 객체 (Typed Object)** | 타입 객체를 참조하는 일반 인스턴스. 인스턴스별 고유 상태 보유 |
| **복사-다운 위임** | 생성 시점에 부모 속성을 복사해 두어 런타임 체인 순회를 피하는 최적화 기법 |
| **팩토리 메서드** | 타입 객체가 직접 인스턴스를 생성하는 패턴. 메모리 할당 전략 캡슐화 가능 |
| **vtable과의 관계** | C++ vtable은 컴파일러가 자동 관리하는 타입 객체 패턴이다 |

---

## 관련 패턴 (See Also)

| 패턴 | 관계 |
|---|---|
| **Flyweight** | 밀접한 사촌. 둘 다 인스턴스 간 데이터 공유. Flyweight는 메모리 절약이 목적, Type Object는 조직화와 유연성이 목적 |
| **Prototype (GoF)** | 같은 고수준 문제(여러 객체 간 데이터/행동 공유)를 다른 방식으로 해결 |
| **State (GoF)** | 유사한 구조. 둘 다 객체가 자신의 일부를 다른 객체에 위임. State는 현재 상태(일시적, 변경 가능), Type Object는 본질적 타입(불변에 가까움)을 나타냄 |
| **Bytecode / Interpreter** | 타입 객체에 코드 로직을 넣어야 할 때 행동을 데이터로 표현하는 수단 |
| **Object Pool** | 팩토리 메서드를 통해 Object Pool과 결합하여 메모리 관리 최적화 |

---

*© 2009-2015 Robert Nystrom*
