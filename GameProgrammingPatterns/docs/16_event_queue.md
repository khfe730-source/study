# Chapter 15: 이벤트 큐 패턴 (Event Queue Pattern)

> **Game Programming Patterns** — Robert Nystrom 저  
> Part V: Decoupling Patterns / Chapter 15

---

## 의도 (Intent)

> 메시지나 이벤트가 **보내지는 시점**과 **처리되는 시점**을 디커플링한다.

---

## 동기 (Motivation)

### 동기식(Synchronous) 오디오 API의 세 가지 문제

게임에 사운드를 추가하는 단순한 API를 만들었다:

```cpp
class Audio
{
public:
    static void playSound(SoundId id, int volume);
};

void Audio::playSound(SoundId id, int volume)
{
    ResourceId resource = loadSound(id);    // 디스크에서 로드
    int channel = findOpenChannel();
    if (channel == -1) return;
    startSound(resource, channel, volume);  // 재생 시작
}
```

메뉴, AI, 전투 코드 곳곳에서 `playSound()`를 호출한다. 그러자 문제가 드러났다:

---

**문제 1: API가 호출자를 블로킹한다**

```
playSound() 호출
    → 사운드 파일 디스크에서 로드 (수십~수백 ms)
    → 로드 완료 후에야 반환
    → 그 동안 게임 전체 프리즈 (프레임 드롭)
```

**문제 2: 요청을 일괄 처리(aggregate)할 수 없다**

```
영웅이 강력한 공격으로 두 적을 동시에 타격
    → 같은 프레임에 동일한 고통 사운드 두 번 재생
    → 파형 합산 = 두 배 음량 → 귀를 찌르는 소음

보스전: 수십 마리 미니언이 한꺼번에 소리 재생 요청
    → 하드웨어 채널 한계 초과 → 사운드 무시/끊김
```

**문제 3: 요청이 잘못된 스레드에서 처리된다**

```
멀티코어 게임 엔진:
    - AI 스레드  → playSound() 호출
    - 렌더 스레드 → playSound() 호출
    - 오디오 스레드 → 놀고 있음!

API가 동기적 → 호출한 스레드에서 즉시 처리
→ 스레드 동기화 없음 → 레이스 컨디션 (race condition)
```

---

### 근본 원인과 해결 방향

모든 문제의 공통 원인: `playSound()`가 "지금 당장 처리하라"는 의미이기 때문이다.

```
문제:  [호출자] ──직접 처리──▶ [오디오 엔진]   (시간 커플링)

해결:  [호출자] ──큐에 추가──▶ [큐] ──나중에──▶ [오디오 엔진]
                               (시간 디커플링)
```

---

## 패턴 구조 (The Pattern)

큐가 알림이나 요청을 **선입선출(FIFO)** 순서로 저장한다. 알림을 보내면 요청을 큐에 넣고 즉시 반환한다. 요청 처리자는 이후 편리한 시점에 큐에서 항목을 꺼내 처리한다. 이로써 발신자와 수신자를 **구조적(정적)으로도, 시간적으로도** 디커플링한다.

---

## 예제 코드 (Sample Code)

### Step 1: 요청 구조체 정의

```cpp
struct PlayMessage
{
    SoundId id;
    int volume;
};
```

### Step 2: 단순 배열 큐

```cpp
class Audio
{
public:
    static void init()   { numPending_ = 0; }

    static void playSound(SoundId id, int volume)
    {
        assert(numPending_ < MAX_PENDING);
        pending_[numPending_].id     = id;
        pending_[numPending_].volume = volume;
        numPending_++;
        // 즉시 반환 — 블로킹 없음
    }

    static void update()  // 게임 루프에서 매 프레임 호출
    {
        for (int i = 0; i < numPending_; i++)
        {
            ResourceId resource = loadSound(pending_[i].id);
            int channel = findOpenChannel();
            if (channel == -1) return;
            startSound(resource, channel, pending_[i].volume);
        }
        numPending_ = 0;
    }

private:
    static const int MAX_PENDING = 16;
    static PlayMessage pending_[MAX_PENDING];
    static int numPending_;
};
```

