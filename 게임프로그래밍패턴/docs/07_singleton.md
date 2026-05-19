# Chapter 6: 싱글턴 패턴 (Singleton Pattern)

> **Game Programming Patterns** — Robert Nystrom 저  
> Part II: Design Patterns Revisited / Chapter 6

---

> **이 챕터는 특이하다.** 이 책의 다른 챕터는 모두 패턴을 *어떻게 사용하는지* 알려준다.  
> **이 챕터는 패턴을 *어떻게 사용하지 않는지* 알려준다.**

---

## 의도 (Intent)

GoF 정의:
> **"클래스에 인스턴스가 하나만 존재하도록 보장하고, 그 인스턴스에 대한 전역 접근점(global access point)을 제공한다."**

이 정의의 **"그리고(and)"** 에 주목하라. 실제로는 두 개의 독립적인 문제를 해결한다:

| 문제 | 설명 |
|---|---|
| **단일 인스턴스 보장** | 클래스 인스턴스가 하나만 존재하도록 컴파일 타임에 강제 |
| **전역 접근점 제공** | 코드베이스 어디서든 그 인스턴스에 접근 가능 |

---

## 고전적 구현

### 기본 구현 (지연 초기화, Lazy Initialization)

```cpp
class FileSystem
{
public:
    static FileSystem& instance()
    {
        if (instance_ == NULL)
            instance_ = new FileSystem();
        return *instance_;
    }

private:
    FileSystem() {}  // private 생성자 → 외부에서 인스턴스 생성 불가
    static FileSystem* instance_;
};
```

### 모던 C++11 구현 (스레드 안전)

```cpp
class FileSystem
{
public:
    static FileSystem& instance()
    {
        static FileSystem* instance = new FileSystem();
        return *instance;
    }

private:
    FileSystem() {}
};
```

> C++11은 로컬 정적 변수(local static variable)의 초기화가 동시성 환경에서도 **단 한 번만 실행됨**을 보장한다.  
> 단, 싱글턴 클래스 자체의 스레드 안전성은 별개의 문제다.

### 플랫폼별 서브클래스 활용

```cpp
class FileSystem
{
public:
    static FileSystem& instance();
    virtual char* readFile(char* path)  = 0;
    virtual void  writeFile(char* path, char* contents) = 0;
protected:
    FileSystem() {}
};

FileSystem& FileSystem::instance()
{
#if PLATFORM == PLAYSTATION3
    static FileSystem* instance = new PS3FileSystem();
#elif PLATFORM == WII
    static FileSystem* instance = new WiiFileSystem();
#endif
    return *instance;
}
```

코드베이스 전체가 `FileSystem::instance()`만 알고, 플랫폼 종속 코드는 구현 파일 안에 캡슐화된다.

---

## 싱글턴의 장점 (Why We Use It)

| 장점 | 설명 |
|---|---|
| **사용 시에만 생성** | 게임이 실제로 요청하기 전까지 인스턴스를 만들지 않음. 메모리·CPU 절약 |
| **런타임 초기화** | 정적 멤버 변수와 달리, 파일에서 읽은 설정값 등 런타임 정보를 활용해 초기화 가능 |
| **초기화 순서 제어** | 다른 싱글턴이 먼저 초기화될 수 있음 (단, 순환 의존 주의) |
| **서브클래싱 가능** | 플랫폼별, 환경별 구현체로 교체 가능 |

---

## 싱글턴의 문제점 (Why We Regret Using It)

### 문제 1: 전역 변수다

싱글턴은 결국 **클래스로 포장된 전역 상태(global state)** 다.

**전역 변수의 세 가지 해악:**

**① 코드 추론을 어렵게 만든다**

```cpp
// 이 함수는 전역 상태에 의존하는가?
void someFunction()
{
    SomeClass::getSomeGlobalData(); // 어디서 이 데이터가 바뀌는지 알 수 없다
}
```

순수 함수(pure function)는 전역 상태를 읽거나 쓰지 않는다. 순수 함수는 추론하기 쉽고, 컴파일러가 최적화하기 쉬우며, 메모이제이션(memoization)도 가능하다. 전역 상태는 이 모든 장점을 날린다.

**② 커플링을 조장한다**

```cpp
// 신입 개발자가 물리 엔진 안에서 오디오를 직접 재생
void Physics::resolveCollision(Entity& a, Entity& b)
{
    // ...
    AudioPlayer::instance().play(CRASH_SOUND); // 물리 ↔ 오디오 커플링 발생!
}
```

