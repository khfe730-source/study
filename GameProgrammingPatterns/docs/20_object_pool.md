# 19장. 오브젝트 풀 (Object Pool)

> **분류**: 최적화 패턴 (Optimization Patterns)

---

## 의도 (Intent)

오브젝트를 개별적으로 할당하고 해제하는 대신, 고정된 풀(pool)에서 재사용함으로써 성능과 메모리 사용을 개선한다.

---

## 동기 (Motivation)

### 파편화의 저주 (The Curse of Fragmentation)

파티클 시스템처럼 오브젝트를 빈번하게 생성/소멸하는 시스템을 구현할 때, 힙(heap) 할당을 반복하면 **메모리 파편화(memory fragmentation)** 문제가 발생한다.

```
힙 메모리 파편화 예시

초기 상태 (14바이트 여유)
┌────────────────────────────────┐
│ [사용 중] [사용 중] [사용 중]    │
│                ↓ 파티클 생성/소멸 반복
└────────────────────────────────┘

파편화 후 (14바이트 여유지만...)
┌─────────────────────────────────────────┐
│[쓰임][7B 빈공간][쓰임][7B 빈공간][쓰임]  │
│                                         │
│  → 12바이트 연속 블록 할당 시도 → 실패!  │
│    총 여유는 14바이트지만, 연속된 12바이트 없음
└─────────────────────────────────────────┘
```

- 콘솔/모바일 환경에서 메모리는 희소하고, 컴팩팅 GC는 사용 불가
- 파편화가 누적되면 결국 게임 전체가 다운된다
- 소크 테스트(soak test, 수일간 연속 실행)에서 크래시 원인의 상당수가 파편화

### 두 마리 토끼

**오브젝트 풀**은 두 요구를 동시에 충족한다:

- 메모리 관리자 입장: 게임 시작 시 큰 메모리 덩어리를 한 번만 할당, 게임 종료까지 해제 없음 → 파편화 없음
- 사용자 입장: 마음껏 오브젝트를 "생성"하고 "소멸"할 수 있음 → 편의성 유지

---

## 패턴 요약

재사용 가능한 오브젝트 컬렉션을 관리하는 풀 클래스를 정의한다. 각 오브젝트는 현재 "사용 중"인지 쿼리할 수 있다. 풀이 초기화될 때 전체 오브젝트 컬렉션을 미리 생성하고(보통 단일 연속 할당으로), 모두 "미사용" 상태로 초기화한다.

새 오브젝트가 필요하면 풀에 요청한다. 풀은 사용 가능한 오브젝트를 찾아 "사용 중"으로 초기화한 후 반환한다. 오브젝트가 더 이상 필요 없으면 "미사용" 상태로 되돌린다. 이렇게 하면 메모리 할당/해제 없이 오브젝트를 자유롭게 생성/소멸할 수 있다.

---

## 언제 사용하는가

- 오브젝트를 빈번하게 생성/소멸해야 할 때
- 오브젝트들의 크기가 비슷할 때
- 힙 할당이 느리거나 메모리 파편화를 유발할 때
- 각 오브젝트가 DB 연결, 네트워크 소켓 같은 비싼 리소스를 캡슐화하고 있을 때

---

## 주의사항 (Keep in Mind)

오브젝트 풀을 사용하면 메모리 관리를 직접 책임지게 된다. 다음 제약사항을 숙지해야 한다.

### 1. 필요 없는 오브젝트에 메모리를 낭비할 수 있다

풀 크기가 너무 작으면 크래시가 발생하고, 너무 크면 다른 용도로 쓸 수 있는 메모리를 낭비한다. 게임 상황에 맞게 풀 크기를 튜닝해야 한다.

### 2. 한 번에 활성화할 수 있는 오브젝트 수가 고정된다

풀이 가득 찼을 때 새 오브젝트를 요청하면 실패한다. 대응 전략:

| 전략 | 설명 | 적합한 경우 |
|------|------|------------|
| **미리 방지** | 풀 크기를 충분히 크게 튜닝 | 보스 같은 중요한 오브젝트 |
| **그냥 생성 안 함** | 요청을 조용히 무시 | 파티클처럼 없어도 눈치채기 어려운 경우 |
| **기존 오브젝트 강제 종료** | 가장 덜 중요한 기존 오브젝트를 교체 | 사운드 풀 (가장 작은 볼륨을 교체) |
| **런타임에 풀 확장** | 메모리 유연성이 있을 때 추가 할당 | 플렉서블한 메모리 환경 |

