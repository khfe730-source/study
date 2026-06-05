# Chapter 16: 서비스 로케이터 패턴 (Service Locator Pattern)

> **Game Programming Patterns** — Robert Nystrom 저  
> Part V: Decoupling Patterns / Chapter 16

---

## 의도 (Intent)

> 서비스를 구현하는 구체적인 클래스에 사용자를 커플링하지 않고, 서비스에 대한 전역 접근점을 제공한다.

---

## 동기 (Motivation)

### 어디서나 필요한 시스템들

게임 코드베이스를 돌아다니며 거의 모든 곳에서 필요한 시스템들이 있다. 메모리 할당자, 로깅, 난수 생성기, 그리고 오디오가 그 예다.

```cpp
// 방법 1: 정적 클래스
AudioSystem::playSound(VERY_LOUD_BANG);

// 방법 2: 싱글턴
AudioSystem::instance()->playSound(VERY_LOUD_BANG);
```

둘 다 목표를 달성하지만, **구체적인 `AudioSystem` 클래스에 직접 커플링**된다. 코드베이스 전체에 100개의 호출 지점이 생기면, 나중에 오디오 시스템을 바꾸거나 테스트용으로 교체하려면 모든 지점을 수정해야 한다.

---

### 전화번호부 비유

```
나쁜 방법: 100명에게 집 주소를 직접 알려줌
  → 이사하면 100명에게 새 주소를 모두 알려야 함

좋은 방법: 전화번호부 (Service Locator)
  → 이름으로 조회하면 현재 주소를 알 수 있음
  → 이사하면 전화회사에만 알리면 됨
  → 실제 주소 대신 사서함(P.O. Box)을 노출할 수도 있음
```

서비스 로케이터는 서비스를 **누가(구체 타입)** 구현하는지와 **어떻게 찾는지(접근 방법)** 모두를 사용자로부터 분리한다.

---

## 패턴 구조 (The Pattern)

**서비스 클래스**가 연산 집합에 대한 추상 인터페이스를 정의한다. **구체적인 서비스 제공자**가 이 인터페이스를 구현한다. 별도의 **서비스 로케이터**가 적절한 제공자를 찾아 서비스에 대한 접근을 제공하는데, 이때 제공자의 구체적인 타입과 위치 탐색 과정을 모두 숨긴다.

```
┌─────────────────────────────────────────────────────────────┐
│                     Locator (서비스 로케이터)                 │
│                                                             │
│  static Audio& getAudio()  { return *service_; }           │
│  static void provide(Audio* s) { service_ = s; }           │
│                                                             │
│  static Audio*     service_     ← 현재 서비스 제공자        │
│  static NullAudio  nullService_ ← 기본 null 서비스         │
└─────────────────────────┬───────────────────────────────────┘
                          │ 반환 (Audio 인터페이스만 노출)
                          ▼
┌───────────────────────────────────────┐
│  Audio (추상 인터페이스 — 서비스)      │
│                                       │
│  playSound(int soundID) = 0           │
│  stopSound(int soundID) = 0           │
│  stopAllSounds() = 0                  │
└──────────┬──────────────┬─────────────┘
           ↓              ↓              ↓
  [ConsoleAudio]   [NullAudio]    [LoggedAudio]
  (실제 구현)      (아무것도      (데코레이터:
                  하지 않음)      로그 + 위임)
```

---

## 예제 코드 (Sample Code)

### 1단계: 서비스 인터페이스

```cpp
class Audio
{
public:
    virtual ~Audio() {}
    virtual void playSound(int soundID)  = 0;
    virtual void stopSound(int soundID)  = 0;
    virtual void stopAllSounds()         = 0;
};
```

### 2단계: 구체적인 서비스 제공자

```cpp
class ConsoleAudio : public Audio
{
public:
    virtual void playSound(int soundID)
    {
        // 콘솔 오디오 API로 사운드 재생
    }
    virtual void stopSound(int soundID)
    {
        // 콘솔 오디오 API로 사운드 정지
    }
    virtual void stopAllSounds()
    {
        // 모든 사운드 정지
    }
};
```

### 3단계: 단순한 서비스 로케이터

```cpp
class Locator
{
public:
    static Audio* getAudio() { return service_; }

    static void provide(Audio* service)
    {
        service_ = service;
    }

private:
    static Audio* service_;
};
```

**초기화 및 사용**:

```cpp
// 게임 시작 시 서비스 등록
ConsoleAudio* audio = new ConsoleAudio();
Locator::provide(audio);

// 코드베이스 어디서든 사용
Audio* audio = Locator::getAudio();
audio->playSound(VERY_LOUD_BANG);
// ConsoleAudio 클래스를 전혀 모름 — Audio 인터페이스만 알면 됨
```

---

