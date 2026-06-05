# Chapter 7: 상태 패턴 (State Pattern)

> **Game Programming Patterns** — Robert Nystrom 저  
> Part II: Design Patterns Revisited / Chapter 7

---

## 의도 (Intent)

> 객체의 내부 상태가 바뀜에 따라 객체의 행동이 바뀔 수 있도록 한다. 객체가 자신의 클래스를 바꾸는 것처럼 보일 것이다.

이 챕터는 GoF 상태 패턴을 넘어, 게임 프로그래밍에서 필수적인 개념인 **유한 상태 머신(FSM)**, **계층적 상태 머신(Hierarchical State Machine)**, **푸시다운 오토마타(Pushdown Automaton)** 까지 다룬다.

---

## 1부: 문제 — 불 플래그의 지옥

플랫포머 게임의 여주인공 입력 처리 코드를 구현한다고 가정하자.

### 1단계: 점프 구현

```cpp
void Heroine::handleInput(Input input)
{
    if (input == PRESS_B)
    {
        yVelocity_ = JUMP_VELOCITY;
        setGraphics(IMAGE_JUMP);
    }
}
```

**버그**: B 버튼을 계속 누르면 공중에서도 점프 가능(에어 점프).

### 2단계: 에어 점프 방지

```cpp
void Heroine::handleInput(Input input)
{
    if (input == PRESS_B)
    {
        if (!isJumping_)
        {
            isJumping_ = true;
            // 점프...
        }
    }
}
```

### 3단계: 웅크리기 추가

```cpp
void Heroine::handleInput(Input input)
{
    if (input == PRESS_B)
    {
        if (!isJumping_ && !isDucking_)  // 플래그 추가
        {
            // 점프...
        }
    }
    else if (input == PRESS_DOWN)
    {
        if (!isJumping_)
        {
            isDucking_ = true;
            setGraphics(IMAGE_DUCK);
        }
    }
    else if (input == RELEASE_DOWN)
    {
        if (isDucking_)
        {
            isDucking_ = false;
            setGraphics(IMAGE_STAND);
        }
    }
}
```

**버그**: 웅크린 채 점프 → 공중에서 아래 버튼 해제 → 점프 중에 서는 그래픽이 출력됨.

### 4단계: 다이브 어택 추가 후 또 버그

```cpp
else if (input == PRESS_DOWN)
{
    if (!isJumping_)
    {
        isDucking_ = true;
        setGraphics(IMAGE_DUCK);
    }
    else
    {
        isJumping_ = false;
        setGraphics(IMAGE_DIVE);  // 다이브 중 에어 점프 가능!
    }
}
```

> **교훈**: 복잡한 분기(branching)와 변경 가능한 상태(mutable state)의 조합은 버그의 온상이다.  
> 코드를 건드릴 때마다 무언가 깨진다. 플래그 조합 중 일부는 유효하지 않지만 컴파일러는 이를 막지 못한다.

---

## 2부: 유한 상태 머신 (Finite State Machine, FSM)

### FSM의 핵심 개념

| 구성 요소 | 설명 |
|---|---|
| **상태 (State)** | 머신이 가질 수 있는 고정된 상태 집합 (서 있기, 점프, 웅크리기, 다이브) |
| **현재 상태** | 한 번에 하나의 상태만 가능 |
| **입력 (Input)** | 머신에 전달되는 이벤트 (버튼 입력 등) |
| **전이 (Transition)** | 입력에 따라 한 상태에서 다른 상태로 이동하는 규칙 |

```
        PRESS_B
STANDING ──────────────▶ JUMPING
    │                        │
    │ PRESS_DOWN        PRESS_DOWN
    ▼                        ▼
DUCKING ◀──────────── DIVING
        RELEASE_DOWN
```

### 구현 1: Enum + Switch

```cpp
enum State
{
    STATE_STANDING,
    STATE_JUMPING,
    STATE_DUCKING,
    STATE_DIVING
};

void Heroine::handleInput(Input input)
{
    switch (state_)
    {
    case STATE_STANDING:
        if (input == PRESS_B)
        {
            state_ = STATE_JUMPING;
            yVelocity_ = JUMP_VELOCITY;
            setGraphics(IMAGE_JUMP);
        }
        else if (input == PRESS_DOWN)
        {
            state_ = STATE_DUCKING;
            setGraphics(IMAGE_DUCK);
        }
        break;

    case STATE_JUMPING:
        if (input == PRESS_DOWN)
        {
            state_ = STATE_DIVING;
            setGraphics(IMAGE_DIVE);
        }
        break;

    case STATE_DUCKING:
        if (input == RELEASE_DOWN)
        {
            state_ = STATE_STANDING;
            setGraphics(IMAGE_STAND);
        }
        break;
    }
}
```

