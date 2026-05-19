# 게임 프로그래밍 패턴 — 전체 요약

> Robert Nystrom 저 / 프로그래머 전문가 수준 한국어 정리  
> 원본: *Game Programming Patterns* (2014)

---

## 책 구성 한눈에 보기

```
게임 프로그래밍 패턴
├── 설계 패턴 재방문 (Design Patterns Revisited)
│   ├── Command
│   ├── Flyweight
│   ├── Observer
│   ├── Prototype
│   ├── Singleton
│   └── State
├── 시퀀싱 패턴 (Sequencing Patterns)
│   ├── Double Buffer
│   ├── Game Loop
│   └── Update Method
├── 행동 패턴 (Behavioral Patterns)
│   ├── Bytecode
│   ├── Subclass Sandbox
│   └── Type Object
├── 디커플링 패턴 (Decoupling Patterns)
│   ├── Component
│   ├── Event Queue
│   └── Service Locator
└── 최적화 패턴 (Optimization Patterns)
    ├── Data Locality
    ├── Dirty Flag
    ├── Object Pool
    └── Spatial Partition
```

---

## 각 챕터 요약

### 00. 서론 및 아키텍처 철학
**핵심**: 좋은 아키텍처 = 변경 비용을 최소화하는 것. 유연성과 성능은 트레이드오프. 추상화는 느리다 — 필요할 때만 쓸 것.

---

### 01. 커맨드 (Command)
**핵심**: 요청을 객체로 캡슐화한다.  
**활용**: 입력 리매핑, 실행 취소/재실행(undo/redo), 리플레이, 커맨드 큐.

```cpp
class Command { virtual void execute() = 0; };
class JumpCommand : public Command { void execute() { jump(); } };
```

**기억할 것**: 함수 포인터나 클로저로도 구현 가능. 실행 취소를 위해 `undo()`를 쌍으로 구현.

---

### 02. 플라이웨이트 (Flyweight)
**핵심**: 다수의 객체가 공유할 수 있는 데이터를 분리해 메모리를 절약한다.  
**내재 상태(intrinsic)** = 공유 가능, 불변 → 단일 인스턴스  
**외재 상태(extrinsic)** = 객체마다 다름 → 각 인스턴스 보유

```
나무 1,000그루: 메시/텍스처(공유) + 위치/크기(개별)
```

**활용**: 지형 타일, 파티클, 총알 등 대량 동종 객체.

---

### 03. 옵저버 (Observer)
**핵심**: 객체 간 1:N 의존성. 상태 변화를 구독자들에게 자동 통보.  
**활용**: 업적 시스템, UI 갱신, 이벤트 로깅.

```cpp
class Observer { virtual void onNotify(Entity& e, Event event) = 0; };
class Subject { void addObserver(Observer*); void notify(Entity&, Event); };
```

**주의**: 연결 리스트 옵저버로 동적 할당 없이 구현 가능. 이벤트 루프/액터 모델로 발전.

---

### 04. 프로토타입 (Prototype)
**핵심**: 기존 객체를 복제해 새 객체를 생성한다.  
**코드 패턴**: `clone()` 가상 함수로 타입별 복제.  
**데이터 패턴**: JSON에서 부모 프로토타입을 참조해 스탯 상속 (copy-down inheritance).

```json
{ "name": "Troll Archer", "prototype": "Troll", "attackDamage": 20 }
```

---

### 05. 싱글턴 (Singleton)
**핵심**: 인스턴스를 하나로 제한 + 전역 접근점 제공.  
**문제점**: 전역 상태 오염, 커플링 증가, 멀티스레드 위험, 테스트 어려움.

**대안** (5가지):
1. 정말 하나가 필요한가? → 클래스를 둘로 분리
2. 이미 존재하는 클래스에서 접근
3. 서비스 로케이터 패턴
4. 의존성 주입
5. 기반 클래스에서 제공

---

### 06. 상태 (State)
**핵심**: 내부 상태에 따라 객체의 행동을 바꾼다.

진화 순서:
```
열거형 + switch → GoF State 패턴 → HSM (계층 상태 기계) → Pushdown Automaton
```

- **HSM**: 슈퍼스테이트(superstate)로 공통 행동 상속. 엔트리/엑시트 핸들러.
- **Pushdown**: 스택으로 이전 상태 기억 (총 장전 후 이전 상태로 복귀 등).

---

### 07. 더블 버퍼 (Double Buffer)
**핵심**: 쓰는 버퍼와 읽는 버퍼를 분리해 불완전한 상태 노출을 방지한다.  
**활용**: 렌더링 프레임버퍼 (tearing 방지), AI 슬랩스틱 방지(같은 프레임 상태 격리).

