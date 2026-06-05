# Redis 심화 정리

---

## 1. Redis 특징

- **In-Memory**: 모든 데이터를 RAM에 저장 → 마이크로초 응답
- **단일 스레드**: 명령어 처리가 순차적 → 원자성 보장, 레이스 컨디션 없음
- **다양한 자료구조**: String, List, Hash, Set, Sorted Set, Stream 등
- **영속성**: RDB 스냅샷 + AOF 로그로 디스크 저장 가능
- **Pub/Sub**: 경량 메시지 브로커
- **Lua Script**: 여러 명령을 원자적으로 실행

---

## 2. 핵심 자료구조

### String
```bash
SET key value EX 3600       # 값 설정 + 만료 1시간
GET key
INCR counter                # 원자적 증가
INCRBY counter 5
SETNX key value             # 없을 때만 SET (분산 락에 활용)
GETSET key newvalue         # 기존값 반환 후 새값 설정
MSET k1 v1 k2 v2            # 다중 설정
MGET k1 k2                  # 다중 조회
```

### List (이중 연결 리스트)
```bash
LPUSH queue task1 task2     # 왼쪽에 추가
RPUSH queue task3           # 오른쪽에 추가
LPOP queue                  # 왼쪽에서 꺼내기 → 스택
RPOP queue                  # 오른쪽에서 꺼내기 → 큐
BRPOP queue 30              # 블로킹 팝 (30초 대기) → 작업 큐에 유용
LRANGE queue 0 -1           # 전체 조회
LLEN queue                  # 길이

# 활용: 최근 알림 N개 유지
LPUSH notifications:user:123 "msg"
LTRIM notifications:user:123 0 99  # 최근 100개만 유지
```

### Hash (필드-값 맵)
```bash
HSET user:123 name "kim" hp 100 gold 5000
HGET user:123 name
HMGET user:123 name hp gold
HGETALL user:123
HINCRBY user:123 gold 500   # 원자적 증가
HDEL user:123 name
HEXISTS user:123 name

# 활용: 유저 세션/상태 저장
HSET session:abc user_id 123 server battle-1 login_ts 1700000000
EXPIRE session:abc 3600
```

### Set (중복 없는 집합)
```bash
SADD online_users user:123
SREM online_users user:123
SISMEMBER online_users user:123   # 존재 여부 O(1)
SCARD online_users                # 원소 수
SMEMBERS online_users             # 전체 (큰 Set에는 SSCAN 사용)
SRANDMEMBER online_users 5        # 랜덤 5개

# 집합 연산
SUNION set1 set2          # 합집합
SINTER set1 set2          # 교집합
SDIFF set1 set2           # 차집합

# 활용: 서버별 접속 유저, 친구 목록 교집합 (공통 친구)
```

### Sorted Set (점수 기반 정렬 집합)
```bash
ZADD ranking 9500 "user:123"
ZADD ranking 8000 "user:456"
ZINCRBY ranking 100 "user:123"    # 점수 증가

# 조회
ZREVRANK ranking "user:123"       # 내림차순 순위 (0-based)
ZRANK ranking "user:123"          # 오름차순 순위
ZSCORE ranking "user:123"         # 점수
ZREVRANGE ranking 0 99 WITHSCORES # Top 100
ZRANGEBYSCORE ranking 8000 9999   # 점수 범위 조회
ZCARD ranking                     # 원소 수

# 활용: 랭킹, 리더보드
ZADD weekly:ranking 1500 "user:123"
ZREVRANK weekly:ranking "user:123"  # 내 순위
ZREVRANGE weekly:ranking 0 9 WITHSCORES  # Top 10
```

### Stream (메시지 스트림, Redis 5.0+)
```bash
XADD game-events * user_id 123 event "kill" target 456
XREAD COUNT 10 STREAMS game-events 0
XGROUP CREATE game-events workers $ MKSTREAM  # 컨슈머 그룹
XREADGROUP GROUP workers w1 COUNT 10 STREAMS game-events >
XACK game-events workers <message-id>

# Kafka 경량 대안. 컨슈머 그룹으로 분산 처리
```

---

## 3. 만료(TTL) 관리

```bash
EXPIRE key 3600         # 초 단위
PEXPIRE key 5000        # 밀리초 단위
EXPIREAT key 1700000000 # Unix 타임스탬프
TTL key                 # 남은 시간 (-1: 만료 없음, -2: 키 없음)
PERSIST key             # 만료 제거
```

