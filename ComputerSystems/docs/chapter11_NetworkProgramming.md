# Chapter 11: Network Programming

## 핵심 주제

모든 네트워크 애플리케이션은 **클라이언트-서버 모델** 위에 구축되며, 동일한 기본 프로그래밍 인터페이스(소켓)를 사용한다. 이 챕터는 네트워크 계층 구조, TCP/IP, 소켓 인터페이스, HTTP 프로토콜을 다루며 간단하지만 실제 동작하는 Tiny Web Server 구현으로 마무리된다.

---

## 11.1 The Client-Server Programming Model

모든 네트워크 애플리케이션은 클라이언트-서버 모델로 동작한다.

**트랜잭션 4단계:**
```
1. 클라이언트가 서버에 요청(request) 전송
2. 서버가 요청을 수신·해석하고 리소스 조작
3. 서버가 클라이언트에 응답(response) 전송
4. 클라이언트가 응답 수신·처리
```

**중요 구분:**
- 클라이언트와 서버는 **프로세스**이지 호스트(머신)가 아님
- 단일 호스트에서 여러 클라이언트/서버가 동시 실행 가능
- 클라이언트와 서버가 같은 호스트 또는 다른 호스트에서 실행 가능

---

## 11.2 Networks

**호스트 관점에서 네트워크:** I/O 장치의 하나. 네트워크 어댑터가 I/O 버스에 연결되어 DMA를 통해 데이터를 메모리로 복사.

**네트워크 계층 구조:**
```
LAN (Local Area Network)  ← Ethernet (가장 일반적)
 ↓ 연결 via 라우터
WAN (Wide Area Network)
 ↓ 연결 via 라우터
인터넷(internet)
```

**Ethernet 구성:**
```
호스트──호스트──호스트
   \    |    /
    ── 허브(Hub) ──          ← 모든 포트에 브로드캐스트
         |
       브리지(Bridge) ──── 다른 세그먼트
```
- 각 Ethernet 어댑터: 전역 고유 48-bit 주소
- **허브(Hub)**: 수신한 모든 비트를 모든 포트로 복사 (무식한 브로드캐스트)
- **브리지(Bridge)**: 목적지에 필요한 포트로만 선택적 전달 (학습 알고리즘)

**인터넷 프로토콜의 역할:**
서로 다른 비호환 LAN/WAN 기술을 연결하기 위해 두 가지 기능 제공:

1. **명명 체계**: 각 호스트에 인터넷 주소(IP 주소) 부여
2. **전달 메커니즘**: 데이터를 **패킷(packet)** 으로 묶음 (헤더 + 페이로드)

**패킷 전달 흐름 (캡슐화):**
```
호스트 A (LAN1)                    라우터                    호스트 B (LAN2)
[데이터]                                                      [데이터]
   ↓                                                              ↑
[IP헤더|데이터] (인터넷 패킷)                           [IP헤더|데이터]
   ↓                                                              ↑
[LAN1헤더|IP헤더|데이터] (프레임)  →  라우터가         [LAN2헤더|IP헤더|데이터]
                                   LAN1 헤더 제거 후
                                   LAN2 헤더 추가 →
```

**캡슐화(Encapsulation)** 가 인터네트워킹의 핵심 통찰이다.

---

## 11.3 The Global IP Internet

TCP/IP 프로토콜 패밀리:
- **IP**: 호스트 간 데이터그램(datagram) 전달, **비신뢰성** (손실/중복 가능)
- **UDP**: IP 확장, 프로세스 간 데이터그램 전달 (비신뢰)
- **TCP**: IP 기반의 신뢰성 있는 **전이중(full-duplex) 연결** 스트림

소켓 인터페이스 + Unix I/O 함수로 애플리케이션 구현.

### 11.3.1 IP 주소

```c
/* IP 주소 구조체 */
struct in_addr {
    uint32_t s_addr;  // 네트워크 바이트 순서(빅엔디언)로 저장
};
```

**네트워크 바이트 순서 변환 (호스트가 리틀엔디언일 때 필수):**
```c
#include <arpa/inet.h>

uint32_t htonl(uint32_t hostlong);   // host → network (32-bit)
uint16_t htons(uint16_t hostshort);  // host → network (16-bit)
uint32_t ntohl(uint32_t netlong);    // network → host (32-bit)
uint16_t ntohs(uint16_t netshort);   // network → host (16-bit)
```