```
Back Buffer (쓰기) ──swap()──→ Front Buffer (읽기/표시)
```

---

### 08. 게임 루프 (Game Loop)
**핵심**: 플랫폼/하드웨어와 무관하게 일정한 속도로 게임을 진행시킨다.

```
while (true) {
    processInput();
    update(elapsed);   // 고정 or 가변 타임스텝
    render(lag/MS_PER_UPDATE);  // 보간 렌더링
}
```

4가지 구현:
1. 최대한 빠르게 (고정 타임스텝)
2. 가변 타임스텝
3. 고정 업데이트 + 가변 렌더링
4. lag 보간 렌더링 ← 권장

---

### 09. 업데이트 메서드 (Update Method)
**핵심**: 각 엔티티가 `update()` 메서드를 가지며, 게임 루프가 모든 엔티티를 매 프레임 순회한다.

**주의사항**:
- 순서 의존성 버그
- `update()` 중 컬렉션 변경 문제
- `deltaTime` 없이 로직 작성 시 프레임레이트 의존성

---

### 10. 바이트코드 (Bytecode)
**핵심**: 인터프리터 패턴의 게임 특화 버전. 스크립트를 바이트코드로 컴파일하고 VM이 실행.

```
스크립트 코드 → 컴파일러 → 바이트코드 → 스택 머신(VM) → 실행
```

**활용**: 스킬/마법 시스템, 모딩 지원, 샌드박스 실행.  
**스택 기반 VM**: `LITERAL`, `GET_HEALTH`, `OP_ADD`, `SET_HEALTH` 등의 명령어.

---

### 11. 하위 클래스 샌드박스 (Subclass Sandbox)
**핵심**: 기반 클래스가 `protected` 연산들을 제공하고, 서브클래스는 그 안에서만 작동한다.

```
Superpower (기반)          SkyLaunch, EarthQuake, ...
├── playSound()            (서브클래스는 외부 시스템에
├── spawnParticles()        직접 접근하지 않음)
└── activate() = 0
```

**효과**: 서브클래스와 외부 시스템의 커플링을 기반 클래스 하나로 집중.

---

### 12. 타입 오브젝트 (Type Object)
**핵심**: 새 클래스를 코드로 추가하지 않고 데이터로 타입을 정의한다.

```
Monster ──→ Breed (타입 정보: maxHealth, attack 등)
Troll1    Dragon Breed
Troll2  ──┘ (공유)
```

**copy-down 상속**: 브리드가 부모 브리드를 참조, 필드 미정의 시 부모 값 사용.  
**활용**: 몬스터 종류, 아이템 종류, 스킬 등 데이터 주도 설계.

---

### 13. 컴포넌트 (Component)
**핵심**: 한 엔티티의 여러 도메인을 독립 컴포넌트로 분리해 결합도를 낮춘다.

```
GameObject
├── InputComponent   (입력 처리)
├── PhysicsComponent (물리)
└── GraphicsComponent(렌더링)
```

**컴포넌트 간 통신**: ① 직접 참조 ② 컨테이너 통해 접근 ③ 메시지/이벤트 시스템  
**ECS의 토대**: 엔티티 = 컴포넌트 묶음.

---

### 14. 이벤트 큐 (Event Queue)
**핵심**: 이벤트 발신과 수신을 시간적으로 디커플링한다. 발신자는 큐에 넣고 즉시 반환.

```
링 버퍼 (Ring Buffer): head/tail 포인터로 순환 배열 구현
[  ][  ][E1][E2][E3][  ][  ]
         ↑tail      ↑head
```

**패턴 비교**:
- Observer: 동기, 즉시 전달
- Event Queue: 비동기, 지연 전달, 집계(aggregation) 가능

---

### 15. 서비스 로케이터 (Service Locator)
**핵심**: 서비스(오디오, 로깅 등)에 전역 접근점을 제공하되, 구현을 숨긴다.

```
Locator::getAudio().playSound(SOUND_JUMP);
//         ↑ 런타임에 NullAudio, ConsoleAudio, LoggedAudio 중 결정
```

**Null 서비스**: 서비스 미등록 시 아무것도 안 하는 기본 구현 → NPE 방지.  
**Decorator**: LoggedAudioService(RealAudioService)로 기능 래핑.

---

### 16. 데이터 지역성 (Data Locality)
**핵심**: CPU 캐시를 활용하도록 데이터를 배치해 메모리 접근 속도를 높인다.