**만료 동작 방식**
- Lazy Expiry: 키 접근 시 만료 여부 확인
- Active Expiry: 주기적으로 랜덤 샘플링하여 만료 키 삭제
→ TTL이 지나도 즉시 삭제되지 않을 수 있음 (메모리 주의)

---

## 4. 분산 락 (Distributed Lock)

```bash
# 단일 Redis 노드
SET lock:resource:1 <random_value> NX EX 10
# NX: 없을 때만 SET
# EX 10: 10초 자동 만료 (데드락 방지)
# random_value: 락 소유자 식별 (다른 프로세스가 해제 못 하도록)

# 해제는 Lua Script로 원자적으로
local val = redis.call('GET', KEYS[1])
if val == ARGV[1] then
    return redis.call('DEL', KEYS[1])
else
    return 0
end
```

**Redlock 알고리즘** (다중 Redis 노드)
- N개(홀수, 보통 5개)의 독립 Redis 노드에 순서대로 락 획득 시도
- N/2 + 1개 이상 성공 + 경과 시간 < TTL → 락 획득
- 실패 시 획득한 모든 노드에서 락 해제

---

## 5. Pub/Sub

```bash
# 발행
PUBLISH chat:room:1 "Hello World"

# 구독 (블로킹)
SUBSCRIBE chat:room:1
PSUBSCRIBE chat:room:*  # 패턴 구독

# 주의사항
# - 메시지 영속성 없음 (구독 중 아닐 때 수신 불가)
# - 전달 보장 없음
# - 영속성 필요하면 Stream 사용
```

**게임 서버 활용**
- 서버 간 이벤트 전파 (채팅, 공지, 서버 상태 변경)
- 여러 서버 인스턴스에 브로드캐스트

---

## 6. Pipeline & Transaction

```bash
# Pipeline: 여러 명령을 한 번에 전송 (RTT 절약)
# 코드 예시 (Go - go-redis)
pipe := rdb.Pipeline()
pipe.Set(ctx, "k1", "v1", 0)
pipe.Set(ctx, "k2", "v2", 0)
pipe.Incr(ctx, "counter")
cmds, err := pipe.Exec(ctx)

# MULTI/EXEC Transaction: 원자적 실행
MULTI
SET k1 v1
INCR counter
EXEC            # 한 번에 실행, 중간에 다른 클라이언트 명령 끼어들기 불가
DISCARD         # 취소

# 주의: MULTI/EXEC는 에러 발생해도 나머지 명령 실행됨
# → 진정한 원자성은 Lua Script 사용
```

---

## 7. Lua Script

복잡한 원자적 연산에 사용. Redis 서버에서 실행되므로 네트워크 RTT 없음.

```lua
-- 아이템 구매: 재화 확인 + 차감 + 아이템 지급 원자적 처리
local gold = tonumber(redis.call('HGET', KEYS[1], 'gold'))
local price = tonumber(ARGV[1])
if gold < price then
    return {err = "insufficient_gold"}
end
redis.call('HINCRBY', KEYS[1], 'gold', -price)
redis.call('SADD', KEYS[2], ARGV[2])  -- 아이템 추가
return {ok = "success"}
```

```bash
EVAL <script> 2 user:123:data user:123:items 100 "sword"
EVALSHA <sha1> 2 ...  # 캐시된 스크립트 실행
SCRIPT LOAD <script>  # 스크립트 사전 로드
```

---

## 8. 영속성 (Persistence)

| 방식 | 설명 | 장점 | 단점 |
|------|------|------|------|
| **RDB** | 주기적 스냅샷 | 컴파일된 파일, 복구 빠름 | 마지막 스냅샷 이후 데이터 손실 |
| **AOF** | 모든 쓰기 명령 로그 | 데이터 손실 최소 | 파일 크기 큼, 재시작 느림 |
| **RDB+AOF** | 병행 사용 | 안전 + 빠른 복구 | 저장 비용 |
| **No persistence** | 캐시 전용 | 최고 성능 | 재시작 시 데이터 소실 |

```bash
# AOF 설정 (redis.conf)
appendonly yes
appendfsync everysec  # always(안전)/everysec(절충)/no(빠름)
```

---

## 9. 고가용성 구성

### Sentinel (장애 감지 + 자동 페일오버)
```
[Primary] ←→ [Replica]
     ↑              ↑
  [Sentinel 1]
  [Sentinel 2]  ← 과반수 합의로 Primary 장애 판단 → 자동 승격
  [Sentinel 3]
```

