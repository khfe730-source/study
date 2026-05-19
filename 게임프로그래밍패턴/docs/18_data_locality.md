# 17장. 데이터 지역성 (Data Locality)

> **분류**: 최적화 패턴 (Optimization Patterns)

---

## 의도 (Intent)

CPU 캐싱을 활용할 수 있도록 데이터를 배치하여 메모리 접근 속도를 높인다.

---

## 동기 (Motivation)

### 거짓말: "프로세서가 빨라지면 게임도 빨라진다"

무어의 법칙(Moore's Law)이 있다. CPU 속도는 해마다 급격히 상승했다. 그런데 하드웨어 엔지니어들이 말하지 않은 것이 있다. **데이터를 가져오는 속도는 전혀 따라오지 못했다**는 사실이다.

```
CPU 속도 / RAM 속도 (1980년 기준 상대치)
────────────────────────────────────────────────────────
1980  : CPU ████  RAM ████   (거의 동등)
1990  : CPU ████████████████████  RAM ██████
2000  : CPU ██████████████████████████████████████  RAM ████████
2010  : CPU ████████...수십 배...████████████████  RAM ██████████
────────────────────────────────────────────────────────
         CPU는 기하급수적 성장, RAM은 완만한 성장
```

오늘날의 하드웨어에서 RAM에서 바이트 하나를 가져오는 데 수백 사이클(cycle)이 걸린다. 그렇다면 CPU는 99% 시간을 데이터를 기다리며 낭비하는 걸까?

실제로 그렇다. 하지만 완충 장치가 있다. 그것이 바로 **CPU 캐시(CPU Cache)**.

---

### 창고 직원 비유 (Warehouse Analogy)

CPU 캐시를 이해하는 가장 좋은 비유다.

```
┌────────────────────────────────────────────────────────┐
│  상황: 당신은 회계사. 종이 상자를 처리하는 게 일.       │
│                                                        │
│  ┌──────────┐       (하루 소요)      ┌──────────────┐  │
│  │  당신의  │ ←─────────────────── │   창고 직원  │  │
│  │  사무실  │    상자 하나씩 배달    │   (지게차)   │  │
│  │ (CPU 레지스터) │                  │   (메모리 버스) │  │
│  └──────────┘                       └──────────────┘  │
│                                             ↑          │
│                                      ┌─────────────┐  │
│                                      │    창고      │  │
│                                      │    (RAM)     │  │
│                                      └─────────────┘  │
└────────────────────────────────────────────────────────┘
```

산업 설계자들이 해결책을 고안했다:
- 상자 하나 요청 시 **주변 상자 50개를 팔레트에 통째로** 가져온다
- 팔레트를 사무실 구석(L1 캐시)에 놓는다
- 다음 상자가 팔레트에 있으면 즉시 꺼낸다 → **캐시 히트(cache hit)**
- 없으면 다시 창고에 가야 한다 → **캐시 미스(cache miss)**

팔레트에 필요한 상자 50개가 모두 있다면? **50배 빠르게 일한다.**

---

### CPU 캐시 구조

```
┌─────────────────────────────────────────────────────┐
│                    CPU Chip                         │
│                                                     │
│  ┌──────────┐  ┌──────────┐  ┌──────────────────┐  │
│  │ Register │  │ L1 Cache │  │    L2 Cache      │  │
│  │ (레지스터) │  │  (매우빠름) │  │    (빠름)         │  │
│  │  <1ns    │  │  ~1ns    │  │    ~4ns          │  │
│  │  수십 바이트│  │  32KB    │  │    256KB         │  │
│  └──────────┘  └──────────┘  └──────────────────┘  │
└─────────────────────────────────────────────────────┘
         ↕ 캐시 미스 시 수백 사이클 지연
┌─────────────────────────────────────────────────────┐
│                L3 Cache / Main RAM                  │
│              ~100ns / 수백 사이클                      │
│              수 MB  /  GB 단위                        │
└─────────────────────────────────────────────────────┘
```

**캐시 라인(Cache Line)**: CPU가 RAM에서 데이터를 읽을 때 한 번에 가져오는 연속 메모리 블록. 보통 **64~128바이트**.

캐시 미스가 발생하면 CPU는 **멈춘다(stall)**. 이는 코드에 `sleepFor500Cycles()`를 삽입한 것과 같다:

```cpp
// 캐시 미스가 많은 코드가 하는 짓과 동일
for (int i = 0; i < NUM_THINGS; i++)
{
    sleepFor500Cycles();  // ← 캐시 미스 비용
    things[i].doStuff();
}
```

---

### 데이터가 곧 성능이다

저자는 동일한 계산을 수행하는 두 프로그램을 만들었다. 차이는 오직 캐시 미스 빈도였다. 결과는 충격적이었다:

> **느린 버전은 빠른 버전보다 50배 느렸다.**

데이터의 메모리 배치 방식이 직접 성능에 영향을 준다. 핵심 원칙:

> **처리하는 데이터들을 메모리상에서 연속적으로 배치하라.**

---

## 패턴 요약

현대 CPU는 메모리 접근 속도를 높이기 위해 캐시를 사용한다. 캐시는 최근에 접근한 메모리 주변을 빠르게 접근할 수 있다. **데이터 지역성(data locality)**을 높임으로써 이를 활용하라. 즉, 처리하는 순서와 동일하게 데이터를 연속된 메모리에 배치하라.

---

## 언제 사용하는가

- 성능 문제가 실제로 존재할 때 (먼저 프로파일링할 것)
- 성능 문제의 원인이 **캐시 미스**임을 확인했을 때
- **Cachegrind** 같은 캐시 프로파일러로 미스 위치를 특정한 뒤 적용한다

빈번하지 않은 코드에 이 최적화를 적용하지 마라. 결과물은 거의 항상 더 복잡하고 유연성이 떨어진다.

---

## 주의사항

이 패턴은 객체지향 설계의 추상화와 근본적으로 충돌한다. 인터페이스/가상 함수를 사용하면 포인터 체이싱(pointer chasing)이 발생하고, 이는 캐시 미스로 이어진다.

- 인터페이스 → 포인터 → 캐시 미스
- 가상 함수 → vtable 조회 → 포인터 → 캐시 미스

**데이터 지역성을 추구할수록 상속과 인터페이스를 포기해야 한다**. 은탄환(silver bullet)은 없다. 트레이드오프를 의식적으로 결정해야 한다.

---

## 샘플 코드 (Sample Code)

### 1. 연속 배열 (Contiguous Arrays)

#### 문제: 포인터 체이싱

게임 엔티티가 AI/Physics/Render 컴포넌트에 대한 포인터를 보유하는 전형적인 구조:

```cpp
class GameEntity
{
public:
    GameEntity(AIComponent* ai,
               PhysicsComponent* physics,
               RenderComponent* render)
    : ai_(ai), physics_(physics), render_(render)
    {}

    AIComponent*      ai()      { return ai_; }
    PhysicsComponent* physics() { return physics_; }
    RenderComponent*  render()  { return render_; }

private:
    AIComponent*      ai_;
    PhysicsComponent* physics_;
    RenderComponent*  render_;
};
```

게임 루프에서의 사용:

```cpp
// ❌ 나쁜 예: 포인터 체이싱
while (!gameOver)
{
    // AI 처리
    for (int i = 0; i < numEntities; i++)
    {
        entities[i]->ai()->update();
        //         ↑           ↑
        //   캐시 미스 1    캐시 미스 2
    }

    // 물리 처리
    for (int i = 0; i < numEntities; i++)
    {
        entities[i]->physics()->update();  // 역시 2번의 캐시 미스
    }

    // 렌더링
    for (int i = 0; i < numEntities; i++)
    {
        entities[i]->render()->render();   // 역시 2번의 캐시 미스
    }
}
```

메모리 레이아웃 시각화:

```
엔티티 배열 (포인터들)     힙 메모리 (무작위 위치)
┌─────────────────┐       ┌────┐  ┌────┐
│ *entity[0]  ────────→   │AI  │  │Phy │  ← 뿔뿔이 흩어져 있음
│ *entity[1]  ─────────────────────→│Rndr│
│ *entity[2]  ──────────────────────────→│AI  │
│    ...      │       ↑                  ↑
└─────────────┘    각 역참조마다 캐시 미스 발생
```

#### 해결: 타입별 연속 배열

```cpp
// ✅ 좋은 예: 타입별 연속 배열
AIComponent*      aiComponents =
    new AIComponent[MAX_ENTITIES];
PhysicsComponent* physicsComponents =
    new PhysicsComponent[MAX_ENTITIES];
RenderComponent*  renderComponents =
    new RenderComponent[MAX_ENTITIES];
```

게임 루프:

```cpp
while (!gameOver)
{
    // AI 처리 — 순차적 메모리 접근
    for (int i = 0; i < numEntities; i++)
    {
        aiComponents[i].update();        // ← 포인터 역참조 없음!
    }

    // 물리 처리
    for (int i = 0; i < numEntities; i++)
    {
        physicsComponents[i].update();
    }

    // 렌더링
    for (int i = 0; i < numEntities; i++)
    {
        renderComponents[i].render();
    }
}
```

메모리 레이아웃 시각화:

```
AI 컴포넌트 배열 (연속)
┌────┬────┬────┬────┬────┬────┐
│AI 0│AI 1│AI 2│AI 3│AI 4│... │  ← 캐시 라인이 순서대로 채워짐
└────┴────┴────┴────┴────┴────┘
  ↑ CPU가 AI[0]을 읽으면 AI[1]~AI[7]도 캐시에 자동으로 올라옴

물리 컴포넌트 배열 (연속)
┌────┬────┬────┬────┬────┬────┐
│Phy0│Phy1│Phy2│Phy3│Phy4│... │
└────┴────┴────┴────┴────┴────┘
```

> `->` 연산자가 적을수록 캐시 친화적이다. 코드에서 역참조 연산자가 보이면 캐시 미스를 의심하라.

**결과: 50배 성능 향상** (저자 직접 측정)

---

### 2. 빽빽한 데이터 (Packed Data)

파티클 시스템에서 활성/비활성 파티클을 효율적으로 처리하는 기법이다.

#### 문제: 비활성 파티클 건너뛰기

```cpp
class Particle
{
public:
    void update() { /* 중력, 바람 등 */ }
    bool isActive() const { return isActive_; }
    // 위치, 속도 등...
private:
    bool isActive_;
};

class ParticleSystem
{
public:
    void update()
    {
        // ❌ 나쁜 예: 비활성 파티클도 캐시에 올라옴
        for (int i = 0; i < numParticles_; i++)
        {
            if (particles_[i].isActive())
            {
                particles_[i].update();
            }
        }
        // 추가 문제: 분기 예측 실패(branch misprediction) → 파이프라인 플러시
    }
private:
    static const int MAX_PARTICLES = 100000;
    int numParticles_;
    Particle particles_[MAX_PARTICLES];
};
```

메모리 상황:

```
particles_ 배열
┌──────┬──────┬──────┬──────┬──────┬──────┬──────┐
│ 활성 │ 비활성│ 비활성│ 활성 │ 비활성│ 활성 │ 비활성│
└──────┴──────┴──────┴──────┴──────┴──────┴──────┘
  ↑처리  ↑건너뜀 ↑건너뜀  ↑처리  ↑건너뜀  ↑처리  ↑건너뜀
캐시에 올라온 데이터 대부분이 낭비됨
```

#### 해결: 활성 상태로 정렬 유지

```cpp
class ParticleSystem
{
public:
    ParticleSystem() : numActive_(0) {}

    void update()
    {
        // ✅ 활성 파티클만 연속적으로 처리
        for (int i = 0; i < numActive_; i++)
        {
            particles_[i].update();
        }
    }

    void activateParticle(int index)
    {
        assert(index >= numActive_);

        // 비활성 영역의 첫 번째 슬롯과 교체
        Particle temp        = particles_[numActive_];
        particles_[numActive_] = particles_[index];
        particles_[index]    = temp;

        numActive_++;
    }

    void deactivateParticle(int index)
    {
        assert(index < numActive_);

        numActive_--;

        // 활성 영역의 마지막 슬롯과 교체
        Particle temp           = particles_[numActive_];
        particles_[numActive_]  = particles_[index];
        particles_[index]       = temp;
    }

private:
    static const int MAX_PARTICLES = 100000;
    int   numActive_;
    Particle particles_[MAX_PARTICLES];
};
```

메모리 레이아웃 (정렬 후):

```
particles_ 배열
┌──────┬──────┬──────┬──────┬──────┬──────┬──────┐
│ 활성 │ 활성 │ 활성 │ 비활성│ 비활성│ 비활성│ 비활성│
└──────┴──────┴──────┴──────┴──────┴──────┴──────┘
 ←── numActive_ = 3 ──→  ←── 처리 안 함 ──────────→

CPU가 캐시에 올린 모든 데이터가 실제로 사용됨!
```

**부가적 장점**: 배열 위치로 활성 여부를 판단할 수 있으므로, `isActive_` 플래그 필드 자체를 없앨 수 있다. 파티클 객체가 더 작아지고 → 캐시 라인당 더 많은 파티클 → 더 빠른 처리.

> 포인터 재할당 대신 데이터를 메모리에서 이동시키는 것이 오히려 빠를 수 있다. 직관과 반대되지만, 프로파일링이 항상 옳다.

---

### 3. 핫/콜드 분리 (Hot/Cold Splitting)

매 프레임 사용하는 데이터(hot)와 가끔 사용하는 데이터(cold)를 분리한다.

#### 문제: 드물게 쓰는 데이터가 캐시를 낭비

```cpp
// ❌ 모든 데이터가 하나의 클래스에
class AIComponent
{
public:
    void update() { /* 매 프레임 animation_, energy_, goalPos_ 사용 */ }

private:
    // ─── HOT: 매 프레임 접근 ───
    Animation* animation_;
    double     energy_;
    Vector     goalPos_;

    // ─── COLD: 죽을 때 한 번만 사용 ───
    LootType   drop_;
    int        minDrops_;
    int        maxDrops_;
    double     chanceOfDrop_;
    // cold 데이터가 캐시 라인을 차지해 hot 데이터 밀어냄
};
```

#### 해결: 핫/콜드 구조체 분리

```cpp
// ✅ Hot 데이터: 매 프레임 캐시에 올라옴
class AIComponent
{
public:
    void update() { /* animation_, energy_, goalPos_ 접근 */ }

private:
    Animation* animation_;    // hot
    double     energy_;       // hot
    Vector     goalPos_;      // hot

    LootDrop*  loot_;         // cold 데이터를 가리키는 포인터 하나만
};

// Cold 데이터: 거의 안 쓰임 (죽을 때만)
class LootDrop
{
    friend class AIComponent;
    LootType drop_;
    int      minDrops_;
    int      maxDrops_;
    double   chanceOfDrop_;
};
```

메모리 레이아웃:

```
AI 컴포넌트 배열 (연속, 작아짐)          LootDrop 배열 (어딘가)
┌──────────────────────┐               ┌─────────────────────┐
│ anim│enrgy│goalPos│* ─────────────→  │ drop│min│max│chance │
│ anim│enrgy│goalPos│* ─────────────→  │ drop│min│max│chance │
│ anim│enrgy│goalPos│* ─────────────→  │ drop│min│max│chance │
└──────────────────────┘               └─────────────────────┘
 ↑ 캐시 라인에 더 많이 들어감              ↑ 평소에는 캐시에 없어도 됨
```

> 핫/콜드 경계는 "매 프레임 접근하는가?" 가 기준이다. 명확하지 않을 때는 블랙 아트에 가깝다. 프로파일러로 측정하며 결정하라.

---

## 설계 결정 (Design Decisions)

### 다형성(Polymorphism)을 어떻게 처리할 것인가?

연속 배열은 동일 크기의 동종(homogenous) 객체를 전제한다. 다형성과 충돌한다.

#### 옵션 A: 다형성을 사용하지 않는다

```cpp
// 단일 구체 타입만 사용
class AIComponent { void update(); };
// 서브클래스 없음
```

- **장점**: 안전하고 빠름. 모든 객체가 동일 크기. 동적 디스패치 비용 없음.
- **단점**: 유연성 없음. 엔티티별 다른 동작을 구현하기 어려움.
- **참고**: Type Object 패턴으로 상속 없이 유연성을 얻을 수 있다.

#### 옵션 B: 타입별 분리 배열

```cpp
// 타입마다 별도 배열
SoldierAI    soldiers[MAX_SOLDIERS];
WizardAI     wizards[MAX_WIZARDS];
BossAI       bosses[MAX_BOSSES];

// 각 배열을 별도 루프로 처리
for (auto& s : soldiers) s.update();
for (auto& w : wizards)  w.update();
for (auto& b : bosses)   b.update();
```

- **장점**: 타입별로 빽빽하게 패킹됨. 정적 디스패치 가능.
- **단점**: 타입 종류가 많아지면 관리 부담. 개방/폐쇄 원칙(OCP) 위반.

#### 옵션 C: 포인터 배열 (기존 방식)

```cpp
BaseAI* aiList[MAX_ENTITIES];  // 포인터 배열
for (int i = 0; i < numEntities; i++)
    aiList[i]->update();        // 가상 함수 + 포인터 역참조 = 캐시 미스
```

- **장점**: 완전한 다형성. 개방성.
- **단점**: 캐시 비친화적. 성능이 중요하지 않은 곳에서만 사용.

---

### 게임 엔티티를 어떻게 표현할 것인가?

컴포넌트 패턴과 데이터 지역성 패턴을 함께 사용할 때 핵심 질문이다.

#### 방식 1: 컴포넌트 포인터를 가진 클래스

```cpp
class GameEntity
{
    AIComponent*      ai_;
    PhysicsComponent* physics_;
    RenderComponent*  render_;
};
```

```
엔티티          컴포넌트 배열
┌─────────┐    ┌────┬────┬────┬────┐
│ *ai  ───────→│AI 0│AI 1│AI 2│... │
│ *phy ───────────────→│Phy1│Phy2│... │
│ *rnd ────────────────────→│Rnd2│... │
└─────────┘    └────┴────┴────┴────┘
```

- 컴포넌트는 연속 배열에 저장 가능
- 엔티티 → 컴포넌트 접근이 포인터 하나
- **단점**: 컴포넌트를 배열에서 이동시키면 엔티티의 포인터가 깨진다

#### 방식 2: 컴포넌트 ID를 가진 클래스

```cpp
class GameEntity
{
    int aiId_;       // 컴포넌트 배열의 인덱스 또는 고유 ID
    int physicsId_;
    int renderId_;
};
```

- 컴포넌트를 자유롭게 배열 내에서 이동 가능
- **단점**: ID→컴포넌트 조회 비용 추가 (해시 테이블 등)

#### 방식 3: 엔티티 = 순수 ID (ECS 방식)

```cpp
// 엔티티는 그냥 숫자
using EntityId = uint32_t;

// 각 컴포넌트가 자신의 엔티티 ID를 알고 있음
class AIComponent
{
    EntityId ownerId_;  // 어느 엔티티 소속인지
    // ...
};

// 컴포넌트 배열이 동일 인덱스 = 동일 엔티티
AIComponent      aiComponents[MAX];     // aiComponents[3]
PhysicsComponent physicsComponents[MAX]; // physicsComponents[3]  } 같은 엔티티
RenderComponent  renderComponents[MAX];  // renderComponents[3]  }
```

```
인덱스  0      1      2      3      4
AI    [AI_0] [AI_1] [AI_2] [AI_3] [AI_4]
Phy   [Phy0] [Phy1] [Phy2] [Phy3] [Phy4]
Rnd   [Rnd0] [Rnd1] [Rnd2] [Rnd3] [Rnd4]
              ↑ 같은 인덱스의 컴포넌트들이 동일 엔티티를 구성
```

- 엔티티 객체 자체가 사라짐. 단순 숫자.
- **장점**: 엔티티는 경량. 생명주기 관리 불필요 (컴포넌트가 모두 파괴되면 엔티티도 소멸).
- **단점**: 컴포넌트 간 상호작용 시 ID 조회 필요. 배열 정렬이 어려워짐 (물리/렌더를 서로 다른 기준으로 정렬 불가).
- **주의**: 배열을 서로 다른 기준으로 정렬하면 인덱스 대응이 깨진다. 일부 엔티티에 물리가 없거나 렌더가 없는 경우도 처리해야 한다.

---

## 전체 구조 요약

```
Data Locality 패턴 핵심 기법
─────────────────────────────────────────────────
┌───────────────────┐
│  문제: 캐시 미스   │
│  (포인터 체이싱)   │
└────────┬──────────┘
         │
   ┌─────▼──────────────────────────────────────┐
   │ 해결책                                      │
   │                                            │
   │  1. 연속 배열       → 포인터 제거           │
   │     (Contiguous Arrays)                    │
   │                                            │
   │  2. 빽빽한 데이터   → 활성 객체를 앞으로    │
   │     (Packed Data)    정렬, 비활성 건너뜀    │
   │                                            │
   │  3. 핫/콜드 분리    → 자주 쓰는 데이터만    │
   │     (Hot/Cold)       캐시에 올라오게 분리   │
   └────────────────────────────────────────────┘
         │
   ┌─────▼────────────────────────────────────┐
   │ 트레이드오프                               │
   │  - 추상화 감소 (인터페이스/상속 포기)       │
   │  - 코드 복잡성 증가                        │
   │  - 먼저 프로파일링, 그 다음 최적화          │
   └──────────────────────────────────────────┘
```

---

## 다른 패턴과의 관계

- **컴포넌트 패턴**: 이 패턴과 가장 자주 함께 쓰인다. 엔티티를 도메인(AI, 물리, 렌더)별로 분리해 연속 배열을 적용하기 쉽게 한다.
- **오브젝트 풀 패턴**: 연속 배열에서 객체를 추가/제거하는 문제를 다룬다.
- **타입 오브젝트 패턴**: 상속 없이 다형성을 구현하는 방법으로, 데이터 지역성과 충돌 없이 동작 다양성을 제공한다.
- **업데이트 메서드 패턴**: 컴포넌트의 `update()` 루프가 데이터 지역성의 직접적인 수혜자다.

---

## 핵심 참고 자료

- **Tony Albrecht** - "Pitfalls of Object-Oriented Programming": 캐시 친화적 게임 데이터 설계를 대중화한 논문
- **Noel Llopis** - 같은 주제의 영향력 있는 블로그 포스트 ("data-oriented design"이라는 용어 사용)
- **Richard Fabian** - "Data-Oriented Design" (책): 이 주제의 심화 전문서
- **Artemis** 게임 엔진: 게임 엔티티를 단순 ID로 표현하는 ECS 아키텍처의 선구자

---

## 요약

| 기법 | 문제 해결 | 트레이드오프 |
|------|----------|------------|
| 연속 배열 | 포인터 체이싱 제거 | 다형성 어려움 |
| 빽빽한 데이터 | 비활성 객체 건너뜀 제거 | 객체가 배열 위치 모름 |
| 핫/콜드 분리 | 불필요한 데이터의 캐시 오염 제거 | 구조 분리 복잡성 |

**단 하나의 원칙**: 처리할 데이터를 메모리에서 이웃하게 배치하라. CPU가 데이터를 페치할 때 다음에 필요한 데이터도 함께 가져오게 하라.