### 3. 각 오브젝트의 메모리 크기가 고정된다

서브클래스나 다양한 크기의 오브젝트를 풀에 담으려면 슬롯 크기를 가장 큰 오브젝트 기준으로 잡아야 한다. 작은 오브젝트를 넣으면 메모리가 낭비된다. 크기 범주별로 풀을 분리하는 것이 일반적인 해결책이다.

### 4. 재사용된 오브젝트는 자동으로 초기화되지 않는다

일반 메모리 관리자는 할당/해제 시 `0xdeadbeef` 같은 값으로 메모리를 채워서 미초기화 버그를 쉽게 발견할 수 있게 한다. 오브젝트 풀은 이 기능이 없다. 게다가 재사용된 슬롯에는 이전 오브젝트의 데이터가 그대로 남아있어, 초기화를 빠뜨렸을 때 "거의 맞는 데이터"처럼 보여 버그를 발견하기 더 어렵다.

**반드시** 오브젝트를 재활용할 때 완전히 초기화하는 코드를 작성할 것. 디버그 빌드에서 반환된 슬롯의 메모리를 의도적으로 클리어하는 기능을 추가하는 것도 좋다.

### 5. GC 환경에서 참조 누수 주의

GC 언어에서 풀을 사용할 때, "미사용" 상태가 된 오브젝트가 다른 오브젝트에 대한 참조를 여전히 보유하고 있으면 GC가 그 오브젝트를 수거하지 못한다. 오브젝트를 반환할 때 참조를 명시적으로 null로 설정해야 한다.

---

## 샘플 코드 (Sample Code)

### 기본 구현

#### Particle 클래스

```cpp
class Particle
{
public:
    Particle()
    : framesLeft_(0)
    {}

    void init(double x, double y,
              double xVel, double yVel, int lifetime)
    {
        x_ = x; y_ = y;
        xVel_ = xVel; yVel_ = yVel;
        framesLeft_ = lifetime;
    }

    void animate()
    {
        if (!inUse()) return;
        framesLeft_--;
        x_ += xVel_;
        y_ += yVel_;
    }

    bool inUse() const { return framesLeft_ > 0; }

private:
    int    framesLeft_;
    double x_, y_;
    double xVel_, yVel_;
};
```

- `framesLeft_ > 0`이면 살아있는 파티클
- 별도의 `isActive_` 플래그 없이 기존 상태 변수를 재활용

#### ParticlePool 클래스 (단순 버전)

```cpp
class ParticlePool
{
public:
    void create(double x, double y,
                double xVel, double yVel, int lifetime)
    {
        // ❌ O(n): 빈 슬롯을 선형 탐색
        for (int i = 0; i < POOL_SIZE; i++)
        {
            if (!particles_[i].inUse())
            {
                particles_[i].init(x, y, xVel, yVel, lifetime);
                return;
            }
        }
        // 풀이 가득 찼으면 그냥 무시 (파티클이므로)
    }

    void animate()
    {
        for (int i = 0; i < POOL_SIZE; i++)
        {
            particles_[i].animate();
        }
    }

private:
    static const int POOL_SIZE = 100;
    Particle particles_[POOL_SIZE];
};
```

```
메모리 레이아웃 (연속 배열)
┌────┬────┬────┬────┬────┬────┬────┬────┐
│ P0 │ P1 │ P2 │ P3 │ P4 │ P5 │ P6 │...│
└────┴────┴────┴────┴────┴────┴────┴────┘
  사용  미사  사용  사용  미사  사용  미사
  중    용    중    중    용    중    용
```

단순하지만 `create()`의 복잡도가 **O(n)**이다. 풀이 크고 거의 가득 찬 경우 성능 문제가 생긴다.

---

### 프리 리스트(Free List)를 이용한 O(1) 최적화

