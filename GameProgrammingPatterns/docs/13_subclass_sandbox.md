# Chapter 12: 서브클래스 샌드박스 패턴 (Subclass Sandbox Pattern)

> **Game Programming Patterns** — Robert Nystrom 저  
> Part IV: Behavioral Patterns / Chapter 12

---

## 의도 (Intent)

> 기반 클래스가 제공하는 일련의 연산들을 사용해, 파생 클래스 안에서 행동을 정의한다.

---

## 동기 (Motivation)

### 슈퍼파워 게임 예제

수십 가지 초능력(superpower)을 선택할 수 있는 히어로 게임을 만든다고 하자. `Superpower` 기반 클래스를 두고, 초능력마다 파생 클래스를 구현하는 계획이다.

여러 프로그래머가 동시에 수백 개의 서브클래스를 작성하면 어떤 일이 벌어질까?

```
① 중복 코드 범람
   - 사운드 재생, 파티클 효과 등 유사한 코드가 여러 서브클래스에 반복됨
   - 조율 없이 작업하면 중복 코드와 노력 낭비 발생

② 게임 엔진 전체와의 무분별한 커플링
   - 렌더러, 오디오, 물리 등 여러 서브시스템에 직접 의존하게 됨
   - 각 서브시스템의 내부 레이어까지 파고들게 됨

③ 시스템 변경 시 다수의 서브클래스가 깨짐
   - 그래픽/오디오/UI 프로그래머가 초능력 코드까지 관리해야 하는 상황

④ 불변 조건(invariant) 강제 불가
   - 오디오 우선순위 큐잉 같은 정책을 100개 서브클래스에 일일이 적용하기 어려움
```

---

### 해결책: 기반 클래스를 경비원으로

기반 클래스가 파생 클래스에게 **허용된 연산들(provided operations)** 만을 `protected` 메서드로 제공한다. 파생 클래스는 이 연산들만 사용해 **샌드박스 메서드(sandbox method)** 를 구현한다.

```
┌─────────────────────────────────────────────────────────┐
│                    Superpower (기반 클래스)               │
│                                                         │
│  [sandbox method]                                       │
│  # activate() = 0   ← 파생 클래스가 반드시 구현          │
│                                                         │
│  [provided operations - protected]                      │
│  # move(x, y, z)         ← 물리 엔진 래핑               │
│  # playSound(id, volume) ← 오디오 엔진 래핑              │
│  # spawnParticles(type)  ← 파티클 시스템 래핑            │
│  # getHeroX/Y/Z()        ← 게임 상태 읽기               │
└────────────────────┬────────────────────────────────────┘
                     │ 상속 (inheritance)
         ┌───────────┼───────────┐
         ▼           ▼           ▼
    [SkyLaunch] [FreezeRay] [HeatRay]
    activate()  activate()  activate()
    (오직 기반 클래스의 provided operations만 호출)
```

**핵심 효과**:

- **커플링 집중**: 100개 파생 클래스 → 게임 엔진 커플링이 `Superpower` 하나로 집중됨
- **코드 재사용**: 공통 연산을 기반 클래스로 추출해 중복 제거
- **불변 조건 강제**: 오디오, 물리 등의 정책을 한 곳에서만 구현하면 모든 서브클래스에 자동 적용

---

## 패턴 구조 (The Pattern)

기반 클래스가 **추상 샌드박스 메서드**와 **여러 제공 연산들**을 정의한다. `protected`로 표시해 파생 클래스 전용임을 명시한다. 각 파생 클래스는 제공 연산들을 사용해 샌드박스 메서드를 구현한다.

---

## 예제 코드 (Sample Code)

### 기반 클래스: Superpower

```cpp
class Superpower
{
public:
    virtual ~Superpower() {}

protected:
    // ① 샌드박스 메서드 — 파생 클래스가 반드시 구현
    virtual void activate() = 0;

    // ② 제공 연산들 — 파생 클래스만 호출 가능 (protected)
    void move(double x, double y, double z)
    {
        // 물리 엔진과의 커플링이 여기에만 존재
    }

    void playSound(SoundId sound, double volume)
    {
        // 오디오 엔진과의 커플링이 여기에만 존재
    }

    void spawnParticles(ParticleType type, int count)
    {
        // 파티클 시스템과의 커플링이 여기에만 존재
    }

    // 히어로 상태 읽기 (읽기 전용 → 안전한 커플링)
    double getHeroX() { /* ... */ }
    double getHeroY() { /* ... */ }
    double getHeroZ() { /* ... */ }
};
```

`activate()`가 샌드박스 메서드다. 순수 가상 함수이므로 파생 클래스는 반드시 구현해야 한다. 다른 `protected` 메서드들은 서브클래스가 자유롭게 사용할 수 있는 도구 세트다.

---

### 파생 클래스: SkyLaunch (공중 도약)

