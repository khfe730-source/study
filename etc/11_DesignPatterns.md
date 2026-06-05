# 디자인 패턴 핵심 정리 (게임 서버 중심)

---

## 1. 패턴 분류

| 분류 | 패턴 | 한 줄 요약 |
|------|------|-----------|
| **생성** | Singleton | 인스턴스 1개 보장 |
| **생성** | Factory Method | 객체 생성을 서브클래스에 위임 |
| **생성** | Abstract Factory | 관련 객체 군(群) 생성 |
| **생성** | Builder | 복잡한 객체 단계별 생성 |
| **구조** | Decorator | 기능 동적 추가 (상속 대안) |
| **구조** | Proxy | 접근 제어, 지연 로딩 |
| **구조** | Facade | 복잡한 서브시스템에 단순 인터페이스 |
| **구조** | Adapter | 호환되지 않는 인터페이스 연결 |
| **행동** | Observer | 상태 변화를 구독자에게 알림 |
| **행동** | Command | 요청을 객체로 캡슐화 |
| **행동** | State | 상태에 따라 행동 변경 |
| **행동** | Strategy | 알고리즘을 교체 가능하게 캡슐화 |
| **행동** | Template Method | 알고리즘 골격 정의, 세부 구현은 서브클래스 |

---

## 2. Singleton (싱글턴)

**목적**: 클래스 인스턴스가 전체 프로그램에서 단 하나임을 보장.

```go
// Go - sync.Once 기반 (스레드 안전)
type Config struct {
    DBHost string
    Port   int
}

var (
    instance *Config
    once     sync.Once
)

func GetConfig() *Config {
    once.Do(func() {
        instance = &Config{DBHost: "localhost", Port: 3306}
    })
    return instance
}
```

**게임 서버 활용**: 설정(Config), DB 커넥션 풀, 로거, 서버 매니저
**주의**: 전역 상태이므로 테스트 어려움. 의존성 주입(DI)으로 대체하는 추세.

---

## 3. Observer (옵저버)

**목적**: 객체의 상태 변화를 여러 구독자에게 자동으로 알림. 발행-구독 패턴.

```go
type EventType string
const (
    PlayerDied  EventType = "player_died"
    ItemDropped EventType = "item_dropped"
)

type Event struct {
    Type    EventType
    Payload interface{}
}

type Handler func(Event)

type EventBus struct {
    handlers map[EventType][]Handler
    mu       sync.RWMutex
}

func (eb *EventBus) Subscribe(t EventType, h Handler) {
    eb.mu.Lock()
    defer eb.mu.Unlock()
    eb.handlers[t] = append(eb.handlers[t], h)
}

func (eb *EventBus) Publish(e Event) {
    eb.mu.RLock()
    defer eb.mu.RUnlock()
    for _, h := range eb.handlers[e.Type] {
        go h(e)  // 비동기 처리
    }
}

// 활용
bus.Subscribe(PlayerDied, func(e Event) {
    // 드롭 아이템 처리
})
bus.Subscribe(PlayerDied, func(e Event) {
    // 퀘스트 업데이트
})
bus.Publish(Event{Type: PlayerDied, Payload: playerID})
```

**게임 서버 활용**: 게임 이벤트 처리, 로그 기록, 퀘스트/업적 진행도 갱신

---

## 4. Command (커맨드)

**목적**: 요청을 객체로 캡슐화. 실행 취소(Undo), 큐잉, 로깅에 활용.

```go
type Command interface {
    Execute() error
    Undo() error
}

// 아이템 지급 커맨드
type GiveItemCommand struct {
    userID int
    itemID int
    repo   ItemRepository
}

func (c *GiveItemCommand) Execute() error {
    return c.repo.AddItem(c.userID, c.itemID)
}

func (c *GiveItemCommand) Undo() error {
    return c.repo.RemoveItem(c.userID, c.itemID)
}

// 커맨드 실행기 (이력 관리)
type CommandExecutor struct {
    history []Command
}

func (e *CommandExecutor) Execute(cmd Command) error {
    if err := cmd.Execute(); err != nil {
        return err
    }
    e.history = append(e.history, cmd)
    return nil
}

func (e *CommandExecutor) UndoLast() error {
    if len(e.history) == 0 { return nil }
    last := e.history[len(e.history)-1]
    e.history = e.history[:len(e.history)-1]
    return last.Undo()
}
```

**게임 서버 활용**: 배치 처리 작업, GM 툴 실행 취소, 이벤트 소싱(Event Sourcing)

---

## 5. State (스테이트)

**목적**: 객체 내부 상태에 따라 행동을 다르게. 복잡한 `if/switch` 대신 상태 클래스로 분리.

```go
type PlayerState interface {
    OnEnter(p *Player)
    Update(p *Player, dt float64)
    OnExit(p *Player)
}

type IdleState struct{}
func (s *IdleState) OnEnter(p *Player) { p.SetAnimation("idle") }
func (s *IdleState) Update(p *Player, dt float64) {
    if p.IsMoving() { p.ChangeState(&MoveState{}) }
    if p.IsAttacking() { p.ChangeState(&AttackState{}) }
}
func (s *IdleState) OnExit(p *Player) {}

type Player struct {
    state PlayerState
    hp    int
}

func (p *Player) ChangeState(next PlayerState) {
    p.state.OnExit(p)
    p.state = next
    p.state.OnEnter(p)
}

func (p *Player) Update(dt float64) {
    p.state.Update(p, dt)
}
```

