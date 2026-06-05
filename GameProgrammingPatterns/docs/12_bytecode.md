# Chapter 11: 바이트코드 패턴 (Bytecode Pattern)

> **Game Programming Patterns** — Robert Nystrom 저  
> Part IV: Behavioral Patterns / Chapter 11

---

## 의도 (Intent)

> 행동을 가상 머신(Virtual Machine)의 명령어로 인코딩함으로써, 데이터만큼의 유연성을 부여한다.

---

## 동기 (Motivation)

### 문제의 배경

현대 게임은 수백만 줄의 C++ 코드로 이루어진다. 이 코드는:

- **높은 안정성** 요구: 콘솔 제조사와 앱 마켓은 크래시에 엄격하다
- **극한의 성능** 요구: 하드웨어를 쥐어짜는 최적화가 필수
- **긴 빌드 타임**: "커피 한 잔"에서 "원두를 직접 로스팅하는" 수준까지

여기에 게임 특유의 제약이 하나 더 있다: **재미**. 플레이어는 새로우면서도 균형 잡힌 경험을 원한다. 이를 위해 수시로 수치를 조정해야 하는데, C++ 코드를 손댈 때마다 리컴파일을 기다려야 한다면 창의적 흐름이 끊긴다.

---

### 마법 주문 예제: 문제 정의

마법 대결 게임을 만든다고 하자. 주문(spell)을 C++ 코드로 직접 정의하면:

```
① 수치 조정할 때마다 엔지니어 개입 필요
② 코드 변경 → 전체 리컴파일 → 재부팅 → 테스트
③ 출시 후 업데이트하려면 실행 파일 자체를 패치해야 함
④ 모딩(Modding) 지원 시 유저에게 소스 코드와 컴파일러 제공 필요
⑤ 유저 작성 주문의 버그가 다른 플레이어의 게임을 크래시시킬 수 있음
```

**결론**: 게임 엔진의 구현 언어는 행동(behavior) 정의에 맞지 않는다. 행동을 **데이터**로 정의하고 싶다.

---

### 인터프리터 패턴(GoF)과의 비교

GoF의 인터프리터 패턴은 표현식을 객체 트리로 표현한다:

```
(1 + 2) * (3 - 4)
            ↓ 파싱(Parsing)
         [Multiply]
         /         \
    [Add]          [Subtract]
    /    \          /       \
 [1]    [2]      [3]       [4]
```

각 노드는 `evaluate()`를 구현하는 객체다:

```cpp
class Expression
{
public:
    virtual double evaluate() = 0;
};

class NumberExpression : public Expression
{
public:
    NumberExpression(double value) : value_(value) {}
    virtual double evaluate() { return value_; }
private:
    double value_;
};

class AdditionExpression : public Expression
{
public:
    AdditionExpression(Expression* left, Expression* right)
    : left_(left), right_(right) {}

    virtual double evaluate()
    {
        double left = left_->evaluate();
        double right = right_->evaluate();
        return left + right;
    }
private:
    Expression* left_;
    Expression* right_;
};
```

우아하지만 **심각한 성능 문제**가 있다:

| 문제 | 상세 |
|---|---|
| **메모리 낭비** | 작은 산술 표현식 하나가 68바이트 이상 차지 (vtable 포인터 포함) |
| **분산된 메모리** | 수많은 작은 객체와 포인터가 메모리 전체에 흩어짐 |
| **캐시 미스** | 포인터 추적 → 데이터 캐시 파괴. 가상 메서드 호출 → 명령 캐시 파괴 |

> Ruby는 15년간 인터프리터 패턴으로 구현했다가 1.9 버전에서 바이트코드로 전환했다.

---

### 해결책: 가상 머신 (Virtual Machine)

실제 머신 코드의 장점:

```
① 밀집(Dense):   연속적인 이진 데이터 블록, 낭비 없음
② 선형(Linear):  명령어가 순서대로 실행됨, 메모리 점프 없음
③ 저수준(Low-level): 각 명령어가 하나의 최소 작업 수행
④ 빠름(Fast):    하드웨어에 직접 구현되어 극도로 빠름
```

단, 실제 머신 코드는 보안상 허용할 수 없다. 해결책: **자체 가상 머신 코드(바이트코드)** 를 정의하고, 게임 안에서 작은 에뮬레이터로 실행한다.