**IP 주소 ↔ 점선 십진수(dotted-decimal) 변환:**
```c
int inet_pton(AF_INET, const char *src, void *dst);
// "128.2.194.242" → 네트워크 바이트 순서의 32-bit 이진 주소
// 반환: 1(성공), 0(잘못된 형식), -1(오류)

const char *inet_ntop(AF_INET, const void *src, char *dst, socklen_t size);
// 네트워크 바이트 순서의 32-bit 이진 주소 → "128.2.194.242"
```

### 11.3.2 인터넷 도메인 이름

```
DNS(Domain Name System) 계층 구조:
        root
     /   |   \
  com   edu   gov
   |      |
google   cmu
          |
         cs   ece
          |
        ics  pdl
```

**도메인 이름 ↔ IP 주소 매핑:**
- 1:1: `whaleshark.ics.cs.cmu.edu` → `128.2.210.175`
- 다:1: 여러 도메인 → 같은 IP (`cs.mit.edu`, `eecs.mit.edu` → `18.62.1.6`)
- 1:다: 하나의 도메인 → 여러 IP (로드 밸런싱: `twitter.com` → 4개 IP)
- 매핑 없음: `edu`, `ics.cs.cmu.edu` 등

**로컬호스트:** `localhost` → 항상 루프백 주소 `127.0.0.1`

### 11.3.3 인터넷 연결

**소켓(Socket):** 연결의 엔드포인트. 소켓 주소 = `IP주소:포트번호`

```
클라이언트 소켓 주소: 128.2.194.242:51213
  ├─ IP: 128.2.194.242 (클라이언트 호스트)
  └─ 포트 51213: 임시 포트(ephemeral port) — 커널이 자동 할당

서버 소켓 주소: 208.216.181.15:80
  ├─ IP: 208.216.181.15 (서버 호스트)
  └─ 포트 80: well-known 포트 (HTTP, /etc/services 에 정의)
```

**연결 = 소켓 쌍(socket pair)으로 고유 식별:**
```
(클라이언트IP:클라이언트포트, 서버IP:서버포트)
(128.2.194.242:51213, 208.216.181.15:80)
```

TCP 연결: **신뢰성**, **전이중**, **점대점(point-to-point)**

**주요 well-known 포트:**
| 서비스 | 포트 |
|--------|------|
| HTTP | 80 |
| HTTPS | 443 |
| SMTP (이메일) | 25 |
| FTP | 21 |
| SSH | 22 |

---

## 11.4 The Sockets Interface

소켓 인터페이스는 Unix I/O 함수와 함께 네트워크 애플리케이션을 구축하는 함수 집합이다.

```
클라이언트                          서버
getaddrinfo                    getaddrinfo
socket                         socket
                               bind
                               listen
open_clientfd ─────────────── open_listenfd
connect          ←연결요청→    accept
rio_writen ──────────────────→ rio_readlineb
rio_readlineb ←────────────── rio_writen
close (EOF) ─────────────────→ rio_readlineb → EOF 감지
                               close
```

### 11.4.1 소켓 주소 구조체

```c
/* Internet 소켓 주소 (IPv4) */
struct sockaddr_in {
    uint16_t       sin_family;  // AF_INET (항상)
    uint16_t       sin_port;    // 포트 번호 (네트워크 바이트 순서)
    struct in_addr sin_addr;    // IP 주소 (네트워크 바이트 순서)
    unsigned char  sin_zero[8]; // sockaddr 크기에 맞추는 패딩
};

/* 제네릭 소켓 주소 (connect/bind/accept에서 요구) */
struct sockaddr {
    uint16_t sa_family;
    char     sa_data[14];
};
typedef struct sockaddr SA;  // 편의용 타입 별칭
```

프로토콜별 소켓 주소 구조체 포인터를 `(SA *)`로 캐스팅해서 전달한다.

### 11.4.2 socket()

```c
#include <sys/socket.h>

int socket(int domain, int type, int protocol);
// 반환: 소켓 디스크립터 / -1 오류

// 사용 (직접 하드코딩 대신 getaddrinfo 파라미터 사용 권장)
clientfd = socket(AF_INET, SOCK_STREAM, 0);
```
반환된 디스크립터는 **부분적으로만 열린** 상태. 클라이언트는 connect, 서버는 bind/listen으로 완성.

### 11.4.3 connect()