**캐시 라인**: 64~128바이트 단위로 RAM → 캐시 로드. 미스 = ~수백 사이클 지연.

3가지 기법:
| 기법 | 문제 해결 |
|------|----------|
| 연속 배열 (Contiguous Arrays) | 포인터 체이싱 제거 → **50배** 성능 향상 사례 |
| 빽빽한 데이터 (Packed Data) | 활성 객체를 앞으로 정렬, 비활성 건너뜀 제거 |
| 핫/콜드 분리 (Hot/Cold Splitting) | 자주 쓰는 데이터만 캐시에 올라오도록 분리 |

---

### 17. 더티 플래그 (Dirty Flag)
**핵심**: 결과가 필요할 때까지 계산을 미뤄 불필요한 재계산을 방지한다.

```
로컬 트랜스폼 변경 → dirty = true
렌더링 시:
  if (dirty) { world = local.combine(parent); dirty = false; }
  else        { 캐시된 world 사용 }
```

**활용**: 씬 그래프 월드 트랜스폼, 텍스트 에디터 미저장 표시, 물리 엔진 sleep 오브젝트.  
**주의**: 모든 기본 데이터 변경 경로에서 반드시 플래그를 세울 것.

---

### 18. 오브젝트 풀 (Object Pool)
**핵심**: 고정 크기 풀을 미리 할당해두고 재사용함으로써 힙 할당/해제 비용과 메모리 파편화를 없앤다.

```cpp
// 프리 리스트: 죽은 오브젝트의 메모리를 포인터로 재활용
union { struct { double xVel, yVel; } live;  Particle* next; } state_;
```

**O(1) create/destroy**: firstAvailable_ 포인터로 빈 슬롯 즉시 접근.  
**주의**: 재사용 시 완전 초기화 필수. GC 환경에서 참조 누수 주의.

---

### 19. 공간 분할 (Spatial Partition)
**핵심**: 오브젝트를 위치 기반 자료구조에 저장해 "근처에 뭐가 있나?" 쿼리를 O(n) → O(log n) 이하로 낮춘다.

**고정 그리드 구현**:
- 셀 = 이중 연결 리스트 (O(1) 삽입/삭제)
- 전투 처리 = 같은 셀 + 인접 4셀만 비교
- 이동 = 셀 경계 넘을 때만 재배치

**자료구조 선택**:
| 자료구조 | 선형 대응 | 적합한 경우 |
|----------|----------|------------|
| Grid | 버킷 정렬 | 균등 분포, 동적 오브젝트 |
| Quadtree / Octree | 트라이 | 밀집도 불균일, 동적 |
| BSP / k-d tree | 이진 탐색 트리 | 정적 지형, 레이캐스팅 |
| BVH | 이진 탐색 트리 | 충돌 감지, 렌더링 컬링 |

---

## 패턴 선택 가이드

```
문제 상황                          추천 패턴
──────────────────────────────────────────────────────────
입력/커맨드를 되돌리고 싶다         Command
수천 개의 동종 객체 메모리 절약      Flyweight
시스템 간 느슨한 이벤트 통신        Observer / Event Queue
데이터로 새 타입을 정의하고 싶다    Prototype / Type Object
전역 서비스에 접근해야 한다         Service Locator (Singleton 대신)
복잡한 AI/애니메이션 상태 관리      State (HSM / Pushdown)
렌더링 깜빡임/불완전 상태 방지      Double Buffer
프레임레이트 독립 게임 루프         Game Loop (lag 보간)
엔티티의 도메인 분리                Component (→ ECS)
게임 로직을 스크립트로 분리         Bytecode
VM 실행 환경 제한                   Subclass Sandbox
지연 계산으로 중복 연산 제거        Dirty Flag
빈번한 생성/소멸, 파편화 방지       Object Pool
위치 기반 쿼리 성능                 Spatial Partition
캐시 미스로 인한 성능 문제          Data Locality
──────────────────────────────────────────────────────────
```

---

## 핵심 원칙 모음

- **커플링을 줄여라**: 변경 비용을 낮추는 것이 아키텍처의 본질
- **데이터가 코드다**: 바이트코드, 타입 오브젝트처럼 로직을 데이터로 표현
- **데이터가 성능이다**: 메모리 배치가 실행 속도를 결정한다 (Data Locality)
- **싱글턴을 피하라**: 전역 상태는 커플링과 버그의 온상
- **먼저 프로파일링하라**: 최적화 패턴은 성능 문제가 확인된 뒤에 적용
- **단순함을 추구하라**: 복잡한 패턴보다 단순한 코드가 유지보수하기 쉽다