```
인터프리터 패턴                   바이트코드 VM
─────────────────────────        ──────────────────────────
객체 트리 (분산, 느림)            연속 바이트 배열 (밀집, 빠름)
메모리: 분산                      메모리: 연속
캐시 효율: 나쁨                   캐시 효율: 좋음
안전성: 높음                      안전성: 높음 (샌드박스)
유연성: 높음                      유연성: 높음 (데이터)
```

---

## 패턴 구조 (The Pattern)

**명령어 집합(instruction set)** 이 수행 가능한 저수준 연산들을 정의한다. 명령어들은 **바이트 시퀀스**로 인코딩된다. **가상 머신**은 이 명령어들을 한 번에 하나씩 실행하며, 중간 값을 위해 **스택**을 사용한다. 명령어를 조합해 복잡한 고수준 행동을 정의할 수 있다.

```
┌──────────────────────────────────────────────────────┐
│                   Virtual Machine                    │
│                                                      │
│  bytecode: [0x00][0x01][0x0A][0x02]...              │
│                ↑                                     │
│            instruction pointer (i)                   │
│                                                      │
│  stack: [  45  |   7  |  11  |     |     ]           │
│              ↑                                       │
│          stackSize_                                  │
│                                                      │
│  switch(instruction) { case 0x00: ... }              │
└──────────────────────────────────────────────────────┘
```

---

## 예제 코드 (Sample Code)

### 1단계: API 설계

주문이 호출할 수 있는 기본 C++ API를 정의한다:

```cpp
// 마법사 스탯 수정
void setHealth(int wizard, int amount);
void setWisdom(int wizard, int amount);
void setAgility(int wizard, int amount);

// 시각/청각 효과
void playSound(int soundId);
void spawnParticles(int particleType);
```

첫 번째 파라미터 `wizard`: 0 = 플레이어, 1 = 상대방

---

### 2단계: 명령어 집합 정의

각 API 호출을 enum 값으로 매핑한다:

```cpp
enum Instruction
{
    INST_SET_HEALTH      = 0x00,
    INST_SET_WISDOM      = 0x01,
    INST_SET_AGILITY     = 0x02,
    INST_PLAY_SOUND      = 0x03,
    INST_SPAWN_PARTICLES = 0x04,
    INST_LITERAL         = 0x05,  // 리터럴 값 푸시
    INST_GET_HEALTH      = 0x06,
    INST_GET_WISDOM      = 0x07,
    INST_GET_AGILITY     = 0x08,
    INST_ADD             = 0x09,
    INST_DIVIDE          = 0x0A,
};
```

주문 = 이 enum 값들의 **바이트 배열**:

```cpp
// 예시: 마법사0의 체력을 100으로 설정
char spell[] = {
    INST_LITERAL, 0,    // wizard = 0
    INST_LITERAL, 100,  // amount = 100
    INST_SET_HEALTH
};
```

---

### 3단계: 스택 머신 (Stack Machine) 구현

```cpp
class VM
{
public:
    VM() : stackSize_(0) {}

    void interpret(char bytecode[], int size)
    {
        for (int i = 0; i < size; i++)
        {
            char instruction = bytecode[i];
            switch (instruction)
            {
                case INST_LITERAL:
                {
                    // 다음 바이트를 리터럴 값으로 스택에 푸시
                    int value = bytecode[++i];
                    push(value);
                    break;
                }

                case INST_SET_HEALTH:
                {
                    int amount = pop();
                    int wizard = pop();
                    setHealth(wizard, amount);
                    break;
                }

                case INST_GET_HEALTH:
                {
                    int wizard = pop();
                    push(getHealth(wizard));
                    break;
                }

                case INST_GET_AGILITY:
                {
                    int wizard = pop();
                    push(getAgility(wizard));
                    break;
                }

                case INST_GET_WISDOM:
                {
                    int wizard = pop();
                    push(getWisdom(wizard));
                    break;
                }

                case INST_ADD:
                {
                    int b = pop();
                    int a = pop();
                    push(a + b);
                    break;
                }

                case INST_DIVIDE:
                {
                    int b = pop();
                    int a = pop();
                    push(a / b);
                    break;
                }

                case INST_PLAY_SOUND:
                    playSound(pop());
                    break;

                case INST_SPAWN_PARTICLES:
                    spawnParticles(pop());
                    break;
            }
        }
    }

private:
    static const int MAX_STACK = 128;
    int stackSize_;
    int stack_[MAX_STACK];

    void push(int value)
    {
        assert(stackSize_ < MAX_STACK);  // 스택 오버플로우 방지
        stack_[stackSize_++] = value;
    }

    int pop()
    {
        assert(stackSize_ > 0);          // 언더플로우 방지
        return stack_[--stackSize_];
    }
};
```

