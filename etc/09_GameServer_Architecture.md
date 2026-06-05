# 게임 서버 아키텍처 핵심 정리

---

## 1. 게임 서버 유형

| 유형 | 특징 | 예시 |
|------|------|------|
| **로비/매치메이킹 서버** | 유저 대기, 매칭, 파티 관리 | 대전 게임 대기실 |
| **배틀/존 서버** | 실시간 게임 로직 처리 | 전투, 던전 |
| **채팅 서버** | 실시간 메시지 브로드캐스트 | 채널 채팅 |
| **게이트웨이 서버** | 인증, 라우팅, 부하 분산 | 프록시 서버 |
| **API 서버** | HTTP REST, 인벤토리, 상점 | 모바일 게임 백엔드 |
| **배치/스케줄 서버** | 랭킹 집계, 리워드 지급 | 크론 작업 |

---

## 2. 게임 서버 아키텍처 패턴

### 모바일 게임 서버 (턴제/SNG 계열)
```
Client (iOS/Android)
    │ HTTPS
    ▼
[API Gateway / Load Balancer]
    │
    ▼
[Game API Server (PHP/Go)]  ←→  [Redis - 세션/캐시]
    │                           [Redis - 분산락/랭킹]
    ▼
[MySQL Primary]
    │
    └── [MySQL Replica × N]  ← 읽기 전용
```

### 실시간 게임 서버 (MMORPG/FPS 계열)
```
Client
    │ TCP/UDP
    ▼
[Gateway Server]  ← 인증, 세션, 라우팅
    │
    ├── [Zone Server 1] ← 특정 맵/룸 담당
    ├── [Zone Server 2]
    └── [Zone Server N]
         │
         ▼
    [Message Queue]  ← 서버 간 이벤트
         │
    [DB / Cache Layer]
```

---

## 3. 동시성 제어 (재화/아이템 처리)

게임에서 가장 중요한 부분. 중복 지급, 아이템 복사 버그 방지.

**방법 1: DB 원자적 UPDATE**
```sql
-- 골드 차감 (부족하면 실패)
UPDATE users 
SET gold = gold - 100 
WHERE user_id = ? AND gold >= 100

-- 영향받은 행이 0이면 실패 처리
```

**방법 2: 낙관적 락 (version)**
```sql
UPDATE users 
SET gold = gold - 100, version = version + 1
WHERE user_id = ? AND version = ? AND gold >= 100
-- 실패 시 재시도 or 오류 반환
```

**방법 3: Redis 분산 락 (여러 서버 환경)**
```
1. SET lock:user:123 1 NX EX 5  → 락 획득 성공 시만 처리
2. 아이템 지급 로직 실행
3. DEL lock:user:123            → 락 해제
```
- NX: 키 없을 때만 SET
- EX 5: 5초 후 자동 만료 (장애 시 영구 잠금 방지)

**방법 4: 큐 직렬화**
유저별 요청을 큐에 넣고 단일 워커가 순서대로 처리. 동시성 문제 원천 차단.

---

## 4. 세션 관리

**Stateless 방식 (JWT)**
```
Client → [서버 A] → JWT 발급 (서명 포함)
Client → [서버 B] → JWT 검증만 (DB 불필요)
```
- 장점: 서버 확장 용이
- 단점: 토큰 강제 무효화 어려움 → Redis 블랙리스트로 해결

**Stateful 방식 (Redis 세션)**
```
Client → [서버 A] → session_id 발급 → Redis에 세션 저장
Client → [서버 B] → Redis에서 session_id 조회
```
- 장점: 즉시 무효화 가능
- 단점: Redis 의존성, 네트워크 I/O

---

## 5. 실시간 동기화 모델

**권위 서버 (Authoritative Server)**
- 서버가 게임 상태의 유일한 진실(Source of Truth)
- 클라이언트는 입력만 전송, 서버가 계산 후 결과 브로드캐스트
- 치팅 방지에 유리
- 예: FPS, MMORPG

**P2P (Peer-to-Peer)**
- 클라이언트끼리 직접 통신 (하나가 호스트 역할)
- 서버 비용 절감, 레이턴시 낮을 수 있음
- 호스트 어드밴티지, 치팅에 취약
- 예: 일부 격투 게임

**Lockstep (동기화 방식)**
- 모든 클라이언트가 같은 입력을 받아 같은 결과를 계산
- 결정론적(Deterministic) 게임 로직 필요
- 예: 전략 게임 (Warcraft 3 방식)

---

## 6. 매치메이킹

```
기본 알고리즘:
1. 유저 MMR/레이팅 수집
2. 대기 큐에 추가
3. 적절한 레이팅 범위 내 N명 모이면 매치 성사
4. 대기 시간 증가 시 범위 확대 (Expansion)
5. 배틀 서버 할당 → 게임 시작

MMR 시스템:
- ELO: 승/패에 따라 포인트 교환 (체스 기원)
- TrueSkill: 불확실성(sigma) 포함, 팀 게임에 적합 (Microsoft)
- Glicko-2: ELO 개선판, 비활성 기간 반영
```