```c
int connect(int clientfd, const struct sockaddr *addr, socklen_t addrlen);
// 반환: 0 성공 / -1 오류

// 연결 성공 시: clientfd 로 읽기/쓰기 가능
// 연결 = (클라이언트IP:임시포트, addr.IP:addr.port)
```

### 11.4.4 bind()

```c
int bind(int sockfd, const struct sockaddr *addr, socklen_t addrlen);
// 서버: 소켓 주소를 소켓 디스크립터에 연결
```

### 11.4.5 listen()

```c
int listen(int sockfd, int backlog);
// 소켓을 능동 소켓(active) → 청취 소켓(listening)으로 변환
// backlog: 커널이 대기열에 쌓을 연결 요청 수 힌트 (보통 1024)
```

### 11.4.6 accept()

```c
int accept(int listenfd, struct sockaddr *addr, int *addrlen);
// 반환: 연결된 디스크립터(connfd) / -1 오류

// listenfd: 청취 디스크립터 (한 번 생성, 서버 수명 동안 유지)
// connfd:   연결된 디스크립터 (클라이언트마다 새로 생성, 서비스 후 닫힘)
```

**두 디스크립터의 구분:**
```
listenfd ─── 클라이언트 연결 요청을 기다리는 끝점 (1개, 서버 전체 수명)
connfd   ─── 실제 데이터 통신에 사용하는 끝점 (클라이언트마다 1개)
→ 이 구분 덕분에 동시(concurrent) 서버 구현 가능 (12장)
```

### 11.4.7 getaddrinfo / getnameinfo

`getaddrinfo`는 도메인 이름/IP 주소, 서비스명/포트 번호를 소켓 주소 구조체로 변환. 프로토콜 독립적(IPv4/IPv6 모두 지원).

```c
#include <netdb.h>

int getaddrinfo(const char *host, const char *service,
                const struct addrinfo *hints,
                struct addrinfo **result);
// 반환: 0 성공 / 비0 오류코드 (gai_strerror로 변환)

void freeaddrinfo(struct addrinfo *result);  // 리스트 메모리 해제

/* addrinfo 구조체 */
struct addrinfo {
    int            ai_flags;     // 힌트 플래그
    int            ai_family;    // AF_INET / AF_INET6
    int            ai_socktype;  // SOCK_STREAM / SOCK_DGRAM
    int            ai_protocol;  // 프로토콜
    char          *ai_canonname; // 정규 호스트명
    size_t         ai_addrlen;   // 소켓 주소 구조체 크기
    struct sockaddr *ai_addr;    // 소켓 주소 구조체 포인터
    struct addrinfo *ai_next;    // 다음 항목 포인터
};
```

**hints 주요 플래그:**
| 플래그 | 설명 |
|--------|------|
| `AI_ADDRCONFIG` | 로컬 호스트 구성에 맞는 주소만 반환 (연결 권장) |
| `AI_CANONNAME` | 첫 항목의 ai_canonname에 정규 이름 설정 |
| `AI_NUMERICSERV` | service를 포트 번호로만 해석 |
| `AI_PASSIVE` | 서버용 와일드카드 주소 반환 (host=NULL과 함께 사용) |

**getaddrinfo의 장점:** ai_addr, ai_addrlen, ai_family 등을 소켓 함수에 직접 전달 가능 → 프로토콜 독립 코드 가능.

```c
/* getnameinfo: 소켓 주소 → 호스트명/서비스명 문자열 (getaddrinfo 역함수) */
int getnameinfo(const struct sockaddr *sa, socklen_t salen,
                char *host, size_t hostlen,
                char *service, size_t servlen, int flags);
// NI_NUMERICHOST: 도메인 이름 대신 점선 십진수 반환
// NI_NUMERICSERV: 서비스 이름 대신 포트 번호 반환
```

**hostinfo 예시 — 도메인 이름 → IP 주소 목록:**
```c
struct addrinfo hints, *listp, *p;
char buf[MAXLINE];

memset(&hints, 0, sizeof(struct addrinfo));
hints.ai_family = AF_INET;        // IPv4만
hints.ai_socktype = SOCK_STREAM;  // 연결만

getaddrinfo(argv[1], NULL, &hints, &listp);
for (p = listp; p; p = p->ai_next) {
    getnameinfo(p->ai_addr, p->ai_addrlen, buf, MAXLINE, NULL, 0, NI_NUMERICHOST);
    printf("%s\n", buf);
}
freeaddrinfo(listp);
```

