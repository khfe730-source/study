# Network 핵심 정리

---

## 1. OSI 7계층 vs TCP/IP 4계층

| OSI | TCP/IP | 프로토콜 | 단위 |
|-----|--------|----------|------|
| 7. Application | Application | HTTP, FTP, DNS, SMTP | 메시지 |
| 6. Presentation | ↑ | SSL/TLS | |
| 5. Session | ↑ | | |
| 4. Transport | Transport | TCP, UDP | 세그먼트/데이터그램 |
| 3. Network | Internet | IP, ICMP, ARP | 패킷 |
| 2. Data Link | Network Access | Ethernet, Wi-Fi | 프레임 |
| 1. Physical | ↑ | | 비트 |

---

## 2. TCP vs UDP

| 항목 | TCP | UDP |
|------|-----|-----|
| 연결 | 연결 지향 (3-way handshake) | 비연결 |
| 신뢰성 | 보장 (재전송, 순서 보장) | 미보장 |
| 속도 | 느림 | 빠름 |
| 흐름/혼잡 제어 | 있음 | 없음 |
| 헤더 크기 | 20~60 bytes | 8 bytes |
| 게임 사용 | 로그인, 결제, 채팅 | 실시간 위치, 전투 (FPS 등) |

**TCP 3-Way Handshake**
```
Client          Server
  |-- SYN ------->|
  |<-- SYN+ACK ---|
  |-- ACK ------->|
       연결 완료
```

**TCP 4-Way Handshake (종료)**
```
Client          Server
  |-- FIN ------->|
  |<-- ACK -------|
  |<-- FIN -------|
  |-- ACK ------->|
```

**TIME_WAIT**: 클라이언트가 마지막 ACK 후 2MSL 동안 대기. 지연 패킷 처리 목적. 서버에서 TIME_WAIT이 대량 쌓이면 포트 고갈 발생.

---

## 3. TCP 상태 전이

```
LISTEN → SYN_RCVD → ESTABLISHED → CLOSE_WAIT → LAST_ACK → CLOSED
                                ↘ FIN_WAIT_1 → FIN_WAIT_2 → TIME_WAIT → CLOSED
```

`ss -s` 또는 `netstat -s`로 TCP 상태 통계 확인.

---

## 4. HTTP/HTTPS

**HTTP 메서드**

| 메서드 | 용도 | 멱등성 | 안전성 |
|--------|------|--------|--------|
| GET | 조회 | O | O |
| POST | 생성 | X | X |
| PUT | 전체 교체 | O | X |
| PATCH | 부분 수정 | X | X |
| DELETE | 삭제 | O | X |

**HTTP 상태 코드**
- 2xx: 성공 (200 OK, 201 Created, 204 No Content)
- 3xx: 리다이렉트 (301 Moved, 302 Found, 304 Not Modified)
- 4xx: 클라이언트 오류 (400 Bad Request, 401 Unauthorized, 403 Forbidden, 404 Not Found, 429 Too Many Requests)
- 5xx: 서버 오류 (500 Internal, 502 Bad Gateway, 503 Service Unavailable, 504 Gateway Timeout)

**HTTP/1.1 vs HTTP/2 vs HTTP/3**

| 버전 | 특징 |
|------|------|
| HTTP/1.1 | Keep-Alive, 파이프라이닝. Head-of-Line Blocking |
| HTTP/2 | 멀티플렉싱, 헤더 압축(HPACK), 서버 푸시 |
| HTTP/3 | QUIC(UDP 기반), 연결 설정 빠름, 패킷 손실 영향 적음 |

---

## 5. HTTPS / TLS

**TLS Handshake (간략)**
1. ClientHello (지원하는 암호화 목록)
2. ServerHello + 인증서 전달
3. 클라이언트가 인증서 검증
4. 세션 키 교환 (RSA 또는 ECDHE)
5. 암호화 통신 시작

**대칭키 vs 비대칭키**
- 대칭키: 암복호화에 같은 키. 빠름 (AES)
- 비대칭키: 공개키/개인키 쌍. 느리지만 키 배포 안전 (RSA, ECDSA)
- TLS: 비대칭키로 세션키 교환 → 이후 대칭키로 통신