전역 접근이 없다면 `#include`를 해도 인스턴스를 얻을 수 없어, 잘못된 커플링이 자연스럽게 방지된다. **접근을 제어하면 커플링을 제어할 수 있다.**

**③ 동시성(Concurrency)에 취약하다**

모든 스레드가 전역 상태를 동시에 읽고 쓸 수 있다. 데드락(deadlock), 레이스 컨디션(race condition) 등 디버깅이 극히 어려운 버그의 온상이 된다.

---

### 문제 2: 두 문제를 항상 같이 해결한다

"단일 인스턴스"와 "전역 접근"은 **독립적인 문제**다. 하나만 필요할 때도 싱글턴은 둘 다 강제한다.

**로거(Logger) 예시:**

```cpp
// 처음에는 편리해 보인다
Log::instance().write("Some event.");

// 나중에 도메인별 로거를 분리하고 싶어졌을 때…
// 싱글턴은 다중 인스턴스를 허용하지 않는다!
// 모든 호출 지점을 변경해야 한다
```

싱글턴 도입 이후 설계 변경이 필요해지면, **클래스 자체와 모든 호출 지점을 동시에 수정**해야 한다.

---

### 문제 3: 지연 초기화가 제어권을 빼앗는다

게임에서 시스템 초기화 시점은 중요하다.

```
오디오 시스템 초기화 = 수백 밀리초 소요
→ 첫 번째 사운드 재생 시 자동 초기화
→ 액션 장면 도중 프레임 드롭 발생 가능
```

또한 메모리 단편화(memory fragmentation) 방지를 위해 힙 레이아웃을 통제해야 한다. 지연 초기화는 이 제어권을 포기하는 것이다.

**지연 초기화를 제거한 구현:**

```cpp
class FileSystem
{
public:
    static FileSystem& instance() { return instance_; }
private:
    FileSystem() {}
    static FileSystem instance_; // 정적 초기화 시점에 생성
};
```

하지만 이렇게 하면 싱글턴의 장점(다형성, 런타임 초기화 등)을 모두 잃는다.  
결국 그냥 **정적 클래스(static class)** 와 다를 게 없다.

> `Foo::instance().bar()` 대신 `Foo::bar()`가 더 간단하다.  
> 정적 클래스로 충분하다면, `instance()` 메서드 자체가 불필요하다.

---

## 싱글턴 대신 사용할 수 있는 것들

### 해결책 0: 클래스가 필요한지 먼저 따져라

"Manager" 클래스의 함정:

```cpp
// 나쁜 설계: BulletManager 싱글턴
class BulletManager
{
public:
    Bullet* create(int x, int y) { ... }
    bool isOnScreen(Bullet& bullet) { ... }
    void move(Bullet& bullet)    { ... }
};

// 좋은 설계: Bullet이 스스로를 관리
class Bullet
{
public:
    Bullet(int x, int y) : x_(x), y_(y) {}
    bool isOnScreen() { return x_ >= 0 && x_ < SCREEN_WIDTH && ...; }
    void move()       { x_ += 5; }
private:
    int x_, y_;
};
```

> OOP의 핵심은 **객체가 스스로를 돌보게 하는 것**이다.  
> Manager/Helper 싱글턴의 상당수는 책임이 잘못 배치된 결과다.

---

### 해결책 1: 단일 인스턴스만 보장하기 (전역 접근 없이)

전역 접근 없이 단일 인스턴스만 강제하는 방법:

```cpp
class FileSystem
{
public:
    FileSystem()
    {
        assert(!instantiated_); // 두 번째 인스턴스 생성 시 즉시 실패
        instantiated_ = true;
    }
    ~FileSystem() { instantiated_ = false; }

private:
    static bool instantiated_;
};
bool FileSystem::instantiated_ = false;
```

누구든 생성할 수 있지만, 두 번 생성하면 `assert`로 즉시 실패한다.  
전역 접근점 없이 단일 인스턴스 요건만 충족한다.

> 단점: 컴파일 타임이 아닌 **런타임에만** 중복 생성을 감지한다.

---

### 해결책 2: 필요한 곳에 전달하기 (Dependency Injection)

가장 단순하고 종종 최선인 방법:

```cpp
// 렌더링이 필요한 곳에 컨텍스트를 직접 전달
void renderScene(GraphicsContext& context)
{
    context.drawMesh(...);
}
```