**왜 배열인가?**

| 자료 구조 | 이유 |
|---|---|
| 연결 리스트 (❌) | 동적 할당, 포인터 오버헤드, 캐시 비친화적 |
| 피보나치 힙 (❌) | 복잡한 구현, 과도한 엔지니어링 |
| **단순 배열 (✅)** | 동적 할당 없음, 캐시 친화적 연속 메모리, 단순 구현 |

---

### Step 3: 링 버퍼(Ring Buffer) — 진정한 큐

단순 배열은 `update()`가 한 번에 전부 처리해야 한다. 한 번에 하나씩 처리하려면 진짜 큐가 필요하다.

**링 버퍼 개념**:

```
초기 상태:
  [_][_][_][_][_][_][_][_]
   ↑
  head = tail = 0

3개 추가 후:
  [A][B][C][_][_][_][_][_]
   ↑        ↑
  head=0   tail=3

A 처리 후:
  [_][B][C][_][_][_][_][_]
      ↑    ↑
    head=1 tail=3

가득 찼을 때 tail이 끝에 도달하면 → 앞으로 되감기(wrap around)!

  [F][G][_][D][E]
      ↑    ↑
    tail  head
```

```cpp
class Audio
{
public:
    static void init()
    {
        head_ = 0;
        tail_ = 0;
    }

    static void playSound(SoundId id, int volume)
    {
        // 큐가 꽉 찼는지 확인 (tail이 head를 따라잡으면 오버플로우)
        assert((tail_ + 1) % MAX_PENDING != head_);

        pending_[tail_].id     = id;
        pending_[tail_].volume = volume;
        tail_ = (tail_ + 1) % MAX_PENDING;  // 모듈러 증가로 wrap-around
    }

    static void update()
    {
        if (head_ == tail_) return;  // 빈 큐

        ResourceId resource = loadSound(pending_[head_].id);
        int channel = findOpenChannel();
        if (channel == -1) return;
        startSound(resource, channel, pending_[head_].volume);

        head_ = (head_ + 1) % MAX_PENDING;  // 처리 후 head 전진
    }

private:
    static const int MAX_PENDING = 16;
    static PlayMessage pending_[MAX_PENDING];
    static int head_;
    static int tail_;
};
```

**링 버퍼 동작 원리**:

```
     head                    tail
      ↓                       ↓
[_][_][D][E][F][G][H][I][J][_][_][_][_][_][_][_]
        ← 처리 대기 중 →

tail이 배열 끝에 도달하면 인덱스 0으로 되돌아감:
tail = (tail + 1) % MAX_PENDING

이렇게 하면 배열이 원형으로 작동:
[K][L][_][D][E][F]...
 ↑       ↑
tail    head
```

**링 버퍼의 장점**:
- 동적 할당 없음
- 원소 이동(shift) 없음 — O(1) 삽입/삭제
- 캐시 친화적 연속 메모리

---

### Step 4: 요청 병합(Aggregation)

```cpp
void Audio::playSound(SoundId id, int volume)
{
    // 이미 큐에 같은 사운드 요청이 있으면 병합
    for (int i = head_; i != tail_; i = (i + 1) % MAX_PENDING)
    {
        if (pending_[i].id == id)
        {
            // 더 큰 볼륨만 유지
            pending_[i].volume = max(volume, pending_[i].volume);
            return;  // 새 항목 추가 불필요
        }
    }

    // 새 요청 추가 (링 버퍼 코드...)
}
```

같은 프레임에 동일한 사운드가 여러 번 요청되면 가장 큰 볼륨 하나로 합쳐진다. 이 병합은 **enqueue 시점**에 이루어진다.

---

### Step 5: 멀티스레드 안전

