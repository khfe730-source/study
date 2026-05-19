# Chapter 3: 플라이웨이트 패턴 (Flyweight Pattern)

> **Game Programming Patterns** — Robert Nystrom 저  
> Part II: Design Patterns Revisited / Chapter 3

---

## 의도 (Intent)

> 공유(sharing)를 통해 많은 수의 세밀한 객체를 효율적으로 지원한다.

**핵심 아이디어**: 다수의 객체 사이에서 **공통 상태(intrinsic state)** 를 단 하나의 인스턴스로 공유해 메모리와 처리 비용을 절감한다.

---

## 동기 (Motivation): 숲을 렌더링하기

수천 그루의 나무로 가득 찬 숲을 실시간으로 렌더링해야 한다고 가정하자.

```cpp
class Tree
{
private:
    Mesh      mesh_;      // 폴리곤 메시 (대용량)
    Texture   bark_;      // 나무껍질 텍스처 (대용량)
    Texture   leaves_;    // 잎 텍스처 (대용량)
    Vector    position_;  // 월드 내 위치
    double    height_;    // 키
    double    thickness_; // 두께
    Color     barkTint_;  // 나무껍질 색조
    Color     leafTint_;  // 잎 색조
};
```

나무가 수천 그루라면 `mesh_`, `bark_`, `leaves_` 같은 **대용량 데이터가 인스턴스마다 중복** 된다.  
이 데이터 전체를 매 프레임 CPU → GPU 버스로 전송하는 것은 현실적으로 불가능하다.

**핵심 관찰**: 수천 그루의 나무는 **거의 동일한 메시와 텍스처** 를 사용한다.  
즉, 대부분의 데이터는 모든 인스턴스에 걸쳐 **동일하다**.

---

## 해결: 상태 분리

객체의 데이터를 두 종류로 분리한다.

### 내재적 상태 (Intrinsic State) — "문맥 독립적(context-free) 데이터"

모든 인스턴스가 **공유**할 수 있는, 인스턴스에 종속되지 않는 데이터.

```cpp
class TreeModel
{
private:
    Mesh    mesh_;   // 공유 메시
    Texture bark_;   // 공유 텍스처
    Texture leaves_; // 공유 텍스처
};
```

게임 전체에서 이 객체는 **단 하나만** 존재한다.

### 외재적 상태 (Extrinsic State) — "인스턴스 고유 데이터"

각 인스턴스마다 **다른** 값을 가지는 데이터.

```cpp
class Tree
{
private:
    TreeModel* model_;    // 공유 모델 포인터 (단 하나를 참조)
    Vector     position_; // 고유 위치
    double     height_;   // 고유 크기
    double     thickness_;
    Color      barkTint_; // 고유 색조
    Color      leafTint_;
};
```

```
TreeModel (1개)          Tree 인스턴스들
┌──────────────┐        ┌──────────┐
│ mesh_        │ ◀───── │ model_*  │
│ bark_        │ ◀───── │ model_*  │
│ leaves_      │ ◀───── │ model_*  │
└──────────────┘        │ position_│
                        │ height_  │
                        │ ...      │
                        └──────────┘
```

---

## GPU 인스턴스 렌더링 (Instanced Rendering)

GPU 수준에서도 동일한 원리를 활용할 수 있다. Direct3D, OpenGL 모두 **인스턴스 렌더링(instanced rendering)** 을 지원한다.

| 스트림 | 내용 |
|---|---|
| **공유 데이터 스트림** | `TreeModel` — 메시와 텍스처, 한 번만 전송 |
| **인스턴스 데이터 스트림** | 각 나무의 위치, 색조, 크기 |

단 하나의 드로우 콜(draw call)로 숲 전체를 렌더링한다.

> 플라이웨이트 패턴은 GoF 패턴 중 **실제 GPU 하드웨어가 직접 지원하는 유일한 패턴** 일 수 있다.

---

## 활용 사례 2: 지형 타일 (Terrain Tiles)