### 11.4.8 Helper 함수: open_clientfd / open_listenfd

**open_clientfd (클라이언트용):**
```c
int open_clientfd(char *hostname, char *port);
// 반환: 연결된 소켓 디스크립터 (Unix I/O 즉시 사용 가능) / -1 오류

int open_clientfd(char *hostname, char *port) {
    int clientfd;
    struct addrinfo hints, *listp, *p;

    memset(&hints, 0, sizeof(struct addrinfo));
    hints.ai_socktype = SOCK_STREAM;
    hints.ai_flags = AI_NUMERICSERV | AI_ADDRCONFIG;
    getaddrinfo(hostname, port, &hints, &listp);

    for (p = listp; p; p = p->ai_next) {
        if ((clientfd = socket(p->ai_family, p->ai_socktype, p->ai_protocol)) < 0)
            continue;
        if (connect(clientfd, p->ai_addr, p->ai_addrlen) != -1)
            break;  // 성공
        close(clientfd);
    }
    freeaddrinfo(listp);
    return (!p) ? -1 : clientfd;
}
```

**open_listenfd (서버용):**
```c
int open_listenfd(char *port);
// 반환: 청취 디스크립터 / -1 오류

// 핵심: AI_PASSIVE 플래그로 와일드카드 주소 사용
// setsockopt(SO_REUSEADDR): "Address already in use" 에러 방지
// 재시작 서버가 바로 연결 수락 가능
```

### 11.4.9 Echo 클라이언트/서버 예시

**에코 클라이언트:**
```c
clientfd = open_clientfd(host, port);
Rio_readinitb(&rio, clientfd);

while (Fgets(buf, MAXLINE, stdin) != NULL) {
    Rio_writen(clientfd, buf, strlen(buf));   // 서버로 전송
    Rio_readlineb(&rio, buf, MAXLINE);         // 에코 수신
    Fputs(buf, stdout);
}
close(clientfd);
```

**반복적 에코 서버 (iterative server):**
```c
listenfd = open_listenfd(argv[1]);
while (1) {
    clientlen = sizeof(struct sockaddr_storage);
    connfd = accept(listenfd, (SA *)&clientaddr, &clientlen);
    getnameinfo((SA *)&clientaddr, clientlen, hostname, MAXLINE, port, MAXLINE, 0);
    printf("Connected to (%s, %s)\n", hostname, port);
    echo(connfd);   // 한 클라이언트 처리
    close(connfd);  // 완료 후 닫기
}

void echo(int connfd) {
    size_t n;
    char buf[MAXLINE];
    rio_t rio;
    Rio_readinitb(&rio, connfd);
    while ((n = Rio_readlineb(&rio, buf, MAXLINE)) != 0) {
        printf("server received %d bytes\n", (int)n);
        Rio_writen(connfd, buf, n);  // 수신한 줄 그대로 에코
    }
}
```

> **EOF on connection**: 연결에서 read가 0을 반환하는 조건. EOF 문자는 없음. 프로세스가 연결의 자기 쪽을 닫으면 상대방이 read 시 0 반환으로 EOF 감지.

> **iterative vs concurrent 서버**: 반복적 서버는 한 번에 한 클라이언트만 처리. 동시(concurrent) 서버는 12장에서.

---

## 11.5 Web Servers

### 11.5.1 웹 기초

**HTTP(HyperText Transfer Protocol):** 텍스트 기반 애플리케이션 레이어 프로토콜.

```
브라우저 → 서버 연결 열기 → 요청 전송 → 서버 응답 → 연결 닫기 → 브라우저 화면 표시
```

**HTML:** 브라우저에 텍스트/그래픽 표시 방법을 지시하는 태그 언어.  
`<a href="http://www.cmu.edu/index.html">Carnegie Mellon</a>` — 하이퍼링크

### 11.5.2 웹 콘텐츠

**MIME 타입:**
| MIME 타입 | 설명 |
|----------|------|
| `text/html` | HTML 페이지 |
| `text/plain` | 일반 텍스트 |
| `image/gif` | GIF 이미지 |
| `image/jpeg` | JPEG 이미지 |
| `image/png` | PNG 이미지 |
| `application/postscript` | PostScript 문서 |

**두 종류의 콘텐츠:**
- **정적 콘텐츠(Static content)**: 디스크 파일을 그대로 반환
- **동적 콘텐츠(Dynamic content)**: 실행 파일 실행 결과를 반환 (CGI)