미사용 오브젝트들을 **연결 리스트**로 연결해두면 빈 슬롯을 O(1)에 찾을 수 있다. 핵심은 별도 메모리를 쓰지 않고 **죽은 파티클의 메모리 공간 자체를 포인터로 재활용**하는 것이다.

#### union을 활용한 Particle 수정

```cpp
class Particle
{
public:
    Particle* getNext() const { return state_.next; }
    void setNext(Particle* next) { state_.next = next; }

    bool inUse() const { return framesLeft_ > 0; }

    bool animate()
    {
        if (!inUse()) return false;
        framesLeft_--;
        x_ += state_.live.xVel;
        y_ += state_.live.yVel;
        return framesLeft_ == 0;  // true이면 이번 프레임에 사망
    }

    void init(double x, double y,
              double xVel, double yVel, int lifetime)
    {
        x_ = x; y_ = y;
        state_.live.xVel = xVel;
        state_.live.yVel = yVel;
        framesLeft_ = lifetime;
    }

private:
    int framesLeft_;
    double x_, y_;

    union
    {
        // 살아있을 때 사용되는 상태
        struct
        {
            double xVel, yVel;
        } live;

        // 죽어있을 때: 다음 빈 파티클을 가리키는 포인터
        Particle* next;
    } state_;
};
```

`union`을 사용해 동일한 메모리 공간을 두 가지 용도로 쓴다:

```
파티클이 살아있을 때 (framesLeft_ > 0)
┌──────────────────────────────────────────┐
│ framesLeft_ │  x_  │  y_  │ xVel │ yVel │
└──────────────────────────────────────────┘

파티클이 죽었을 때 (framesLeft_ == 0)
┌──────────────────────────────────────────┐
│ framesLeft_ │  x_  │  y_  │    *next     │
└──────────────────────────────────────────┘
                              ↑ 다음 빈 슬롯 포인터
                                (xVel/yVel 공간 재활용)
```

#### ParticlePool (프리 리스트 버전)

```cpp
class ParticlePool
{
public:
    ParticlePool()
    {
        // 초기화: 모든 파티클을 프리 리스트로 연결
        firstAvailable_ = &particles_[0];
        for (int i = 0; i < POOL_SIZE - 1; i++)
        {
            particles_[i].setNext(&particles_[i + 1]);
        }
        particles_[POOL_SIZE - 1].setNext(NULL);
    }

    void create(double x, double y,
                double xVel, double yVel, int lifetime)
    {
        assert(firstAvailable_ != NULL);  // 풀이 가득 찼을 때 처리

        // ✅ O(1): 프리 리스트 헤드에서 즉시 꺼냄
        Particle* newParticle = firstAvailable_;
        firstAvailable_ = newParticle->getNext();

        newParticle->init(x, y, xVel, yVel, lifetime);
    }

    void animate()
    {
        for (int i = 0; i < POOL_SIZE; i++)
        {
            if (particles_[i].animate())
            {
                // 파티클이 이번 프레임에 사망 → 프리 리스트 앞에 추가
                particles_[i].setNext(firstAvailable_);
                firstAvailable_ = &particles_[i];
            }
        }
    }

private:
    static const int POOL_SIZE = 100;
    Particle  particles_[POOL_SIZE];
    Particle* firstAvailable_;
};
```

프리 리스트 동작 시각화:

```
초기 상태 (모두 미사용)
firstAvailable_ → [P0] → [P1] → [P2] → [P3] → ... → [P99] → NULL

P0, P1 create() 후
firstAvailable_ → [P2] → [P3] → ... → [P99] → NULL
[P0]: 사용 중 (살아있음)
[P1]: 사용 중 (살아있음)

P0 사망 후 (animate()에서 반환)
firstAvailable_ → [P0] → [P2] → [P3] → ... → [P99] → NULL
[P0]: 미사용 (next = &P2)
[P1]: 사용 중
```

---

## 설계 결정 (Design Decisions)

### 오브젝트가 풀에 결합되어 있는가?

#### 오브젝트가 풀을 안다 (Coupled)

```cpp
class Particle
{
    friend class ParticlePool;  // 풀만 생성 가능하도록

private:
    Particle() : inUse_(false) {}
    bool inUse_;
};

class ParticlePool
{
    Particle pool_[100];
};
```