### 나쁜 구현 — enum + switch

```cpp
enum Terrain { TERRAIN_GRASS, TERRAIN_HILL, TERRAIN_RIVER };

class World
{
private:
    Terrain tiles_[WIDTH][HEIGHT];
};

int World::getMovementCost(int x, int y)
{
    switch (tiles_[x][y])
    {
        case TERRAIN_GRASS: return 1;
        case TERRAIN_HILL:  return 3;
        case TERRAIN_RIVER: return 2;
    }
}

bool World::isWater(int x, int y)
{
    switch (tiles_[x][y])
    {
        case TERRAIN_GRASS: return false;
        case TERRAIN_HILL:  return false;
        case TERRAIN_RIVER: return true;
    }
}
```

**문제점**: 지형 타입에 관한 데이터가 여러 메서드에 흩어져 있다. 객체지향의 캡슐화 원칙을 위반한다.

---

### 좋은 구현 — Flyweight 적용

**1단계 — Terrain 클래스 정의 (불변 객체)**

```cpp
class Terrain
{
public:
    Terrain(int movementCost, bool isWater, Texture texture)
        : movementCost_(movementCost),
          isWater_(isWater),
          texture_(texture)
    {}

    int           getMovementCost() const { return movementCost_; }
    bool          isWater()         const { return isWater_; }
    const Texture& getTexture()     const { return texture_; }

private:
    int     movementCost_;
    bool    isWater_;
    Texture texture_;
};
```

> 모든 메서드가 `const`인 것에 주목하라.  
> 하나의 객체가 여러 컨텍스트에서 공유되므로, **플라이웨이트 객체는 거의 항상 불변(immutable)** 이어야 한다.  
> 만약 수정이 가능하다면 공유하는 모든 곳에 영향이 미친다.

**2단계 — World가 Terrain 인스턴스를 직접 보유**

```cpp
class World
{
public:
    World()
        : grassTerrain_(1, false, GRASS_TEXTURE),
          hillTerrain_ (3, false, HILL_TEXTURE),
          riverTerrain_(2, true,  RIVER_TEXTURE)
    {}

private:
    Terrain grassTerrain_;
    Terrain hillTerrain_;
    Terrain riverTerrain_;

    Terrain* tiles_[WIDTH][HEIGHT]; // 포인터 그리드
};
```

**3단계 — 지형 생성 시 포인터 할당**

```cpp
void World::generateTerrain()
{
    for (int x = 0; x < WIDTH; x++)
    {
        for (int y = 0; y < HEIGHT; y++)
        {
            tiles_[x][y] = (random(10) == 0)
                           ? &hillTerrain_
                           : &grassTerrain_;
        }
    }
    // 강 한 줄 그리기
    int x = random(WIDTH);
    for (int y = 0; y < HEIGHT; y++)
        tiles_[x][y] = &riverTerrain_;
}
```

**4단계 — 깔끔한 API로 접근**

```cpp
const Terrain& World::getTile(int x, int y) const
{
    return *tiles_[x][y];
}

// 사용 측
int cost = world.getTile(2, 3).getMovementCost();
```

```
World
┌─────────────────────────────────────────┐
│  grassTerrain_  hillTerrain_  river...  │
│  (Terrain 객체, 각 1개씩)                 │
│                                         │
│  tiles_[WIDTH][HEIGHT]                  │
│  [ &grass, &grass, &hill, &river, ... ] │
└─────────────────────────────────────────┘
```

---

## enum 방식과 비교

| 항목 | enum + switch | Flyweight (Terrain 객체) |
|---|---|---|
| **데이터 응집도** | 메서드마다 흩어짐 | 클래스에 캡슐화 |
| **확장성** | 새 타입 추가 시 모든 switch 수정 | 새 Terrain 인스턴스만 추가 |
| **메모리** | 타일당 enum 값 1개 | 타일당 포인터 1개 |
| **성능** | 직접 접근 | 포인터 간접 참조 (캐시 미스 가능) |
| **가독성** | 낮음 | 높음 |

