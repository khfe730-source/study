# Chapter 8: 더블 버퍼 패턴 (Double Buffer Pattern)

> **Game Programming Patterns** — Robert Nystrom 저  
> Part III: Sequencing Patterns / Chapter 8

---

## 의도 (Intent)

> 일련의 순차적 연산이 순간적으로 또는 동시에 발생하는 것처럼 보이게 한다.

---

## 동기 (Motivation)

컴퓨터는 본질적으로 **순차적(sequential)** 으로 동작한다. 하지만 사용자는 여러 작업이 한 번에 발생하는 것처럼 보이길 원한다.

렌더링이 대표적인 예다. 게임이 화면을 픽셀 단위로 순서대로 그리는 동안, 디스플레이 드라이버가 그 버퍼를 동시에 읽는다면 **테어링(tearing)** 이 발생한다.

---

## 컴퓨터 그래픽의 동작 원리

모니터는 **프레임버퍼(framebuffer)** — 메모리의 픽셀 배열 — 를 읽어 화면을 그린다.  
초당 60회 스캔하며, 각 픽셀의 색상값을 이 배열에서 읽는다.

**문제: 테어링(Tearing)**

```
게임이 프레임버퍼에 픽셀을 순서대로 쓰는 동안
비디오 드라이버가 동시에 같은 버퍼를 읽으면...

[미완성 얼굴: 눈은 그려졌고, 입은 아직 안 그려짐]
                   ↑
            드라이버가 이 시점에 읽으면
            반만 그려진 얼굴이 화면에 출력됨
```

**해결 아이디어 — 두 개의 무대(Stage A / Stage B):**

- 관객(디스플레이)은 **무대 A**를 본다
- 그 사이 **무대 B**에서 다음 장면을 몰래 준비한다
- 준비 완료 → 조명을 **순간적으로** 전환한다
- 이제 **무대 B**가 보이고, **무대 A**에서 그 다음 장면을 준비한다

이것이 **더블 버퍼**의 원리다.

---

## 패턴 구조 (The Pattern)

```
┌────────────────────────────────────────────────┐
│              Buffered Object                   │
│                                                │
│  current buffer  ←── 외부에서 읽기 (Read)       │
│  next buffer     ←── 내부에서 쓰기 (Write)      │
│                                                │
│  swap() → current ↔ next 즉시 교환             │
└────────────────────────────────────────────────┘
```

| 버퍼 | 역할 |
|---|---|
| **current buffer** | 외부 코드(비디오 드라이버 등)가 읽는 버퍼. 항상 완성된 상태 |
| **next buffer** | 내부에서 점진적으로 수정하는 버퍼. 외부에 노출되지 않음 |
| **swap()** | 수정 완료 후 두 버퍼를 즉시 교환. 원자적(atomic)으로 실행 |

---

## 예제 1: 그래픽 시스템

### 버퍼 클래스

```cpp
class Framebuffer
{
public:
    Framebuffer() { clear(); }

    void clear()
    {
        for (int i = 0; i < WIDTH * HEIGHT; i++)
            pixels_[i] = WHITE;
    }

    void draw(int x, int y)
    {
        pixels_[(WIDTH * y) + x] = BLACK;
    }

    const char* getPixels() { return pixels_; }

private:
    static const int WIDTH  = 160;
    static const int HEIGHT = 120;
    char pixels_[WIDTH * HEIGHT];
};
```

### 더블 버퍼를 적용한 Scene

```cpp
class Scene
{
public:
    Scene()
        : current_(&buffers_[0]),
          next_   (&buffers_[1])
    {}

    void draw()
    {
        next_->clear();    // next 버퍼에 그린다

        next_->draw(1, 1);
        next_->draw(4, 1);
        next_->draw(1, 3);
        next_->draw(2, 4);
        next_->draw(3, 4);
        next_->draw(4, 3);

        swap();            // 그리기 완료 후 버퍼 교환
    }

    Framebuffer& getBuffer() { return *current_; } // 항상 current 반환

private:
    void swap()
    {
        Framebuffer* temp = current_;
        current_ = next_;
        next_    = temp;
    }

    Framebuffer  buffers_[2]; // 실제 버퍼 두 개
    Framebuffer* current_;    // 현재 표시 버퍼
    Framebuffer* next_;       // 다음 작업 버퍼
};
```

```
프레임 1: next(B)에 그리기 → swap() → current=B, next=A
프레임 2: next(A)에 그리기 → swap() → current=A, next=B
프레임 3: next(B)에 그리기 → swap() → ...

비디오 드라이버는 항상 current를 읽는다 → 테어링 없음
```