---

### 4단계: 복합 표현식 실행 추적

**목표 주문**: "플레이어 마법사의 체력을 현재 체력 + (민첩 + 지혜) / 2 로 설정"

```cpp
// C++로 표현하면:
setHealth(0, getHealth(0) + (getAgility(0) + getWisdom(0)) / 2);
```

**바이트코드 + 스택 변화 (체력=45, 민첩=7, 지혜=11):**

```
명령어              스택 상태             설명
──────────────────────────────────────────────────────
LITERAL 0          [0]                  마법사 인덱스
LITERAL 0          [0, 0]               마법사 인덱스 (get_health 용)
GET_HEALTH         [0, 45]              현재 체력 획득
LITERAL 0          [0, 45, 0]           마법사 인덱스
GET_AGILITY        [0, 45, 7]           민첩 획득
LITERAL 0          [0, 45, 7, 0]        마법사 인덱스
GET_WISDOM         [0, 45, 7, 11]       지혜 획득
ADD                [0, 45, 18]          민첩 + 지혜
LITERAL 2          [0, 45, 18, 2]       나누기 인수
DIVIDE             [0, 45, 9]           평균 계산
ADD                [0, 54]              체력 + 평균
SET_HEALTH         []                   체력을 54로 설정
```

> 처음에 푸시한 마법사 인덱스(0)가 스택 바닥에서 기다리다가 마지막 `SET_HEALTH`에서 사용된다. 스택이 순서를 암묵적으로 관리한다.

---

## 샌드박스 보안

VM의 보안은 구조적으로 보장된다:

```
보안 속성
├── 명령어 집합이 허용하는 것만 실행 가능
│   → 파일 시스템, 네트워크 등 미노출 API에는 접근 불가
├── 스택 크기 고정 → 메모리 사용량 제어
├── assert로 스택 오버/언더플로우 방지
└── 명령어 카운터로 무한 루프 제한 가능
```

```cpp
// 실행 시간 제한 예시
int executedCount = 0;
const int MAX_INSTRUCTIONS = 10000;

for (int i = 0; i < size; i++)
{
    if (++executedCount > MAX_INSTRUCTIONS) break;  // 타임아웃
    // ... 명령어 실행
}
```

---

## 도구 체인 (Toolchain)

바이트코드는 저수준이어서 직접 작성하기 어렵다. **프론트엔드 도구**가 필요하다:

```
사용자가 작성한 고수준 코드/GUI
            ↓
    [컴파일러 / 저작 도구]
            ↓
    [바이트코드 파일 (.spell)]
            ↓
    [VM이 로드하여 실행]
```

### 텍스트 기반 언어 vs GUI 도구

| 방식 | 장점 | 단점 |
|---|---|---|
| **텍스트 기반 언어** | 보편적 파일 형식, 이식성, 개발자 친화적 | 문법 설계 어려움, 파서 구현 필요, 오류 처리 복잡, 비기술 사용자 진입 장벽 |
| **그래픽 저작 도구** | 잘못된 프로그램 원천 차단, 사용자 친화적, 실시간 오류 방지 | UI 구현 부담, 플랫폼 의존성 |

> **저자 권장**: 디자이너가 사용한다면 GUI 도구를 만들어라. 텍스트 파일은 세미콜론 하나에 컴파일러가 비명을 지른다. 사람은 실수를 하게 되어 있다 — 그것을 포용하고 되돌리기(undo) 같은 기능으로 지원하라.

---

## 설계 결정 (Design Decisions)

### 1. 스택 기반 vs 레지스터 기반 VM