**게임 서버 활용**: 유저 접속 상태 (로비/게임중/대기), 몬스터 AI, 매치 진행 상태

---

## 6. Strategy (스트래티지)

**목적**: 알고리즘을 인터페이스로 정의하고 런타임에 교체 가능하게.

```go
type MatchStrategy interface {
    FindMatch(players []*Player) [][]*Player
}

type RatingBasedStrategy struct{ Range int }
func (s *RatingBasedStrategy) FindMatch(players []*Player) [][]*Player {
    // MMR 기반 매칭 로직
}

type RandomStrategy struct{}
func (s *RandomStrategy) FindMatch(players []*Player) [][]*Player {
    // 랜덤 매칭 로직
}

type MatchMaker struct {
    strategy MatchStrategy
}

func (m *MatchMaker) SetStrategy(s MatchStrategy) {
    m.strategy = s
}

func (m *MatchMaker) Run(players []*Player) [][]*Player {
    return m.strategy.FindMatch(players)
}
```

**게임 서버 활용**: 매치메이킹 알고리즘, 보상 계산 공식, 피해량 계산

---

## 7. Factory Method (팩토리 메서드)

**목적**: 객체 생성 로직을 서브클래스에 위임. 타입에 따라 다른 객체를 생성.

```go
type Monster interface {
    Attack() int
    HP() int
}

type Goblin struct{}
func (g *Goblin) Attack() int { return 10 }
func (g *Goblin) HP() int     { return 50 }

type Dragon struct{}
func (d *Dragon) Attack() int { return 100 }
func (d *Dragon) HP() int     { return 1000 }

// 팩토리 함수
func NewMonster(monsterType string) (Monster, error) {
    switch monsterType {
    case "goblin":  return &Goblin{}, nil
    case "dragon":  return &Dragon{}, nil
    default:        return nil, fmt.Errorf("unknown monster: %s", monsterType)
    }
}
```

**게임 서버 활용**: 몬스터/아이템/스킬 생성, 핸들러 등록

---

## 8. Decorator (데코레이터)

**목적**: 상속 없이 객체에 기능을 동적으로 추가. HTTP 미들웨어의 원리.

```go
type Handler func(ctx context.Context, req Request) Response

// 로깅 미들웨어
func WithLogging(h Handler) Handler {
    return func(ctx context.Context, req Request) Response {
        start := time.Now()
        resp := h(ctx, req)
        log.Printf("%s took %v", req.Method, time.Since(start))
        return resp
    }
}

// 인증 미들웨어
func WithAuth(h Handler) Handler {
    return func(ctx context.Context, req Request) Response {
        if !validateToken(req.Token) {
            return Response{Code: 401}
        }
        return h(ctx, req)
    }
}

// 체이닝
handler = WithLogging(WithAuth(baseHandler))
```

**게임 서버 활용**: HTTP 미들웨어 (인증/로깅/레이트리밋), 스킬 버프 스택

---

## 9. Proxy (프록시)

**목적**: 실제 객체 접근을 제어. 지연 초기화, 캐싱, 접근 제어.

```go
type UserRepository interface {
    GetUser(id int) (*User, error)
}

// 캐싱 프록시
type CachedUserRepository struct {
    real  UserRepository
    cache map[int]*User
    mu    sync.RWMutex
}

func (r *CachedUserRepository) GetUser(id int) (*User, error) {
    r.mu.RLock()
    if u, ok := r.cache[id]; ok {
        r.mu.RUnlock()
        return u, nil
    }
    r.mu.RUnlock()

    u, err := r.real.GetUser(id)
    if err != nil { return nil, err }

    r.mu.Lock()
    r.cache[id] = u
    r.mu.Unlock()
    return u, nil
}
```

**게임 서버 활용**: DB 접근 캐싱, 원격 서버 접근 추상화

---

## 10. 자주 나오는 면접 질문

**Q. 싱글턴이 안티패턴인 이유?**
전역 상태를 만들어 테스트 격리가 어렵고, 의존성이 숨겨짐. 멀티스레드 환경에서 동기화 필요. 의존성 주입(DI)으로 대체하면 테스트하기 쉬워짐.

**Q. Strategy vs State 차이?**
Strategy: 외부에서 알고리즘을 주입, 교체. State: 객체 스스로 내부 상태에 따라 행동 변경. State는 상태 간 전이가 핵심.

**Q. Observer vs Pub/Sub 차이?**
Observer: 발행자가 구독자를 직접 참조 (같은 프로세스). Pub/Sub: 브로커(채널)를 통해 완전히 분리 (다른 서비스/프로세스 가능).

**Q. SOLID 원칙?**
- **S**ingle Responsibility: 한 클래스는 하나의 책임
- **O**pen/Closed: 확장에 열려있고, 수정에 닫혀있어야
- **L**iskov Substitution: 서브타입은 기반 타입을 대체 가능해야
- **I**nterface Segregation: 사용하지 않는 인터페이스 의존 금지
- **D**ependency Inversion: 구체가 아닌 추상에 의존