---

## 예제 2: 비(非)그래픽 — 슬랩스틱 AI

더블 버퍼는 그래픽만의 문제가 아니다. **수정 중인 상태를 자기 자신이 읽는 경우**에도 발생한다.

### 문제: 업데이트 순서에 따라 결과가 달라진다

배우 3명(Harry → Baldy → Chump → Harry 순환)이 서로 뺨을 때리는 시스템:

```cpp
class Actor
{
public:
    Actor() : slapped_(false) {}
    virtual void update() = 0;
    void reset()        { slapped_ = false; }
    void slap()         { slapped_ = true; }
    bool wasSlapped()   { return slapped_; }
private:
    bool slapped_;
};

void Stage::update()
{
    for (int i = 0; i < NUM_ACTORS; i++)
    {
        actors_[i]->update(); // 업데이트 중 slap() 호출 가능
        actors_[i]->reset();
    }
}
```

**배열 순서: Harry[0] → Baldy[1] → Chump[2]**
```
Harry 업데이트: slapped → Baldy 뺨 때림
Baldy 업데이트: slapped → Chump 뺨 때림
Chump 업데이트: slapped → Harry 뺨 때림
→ 한 프레임에 전파 완료
```

**배열 순서 변경: Chump[0] → Baldy[1] → Harry[2]**
```
Chump 업데이트: not slapped → 무행동
Baldy 업데이트: not slapped → 무행동
Harry 업데이트: slapped → Baldy 뺨 때림
→ 한 프레임에 하나만 전파
```

> **같은 로직, 다른 결과.** 배열 내 순서가 결과에 영향을 준다.  
> 이것은 업데이트 도중 읽는 상태(`slapped_`)와 쓰는 상태가 같기 때문이다.

---

### 해결: 각 액터에 더블 버퍼 적용

```cpp
class Actor
{
public:
    Actor() : currentSlapped_(false), nextSlapped_(false) {}

    virtual void update() = 0;

    void swap()
    {
        currentSlapped_ = nextSlapped_; // next → current 복사
        nextSlapped_ = false;           // next 초기화
    }

    void slap()         { nextSlapped_ = true; }      // next에 쓰기
    bool wasSlapped()   { return currentSlapped_; }   // current에서 읽기

private:
    bool currentSlapped_; // 읽기용
    bool nextSlapped_;    // 쓰기용
};

void Stage::update()
{
    for (int i = 0; i < NUM_ACTORS; i++)
        actors_[i]->update();   // 모든 액터 업데이트 (next에만 쓰기)

    for (int i = 0; i < NUM_ACTORS; i++)
        actors_[i]->swap();     // 모든 상태 일괄 교환
}
```

이제 업데이트 루프 전체가 끝나고 나서야 상태가 교환된다.  
배열 순서에 관계없이 **항상 같은 결과**가 나온다.

---

## 설계 결정 (Design Decisions)

### 1. 버퍼를 어떻게 교환할 것인가?

#### 방법 A: 포인터/참조 교환 (Pointer Swap)

```cpp
void swap()
{
    Framebuffer* temp = current_;
    current_ = next_;
    next_    = temp;
}
```

| 항목 | 내용 |
|---|---|
| **속도** | 포인터 두 개 교환 — 매우 빠름 |
| **단점** | 외부 코드가 버퍼 내부에 **영구 포인터**를 저장할 수 없음 |
| **주의** | `next` 버퍼의 기존 데이터는 **두 프레임 전** 데이터 (A→B→A 교대 방식) |

> **모션 블러(motion blur)** 효과: 이전 프레임 데이터를 현재 프레임과 블렌딩해 사실적인 잔상 표현 가능. 포인터 스왑 방식에서 자연스럽게 활용됨.

#### 방법 B: 데이터 복사 (Data Copy)

```cpp
void swap()
{
    memcpy(currentBuffer_, nextBuffer_, BUFFER_SIZE);
}
```

| 항목 | 내용 |
|---|---|
| **장점** | `next` 버퍼의 기존 데이터가 **한 프레임 전** 데이터 (더 최신) |
| **단점** | 버퍼가 크면 복사 비용이 크다 |
| **사용 시기** | 상태가 작을 때 (슬랩스틱 예제의 `bool` 하나처럼) |

---

### 2. 버퍼의 단위는 얼마나 세밀한가?

#### 단일 모놀리식 버퍼 (Monolithic Buffer)