| 특성 | 스택 기반 (Stack-based) | 레지스터 기반 (Register-based) |
|---|---|---|
| **명령어 크기** | 작음 (보통 1바이트) | 큼 (Lua는 32비트) |
| **코드 생성 복잡도** | 단순 | 복잡 |
| **명령어 수** | 많음 (값 이동 명령 필요) | 적음 |
| **성능** | 약간 낮음 | 약간 높음 |
| **대표 사례** | JVM, .NET CLR, CPython | Lua 5.1+, LuaJIT |

> **권장**: 스택 기반 VM. 구현이 단순하고 코드 생성이 쉽다.

---

### 2. 명령어 종류

| 카테고리 | 설명 | 예시 |
|---|---|---|
| **외부 기본 연산** | VM에서 게임 엔진으로 나가는 명령 | `SET_HEALTH`, `PLAY_SOUND` |
| **내부 기본 연산** | VM 내부 값 조작 | `LITERAL`, `ADD`, `DIVIDE` |
| **제어 흐름** | 조건 분기, 루프 | `JUMP`, `JUMP_IF` (goto와 동일) |
| **추상화** | 재사용 가능한 프로시저 | `CALL`, `RETURN` (두 번째 리턴 스택 사용) |

**점프(Jump) 구현 원리**:

```cpp
case INST_JUMP:
{
    // 다음 바이트가 점프할 오프셋
    int offset = bytecode[++i];
    i += offset;  // 명령어 포인터를 직접 수정
    break;
}
// 모든 고수준 제어 흐름(if/while/for)은 결국 jump로 구현됨
```

---

### 3. 값 표현 방식

| 방식 | 장점 | 단점 | 적합한 경우 |
|---|---|---|---|
| **단일 타입** | 단순, 빠름 | 타입 다양성 없음 | 숫자만 필요할 때 |
| **태그드 유니온** | 런타임 타입 확인, 동적 타입 | 메모리 오버헤드 | 대부분의 스크립팅 언어 |
| **비태그 유니온** | 최소 메모리, 빠름 | 타입 안전성 없음, 위험 | 정적 타입 언어 |
| **인터페이스(다형성)** | 확장 가능, OOP 친화 | 힙 할당, 가상 호출 오버헤드 | 성능이 중요하지 않을 때 |

**태그드 유니온 예시** (권장):

```cpp
enum ValueType
{
    TYPE_INT,
    TYPE_DOUBLE,
    TYPE_STRING
};

struct Value
{
    ValueType type;
    union
    {
        int    intValue;
        double doubleValue;
        char*  stringValue;
    };
};
```

> **권장**: 단일 타입으로 충분하면 그렇게 하고, 아니라면 태그드 유니온을 사용하라. 대부분의 언어 인터프리터가 이 방식을 채택한다.

---

### 4. 바이트코드 생성 방법

```
텍스트 언어                           그래픽 도구
─────────────────────────             ─────────────────────────
[사용자가 텍스트 파일 작성]             [사용자가 GUI에서 블록 조립]
        ↓                                      ↓
  [렉서 (Lexer)]                       [UI 이벤트 핸들러]
        ↓                                      ↓
  [파서 (Parser)]                      [내부 AST 구성]
        ↓                                      ↓
  [AST 생성]                           [코드 생성기]
        ↓                                      ↓
  [코드 생성기]                         [바이트코드]
        ↓
  [바이트코드]
```

> 내부적으로 두 방식 모두 **AST(추상 구문 트리)**를 거친다. 이 트리를 인터프리터 패턴으로 즉시 실행하는 대신, 순회하며 바이트코드 명령어를 출력한다.

---

## 인터프리터 패턴과의 관계

```
인터프리터 패턴과 바이트코드의 협력:

 ┌─────────────────────────────────────────────┐
 │            저작 도구 (Authoring Tool)         │
 │                                             │
 │  [텍스트 / GUI]                             │
 │       ↓                                     │
 │  [파서 → AST] ← 인터프리터 패턴             │
 │       ↓                                     │
 │  [AST 순회 + 바이트코드 출력]               │
 └─────────────────────────────────────────────┘
                     ↓
            [바이트코드 파일]
                     ↓
 ┌─────────────────────────────────────────────┐
 │              게임 런타임                     │
 │                                             │
 │  [VM이 바이트코드 로드 및 실행]              │
 └─────────────────────────────────────────────┘
```

