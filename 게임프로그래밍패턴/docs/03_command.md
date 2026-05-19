# Chapter 2: 커맨드 패턴 (Command Pattern)

> **Game Programming Patterns** — Robert Nystrom 저  
> Part II: Design Patterns Revisited / Chapter 2

---

## 의도 (Intent)

> **커맨드(Command)는 메서드 호출을 실체화(reify)한 것이다.**

GoF의 정의는 다음과 같다:
> *"요청을 객체로 캡슐화하여, 클라이언트를 다양한 요청으로 매개변수화하고, 요청을 큐에 넣거나 로그로 남기며, 실행 취소 가능한 연산을 지원할 수 있게 한다."*

더 단순하게 표현하면:

> **"커맨드는 콜백(callback)의 객체지향적 대체물이다."** — Gang of Four

**실체화(reify)**: 어떤 개념을 변수에 담거나 함수에 넘길 수 있는 **데이터(객체)** 로 만드는 것.  
즉, 커맨드 패턴은 메서드 호출을 객체로 감싸서 일급 시민(first-class citizen)으로 취급하는 것이다.

콜백(callback), 일급 함수(first-class function), 함수 포인터(function pointer), 클로저(closure), 부분 적용 함수(partially applied function) 등과 같은 범주에 속한다.

---

## 활용 사례 1: 입력 설정 (Configuring Input)

### 문제

모든 게임에는 원시 입력(raw input)을 게임 액션(game action)으로 변환하는 코드가 있다.

```cpp
void InputHandler::handleInput()
{
    if (isPressed(BUTTON_X)) jump();
    else if (isPressed(BUTTON_Y)) fireGun();
    else if (isPressed(BUTTON_A)) swapWeapon();
    else if (isPressed(BUTTON_B)) lurchIneffectively();
}
```

이 코드는 **입력과 액션이 하드코딩** 되어 있어, 사용자가 키 배열을 커스터마이징하는 기능을 지원할 수 없다.

### 해결: Command 패턴 적용

**1단계 — 커맨드 기반 클래스 정의**

```cpp
class Command
{
public:
    virtual ~Command() {}
    virtual void execute() = 0;
};
```

> `execute()` 하나만 있는 반환값 없는 인터페이스 — 커맨드 패턴의 전형적인 형태다.

**2단계 — 각 액션을 서브클래스로 구현**

```cpp
class JumpCommand : public Command
{
public:
    virtual void execute() { jump(); }
};

class FireCommand : public Command
{
public:
    virtual void execute() { fireGun(); }
};
```

**3단계 — InputHandler에서 커맨드 포인터를 보관**

```cpp
class InputHandler
{
public:
    void handleInput();
private:
    Command* buttonX_;
    Command* buttonY_;
    Command* buttonA_;
    Command* buttonB_;
};

void InputHandler::handleInput()
{
    if (isPressed(BUTTON_X)) buttonX_->execute();
    else if (isPressed(BUTTON_Y)) buttonY_->execute();
    else if (isPressed(BUTTON_A)) buttonA_->execute();
    else if (isPressed(BUTTON_B)) buttonB_->execute();
}
```

이제 버튼과 액션 사이에 **간접 계층(layer of indirection)** 이 생겼다.  
런타임에 버튼에 바인딩된 커맨드 객체를 교체하는 것만으로 키 배열을 변경할 수 있다.

```
[버튼 입력] ──→ [Command 포인터] ──→ [execute()] ──→ [실제 액션]
```

> **Null Object 패턴**: 아무것도 하지 않는 `execute()`를 가진 커맨드 클래스를 만들면, `NULL` 체크 없이 "아무 동작 안 함"을 표현할 수 있다.

---

## 활용 사례 2: 액터에게 명령 내리기 (Directions for Actors)

### 문제

앞선 예시의 커맨드들은 암묵적으로 플레이어 캐릭터만을 대상으로 한다.  
`JumpCommand`는 플레이어만 점프시킬 수 있다.

### 해결: 액터(Actor)를 파라미터로 전달

```cpp
class Command
{
public:
    virtual ~Command() {}
    virtual void execute(GameActor& actor) = 0;
};

class JumpCommand : public Command
{
public:
    virtual void execute(GameActor& actor)
    {
        actor.jump();
    }
};
```

**InputHandler는 커맨드를 반환하도록 변경**