Boolean 플래그 방식과의 차이:
- 유효하지 않은 상태 조합이 원천 차단된다 (`isJumping_ && isDucking_` 같은 불가능한 조합 없음)
- 각 상태의 코드가 한 곳에 모인다

**한계**: 웅크림 차징(charge) 같은 **상태별 데이터**가 필요해지면, `Heroine` 클래스에 `chargeTime_` 같은 필드가 쌓이기 시작한다.

```cpp
void Heroine::update()
{
    if (state_ == STATE_DUCKING)
    {
        chargeTime_++;         // 웅크릴 때만 의미 있는 필드가 Heroine에 존재
        if (chargeTime_ > MAX_CHARGE) superBomb();
    }
}
```

---

## 3부: GoF 상태 패턴 (State Pattern)

### 구조

**① 상태 인터페이스 정의**

```cpp
class HeroineState
{
public:
    virtual ~HeroineState() {}
    virtual void handleInput(Heroine& heroine, Input input) {}
    virtual void update(Heroine& heroine) {}
};
```

**② 각 상태를 클래스로 구현**

```cpp
class DuckingState : public HeroineState
{
public:
    DuckingState() : chargeTime_(0) {}

    virtual void handleInput(Heroine& heroine, Input input)
    {
        if (input == RELEASE_DOWN)
        {
            heroine.setGraphics(IMAGE_STAND);
        }
    }

    virtual void update(Heroine& heroine)
    {
        chargeTime_++;
        if (chargeTime_ > MAX_CHARGE)
            heroine.superBomb();
    }

private:
    int chargeTime_; // 이 데이터는 이 상태에만 속한다
};
```

> `chargeTime_`이 `Heroine`이 아닌 `DuckingState`에 속한다. 데이터가 논리적으로 속한 곳에 위치하게 된다.

**③ Heroine은 상태 객체에 위임**

```cpp
class Heroine
{
public:
    virtual void handleInput(Input input)
    {
        state_->handleInput(*this, input);
    }
    virtual void update()
    {
        state_->update(*this);
    }

private:
    HeroineState* state_;
};
```

상태 전환 = `state_` 포인터를 다른 `HeroineState` 객체로 교체.

---

### 상태 객체는 어디에 두는가?

#### 방법 1: 정적 상태 (Static States)

상태 객체가 인스턴스별 데이터가 없다면, 단 하나의 인스턴스를 공유한다 (플라이웨이트 패턴과 동일):

```cpp
class HeroineState
{
public:
    static StandingState standing;
    static DuckingState  ducking;
    static JumpingState  jumping;
    static DivingState   diving;
};

// 전환 시
heroine.state_ = &HeroineState::jumping;
```

메모리 할당 없이 효율적. 상태에 필드가 없을 때 적합.

> 상태에 가상 메서드가 하나뿐이고 필드가 없다면, **상태 함수 포인터**로 더 단순하게 구현 가능하다.

#### 방법 2: 인스턴스 상태 (Instantiated States)

`DuckingState`처럼 인스턴스별 데이터(`chargeTime_`)가 있다면, 전환 시 새 인스턴스를 생성한다:

```cpp
void Heroine::handleInput(Input input)
{
    HeroineState* state = state_->handleInput(*this, input);
    if (state != NULL)
    {
        delete state_;   // 이전 상태 해제
        state_ = state;  // 새 상태로 교체
        state_->enter(*this); // 진입 액션 실행
    }
}

// StandingState에서 DuckingState로 전환
HeroineState* StandingState::handleInput(Heroine& heroine, Input input)
{
    if (input == PRESS_DOWN)
    {
        return new DuckingState(); // 새 인스턴스 반환
    }
    return NULL;
}
```

> `handleInput()` 실행 중에는 현재 상태 객체를 `delete`하면 안 된다(`this` 파괴).  
> 반환값으로 새 상태를 넘기고, Heroine이 교체하는 구조가 안전하다.

---

### 진입/퇴장 액션 (Enter / Exit Actions)