---

## 6. 소켓 프로그래밍

```
서버 소켓 흐름:
socket() → bind() → listen() → accept() → read()/write() → close()

클라이언트 소켓 흐름:
socket() → connect() → read()/write() → close()
```

**TCP 소켓 옵션 (게임 서버 중요)**
```
SO_REUSEADDR: TIME_WAIT 상태 포트 재사용
SO_KEEPALIVE: 유휴 연결 생존 확인
TCP_NODELAY: Nagle 알고리즘 비활성화 (실시간 게임에서 지연 최소화)
SO_SNDBUF / SO_RCVBUF: 송수신 버퍼 크기
SO_LINGER: close() 시 데이터 전송 보장 여부
```

**Nagle 알고리즘**: 작은 패킷들을 모아서 보내는 최적화. 실시간 게임에서는 `TCP_NODELAY`로 비활성화.

---

## 7. 웹소켓 (WebSocket)

```
HTTP Upgrade 요청 → 101 Switching Protocols → 양방향 전이중 통신
```

- HTTP 연결을 업그레이드하여 지속적인 양방향 통신
- 게임 채팅, 실시간 알림, 라이브 이벤트에 활용
- HTTP 폴링 대비 오버헤드 적음

---

## 8. 로드 밸런싱

**알고리즘**
- Round Robin: 순서대로 분배
- Weighted Round Robin: 서버 성능에 따라 가중치
- Least Connections: 현재 연결 수가 가장 적은 서버
- IP Hash: 같은 IP는 항상 같은 서버 (세션 유지)

**L4 vs L7 로드 밸런서**
- L4: IP + 포트 기반. 빠름. TCP/UDP 레벨
- L7: HTTP 헤더, URL, 쿠키 기반. 스마트 라우팅 가능

---

## 9. DNS

```
브라우저 → OS 캐시 → Local DNS → Root DNS → TLD DNS → Authoritative DNS
```

- **A 레코드**: 도메인 → IPv4
- **AAAA 레코드**: 도메인 → IPv6
- **CNAME**: 도메인 → 도메인 별칭
- **MX**: 메일 서버
- **TTL**: 캐시 유효 시간

---

## 10. 게임 서버 네트워크 패턴

**게임 서버 구성 예시**
```
Client
  ↓
[CDN / DDoS Protection]
  ↓
[L7 Load Balancer]
  ↓
[Gateway Server] ← 인증, 라우팅
  ↓
[Zone Server] [Battle Server] [Chat Server]
  ↓
[DB] [Cache(Redis)] [Message Queue]
```

**패킷 설계**
- 헤더: 패킷 크기, 패킷 타입(opcode), 시퀀스 번호
- 암호화: TLS 또는 커스텀 암호화
- 압축: zlib, LZ4 (대용량 패킷)

**UDP 신뢰성 구현 (RUDP)**
- 시퀀스 번호로 순서 보장
- ACK로 수신 확인
- 타임아웃 후 재전송
- 예: ENet, RakNet, KCP

---

## 11. 자주 나오는 면접 질문

**Q. HTTP는 Stateless인데 로그인 상태는 어떻게 유지?**
쿠키/세션 또는 JWT 토큰 사용. 서버는 세션 ID나 토큰으로 클라이언트 식별.

**Q. DDoS 방어 방법?**
CDN/WAF(CloudFlare 등), IP 차단, Rate Limiting, TCP SYN Cookie, Anycast 라우팅.

**Q. Keep-Alive란?**
HTTP/1.1에서 하나의 TCP 연결로 여러 HTTP 요청을 처리하는 것. 연결 수립 오버헤드 감소.

**Q. CORS란?**
Cross-Origin Resource Sharing. 다른 출처의 리소스 요청 시 브라우저 보안 정책. 서버에서 `Access-Control-Allow-Origin` 헤더로 허용.

**Q. 게임에서 TCP vs UDP 선택 기준?**
신뢰성 필요한 데이터(로그인, 아이템, 결제)는 TCP. 실시간성이 중요하고 약간의 패킷 손실을 허용하는 데이터(위치, 공격 판정)는 UDP.