```cpp
Framebuffer buffers_[2]; // 그래픽 예제
```

- 교환이 단순하다 (포인터 두 개)
- 대용량 데이터에 적합

#### 객체 분산 버퍼 (Distributed Buffer)

```cpp
bool slapped_[2]; // 각 Actor마다 (슬랩스틱 예제)
```

- 교환 시 모든 객체를 순회해야 한다
- 상태가 논리적으로 각 객체에 속할 때 자연스럽다

#### 최적화: 정적 인덱스를 이용한 분산 버퍼

객체별 포인터 스왑과 분산 버퍼의 장점을 결합한다:

```cpp
class Actor
{
public:
    static void init() { current_ = 0; }
    static void swap() { current_ = next(); }  // 단 한 번 호출로 전체 교환

    void slap()       { slapped_[next()]    = true; }
    bool wasSlapped() { return slapped_[current_]; }

private:
    static int  current_;                // 모든 Actor가 공유하는 현재 인덱스
    static int  next()  { return 1 - current_; }

    bool slapped_[2];  // [current_] = 읽기, [next()] = 쓰기
};
```

`swap()`이 정적 메서드이므로 **한 번만 호출하면 모든 Actor의 상태가 동시에 교환**된다. 포인터 스왑과 동등한 성능을 분산 버퍼에서 달성한다.

---

## 언제 사용할까? (When to Use It)

다음 조건이 모두 해당할 때:

1. **상태가 점진적으로 수정된다**
2. **수정 중에 같은 상태에 접근할 수 있다** (다른 스레드, 인터럽트, 또는 같은 코드)
3. **접근하는 코드에서 작업 중인 상태를 보이지 않게** 하고 싶다
4. **읽기 시 쓰기가 완료될 때까지 기다리고 싶지 않다**

---

## 주의사항 (Keep in Mind)

- **교환 자체도 시간이 든다**: 교환은 원자적이어야 한다. 교환 시간이 수정 시간보다 길면 의미가 없다.
- **메모리가 두 배 필요하다**: 메모리가 제한된 환경에서는 부담이 크다.
- **포인터 스왑 시 영구 포인터 불가**: 외부 코드가 버퍼 내 특정 주소를 저장하면 교환 후 잘못된 데이터를 가리킨다.
- **포인터 스왑 시 데이터는 두 프레임 전**: 버퍼를 재사용할 때 예상보다 오래된 데이터임에 주의한다.

---

## 패턴 요약 다이어그램

```
         쓰기 (Write)          읽기 (Read)
              ↓                     ↓
┌─────────────────────────────────────────────┐
│  [next buffer]          [current buffer]    │
│   (작업 중)              (완성된 상태)        │
│                                             │
│         ←── swap() ───▶                    │
│  (완성 후 포인터/데이터 교환)                  │
└─────────────────────────────────────────────┘

swap() 후:
[current buffer] ← 방금 완성한 next buffer
[next buffer]    ← 이제 다음 프레임을 그릴 공간
```

---

## 핵심 정리

| 개념 | 설명 |
|---|---|
| **테어링 (Tearing)** | 수정 중인 버퍼를 읽어 발생하는 반만 그려진 화면 현상 |
| **프레임버퍼 (Framebuffer)** | 픽셀 색상값을 저장하는 메모리 배열 |
| **포인터 스왑** | 포인터만 교환. 매우 빠르지만 영구 포인터 저장 불가 |
| **데이터 복사** | 실제 데이터를 복사. 느리지만 최신 데이터 유지 |
| **정적 인덱스 기법** | 분산 버퍼에서 단일 정적 변수 교환으로 전체 스왑 성능 달성 |
| **원자적 교환 (Atomic Swap)** | swap() 중에는 어떤 코드도 두 버퍼 중 어느 것도 접근하면 안 됨 |

---

## 관련 패턴 (See Also)

| 패턴 | 관계 |
|---|---|
| **Update Method** | 더블 버퍼와 자주 함께 사용됨. 매 프레임 액터를 업데이트하는 패턴 |
| **Game Loop** | 프레임마다 draw → swap을 호출하는 메인 루프 |
| **Object Pool** | 동적 버퍼 할당 시 메모리 단편화 방지 |

---

## 실제 API 사례

| API | 더블 버퍼 메서드 |
|---|---|
| **OpenGL** | `swapBuffers()` |
| **Direct3D** | Swap Chain |
| **XNA Framework** | `endDraw()` 내부 프레임버퍼 교환 |

---

*© 2009-2015 Robert Nystrom*