상태 전환 시 그래픽 변경 등의 처리를 **각 상태가 직접 소유**하도록 한다.

```cpp
class StandingState : public HeroineState
{
public:
    virtual void enter(Heroine& heroine)
    {
        heroine.setGraphics(IMAGE_STAND); // 이 상태로 들어올 때 항상 실행
    }
};
```

장점: 어느 상태에서 전환되어 오든, `enter()`는 항상 실행된다.  
여러 전환이 같은 상태로 진입할 때 코드 중복을 방지한다.

```
어떤 상태에서든 → StandingState 진입 → enter() 한 번만 정의
```

---

### Strategy / Type Object 패턴과의 비교

세 패턴 모두 주 객체가 하위 객체에 위임하는 구조이지만 의도가 다르다:

| 패턴 | 의도 |
|---|---|
| **Strategy** | 주 클래스로부터 일부 행동을 디커플링 |
| **Type Object** | 같은 타입 객체를 참조함으로써 여러 객체가 유사하게 동작하게 함 |
| **State** | 위임하는 객체를 바꿔 주 객체의 행동 자체를 변경 |

---

## 4부: FSM의 한계와 확장

### FSM의 근본적 한계

FSM은 **튜링 완전(Turing complete)이 아니다.** 표현력이 제한되어 있어, 복잡한 AI에는 적합하지 않다.

### 확장 1: 병렬 상태 머신 (Concurrent State Machines)

여주인공이 총을 들고 다닐 수 있다면, 기존 상태(서기/점프/웅크리기)와 장비 상태(비무장/무장)의 조합이 필요하다.

단일 FSM으로 모든 조합을 표현하면: **n개 행동 × m개 장비 = n×m 개 상태**

**해결**: 독립적인 두 개의 상태 머신을 병렬로 운용한다.

```cpp
class Heroine
{
private:
    HeroineState* state_;       // 행동 상태 머신
    HeroineState* equipment_;   // 장비 상태 머신
};

void Heroine::handleInput(Input input)
{
    state_->handleInput(*this, input);      // 두 머신 모두에 입력 전달
    equipment_->handleInput(*this, input);
}
```

```
[행동 FSM]                [장비 FSM]
서기 ↔ 점프 ↔ 웅크리기   비무장 ↔ 무장
(n개 상태)                (m개 상태)
총 n+m 개 (조합 없이)
```

두 상태가 상호작용해야 하는 경우(예: 점프 중에는 발사 불가)는 한 상태에서 다른 머신의 상태를 `if`로 확인하는 방식으로 처리한다.

---

### 확장 2: 계층적 상태 머신 (Hierarchical State Machine, HSM)

서기, 걷기, 달리기, 슬라이딩 상태가 모두 B 버튼으로 점프하고, 아래 버튼으로 웅크린다면?

→ 이 공통 동작을 각 상태에 중복 작성하는 대신, **수퍼스테이트(superstate)** 에 한 번만 정의한다.

```cpp
// 수퍼스테이트: 지상 상태 공통 처리
class OnGroundState : public HeroineState
{
public:
    virtual void handleInput(Heroine& heroine, Input input)
    {
        if (input == PRESS_B)   { /* 점프 */ }
        else if (input == PRESS_DOWN) { /* 웅크리기 */ }
    }
};

// 서브스테이트: 고유 동작만 처리하고, 나머지는 수퍼스테이트에 위임
class DuckingState : public OnGroundState
{
public:
    virtual void handleInput(Heroine& heroine, Input input)
    {
        if (input == RELEASE_DOWN)
        {
            // 일어서기...
        }
        else
        {
            OnGroundState::handleInput(heroine, input); // 수퍼스테이트에 위임
        }
    }
};
```

```
        OnGroundState (수퍼스테이트)
        - PRESS_B → 점프
        - PRESS_DOWN → 웅크리기
        ┌──────┬──────┬──────┐
     Standing Walking Running DuckingState
     (고유 동작 없음)          - RELEASE_DOWN → 일어서기
```

클래스 상속을 사용하지 않는 경우, **상태 스택(stack of states)** 으로도 구현 가능하다. 현재 상태(스택 최상단)가 처리하지 못한 입력은 아래 상태로 전달된다.

---

### 확장 3: 푸시다운 오토마타 (Pushdown Automaton)