### 4단계: Null 서비스 — 항상 유효한 객체 보장

서비스를 등록하기 전에 사용하면 `NULL` 반환 → 크래시. 이를 **Null Object 패턴**으로 해결한다:

```cpp
// 아무것도 하지 않는 null 구현
class NullAudio : public Audio
{
public:
    virtual void playSound(int soundID) { /* 아무것도 안 함 */ }
    virtual void stopSound(int soundID) { /* 아무것도 안 함 */ }
    virtual void stopAllSounds()        { /* 아무것도 안 함 */ }
};

class Locator
{
public:
    static void initialize() { service_ = &nullService_; }

    // 포인터 대신 참조 반환 → NULL 불가 보장
    static Audio& getAudio() { return *service_; }

    static void provide(Audio* service)
    {
        if (service == NULL)
            service_ = &nullService_;  // NULL 등록 시 null 서비스로 복귀
        else
            service_ = service;
    }

private:
    static Audio*    service_;
    static NullAudio nullService_;  // 정적 null 서비스 (항상 유효)
};
```

```cpp
// 초기화
Locator::initialize();  // null 서비스로 시작

// 나중에 실제 서비스 등록
Locator::provide(new ConsoleAudio());

// 개발 중 오디오 비활성화: 그냥 provide하지 않으면 됨
// → null 서비스가 무음으로 동작
```

**Null 서비스의 활용**:
- 아직 구현되지 않은 시스템이 있어도 게임이 계속 실행됨
- 개발 중 특정 시스템을 조용히 비활성화 가능
- 디버깅 시 귀를 찢는 사운드 루프 방지

---

### 5단계: 로깅 데코레이터

**문제**: 특정 시스템의 API 호출만 선택적으로 로깅하고 싶다.

**해결**: 동일한 인터페이스를 구현하는 데코레이터로 기존 서비스를 감싼다:

```cpp
class LoggedAudio : public Audio
{
public:
    LoggedAudio(Audio& wrapped) : wrapped_(wrapped) {}

    virtual void playSound(int soundID)
    {
        log("play sound");           // 로깅
        wrapped_.playSound(soundID); // 실제 동작에 위임
    }

    virtual void stopSound(int soundID)
    {
        log("stop sound");
        wrapped_.stopSound(soundID);
    }

    virtual void stopAllSounds()
    {
        log("stop all sounds");
        wrapped_.stopAllSounds();
    }

private:
    void log(const char* message)
    {
        // 로그 기록
    }

    Audio& wrapped_;  // 실제 서비스에 대한 참조
};
```

**로깅 활성화/비활성화**:

```cpp
void enableAudioLogging()
{
    // 기존 서비스를 LoggedAudio로 감싸서 교체
    Audio* service = new LoggedAudio(Locator::getAudio());
    Locator::provide(service);
}

void disableAudioLogging()
{
    // 원래 서비스로 복구 (or 다시 null 서비스로)
    Locator::provide(originalAudio);
}
```

**데코레이터 체이닝**:

```
실제 동작:  [LoggedAudio] → [ConsoleAudio] → 하드웨어
로그 없음:  [ConsoleAudio] → 하드웨어
무음 모드:  [NullAudio] → 아무것도 안 함
로그 + 무음: [LoggedAudio] → [NullAudio] → 로그만, 소리 없음
```

---

## 설계 결정 (Design Decisions)

### 1. 서비스를 어떻게 찾는가?

| 방식 | 장점 | 단점 |
|---|---|---|
| **외부 코드가 등록** (권장) | 빠름, 단순, 런타임에 교체 가능, 생성자 파라미터 제어 가능 | 초기화 순서 의존 (등록 전 사용 시 실패) |
| **컴파일 타임 바인딩** | 가장 빠름, 서비스 보장 | 재컴파일 필요, 런타임 교체 불가 |
| **런타임 설정 파일** | 재컴파일 없이 교체, 비프로그래머 변경 가능 | 복잡, 느림 (리플렉션 비용) |

```cpp
// 컴파일 타임 바인딩 예시
class Locator
{
private:
#if DEBUG
    static DebugAudio   service_;
#else
    static ReleaseAudio service_;
#endif
public:
    static Audio& getAudio() { return service_; }
};
```

---

### 2. 서비스를 찾지 못했을 때?

| 방식 | 특징 |
|---|---|
| **NULL 반환** | 사용자가 직접 처리. 모든 호출 지점에서 null 체크 필요 |
| **assert로 중단** | 로케이터의 계약 위반을 명시. 출시 게임에서 현실적 |
| **Null 서비스 반환** (권장) | 사용자 코드 단순화. 개발 중 편리. 의도치 않은 누락 디버깅이 어려울 수 있음 |