```cpp
Command* InputHandler::handleInput()
{
    if (isPressed(BUTTON_X)) return buttonX_;
    if (isPressed(BUTTON_Y)) return buttonY_;
    if (isPressed(BUTTON_A)) return buttonA_;
    if (isPressed(BUTTON_B)) return buttonB_;
    return NULL;
}
```

**실행은 외부에서 액터와 함께 처리**

```cpp
Command* command = inputHandler.handleInput();
if (command)
{
    command->execute(actor);
}
```

### 이 구조의 장점

커맨드와 그것을 수행하는 액터를 **분리**함으로써 얻는 이점:

| 활용 | 설명 |
|---|---|
| **플레이어 제어 전환** | `actor`를 다른 캐릭터로 바꾸면 누구든 플레이어처럼 제어 가능 |
| **AI 연동** | AI 엔진이 `Command` 객체를 생성해 액터에게 전달하면, AI와 액터 코드가 완전히 분리됨 |
| **데모 모드** | AI가 생성한 커맨드로 플레이어 캐릭터를 자동 조종 |
| **네트워크 멀티플레이** | 커맨드를 직렬화(serialize)해 네트워크로 전송하고, 수신 측에서 재생(replay) 가능 |

```
[InputHandler / AI] ──→ [Command Stream] ──→ [Dispatcher] ──→ [Actor]
    (생산자)                  (큐/스트림)          (소비자)
```

> 큐 중간에 커맨드 스트림을 두면 생산자(producer)와 소비자(consumer)가 완전히 디커플링된다.  
> 자세한 내용은 **Event Queue 패턴** 참고.

---

## 활용 사례 3: 실행 취소와 다시 실행 (Undo and Redo)

커맨드 패턴의 가장 유명한 활용. 커맨드 객체가 어떤 일을 **할 수 있다면**, 그 일을 **되돌리는 것**도 쉽게 추가할 수 있다.

### undo() 메서드 추가

```cpp
class Command
{
public:
    virtual ~Command() {}
    virtual void execute() = 0;
    virtual void undo() = 0;
};
```

### MoveUnitCommand 구현

```cpp
class MoveUnitCommand : public Command
{
public:
    MoveUnitCommand(Unit* unit, int x, int y)
        : unit_(unit),
          xBefore_(0), yBefore_(0),
          x_(x), y_(y)
    {}

    virtual void execute()
    {
        // 이전 위치를 기억해 두어야 undo가 가능하다
        xBefore_ = unit_->x();
        yBefore_ = unit_->y();
        unit_->moveTo(x_, y_);
    }

    virtual void undo()
    {
        unit_->moveTo(xBefore_, yBefore_);
    }

private:
    Unit* unit_;
    int xBefore_, yBefore_;  // undo를 위해 저장하는 이전 상태
    int x_, y_;              // 목표 위치
};
```

### 입력 처리: 매 이동마다 새 커맨드 인스턴스 생성

```cpp
Command* handleInput()
{
    Unit* unit = getSelectedUnit();

    if (isPressed(BUTTON_UP)) {
        int destY = unit->y() - 1;
        return new MoveUnitCommand(unit, unit->x(), destY);
    }
    if (isPressed(BUTTON_DOWN)) {
        int destY = unit->y() + 1;
        return new MoveUnitCommand(unit, unit->x(), destY);
    }
    // ...
    return NULL;
}
```

> C++에서는 이 커맨드를 실행한 코드가 메모리 해제 책임도 진다.

### 다단계 Undo/Redo 구현

커맨드 리스트(list)와 현재 위치를 가리키는 포인터로 구현한다.

```
[Cmd1] [Cmd2] [Cmd3*] [Cmd4] [Cmd5]
                 ↑
              current
```

| 동작 | 처리 |
|---|---|
| **새 커맨드 실행** | 리스트에 추가, `current` 전진, `current` 이후 항목 삭제 |
| **Undo (Ctrl+Z)** | `current->undo()` 호출, `current` 후퇴 |
| **Redo (Ctrl+Y)** | `current` 전진, `current->execute()` 호출 |

> 리플레이(replay) 시스템도 같은 원리: 매 프레임의 커맨드를 기록해두고, 재생 시 동일한 커맨드를 순서대로 실행한다.  
> 전체 게임 상태를 스냅샷으로 저장하는 것보다 훨씬 메모리 효율적이다.