- **장점**: 구현 단순. 오브젝트를 풀 밖에서 생성하는 것을 방지 가능. 기존 상태를 `inUse()` 로직에 재활용 가능.
- **단점**: 특정 풀 구현에 오브젝트가 종속됨.

#### 오브젝트가 풀을 모른다 (Decoupled)

```cpp
template <class TObject>
class GenericPool
{
private:
    static const int POOL_SIZE = 100;

    TObject pool_[POOL_SIZE];
    bool    inUse_[POOL_SIZE];  // 사용 여부를 외부에서 별도 관리
};
```

- **장점**: 어떤 타입이든 풀링 가능. 범용 풀 클래스 구현 가능.
- **단점**: `inUse_` 배열을 별도로 유지해야 함 → 메모리 추가 사용.

---

### 누가 재사용 오브젝트를 초기화하는가?

#### 풀이 내부에서 초기화

```cpp
class ParticlePool
{
public:
    // 초기화 방식마다 오버로드 제공
    void create(double x, double y);
    void create(double x, double y, double angle);
    void create(double x, double y, double xVel, double yVel);
};
```

- **장점**: 오브젝트를 풀 안에 완전히 캡슐화 가능.
- **단점**: 오브젝트의 초기화 방식이 늘어날 때마다 풀의 인터페이스도 변경해야 함.

#### 외부 코드가 초기화

```cpp
class ParticlePool
{
public:
    Particle* create()
    {
        // 사용 가능한 파티클 포인터 반환
        // 풀이 가득 찼으면 NULL 반환
    }
};

// 사용 예
ParticlePool pool;
pool.create()->init(1, 2);
pool.create()->init(1, 2, 0.3);

// NULL 체크 필요
Particle* p = pool.create();
if (p != NULL) p->init(1, 2);
```

- **장점**: 풀의 인터페이스가 단순. 오브젝트의 초기화 방식 변경에 풀이 영향받지 않음.
- **단점**: 호출자가 NULL 체크를 직접 해야 함.

---

## Flyweight 패턴과의 비교

| 구분 | Flyweight | Object Pool |
|------|-----------|-------------|
| **재사용 방식** | 동시에 여러 소유자가 공유 | 시간을 두고 순차적으로 재사용 |
| **목적** | 중복 메모리 절약 | 할당/해제 비용 절감, 파편화 방지 |
| **동시 공유** | O | X |
| **예시** | 나무 메시 공유 | 파티클 풀, 사운드 인스턴스 풀 |

---

## 다른 패턴과의 관계

- **데이터 지역성 패턴**: 동일 타입 오브젝트를 연속된 배열에 담는 것은 캐시 친화적이다. 오브젝트 풀과 데이터 지역성은 자연스럽게 함께 사용된다.
- **업데이트 메서드 패턴**: 풀의 `animate()` 루프가 전형적인 업데이트 메서드 패턴이다.
- **더티 플래그 패턴**: 물리 엔진에서 sleep 상태 오브젝트를 풀처럼 관리하는 방식과 연관된다.

---

## 요약

```
Object Pool 핵심 구조
────────────────────────────────────────────────────────
                    ParticlePool
 ┌──────────────────────────────────────────────┐
 │ particles_[POOL_SIZE]  (연속 배열, 선할당)     │
 │ ┌────┬────┬────┬────┬────┬────┬────┬────┐   │
 │ │ P0 │ P1 │ P2 │ P3 │ P4 │ P5 │ P6 │...│   │
 │ └────┴────┴────┴────┴────┴────┴────┴────┘   │
 │  사용  미사  사용  사용  미사  사용  미사       │
 │  중    용    중    중    용    중    용         │
 │       ↑──────────────↑──────────↑            │
 │  firstAvailable_ 프리 리스트로 연결            │
 └──────────────────────────────────────────────┘

create()  → O(1): firstAvailable_에서 꺼냄
destroy() → O(1): firstAvailable_ 앞에 추가
animate() → O(n): 전체 순회 (사용 중인 것만 처리)

장점: 파편화 없음, O(1) 생성/소멸
단점: 최대 오브젝트 수 고정, 미초기화 버그 위험
────────────────────────────────────────────────────────
```