```cpp
class SkyLaunch : public Superpower
{
protected:
    virtual void activate()
    {
        if (getHeroZ() == 0)
        {
            // 지상에 있을 때: 공중으로 도약
            playSound(SOUND_SPROING, 1.0f);
            spawnParticles(PARTICLE_DUST, 10);
            move(0, 0, 20);
        }
        else if (getHeroZ() < 10.0f)
        {
            // 지상 근처: 이중 점프
            playSound(SOUND_SWOOP, 1.0f);
            move(0, 0, getHeroZ() + 20);
        }
        else
        {
            // 높은 공중: 다이브 어택
            playSound(SOUND_DIVE, 0.7f);
            spawnParticles(PARTICLE_SPARKLES, 1);
            move(0, 0, -getHeroZ());
        }
    }
};
```

`SkyLaunch`는 `Superpower`만 `#include`하면 된다. 물리 엔진, 오디오, 파티클 시스템을 직접 알 필요가 없다.

---

## 아키텍처 구조 다이어그램

```
       [물리 엔진]   [오디오 엔진]   [파티클 시스템]   [게임 상태]
            ↑              ↑               ↑              ↑
            └──────────────┴───────────────┴──────────────┘
                                   │
                           커플링이 여기에만 집중됨
                                   │
                    ┌──────────────▼──────────────┐
                    │       Superpower             │
                    │   (기반 클래스 - 게이트키퍼) │
                    └──────────────┬──────────────┘
                 ┌─────────────────┼─────────────────┐
                 ▼                 ▼                 ▼
          [SkyLaunch]       [FreezeRay]        [HeatRay]
          [Teleport]        [MindControl]      [Invisibility]
          ...               ...               ...
          (100개 파생 클래스 — 외부 시스템을 직접 모름)
```

> **얕고 넓은 클래스 계층(shallow but wide hierarchy)**: 상속 깊이는 얕지만, `Superpower`에 직접 매달린 서브클래스가 많다. 기반 클래스에 투자한 노력이 넓은 범위에 혜택을 준다.

---

## 설계 결정 (Design Decisions)

### 1. 어떤 연산을 제공할 것인가?

| 제공 정도 | 특징 |
|---|---|
| **최소** (샌드박스 메서드만) | 서브클래스가 외부 시스템에 직접 접근. 패턴 효과 없음 |
| **최대** (모든 필요 연산 제공) | 서브클래스는 기반 클래스만 알면 됨. 기반 클래스가 거대해짐 |
| **중간** (일부만 제공) | 현실적인 균형점 |

**제공 연산 선택 기준**:

```
① 여러 서브클래스가 공통으로 사용하는가?
   → 하나 또는 소수만 사용하면 기반 클래스에 둘 가치가 낮다.

② 상태를 변경하는가?
   → 상태 변경 연산은 기반 클래스 래핑이 더 안전하다.
   → 읽기 전용은 직접 외부 시스템 호출도 무방하다.

③ 단순 포워딩인가?
   void playSound(SoundId s) { soundEngine_.play(s); }
   → 단순 포워딩이지만, soundEngine_ 필드를 서브클래스에서 숨기는 값이 있다.
```

---

### 2. 연산을 직접 제공 vs 헬퍼 객체를 통해 제공

**문제**: 제공 연산이 많아지면 기반 클래스가 비대해진다.

**해결**: 관련 연산들을 별도 클래스로 분리하고, 기반 클래스는 그 객체에 대한 접근자만 제공한다.

```cpp
// 나쁜 예: Superpower에 오디오 메서드가 직접 잔뜩 있음
class Superpower
{
protected:
    void playSound(SoundId s, double v) { /* ... */ }
    void stopSound(SoundId s)           { /* ... */ }
    void setVolume(SoundId s, double v) { /* ... */ }
    // ... 더 많은 오디오 메서드
};

// 좋은 예: 헬퍼 객체 분리
class SoundPlayer
{
public:
    void playSound(SoundId s, double v) { /* ... */ }
    void stopSound(SoundId s)           { /* ... */ }
    void setVolume(SoundId s, double v) { /* ... */ }
};

class Superpower
{
protected:
    SoundPlayer& getSoundPlayer() { return soundPlayer_; }
    // 메서드 3개 → getter 1개로 줄어듦

private:
    SoundPlayer soundPlayer_;
};

// 서브클래스에서 사용
class SkyLaunch : public Superpower
{
protected:
    virtual void activate()
    {
        getSoundPlayer().playSound(SOUND_SPROING, 1.0f);
        // ...
    }
};
```

**헬퍼 객체 분리의 장점**:
- 기반 클래스의 메서드 수 감소
- 헬퍼 클래스는 독립적으로 유지·수정 가능
- `Superpower`와 외부 시스템 간 커플링 감소

---

### 3. 기반 클래스가 필요한 상태를 어떻게 얻는가?

#### 방법 A: 생성자에서 전달