---

## 언제 사용할까? (When to Use It)

다음 조건이 모두 해당할 때:

```
① 정의해야 할 행동이 많다
② 구현 언어(C++)가 맞지 않는다:
   - 너무 저수준 → 반복적이고 오류 발생 쉬움
   - 컴파일 시간이 너무 길다
   - 보안/샌드박스가 필요하다
```

**사용하지 말아야 할 때**:

- 성능이 critical한 엔진 코어 (바이트코드는 네이티브 코드보다 느림)
- 행동이 단순하고 변경이 드물 때 (오버엔지니어링)

---

## 주의사항 (Keep in Mind)

### 언어는 자란다

작게 시작한 바이트코드 시스템은 필연적으로 커진다. "아주 작게 만들겠다"는 말을 믿지 마라. 기능이 추가되면서 즉흥적으로 성장한 언어는 아키텍처가 엉망이 된다.

> **해결**: 처음부터 범위를 엄격히 제한하라. 기능 추가 요청에 보수적으로 대응하라.

### 프론트엔드가 필요하다

바이트코드는 저수준이다. 사용자가 직접 바이트코드를 작성하는 것은 어셈블리를 작성하는 것과 같다. **반드시 고수준 저작 도구를 함께 제공해야 한다.** 저작 도구 없이 바이트코드만 만드는 것은 반쪽짜리다.

### 디버거를 잃는다

기존의 디버거, 정적 분석기, 프로파일러는 모두 C++/네이티브 코드용이다. 자체 VM을 만들면 이런 도구들을 사용할 수 없다. **VM 전용 디버깅 도구** 개발에 투자하라 — 특히 모딩을 지원한다면 필수다.

---

## 핵심 정리

| 개념 | 설명 |
|---|---|
| **바이트코드 (Bytecode)** | 가상 머신이 실행하는 합성 이진 명령어 시퀀스 |
| **가상 머신 (VM)** | 바이트코드를 해석하고 실행하는 에뮬레이터 |
| **스택 머신 (Stack Machine)** | 명령어 인수를 스택으로 주고받는 VM 아키텍처 |
| **명령어 집합 (Instruction Set)** | VM이 실행 가능한 저수준 연산의 집합. 샌드박스의 경계 |
| **리터럴 명령어 (INST_LITERAL)** | 바이트코드 스트림에서 다음 바이트를 값으로 스택에 푸시 |
| **태그드 유니온 (Tagged Union)** | 타입 태그와 실제 값을 함께 저장하는 동적 타입 값 표현 방식 |
| **샌드박스 (Sandbox)** | VM이 허용된 명령어만 실행하도록 격리하는 보안 모델 |
| **레지스터 기반 VM** | 스택 대신 인덱싱된 레지스터로 값을 참조하는 VM (Lua) |

---

## 관련 패턴 (See Also)

| 패턴 | 관계 |
|---|---|
| **Interpreter (GoF)** | 바이트코드의 형제 패턴. 저작 도구 내부에서 AST를 표현하는 데 사용됨. 바이트코드는 Interpreter 패턴의 성능 문제를 해결 |
| **Update Method** | 바이트코드로 구현된 행동은 매 프레임 VM을 통해 실행될 수 있음 |
| **Data Locality** | 연속 바이트 배열인 바이트코드는 캐시 친화적이어서 Interpreter 패턴보다 빠름 |

---

## 실제 사례

| 시스템 | 설명 |
|---|---|
| **Lua** | 가장 널리 사용되는 게임 스크립팅 언어. 레지스터 기반 바이트코드 VM |
| **Java Virtual Machine (JVM)** | 1바이트 명령어를 사용하는 스택 기반 VM의 대표 사례 |
| **.NET CLR** | JVM과 유사한 스택 기반 VM. C#, F# 등 실행 |
| **Kismet (Unreal)** | UnrealEd에 내장된 그래픽 스크립팅 도구 |
| **Wren** | 저자(Robert Nystrom)가 직접 만든 스택 기반 바이트코드 인터프리터 |
| **Henry Hatsworth** | 저자가 그래픽 UI 기반 스크립팅 시스템으로 구현한 실제 게임 |

---

*© 2009-2015 Robert Nystrom*