---

## 함수형 언어에서의 커맨드 패턴

C++에서는 클래스로 구현하지만, 클로저(closure)를 지원하는 언어라면 더 간결하게 표현 가능하다.

### JavaScript 예시 (클로저 활용)

```javascript
// 단순 커맨드
function makeMoveUnitCommand(unit, x, y) {
    return function() {
        unit.moveTo(x, y);
    }
}

// undo 지원 커맨드 (클로저 쌍 사용)
function makeMoveUnitCommand(unit, x, y) {
    var xBefore, yBefore;
    return {
        execute: function() {
            xBefore = unit.x();
            yBefore = unit.y();
            unit.moveTo(x, y);
        },
        undo: function() {
            unit.moveTo(xBefore, yBefore);
        }
    };
}
```

> 커맨드 패턴은 클로저가 없는 언어에서 클로저를 에뮬레이션하는 방법이기도 하다.  
> 다만 여러 연산(execute + undo 등)이 필요한 경우엔 클래스 기반이 여전히 유용하다.

---

## 패턴 구조 요약

```
        ┌─────────────┐
        │   Command   │  ← 추상 기반 클래스
        │─────────────│
        │ execute()   │
        │ undo()      │
        └──────┬──────┘
               │ 상속
    ┌──────────┼──────────┐
    ▼          ▼          ▼
JumpCommand  FireCommand  MoveUnitCommand
execute()    execute()    execute()
             undo()       undo()
```

---

## 언제 사용할까? (When to Use It)

- **입력 키 재매핑(key remapping)** — 버튼과 액션을 런타임에 분리해야 할 때
- **실행 취소/다시 실행(undo/redo)** — 레벨 에디터, 전략 게임 등
- **AI와 플레이어 제어 통합** — 동일한 인터페이스로 AI와 플레이어 입력을 처리할 때
- **리플레이 시스템(replay system)** — 커맨드 스트림을 기록·재생
- **네트워크 동기화** — 커맨드를 직렬화해 전송

---

## 주의사항 (Keep in Mind)

- 모든 상태 변경이 반드시 커맨드를 통해야 undo/redo가 완벽하게 동작한다. 이 규율을 지키는 것이 까다로울 수 있다.
- **재사용 가능한 커맨드** vs **일회성 커맨드**의 구분이 중요하다.
  - 재사용: 입력 핸들러가 단일 커맨드 객체를 보관하고 반복 호출
  - 일회성: 매 실행마다 새 인스턴스 생성 (undo 지원 시 적합)
- C++에서는 커맨드 객체의 **메모리 관리** 책임을 명확히 해야 한다.

---

## 관련 패턴 (See Also)

| 패턴 | 관계 |
|---|---|
| **Subclass Sandbox** | 커맨드 클래스가 많아질 때, `execute()` 내에서 공통 기반 클래스의 고수준 메서드를 조합해 구현 |
| **Chain of Responsibility** (GoF) | 커맨드를 받을 객체가 계층 구조일 때, 객체가 커맨드를 직접 처리하거나 하위 객체에 위임 |
| **Flyweight** (GoF) | 상태가 없는 커맨드(`JumpCommand` 등)는 인스턴스를 하나만 공유 가능 |
| **Event Queue** | 커맨드 스트림을 큐로 관리할 때 |
| **Memento** (GoF) | undo를 위한 상태 저장에 사용을 고려할 수 있으나, 커맨드가 일부 상태만 변경할 경우 직접 저장이 더 효율적 |

---

## 핵심 정리

| 개념 | 설명 |
|---|---|
| **실체화 (Reification)** | 메서드 호출을 객체로 만들어 변수에 담고 전달할 수 있게 함 |
| **간접 계층 (Indirection)** | 입력과 액션 사이에 커맨드 객체를 두어 런타임에 동작을 교체 가능하게 함 |
| **커맨드 스트림 (Command Stream)** | 커맨드를 큐/스트림으로 관리해 생산자와 소비자를 디커플링 |
| **undo 구현 원리** | execute() 전 이전 상태를 저장, undo()에서 복원 |
| **다단계 undo** | 커맨드 리스트 + current 포인터로 undo/redo 히스토리 관리 |

---

*© 2009-2015 Robert Nystrom*