**URL 구조:**
```
http://www.google.com:80/index.html
│      │              │  └── URI suffix (서버가 사용)
│      │              └── 포트 (기본값 80)
│      └── 호스트명 (클라이언트가 연결에 사용)
└── 프로토콜

동적 콘텐츠 URL:
http://bluefish.ics.cs.cmu.edu:8000/cgi-bin/adder?15000&213
                                    └── 파일명    └── 인수 (? 구분, & 연결)
```

### 11.5.3 HTTP 트랜잭션

**HTTP 요청 형식:**
```
GET / HTTP/1.1                    ← 요청 줄: method URI version
Host: www.aol.com                 ← 요청 헤더 (HTTP/1.1에서 Host 필수)
                                  ← 빈 줄: 헤더 종료
```

**HTTP 응답 형식:**
```
HTTP/1.0 200 OK                   ← 응답 줄: version status-code status-message
MIME-Version: 1.0                 ← 응답 헤더들
Date: Mon, 8 Jan 2010 4:59:42 GMT
Content-Type: text/html           ← MIME 타입 (중요!)
Content-Length: 42092             ← 바이트 크기 (중요!)
                                  ← 빈 줄: 헤더 종료
<html>...</html>                  ← 응답 본문
```

**주요 HTTP 상태 코드:**
| 코드 | 메시지 | 의미 |
|------|--------|------|
| 200 | OK | 성공 |
| 301 | Moved permanently | 콘텐츠가 Location 헤더로 이동 |
| 400 | Bad request | 서버가 요청 이해 불가 |
| 403 | Forbidden | 서버에 접근 권한 없음 |
| 404 | Not found | 파일을 찾을 수 없음 |
| 501 | Not implemented | 요청 메서드 미지원 |
| 505 | HTTP version not supported | HTTP 버전 미지원 |

### 11.5.4 동적 콘텐츠 제공 (CGI)

**CGI(Common Gateway Interface):** 서버가 CGI 프로그램을 실행하고 출력을 클라이언트에 전달하는 표준.

**GET 요청의 인수 전달:**
```
GET /cgi-bin/adder?15000&213 HTTP/1.1
              ↑     ↑
       실행 파일  '?' 뒤에 인수들 ('&' 구분, 공백 → %20)
```

**서버 → CGI 프로그램 인수 전달 메커니즘:**
```c
// 서버가 fork() 후 execve() 전에:
setenv("QUERY_STRING", "15000&213", 1);  // QUERY_STRING 환경변수 설정

// CGI 프로그램에서:
char *buf = getenv("QUERY_STRING");  // "15000&213" 읽기
```

**주요 CGI 환경 변수:**
| 환경 변수 | 내용 |
|----------|------|
| `QUERY_STRING` | 프로그램 인수 |
| `SERVER_PORT` | 서버 포트 번호 |
| `REQUEST_METHOD` | GET 또는 POST |
| `REMOTE_HOST` | 클라이언트 도메인 이름 |
| `REMOTE_ADDR` | 클라이언트 IP 주소 |
| `CONTENT_TYPE` | POST: 요청 본문 MIME 타입 |
| `CONTENT_LENGTH` | POST: 요청 본문 크기 |

**CGI 출력 처리:**
```c
// 서버가 execve 전에:
dup2(connfd, STDOUT_FILENO);  // CGI 프로그램의 stdout → 소켓으로 리다이렉션

// CGI 프로그램(adder.c)이 직접 HTTP 응답 생성:
printf("Content-length: %d\r\n", strlen(content));
printf("Content-type: text/html\r\n\r\n");  // \r\n 으로 빈 줄
printf("%s", content);
// → stdout이 소켓으로 리다이렉션되어 클라이언트에 직접 전달
```

**동적 콘텐츠 제공 전체 흐름:**
```c
void serve_dynamic(int fd, char *filename, char *cgiargs) {
    // 자식 프로세스 생성
    if (fork() == 0) {
        setenv("QUERY_STRING", cgiargs, 1);
        dup2(fd, STDOUT_FILENO);      // stdout → 소켓
        execve(filename, argv, environ);  // CGI 프로그램 실행
    }
    wait(NULL);  // 자식 완료 대기
}
```

---

## 11.6 Tiny Web Server (250줄)

여러 개념(프로세스 제어, Unix I/O, 소켓, HTTP)을 결합한 실제 동작하는 웹 서버.