---

## 7. 패킷 설계

**헤더 구조 예시**
```
┌─────────────────────────────────────┐
│ Length (2 bytes) - 패킷 전체 크기  │
│ OpCode (2 bytes) - 패킷 타입       │
│ Sequence (4 bytes) - 시퀀스 번호   │
│ Session ID (8 bytes)                │
├─────────────────────────────────────┤
│ Payload (가변 길이)                 │
└─────────────────────────────────────┘
```

**직렬화 포맷 비교**

| 포맷 | 크기 | 속도 | 특징 |
|------|------|------|------|
| JSON | 큼 | 느림 | 가독성 좋음, 디버깅 용이 |
| MessagePack | 중간 | 빠름 | JSON 바이너리 버전 |
| Protobuf | 작음 | 매우 빠름 | 스키마 정의 필요, Google |
| FlatBuffers | 작음 | 매우 빠름 | 역직렬화 없이 접근 가능 |

게임 서버 권장: **Protobuf** (네트워크 대역폭 절약, 타입 안전성)

---

## 8. 게임 서버 성능 지표

| 지표 | 설명 | 목표 (예시) |
|------|------|-----------|
| **CCU** | 동시 접속자 수 | 100K CCU |
| **TPS** | 초당 트랜잭션 수 | 10,000 TPS |
| **Latency** | 응답 지연 | API < 100ms, 실시간 < 50ms |
| **Tick Rate** | 서버 업데이트 주기 | 10~60 tick/s |
| **Packet Loss** | 패킷 손실률 | < 1% |

---

## 9. 라이브 서비스 운영

**무중단 배포 (Rolling Update)**
```
1. 신버전 서버 N대 추가 시작
2. 헬스체크 통과 후 LB에 등록
3. 구버전 서버를 LB에서 제거
4. Graceful Shutdown: 기존 연결 처리 완료 후 종료
```

**Graceful Shutdown 구현 (Go 예시)**
```go
quit := make(chan os.Signal, 1)
signal.Notify(quit, syscall.SIGTERM, syscall.SIGINT)
<-quit

ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
defer cancel()
server.Shutdown(ctx)  // 새 요청 거부, 기존 요청 처리 완료
```

**서버 점검 패턴**
- 점검 플래그 ON → 신규 접속 거부, 기존 유저 안내
- 배포 완료 → 점검 플래그 OFF
- Redis에 점검 상태 저장 → 모든 서버 실시간 반영

---

## 10. 보안

**게임 보안 주요 이슈**

| 위협 | 대응 |
|------|------|
| 스피드핵, 무적핵 | 서버 권위 검증, 이동 속도/수치 범위 검사 |
| 패킷 조작 | HMAC 서명, TLS 암호화 |
| DDoS | Cloud Armor, Rate Limiting |
| 계정 도용 | 2FA, 비정상 로그인 탐지 |
| SQL Injection | Prepared Statement |
| 중복 결제/구매 | 멱등성 키, 분산 락 |

**API 인증 플로우**
```
1. 로그인 → 서버가 Access Token(짧은 만료) + Refresh Token(긴 만료) 발급
2. API 요청 시 Authorization: Bearer <access_token>
3. Access Token 만료 시 Refresh Token으로 갱신
4. Refresh Token도 만료 시 재로그인
```

---

## 11. Redis 활용 패턴 (게임)

```
# 랭킹 (Sorted Set)
ZADD ranking 9500 "user:123"
ZREVRANK ranking "user:123"   → 순위
ZREVRANGE ranking 0 99 WITHSCORES  → Top 100

# 쿨다운 (TTL)
SET cooldown:user:123:skill:fireball 1 EX 5  → 5초 쿨다운

# 세션 (Hash)
HSET session:abc123 user_id 123 server_id 1 login_time 1700000000
EXPIRE session:abc123 3600

# 접속자 수 (Set)
SADD online_users user:123
SCARD online_users  → 접속자 수

# Pub/Sub (채팅)
PUBLISH chat:channel:1 "메시지"
SUBSCRIBE chat:channel:1
```

---

## 12. 자주 나오는 면접 질문

**Q. CCU 10만 이상 처리 방법?**
수평 확장(Scale-out) + LB. 세션은 Redis에 저장(Stateless 서버). DB 읽기 부하는 Read Replica. 캐시 레이어로 DB 부하 감소.

**Q. 아이템 중복 지급 버그를 막는 방법?**
멱등성 키(요청 ID) + 분산 락 또는 DB 유니크 제약. 같은 요청 ID로 중복 호출 시 동일 결과 반환.

**Q. 서버 간 유저 이동 (존 이동) 처리?**
유저 상태를 DB에 저장 → 새 서버에서 로드. 이전 서버에서 세션 무효화. Redis에 현재 서버 정보 기록.

**Q. 핫스팟(Hot Key) 문제?**
특정 데이터(인기 아이템, 공지사항)에 요청 집중 → 로컬 캐시(in-process) 활용, 데이터 샤딩, Read Replica로 분산.