```cpp
// 핵심: 큐 수정 연산을 상호 배제(mutual exclusion)로 보호
void Audio::playSound(SoundId id, int volume)
{
    lock(queueMutex_);      // 잠금 획득
    // ... 큐에 추가 ...
    unlock(queueMutex_);    // 잠금 해제
    // 잠금 구간이 매우 짧아 블로킹 최소화
}

void Audio::update()        // 오디오 전용 스레드에서 실행
{
    waitForCondition();     // 큐에 항목이 생길 때까지 대기 (CPU 낭비 없음)
    lock(queueMutex_);
    // ... 큐에서 꺼내기 ...
    unlock(queueMutex_);
    // 실제 사운드 재생 (잠금 밖에서)
}
```

---

## 전체 구조 다이어그램

```
[UI 코드]   [AI 코드]   [전투 코드]   [물리 코드]
    │            │            │             │
    └────────────┴────────────┴─────────────┘
                         │
                  playSound(id, volume)
                  (즉시 반환, 논블로킹)
                         │
                         ▼
        ┌────────────────────────────────┐
        │         Event Queue            │
        │  [req1][req2][req3][req4]...   │
        │   head→               ←tail   │
        └────────────────────────────────┘
                         │
                  update() — 편리한 시점에 처리
                  (게임 루프 or 오디오 전용 스레드)
                         │
                         ▼
                  [오디오 엔진]
                  - 중복 병합
                  - 우선순위 처리
                  - 채널 관리
```

---

## 설계 결정 (Design Decisions)

### 1. 이벤트(Event) vs 메시지(Message)

| 구분 | 의미 | 청취자 수 | 범위 |
|---|---|---|---|
| **이벤트** | "이미 발생한 일" (예: "몬스터 사망") | 여러 청취자 (비동기 Observer) | 전역/넓음 |
| **메시지** | "앞으로 할 일" (예: "사운드 재생") | 단일 청취자 (비동기 API) | 좁음/단일 서비스 |

---

### 2. 큐에서 누가 읽는가?

| 방식 | 특징 | 적합한 경우 |
|---|---|---|
| **단일 수신 (Single-cast)** | 큐가 특정 클래스의 구현 세부 사항. 캡슐화 높음 | 오디오 엔진처럼 단일 서비스 API |
| **브로드캐스트 (Broadcast)** | 모든 청취자가 이벤트를 받음. 청취자 없으면 이벤트 소실 | 전역 이벤트 버스 |
| **작업 큐 (Work Queue)** | 여러 청취자 중 하나가 처리. 스케줄링 필요 | 스레드 풀에 작업 분배 |

---

### 3. 큐에 누가 쓰는가?

| 방식 | 특징 |
|---|---|
| **단일 발신자** | 이벤트 출처가 명확. Observer 패턴의 비동기 버전 |
| **다중 발신자** | 어디서든 큐에 접근 가능. 피드백 루프 위험 증가. 이벤트에 발신자 정보 포함 필요 |

---

### 4. 큐 내 객체의 생명주기

| 방식 | 설명 | C++ 타입 |
|---|---|---|
| **소유권 이전** | 큐가 메시지 소유. 수신자가 해제 | `unique_ptr<T>` |
| **공유 소유권** | 참조 카운팅. 아무도 없을 때 자동 해제 | `shared_ptr<T>` |
| **큐 소유** | 메시지가 큐 내부 슬롯에 존재. Object Pool과 동일 구조 | 링 버퍼 배열 |

---

## 주의사항 (Keep in Mind)

### 전역 이벤트 큐 = 전역 변수의 위험

"중앙 이벤트 버스"는 강력하지만 전역 상태의 모든 문제를 안고 있다. 코드베이스 어디서든 이벤트를 보내고 받을 수 있어 의존성이 은밀하게 생긴다.

### 이벤트 처리 시 세계 상태가 달라져 있을 수 있다

```
프레임 N: "Entity #42 사망" 이벤트를 큐에 추가
프레임 N+3: 이벤트 처리 시점

→ Entity #42는 이미 메모리에서 해제됨!
→ 주변 적들도 이미 이동함!
```