```cpp
class Superpower
{
public:
    Superpower(ParticleSystem* particles)
    : particles_(particles)
    {}

private:
    ParticleSystem* particles_;
};

class SkyLaunch : public Superpower
{
public:
    SkyLaunch(ParticleSystem* particles)
    : Superpower(particles)  // 파생 클래스도 알아야 함!
    {}
};
```

**문제**: 파생 클래스 생성자가 기반 클래스의 구현 세부 사항을 알아야 한다. 상태 추가 시 모든 파생 클래스 생성자 수정 필요.

---

#### 방법 B: 2단계 초기화 (권장)

```cpp
class Superpower
{
public:
    void init(ParticleSystem* particles)  // 별도 초기화 메서드
    {
        particles_ = particles;
    }
    // 파라미터 없는 생성자
};

class SkyLaunch : public Superpower
{
    // 생성자에서 파티클 시스템을 전혀 모름 ✓
};

// 사용 시: 팩토리 함수로 캡슐화
Superpower* createSkyLaunch(ParticleSystem* particles)
{
    Superpower* power = new SkyLaunch();
    power->init(particles);  // 반드시 호출해야 함
    return power;
}
```

> `private` 생성자 + `friend` 팩토리 함수 조합으로 `init()` 호출 강제 가능.

---

#### 방법 C: 정적 상태 (모든 인스턴스가 공유)

```cpp
class Superpower
{
public:
    static void init(ParticleSystem* particles)
    {
        particles_ = particles;  // 한 번만 초기화
    }

private:
    static ParticleSystem* particles_;  // 모든 인스턴스 공유
};

// 게임 시작 시 한 번만 호출
Superpower::init(particleSystem);
```

**장점**: 인스턴스마다 포인터 저장 불필요 → 메모리 절약  
**단점**: 싱글턴과 유사한 문제 (공유 상태로 인한 추론 어려움)

---

#### 방법 D: 서비스 로케이터 (Service Locator)

```cpp
class Superpower
{
protected:
    void spawnParticles(ParticleType type, int count)
    {
        // 외부에서 주입받지 않고, 스스로 획득
        ParticleSystem& particles = Locator::getParticles();
        particles.spawn(type, count);
    }
};
```

외부 코드가 미리 상태를 주입하는 대신, 기반 클래스가 필요할 때 직접 가져온다. Service Locator 패턴과 연동.

---

## 주의사항 (Keep in Mind)

### 취약한 기반 클래스 문제 (Brittle Base Class Problem)

서브클래스들이 기반 클래스를 통해 게임 엔진에 접근하므로, 기반 클래스는 모든 파생 클래스가 필요한 시스템에 커플링된다. 기반 클래스를 변경하면 많은 것이 깨질 수 있다.

```
단점: Superpower는 변경하기 어렵다 (너무 많은 것이 의존)
장점: 파생 클래스 100개는 외부 세계로부터 격리됨
→ 코드베이스의 대부분인 파생 클래스가 유지보수하기 쉬워진다
```

기반 클래스가 "코드 스튜 냄비"가 될 것 같으면 **Component 패턴**으로 책임을 분산시켜라.

---

## 핵심 정리

| 개념 | 설명 |
|---|---|
| **샌드박스 메서드 (Sandbox Method)** | 파생 클래스가 반드시 구현해야 하는 추상 `protected` 메서드. 행동의 구현 장소 |
| **제공 연산 (Provided Operations)** | 기반 클래스가 파생 클래스에게 제공하는 `protected` 메서드들. 외부 시스템의 래퍼 |
| **얕고 넓은 계층 (Shallow & Wide)** | 상속 깊이는 얕고, 단일 기반 클래스에 많은 서브클래스가 매달린 구조 |
| **커플링 집중** | 파생 클래스 → 외부 시스템 커플링을 기반 클래스 하나로 집중시키는 전략 |
| **취약한 기반 클래스 문제** | 많은 파생 클래스가 기반 클래스에 의존하므로, 기반 클래스 변경이 위험해지는 현상 |

---

## 관련 패턴 (See Also)

| 패턴 | 관계 |
|---|---|
| **Update Method** | `update()` 자체가 샌드박스 메서드인 경우가 많음 |
| **Template Method (GoF)** | 역할이 반전된 패턴. Template Method는 기반 클래스가 알고리즘을 가지고 파생 클래스가 기본 연산을 구현. Subclass Sandbox는 파생 클래스가 알고리즘(activate)을 가지고 기반 클래스가 기본 연산을 제공 |
| **Facade (GoF)** | 변형된 파사드 패턴. 기반 클래스가 전체 게임 엔진을 서브클래스로부터 숨기는 파사드 역할 |
| **Component** | 기반 클래스가 비대해질 때, 제공 연산들을 별도 컴포넌트로 분리하는 대안 |
| **Service Locator** | 기반 클래스가 필요한 상태를 외부 주입 없이 스스로 획득하는 방법 |
| **Bytecode, Type Object** | 초능력처럼 파생 클래스가 매우 많다면, 데이터 기반 접근법이 더 나을 수 있음 |

---

*© 2009-2015 Robert Nystrom*