**전체 구조:**
```c
// main: 반복적 서버 루프
listenfd = open_listenfd(port);
while (1) {
    connfd = accept(listenfd, ...);
    doit(connfd);   // HTTP 트랜잭션 처리
    close(connfd);
}
```

**doit 함수 흐름:**
```c
void doit(int fd) {
    // 1. 요청 줄 읽기: "GET /index.html HTTP/1.1"
    Rio_readlineb(&rio, buf, MAXLINE);
    sscanf(buf, "%s %s %s", method, uri, version);

    // 2. GET만 지원, 나머지는 501 오류
    if (strcasecmp(method, "GET")) { clienterror(..., "501", ...); return; }

    // 3. 요청 헤더 읽고 무시
    read_requesthdrs(&rio);

    // 4. URI 파싱 → 파일명 + CGI 인수, 정적/동적 판별
    is_static = parse_uri(uri, filename, cgiargs);
    // cgi-bin 포함 → 동적, 나머지 → 정적

    // 5. 파일 존재 확인 (stat)
    if (stat(filename, &sbuf) < 0) { clienterror(..., "404", ...); return; }

    // 6. 정적: 파일 읽어서 전송
    if (is_static) serve_static(fd, filename, sbuf.st_size);
    // 7. 동적: fork + execve + dup2(stdout→소켓)
    else serve_dynamic(fd, filename, cgiargs);
}
```

**serve_static 핵심:**
```c
void serve_static(int fd, char *filename, int filesize) {
    // 응답 헤더 전송
    sprintf(buf, "HTTP/1.0 200 OK\r\n");
    sprintf(buf, "Content-type: %s\r\n", filetype);  // HTML, JPEG 등
    sprintf(buf, "Content-length: %d\r\n\r\n", filesize);
    Rio_writen(fd, buf, strlen(buf));

    // 파일 내용 전송 (mmap으로 효율적으로)
    srcfd = open(filename, O_RDONLY, 0);
    srcp = mmap(0, filesize, PROT_READ, MAP_PRIVATE, srcfd, 0);
    Rio_writen(fd, srcp, filesize);
    munmap(srcp, filesize);
}
```

---

## 소켓 프로그래밍 전체 흐름 요약

```
【클라이언트】                              【서버】
getaddrinfo(host, port, hints, &list)    getaddrinfo(NULL, port, hints, &list)
socket(ai_family, ai_socktype, ...)      socket(ai_family, ai_socktype, ...)
                                         setsockopt(SO_REUSEADDR)
                                         bind(sockfd, ai_addr, ai_addrlen)
                                         listen(sockfd, LISTENQ)
connect(clientfd, ai_addr, ai_addrlen)  connfd = accept(listenfd, &clientaddr, ...)
freeaddrinfo(list)                       freeaddrinfo(list)

↓ 연결 수립 완료 ↓

Rio_writen(clientfd, ...)  ──────────→  Rio_readlineb(&rio, buf, ...)
Rio_readlineb(&rio, ...)  ←──────────  Rio_writen(connfd, ...)

close(clientfd)                          close(connfd)
                                         ↑ loop back to accept
```

---

## 요약

| 주제 | 핵심 |
|------|------|
| 클라이언트-서버 모델 | 요청→처리→응답 4단계 트랜잭션, 프로세스 단위 |
| 네트워크 계층 | LAN(Ethernet/허브/브리지) → WAN → 인터넷(라우터) |
| 인터넷 프로토콜 | IP 주소(32-bit), 패킷/헤더/페이로드, 캡슐화 |
| IP 주소 | 네트워크 바이트 순서, htonl/ntohl, inet_pton/ntop |
| 도메인 이름 | DNS, 1:1/다:1/1:다 매핑, localhost=127.0.0.1 |
| 소켓 연결 | 소켓 쌍(4-tuple)으로 연결 고유 식별 |
| 소켓 API | socket→connect(클), socket→bind→listen→accept(서) |
| getaddrinfo | 프로토콜 독립적, addrinfo 연결 리스트, freeaddrinfo |
| HTTP | 요청(method URI version + 헤더), 응답(version 상태코드 + 헤더 + 본문) |
| CGI | QUERY_STRING 환경변수, dup2(connfd, stdout), fork+execve |
| Tiny 서버 | 250줄, GET만 지원, 정적/동적 콘텐츠, 반복적 서버 |