**해결**: 큐에 넣을 때 필요한 데이터를 복사해서 함께 저장한다. 동기식과 달리 "무언가 발생했다"만으로는 부족하다.

### 피드백 루프 주의

```
A가 이벤트 발송
    → B가 수신, 이벤트 발송
        → A가 수신, 이벤트 발송
            → B가 수신...  (무한 루프)

동기 시스템: 스택 오버플로우 → 즉시 크래시 (발견 쉬움)
비동기 큐:  스택은 정상 → 이벤트가 조용히 쌓임 (발견 어려움)
```

**권고**: 이벤트 처리 중에는 같은 큐에 이벤트를 보내지 않는다.

---

## Push vs Pull 모델

```
Push 모델 (호출자가 밀어 넣음):
[A 코드] → playSound() → 즉시 처리 (동기)
           "지금 처리해줘"

Pull 모델 (처리자가 당겨 가져옴):
[A 코드] → playSound() → 큐에 추가
                          ...나중에...
[오디오 엔진] → update() → 큐에서 꺼내 처리
               "내가 편할 때 처리할게"

큐 = Push와 Pull 사이의 버퍼
```

큐는 처리자(수신자)에게 제어권을 준다 — 처리 지연, 병합, 폐기 모두 가능. 단, 발신자는 응답을 받을 수 없다. 응답이 필요하다면 큐는 맞지 않는다.

---

## 핵심 정리

| 개념 | 설명 |
|---|---|
| **이벤트 큐 (Event Queue)** | 요청/알림을 저장했다가 나중에 처리하는 FIFO 버퍼 |
| **링 버퍼 (Ring Buffer)** | 배열을 원형으로 사용해 동적 할당 없이 큐를 구현하는 기법 |
| **시간 디커플링** | 이벤트를 보내는 시점과 처리하는 시점을 분리. Observer는 구조적 디커플링만 제공 |
| **요청 병합 (Aggregation)** | 동일한 요청이 여러 번 들어오면 하나로 합쳐 처리 |
| **브로드캐스트 큐** | 하나의 이벤트를 등록된 모든 청취자에게 전달 |
| **작업 큐 (Work Queue)** | 여러 처리자 중 하나에게 작업을 분배. 스레드 풀 패턴의 기반 |
| **피드백 루프** | 이벤트 처리 중 같은 이벤트를 재발송해 무한 순환하는 위험 |

---

## 관련 패턴 (See Also)

| 패턴 | 관계 |
|---|---|
| **Observer (GoF)** | 이벤트 큐의 비동기 버전. Observer는 동기적, Event Queue는 시간 디커플링 추가 |
| **Command (GoF)** | "메시지"는 커맨드와 개념적으로 동일. 커맨드를 큐에 담아 나중에 실행 |
| **Update Method** | `update()`에서 큐를 처리하는 패턴. 두 패턴이 함께 사용됨 |
| **Object Pool** | 큐의 백킹 스토어를 오브젝트 풀로 구현하면 메모리 할당 비용 제거 |
| **State (GoF)** | 상태 머신이 비동기적으로 입력에 반응하려면 입력 이벤트를 큐에 담아 처리 |
| **Service Locator** | 어떤 코드가 처리하든 상관없이 메시지를 보내는 경우, Service Locator와 유사한 구조 |

---

## 실제 사례

| 시스템 | 이벤트 큐 |
|---|---|
| **OS GUI 이벤트 루프** | `getNextEvent()` — 사용자 입력 큐 |
| **Go 언어** | 내장 `channel` 타입 — 고루틴 간 메시지 큐 |
| **Actor 모델** | 각 액터가 메일박스(큐)를 보유. Erlang, Akka 등 |
| **pub/sub 시스템** | Redis Pub/Sub, RabbitMQ 등 분산 이벤트 큐 |
| **게임 중앙 이벤트 버스** | 전투 시스템 → 큐 → 튜토리얼 시스템 (서로 모름) |

---

*© 2009-2015 Robert Nystrom*