**문제**: FSM은 히스토리(history)가 없다. 이전 상태를 기억하지 못한다.

**시나리오**: 서기/달리기/점프 중 어디서든 총을 발사하면 발사 애니메이션을 재생하고, 끝나면 **이전 상태로 돌아가야** 한다.

FSM 방식의 한계:
```
서기_발사, 달리기_발사, 점프_발사 ... 상태가 폭발적으로 증가
→ 각각 이전 상태 하드코딩 필요
```

**해결**: 상태 스택(state stack) 사용

| 연산 | 설명 |
|---|---|
| **Push** | 새 상태를 스택에 추가. 이전 상태는 스택에 보존 |
| **Pop** | 현재 상태를 스택에서 제거. 자동으로 이전 상태로 복귀 |

```
발사 버튼 → FiringState PUSH
           스택: [StandingState, FiringState ← 현재]

발사 완료 → FiringState POP
           스택: [StandingState ← 현재]  (자동 복귀!)
```

```
┌─────────────────────────────────────────────┐
│ FSM            vs       Pushdown Automaton  │
│                                             │
│ state_           stack:                     │
│ ┌─────────┐      ┌─────────┐ ← top (현재)  │
│ │Standing │      │ Firing  │               │
│ └─────────┘      ├─────────┤               │
│                  │Standing │               │
│                  └─────────┘               │
└─────────────────────────────────────────────┘
```

---

## FSM을 사용하기 좋은 상황

FSM이 적합한 조건:

1. 내부 상태에 따라 행동이 바뀌는 엔티티가 있다
2. 상태를 비교적 적은 수의 뚜렷한 선택지로 구분할 수 있다
3. 시간에 따라 일련의 입력이나 이벤트에 반응한다

**주요 활용 사례**: AI, 사용자 입력 처리, 메뉴 내비게이션, 텍스트 파싱, 네트워크 프로토콜, 비동기 행동

> 복잡한 게임 AI에는 **행동 트리(behavior tree)**, **플래닝 시스템(planning system)** 이 더 적합하다.

---

## 패턴 구조 요약

```
    ┌────────────────────────┐
    │        Heroine         │  ← Context
    │────────────────────────│
    │ - state_: HeroineState*│
    │ + handleInput(Input)   │──▶ state_->handleInput(*this, input)
    │ + update()             │──▶ state_->update(*this)
    └────────────────────────┘

    ┌────────────────────────┐
    │     HeroineState       │  ← State (인터페이스)
    │────────────────────────│
    │ + handleInput(...)     │
    │ + update(...)          │
    │ + enter(...)           │
    └────────────────────────┘
               ▲
    ┌──────────┼──────────┐
    │          │          │
StandingState DuckingState JumpingState
(enter: 서기) (chargeTime_) (...)
```

---

## 핵심 정리

| 개념 | 설명 |
|---|---|
| **FSM (유한 상태 머신)** | 고정된 상태 집합, 단일 현재 상태, 입력에 따른 전이로 구성 |
| **Enum + Switch** | FSM의 가장 단순한 구현. 단순한 경우에 적합 |
| **GoF State 패턴** | 각 상태를 클래스로 분리. 상태별 데이터와 행동의 캡슐화 |
| **진입/퇴장 액션** | `enter()` / `exit()` 메서드로 상태 전환 시 공통 동작 처리 |
| **정적 상태** | 필드 없는 상태 객체는 단 하나의 인스턴스를 공유 (Flyweight) |
| **병렬 상태 머신** | 독립적인 두 차원의 상태를 별개의 FSM으로 모델링 |
| **계층적 상태 머신** | 수퍼스테이트로 공통 행동을 공유, 서브스테이트에서 특수 행동 추가 |
| **푸시다운 오토마타** | 상태 스택으로 히스토리 관리. push/pop으로 이전 상태 복귀 가능 |

---

## 관련 패턴 (See Also)

| 패턴 | 관계 |
|---|---|
| **Flyweight** | 필드 없는 상태 객체를 단일 인스턴스로 공유할 때 |
| **Strategy** (GoF) | 상태 패턴과 구조 유사하지만 의도 다름. 행동 교체가 목적 |
| **Update Method** | FSM의 각 상태가 `update()`를 가질 때 자연스럽게 연결 |
| **Object Pool** | 상태 객체를 동적 할당할 때 단편화 방지 |

---

*© 2009-2015 Robert Nystrom*