### Cluster (수평 확장 + 샤딩)
```
- 16384개 해시 슬롯을 N개 마스터에 분배
- 각 마스터는 1개 이상의 레플리카 보유
- CRC16(key) % 16384 로 슬롯 결정
- 리샤딩 중에도 서비스 가능

redis-cli --cluster create \
  node1:6379 node2:6379 node3:6379 \
  --cluster-replicas 1
```

---

## 10. 메모리 관리

```bash
# 메모리 정책 (redis.conf)
maxmemory 4gb
maxmemory-policy allkeys-lru  # 전체 키 중 LRU 제거

# 정책 종류
# noeviction       : 꽉 차면 에러 (기본)
# allkeys-lru      : 전체 키 중 LRU
# volatile-lru     : TTL 있는 키 중 LRU
# allkeys-lfu      : 전체 키 중 LFU (접근 빈도)
# volatile-ttl     : TTL 짧은 키 먼저 제거
# allkeys-random   : 랜덤 제거

# 메모리 사용량 확인
INFO memory
MEMORY USAGE key        # 특정 키 메모리 사용량
```

---

## 11. 모니터링 명령어

```bash
INFO all                        # 전체 통계
INFO replication                # 복제 상태
MONITOR                         # 실시간 명령어 스트림 (디버깅용, 성능 영향)
SLOWLOG GET 10                  # 슬로우 쿼리 상위 10개
CLIENT LIST                     # 연결된 클라이언트
DBSIZE                          # 키 개수
DEBUG SLEEP 0                   # 응답 테스트

# 핫 키 분석
redis-cli --hotkeys             # 접근 빈도 높은 키
redis-cli --bigkeys             # 크기 큰 키
```

---

## 12. 게임 서버 활용 패턴 심화

```bash
# 쿨다운 관리 (스킬/액션 제한)
SET cooldown:user:123:fireball 1 EX 5   # 5초 쿨다운
EXISTS cooldown:user:123:fireball        # 사용 가능 여부

# 중복 요청 방지 (멱등성 키)
SET idempotent:purchase:req-abc123 1 NX EX 300  # 5분 내 중복 차단

# 일일 퀘스트 카운터 (자정 만료)
INCR quest:user:123:kill_count
EXPIREAT quest:user:123:kill_count <내일자정 Unix timestamp>

# 서버 간 유저 위치 (어느 서버에 접속 중)
HSET user_location user:123 "battle-server-02"
EXPIRE user_location 60  # 60초마다 갱신

# Rate Limiting (초당 요청 제한)
local count = redis.call('INCR', KEYS[1])
if count == 1 then
    redis.call('EXPIRE', KEYS[1], ARGV[1])
end
if count > tonumber(ARGV[2]) then
    return 0  -- 제한 초과
end
return 1

# 롤링 랭킹 (7일 슬라이딩 윈도우)
-- 매일 새로운 키에 점수 추가 + 7일치 합산
ZADD score:20241201 100 "user:123"
ZUNIONSTORE weekly_ranking 7 score:20241201 score:20241130 ... WEIGHTS 1 1 ...
```

---

## 13. 자주 나오는 면접 질문

**Q. Redis가 빠른 이유?**
In-Memory 저장, 단일 스레드로 락 오버헤드 없음, 간단한 자료구조로 O(1)~O(log n) 연산, I/O multiplexing(epoll)으로 다수 클라이언트 처리.

**Q. Redis 단일 스레드인데 느리지 않나?**
명령어 처리는 단일 스레드지만 I/O는 epoll 기반 비동기. 메모리 연산이라 CPU 병목이 거의 없음. Redis 6.0+에서 I/O 스레드 멀티화.

**Q. Cache Stampede(캐시 스탬피드)란?**
캐시 만료 시 다수의 요청이 동시에 DB에 몰리는 현상. 해결책: 락으로 한 요청만 DB 조회 후 캐시 갱신, 만료 시간에 랜덤 지터(jitter) 추가.

**Q. Redis Cluster에서 MGET이 안 되는 이유?**
다른 슬롯에 있는 키들은 다른 노드에 분산되어 있어 단일 명령으로 처리 불가. 해시 태그 `{tag}key` 로 같은 슬롯에 배치하거나 파이프라인 사용.

**Q. 캐시 전략 종류?**
- Cache-Aside(Lazy): 앱이 캐시 miss 시 DB 조회 후 캐시 저장. 가장 일반적
- Write-Through: 쓰기 시 캐시+DB 동시 갱신. 일관성 좋음
- Write-Behind: 캐시에만 쓰고 비동기로 DB 반영. 빠르지만 유실 위험
- Read-Through: 캐시가 DB 조회를 대행