```cpp
// assert 방식
static Audio& getAudio()
{
    Audio* service = nullptr;
    // ... 탐색 ...
    assert(service != nullptr);  // "서비스 없음 = 로케이터 버그"
    return *service;
}
```

---

### 3. 서비스의 접근 범위

| 방식 | 장점 | 단점 |
|---|---|---|
| **전역 접근** | 코드베이스 전체가 동일 서비스 사용. 중복 인스턴스 방지 | 어디서든 사용 가능 → 의존성 추적 어려움 |
| **클래스 범위 제한** | 커플링 제어. 특정 도메인만 접근 | 무관한 클래스들이 각자 참조 보유 가능 |

```cpp
// 접근을 특정 클래스 계층으로 제한
class OnlineGameBase
{
protected:
    static NetworkService& getNetwork() { return *service_; }

private:
    static NetworkService* service_;
};

// OnlineGameBase를 상속한 클래스만 NetworkService에 접근 가능
```

**가이드라인**: 특정 도메인에만 사용되는 서비스(네트워크 등)는 해당 클래스 계층으로 제한하고, 광범위하게 사용되는 서비스(로깅 등)는 전역으로 둔다.

---

## 싱글턴과의 비교

| | 싱글턴 | 서비스 로케이터 |
|---|---|---|
| **구체 클래스 커플링** | 있음 (클래스 자체가 싱글턴) | 없음 (인터페이스만 노출) |
| **교체 가능성** | 어려움 | 쉬움 (`provide()` 한 번) |
| **Null 서비스** | 불가 | 가능 |
| **테스트 편의** | 어려움 | 쉬움 (Mock 서비스 주입) |
| **런타임 교체** | 불가 | 가능 |
| **복잡도** | 낮음 | 중간 |

---

## 전체 구조 요약

```
게임 시작:
  Locator::initialize()       → null 서비스로 기본 설정
  Locator::provide(new ConsoleAudio()) → 실제 서비스 등록

개발 중 디버깅:
  enableAudioLogging()        → LoggedAudio 데코레이터 적용

특정 기능 비활성화:
  Locator::provide(nullptr)   → null 서비스로 복귀

호출 코드 (어디서든):
  Audio& audio = Locator::getAudio();
  audio.playSound(BANG);
  // ConsoleAudio, LoggedAudio, NullAudio 중 뭔지 알 필요 없음
```

---

## 주의사항 (Keep in Mind)

**의존성이 런타임까지 드러나지 않는다**: 싱글턴이나 정적 클래스는 코드만 봐도 의존성이 명확하다. 서비스 로케이터는 런타임에 바인딩되므로, 코드만 봐서는 어떤 서비스가 실제로 사용되는지 알기 어렵다.

**아껴서 사용하라**: 어떤 코드든 접근 가능하게 만드는 것은 항상 위험하다. 대부분의 경우 **의존성 주입(Dependency Injection)** — 객체를 파라미터로 직접 전달하는 것 — 이 더 명확하고 테스트하기 쉽다. 서비스 로케이터는 진정으로 "앰비언트(ambient)"한 시스템, 즉 게임 환경의 기본 속성 같은 것(오디오, 로깅)에만 사용하라.

---

## 핵심 정리

| 개념 | 설명 |
|---|---|
| **서비스 로케이터** | 서비스의 구체 타입과 위치 탐색 과정을 숨기는 전역 접근 객체 |
| **Null Object 패턴** | 서비스 없을 때 NULL 대신 "아무것도 안 하는" 구현 반환. 호출자 보호 |
| **데코레이터 (Decorator)** | 기존 서비스를 감싸 추가 동작(로깅 등) 삽입. 서비스 교체 불필요 |
| **의존성 주입** | 서비스를 전역에서 끌어오지 않고 파라미터로 전달. 더 명확하지만 번거로움 |
| **컴파일 타임 바인딩** | 빌드 시 서비스를 결정. 빠르지만 유연성 없음 |
| **런타임 등록** | 초기화 코드가 서비스를 주입. 유연하지만 초기화 순서 주의 |

---

## 관련 패턴 (See Also)

| 패턴 | 관계 |
|---|---|
| **Singleton (GoF)** | 형제 패턴. 둘 다 전역 접근 제공. Service Locator는 구체 타입으로부터 추가 디커플링 |
| **Decorator (GoF)** | LoggedAudio처럼 서비스에 기능을 덧씌우는 데 사용. 서비스 교체 없이 동작 추가 |
| **Null Object** | 서비스를 찾지 못했을 때 NULL 대신 반환하는 패턴 |
| **Component** | Unity의 `GetComponent()`가 Component + Service Locator의 결합 |
| **Subclass Sandbox** | 기반 클래스가 서비스 로케이터를 통해 서비스를 획득해 파생 클래스에 제공 |

---

*© 2009-2015 Robert Nystrom*