**의존성 주입(Dependency Injection)**: 코드가 전역에서 의존성을 끌어오는 대신, 필요한 의존성을 파라미터로 밀어 넣는다.

로깅처럼 여러 곳에 흩어진 **횡단 관심사(cross-cutting concern)** 에는 다소 어색할 수 있다.

---

### 해결책 3: 기반 클래스에서 제공받기

게임 오브젝트 계층 구조를 활용한다:

```cpp
class GameObject
{
protected:
    Log& getLog() { return log_; } // 파생 클래스만 접근 가능

private:
    static Log& log_;
};

class Enemy : public GameObject
{
    void doSomething()
    {
        getLog().write("I can log!"); // 전역 접근 없이 로거 사용
    }
};
```

`GameObject` 외부에서는 `Log`에 접근할 수 없고, 모든 파생 클래스는 보호된 메서드로 접근한다.  
→ **Subclass Sandbox 패턴** 참고.

---

### 해결책 4: 이미 전역인 것에 얹기

코드베이스에 이미 존재하는 전역 객체(예: `Game`, `World`)를 활용한다:

```cpp
class Game
{
public:
    static Game& instance() { return instance_; }

    Log&         getLog()         { return *log_; }
    FileSystem&  getFileSystem()  { return *fileSystem_; }
    AudioPlayer& getAudioPlayer() { return *audioPlayer_; }

private:
    static Game instance_;
    Log*         log_;
    FileSystem*  fileSystem_;
    AudioPlayer* audioPlayer_;
};

// 사용
Game::instance().getAudioPlayer().play(VERY_LOUD_BANG);
```

전역 클래스의 수를 `Game` 하나로 줄인다. `Log`, `FileSystem`, `AudioPlayer`는 전역이 아니므로, 나중에 여러 인스턴스를 허용하는 설계로 변경해도 이들 클래스는 수정이 불필요하다.

---

### 해결책 5: 서비스 로케이터 (Service Locator)

전역 접근을 전담하는 클래스를 별도로 정의한다.  
→ **Service Locator 패턴** (별도 챕터)에서 자세히 다룬다.

---

## 의사결정 가이드

```
전역 접근이 필요한가?
├── 아니오 → assert로 단일 인스턴스만 보장
└── 예 → 왜 전역이 필요한가?
         ├── 여러 시스템이 같은 객체를 공유해야 한다
         │   ├── 파라미터로 전달 가능한가? → Dependency Injection
         │   ├── 게임 오브젝트 계층이 있는가? → Base Class 방식
         │   ├── 이미 전역인 Game/World 객체가 있는가? → 거기에 얹기
         │   └── 그래도 전역이 필요하다 → Service Locator
         └── 그냥 어디서든 쓰기 편해서 → ⛔ 재고하라
```

---

## 핵심 정리

| 개념 | 설명 |
|---|---|
| **싱글턴 = 전역 상태** | 클래스로 포장됐을 뿐, 전역 변수의 모든 문제를 그대로 가진다 |
| **두 문제의 분리** | "단일 인스턴스"와 "전역 접근"은 독립적인 문제. 필요한 것만 해결하라 |
| **지연 초기화의 위험** | 게임에서는 초기화 시점과 메모리 레이아웃이 중요하다 |
| **Manager 패턴 경계** | Manager 싱글턴의 대부분은 책임이 잘못 배치된 설계의 결과다 |
| **순수 함수 (Pure Function)** | 전역 상태를 읽거나 쓰지 않는 함수. 추론·최적화·테스트가 용이하다 |
| **횡단 관심사 (Cross-Cutting Concern)** | 로깅처럼 코드베이스 전반에 흩어지는 관심사. 해결이 까다롭다 |
| **의존성 주입 (Dependency Injection)** | 의존성을 전역에서 끌어오지 않고 파라미터로 밀어 넣는 방식 |

---

## 관련 패턴 (See Also)

| 패턴 | 관계 |
|---|---|
| **Service Locator** | 전역 접근을 전담 클래스에 위임하는 더 유연한 방식 |
| **Subclass Sandbox** | 기반 클래스를 통해 파생 클래스에게만 공유 상태 접근을 허용 |
| **Object Pool** | 메모리 단편화 문제를 이해하는 데 도움이 되는 관련 챕터 |

---

*© 2009-2015 Robert Nystrom*