> 성능에 대한 황금 원칙: **먼저 프로파일링하라(profile first)**.  
> 저자의 테스트에서 플라이웨이트는 enum 방식보다 오히려 빠르게 나왔다.  
> 메모리 배치에 따라 결과가 달라지므로, 추측으로 최적화하지 말 것.  
> 캐시 미스에 대한 자세한 내용은 **Data Locality 패턴** 참고.

---

## Type Object 패턴과의 차이

플라이웨이트와 **Type Object 패턴**은 구조가 유사해 보이지만 **의도가 다르다**.

| 항목 | Flyweight | Type Object |
|---|---|---|
| **목적** | 메모리/성능 효율 | 클래스 수 최소화, 타입 시스템 확장 |
| **공유** | 주 목적 | 부수적 이점 |
| **관점** | 최적화 | 설계 구조 |

---

## 패턴 구조 요약

```
        ┌──────────────────────────┐
        │      Flyweight           │
        │  (내재적 상태만 포함)      │
        │  - intrinsicState        │
        │  + operation(extrinsic)  │
        └──────────────────────────┘
                    ▲
        ┌───────────┴───────────┐
        │                       │
  Client (외재적 상태 보유)    FlyweightFactory
  - extrinsicState             - flyweights[]
  - *flyweight                 + getFlyweight()
```

---

## 언제 사용할까? (When to Use It)

- 매우 **많은 수의 동일하거나 유사한 객체** 를 생성해야 할 때
- 객체 생성 비용이 **메모리** 또는 **전송 대역폭** 측면에서 클 때
- 객체의 대부분 상태를 **외재적(extrinsic)** 으로 만들 수 있을 때
- enum + switch 구조를 사용하고 있고, 이를 객체지향적으로 리팩터링하고 싶을 때

---

## 주의사항 (Keep in Mind)

- **플라이웨이트 객체는 불변(immutable)이어야 한다.** 공유 상태를 수정하면 해당 플라이웨이트를 참조하는 모든 곳에 영향이 미친다.
- 외재적 상태를 **클라이언트가 직접 관리**해야 하므로, 설계가 다소 복잡해진다.
- 포인터 역참조(pointer dereference)로 인한 **캐시 미스** 가능성을 인지해야 한다.

---

## 관련 패턴 (See Also)

| 패턴 | 관계 |
|---|---|
| **Factory Method** (GoF) | 플라이웨이트를 온디맨드(on-demand)로 생성할 때, 이미 존재하는 인스턴스를 먼저 찾아 반환하는 생성 로직을 캡슐화 |
| **Object Pool** | 이미 생성된 플라이웨이트 인스턴스 풀을 관리하는 데 활용 |
| **Type Object** | 구조적으로 유사하나 의도가 다름. 클래스 수 최소화가 목적 |
| **State** (GoF) | 상태 머신에서 상태 객체가 인스턴스별 필드 없이 메서드와 ID만 가진다면, 동일 상태 객체를 여러 상태 머신에서 공유 가능 |
| **Data Locality** | 포인터 간접 참조로 인한 캐시 미스 문제를 다루는 패턴 |

---

## 핵심 정리

| 개념 | 설명 |
|---|---|
| **내재적 상태 (Intrinsic State)** | 인스턴스 간 공유 가능한 문맥 독립적 데이터. 플라이웨이트 객체 내부에 저장 |
| **외재적 상태 (Extrinsic State)** | 인스턴스마다 고유한 데이터. 클라이언트가 보유하고 플라이웨이트에 전달 |
| **불변성 (Immutability)** | 공유 객체는 반드시 불변이어야 한다 |
| **인스턴스 렌더링 (Instanced Rendering)** | GPU가 하드웨어 수준에서 플라이웨이트 원리를 구현한 방식 |
| **프로파일 우선 (Profile First)** | 최적화 전 반드시 측정하라. 직관과 실제 성능은 다를 수 있다 |

---

*© 2009-2015 Robert Nystrom*
