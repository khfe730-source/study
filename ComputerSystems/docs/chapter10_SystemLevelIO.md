# Chapter 10: System-Level I/O

## 핵심 주제

I/O는 메인 메모리와 외부 장치(디스크, 터미널, 네트워크) 사이에서 데이터를 복사하는 과정이다. 표준 C 라이브러리(`printf`, `scanf`)나 C++의 스트림 연산자 같은 고수준 I/O 함수는 모두 커널의 **Unix I/O** 위에 구현된다. 직접 Unix I/O를 이해해야 하는 이유:
- 파일 메타데이터 접근 (`stat`, 파일 크기/생성 시간 등) — 표준 I/O는 불가
- 네트워크 소켓 I/O — 표준 I/O는 소켓과 궁합이 나쁨
- 프로세스 생성, 공유 파일, I/O 리다이렉션 등 시스템 개념 이해

---

## 10.1 Unix I/O

리눅스에서 **파일**은 m bytes의 시퀀스 B₀, B₁, ..., B_{m-1}이다. 모든 I/O 장치(네트워크, 디스크, 터미널)는 파일로 모델링된다 → 단일 인터페이스로 모든 I/O 처리.

**Unix I/O 핵심 연산:**

```
파일 열기 (Opening):
  커널이 파일 디스크립터(fd, 작은 양의 정수) 반환
  각 프로세스는 기본 3개 디스크립터로 시작:
    0 = stdin (STDIN_FILENO)
    1 = stdout (STDOUT_FILENO)
    2 = stderr (STDERR_FILENO)

파일 위치 변경 (Seeking):
  커널이 각 열린 파일마다 현재 파일 위치 k (초기값 0) 유지
  lseek()로 명시적 변경 가능

읽기/쓰기 (Reading/Writing):
  read: 파일 위치 k에서 n bytes를 메모리로 복사, k += n
  write: 메모리에서 n bytes를 파일 위치 k에 복사, k += n
  k ≥ 파일 크기 m이면 EOF 조건 발생 (EOF 문자는 없음)

파일 닫기 (Closing):
  커널이 열린 파일 관련 자료구조 해제, 디스크립터 반환
  프로세스 종료 시 커널이 모든 열린 파일 자동 닫음
```

---

## 10.2 Files

**파일 타입:**

| 타입 | 설명 |
|------|------|
| **Regular file** | 임의 데이터. 커널에게 텍스트/바이너리 구분 없음. 텍스트 파일은 `\n`(LF, 0x0a)으로 줄 구분 |
| **Directory** | 링크(파일명→파일) 배열. `.`(자신)과 `..`(부모) 항상 포함 |
| **Socket** | 네트워크를 통한 다른 프로세스와의 통신 파일 (11장) |
| Named pipe, symbolic link, device | 기타 타입 |

**디렉토리 계층:**
```
/ (루트)
├── bin/
├── dev/
├── home/
│   ├── droh/
│   │   └── hello.c
│   └── bryant/
└── usr/
    ├── include/
    │   └── stdio.h
    └── bin/
```

**경로명(Pathname):**
- 절대 경로: `/home/droh/hello.c` (루트에서 시작)
- 상대 경로: `./hello.c` (현재 작업 디렉토리 기준)
- `cd` 명령어로 현재 작업 디렉토리 변경

---

## 10.3 Opening and Closing Files

```c
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>

int open(char *filename, int flags, mode_t mode);
// 반환: 열린 파일 디스크립터 (항상 현재 프로세스에서 열려있지 않은 최솟값) / -1 오류
```

**flags 인수 — 접근 모드 (하나 필수):**
| 플래그 | 의미 |
|--------|------|
| `O_RDONLY` | 읽기 전용 |
| `O_WRONLY` | 쓰기 전용 |
| `O_RDWR` | 읽기/쓰기 |

**추가 플래그 (OR로 조합):**
| 플래그 | 의미 |
|--------|------|
| `O_CREAT` | 파일이 없으면 빈 파일 생성 |
| `O_TRUNC` | 파일이 이미 있으면 0으로 자름 |
| `O_APPEND` | 매 쓰기 전 파일 끝으로 위치 이동 |

```c
// 예시
fd = open("foo.txt", O_RDONLY, 0);                   // 읽기 전용으로 열기
fd = open("foo.txt", O_WRONLY|O_APPEND, 0);          // 추가 쓰기
fd = open("new.txt", O_CREAT|O_TRUNC|O_WRONLY, DEF_MODE); // 새 파일 생성
```

**mode 인수 — 접근 권한 비트 (파일 생성 시):**
```
S_IRUSR  S_IWUSR  S_IXUSR  → owner read/write/execute
S_IRGRP  S_IWGRP  S_IXGRP  → group read/write/execute
S_IROTH  S_IWOTH  S_IXOTH  → others read/write/execute

실제 권한 = mode & ~umask
```

```c
#include <unistd.h>
int close(int fd);  // 디스크립터 닫기, 이미 닫힌 디스크립터 닫기는 오류
```

> 핵심: `open`은 항상 **현재 열려있지 않은 가장 작은** 디스크립터 번호를 반환한다.

---

## 10.4 Reading and Writing Files

```c
#include <unistd.h>

ssize_t read(int fd, void *buf, size_t n);
// 반환: 실제 읽은 bytes / 0 (EOF) / -1 (오류)

ssize_t write(int fd, const void *buf, size_t n);
// 반환: 실제 쓴 bytes / -1 (오류)
```

**Short count (요청보다 적게 전송) 발생 조건:**
| 상황 | 설명 |
|------|------|
| EOF 만남 | 파일에 20 bytes 남았는데 50 bytes 요청 → 20 반환, 다음엔 0 |
| 터미널 텍스트 읽기 | 한 번에 한 텍스트 줄만 전송 |
| 네트워크 소켓 | 내부 버퍼링, 네트워크 지연으로 short count 발생 |
| Linux pipe | 프로세스 간 통신 메커니즘에서 발생 가능 |

> **디스크 파일**: EOF를 제외하면 short count 없음. **네트워크**: 반드시 처리해야 함.

```c
// 표준 입력을 1 byte씩 표준 출력으로 복사
while (Read(STDIN_FILENO, &c, 1) != 0)
    Write(STDOUT_FILENO, &c, 1);
```

---

## 10.5 Robust Reading and Writing — Rio Package

Short count를 자동으로 처리하는 I/O 패키지. 네트워크 프로그래밍에서 필수.

### 10.5.1 비버퍼 함수 (Unbuffered)

```c
#include "csapp.h"

ssize_t rio_readn(int fd, void *usrbuf, size_t n);
// n bytes 모두 읽을 때까지 반복. EOF 시에만 short count 허용.

ssize_t rio_writen(int fd, void *usrbuf, size_t n);
// n bytes 모두 쓸 때까지 반복. Short count 절대 없음.
```

**구현 (신호 핸들러 복귀로 인한 EINTR 자동 재시작):**
```c
ssize_t rio_readn(int fd, void *usrbuf, size_t n) {
    size_t nleft = n;
    ssize_t nread;
    char *bufp = usrbuf;

    while (nleft > 0) {
        if ((nread = read(fd, bufp, nleft)) < 0) {
            if (errno == EINTR)  nread = 0;  // 시그널 핸들러 복귀 → 재시도
            else                 return -1;
        } else if (nread == 0)
            break;               // EOF
        nleft -= nread;
        bufp += nread;
    }
    return (n - nleft);
}
```

### 10.5.2 버퍼 함수 (Buffered)

파일 내용을 **애플리케이션 레벨 버퍼**에 캐시. 시스템 콜 횟수 최소화.

```c
void    rio_readinitb(rio_t *rp, int fd);         // 버퍼 초기화, 디스크립터 연결
ssize_t rio_readlineb(rio_t *rp, void *usrbuf, size_t maxlen); // 텍스트 한 줄 읽기
ssize_t rio_readnb(rio_t *rp, void *usrbuf, size_t n);         // n bytes 읽기
```

**rio_t 구조:**
```c
#define RIO_BUFSIZE 8192
typedef struct {
    int   rio_fd;             // 연결된 파일 디스크립터
    int   rio_cnt;            // 내부 버퍼의 미읽은 bytes 수
    char *rio_bufptr;         // 다음 미읽은 byte 포인터
    char  rio_buf[RIO_BUFSIZE]; // 내부 버퍼 (8KB)
} rio_t;
```

**텍스트 파일을 줄 단위로 복사:**
```c
rio_t rio;
char buf[MAXLINE];

Rio_readinitb(&rio, STDIN_FILENO);
while ((n = Rio_readlineb(&rio, buf, MAXLINE)) != 0)
    Rio_writen(STDOUT_FILENO, buf, n);
```

**rio_readlineb와 rio_readnb는 같은 디스크립터에서 교대로 사용 가능 (thread-safe).**
단, 버퍼 함수와 비버퍼 `rio_readn`은 같은 디스크립터에 혼용 금지.

---

## 10.6 Reading File Metadata

```c
#include <sys/stat.h>

int stat(const char *filename, struct stat *buf); // 파일명으로 메타데이터 조회
int fstat(int fd, struct stat *buf);               // 디스크립터로 메타데이터 조회
// 반환: 0 성공 / -1 오류
```

**stat 구조체 주요 필드:**
```c
struct stat {
    dev_t   st_dev;    // 장치
    ino_t   st_ino;    // inode 번호
    mode_t  st_mode;   // 파일 타입 + 접근 권한 비트
    nlink_t st_nlink;  // 하드 링크 수
    uid_t   st_uid;    // 소유자 UID
    gid_t   st_gid;    // 그룹 GID
    off_t   st_size;   // 파일 크기 (bytes)
    time_t  st_atime;  // 마지막 접근 시간
    time_t  st_mtime;  // 마지막 수정 시간
    time_t  st_ctime;  // 마지막 변경 시간
};
```

**파일 타입 판별 매크로 (`st_mode` 기반):**
```c
S_ISREG(m)   // 일반 파일?
S_ISDIR(m)   // 디렉토리?
S_ISSOCK(m)  // 소켓?

// 사용 예
struct stat st;
stat(argv[1], &st);
if (S_ISREG(st.st_mode))  type = "regular";
else if (S_ISDIR(st.st_mode)) type = "directory";
if (st.st_mode & S_IRUSR) readok = "yes";  // 소유자 읽기 권한 확인
```

---

## 10.7 Reading Directory Contents

```c
#include <dirent.h>

DIR           *opendir(const char *name);       // 디렉토리 스트림 열기
struct dirent *readdir(DIR *dirp);              // 다음 항목 반환 / NULL (끝 또는 오류)
int            closedir(DIR *dirp);             // 스트림 닫기

struct dirent {
    ino_t d_ino;        // inode 번호
    char  d_name[256];  // 파일명
};
```

```c
// 디렉토리 내용 출력
DIR *streamp = opendir(argv[1]);
errno = 0;
struct dirent *dep;
while ((dep = readdir(streamp)) != NULL)
    printf("Found file: %s\n", dep->d_name);
if (errno != 0)
    unix_error("readdir error");
closedir(streamp);
```

> `readdir`의 NULL 반환이 오류인지 끝인지 구분하려면 `errno` 변경 여부를 확인해야 한다.

---

## 10.8 Sharing Files

커널은 열린 파일을 세 가지 자료구조로 표현한다:

```
프로세스 A의        파일 테이블               v-node 테이블
디스크립터 테이블   (모든 프로세스 공유)        (모든 프로세스 공유)
┌──────────┐       ┌──────────────────┐     ┌──────────────┐
│ fd 0 ────┼──────→│ pos=0, refcnt=1  │────→│ st_mode      │
│ fd 1 ────┼──────→│ pos=0, refcnt=1  │────→│ st_size 등   │
│ fd 2     │       └──────────────────┘     │ (File A 정보) │
│ fd 3 ──┐ │       ┌──────────────────┐     └──────────────┘
│ fd 4 ──┘ │──────→│ pos=0, refcnt=1  │────→┌──────────────┐
└──────────┘       └──────────────────┘     │ (File B 정보) │
                                             └──────────────┘
```

**세 자료구조:**

| 자료구조 | 공유 | 설명 |
|---------|------|------|
| **Descriptor table** | 프로세스마다 독립 | fd → 파일 테이블 항목 포인터 |
| **File table** | 모든 프로세스 공유 | 파일 위치(pos), 참조 카운트(refcnt), v-node 포인터 |
| **v-node table** | 모든 프로세스 공유 | stat 구조체 대부분의 정보 (st_mode, st_size 등) |

**같은 파일을 두 번 open한 경우:**
```c
fd1 = open("foobar.txt", O_RDONLY, 0);  // 각자 독립된 파일 테이블 항목
fd2 = open("foobar.txt", O_RDONLY, 0);  // → 독립된 파일 위치(pos) 보유
// fd1에서 읽어도 fd2의 위치는 바뀌지 않음
```

**fork()와 파일 공유:**
```
fork() 후:
- 자식은 부모 디스크립터 테이블의 복사본 획득
- 부모와 자식이 동일한 파일 테이블 항목 공유 (refcnt += 1)
- → 같은 파일 위치 공유! (자식이 읽으면 부모의 파일 위치도 이동)
- 파일 테이블 항목 삭제: refcnt = 0이 될 때 (부모+자식 모두 close)
```

```c
// fork 예시: 자식이 읽으면 부모의 위치도 이동
fd = open("foobar.txt", O_RDONLY, 0);  // foobar
if (Fork() == 0) {
    Read(fd, &c, 1);  // 'f' 읽음, pos=1로 이동
    exit(0);
}
Wait(NULL);
Read(fd, &c, 1);       // pos=1이므로 'o' 읽음 → c = 'o'
```

---

## 10.9 I/O Redirection

```c
#include <unistd.h>

int dup2(int oldfd, int newfd);
// newfd를 oldfd의 복사본으로 만듦 (oldfd가 가리키는 파일로 리다이렉션)
// newfd가 이미 열려있으면 먼저 닫은 후 복사
// 반환: 음수 아닌 디스크립터 / -1 오류
```

**동작 원리:**
```
before dup2(4, 1):
fd1(stdout) → File A (터미널)       refcnt=1
fd4         → File B (디스크 파일)  refcnt=1

after dup2(4, 1):
fd1(stdout) → File B (디스크 파일)  refcnt=2
fd4         → File B               
File A → 닫힘 (refcnt=0, 삭제)

→ 이후 stdout에 쓰는 모든 데이터가 File B로 리다이렉션됨
```

**셸에서 리다이렉션 (`ls > foo.txt`)의 내부 구현:**
```c
// 셸이 fork 후 execve 전에 수행:
int fd = open("foo.txt", O_WRONLY|O_CREAT|O_TRUNC, DEF_MODE);
dup2(fd, STDOUT_FILENO);  // stdout(1)을 foo.txt로 리다이렉션
close(fd);
execve("ls", argv, envp);  // ls의 stdout은 이제 foo.txt
```

**표준 입력을 디스크립터 5로 리다이렉션:**
```c
dup2(5, STDIN_FILENO);  // or dup2(5, 0)
```

---

## 10.10 Standard I/O

C 표준 라이브러리가 제공하는 고수준 I/O. Unix I/O 위에 구현됨.

**스트림(Stream):** `FILE*` 구조체 포인터. **파일 디스크립터 + 스트림 버퍼**를 추상화.

```c
// 기본 제공 스트림 (fd 0, 1, 2에 대응)
extern FILE *stdin;   // descriptor 0
extern FILE *stdout;  // descriptor 1
extern FILE *stderr;  // descriptor 2
```

**주요 함수:**
```c
// 파일 열기/닫기
FILE *fopen(const char *path, const char *mode);  // "r", "w", "a", "rb" 등
int   fclose(FILE *fp);

// 바이트 읽기/쓰기
size_t fread(void *ptr, size_t size, size_t n, FILE *fp);
size_t fwrite(const void *ptr, size_t size, size_t n, FILE *fp);

// 문자열 읽기/쓰기
char *fgets(char *s, int n, FILE *fp);
int   fputs(const char *s, FILE *fp);

// 형식 있는 I/O
int scanf(const char *fmt, ...);
int printf(const char *fmt, ...);
int fscanf(FILE *fp, const char *fmt, ...);
int fprintf(FILE *fp, const char *fmt, ...);
int sscanf(const char *s, const char *fmt, ...);   // 문자열에서 파싱
int sprintf(char *s, const char *fmt, ...);        // 문자열로 포맷

// 디스크립터 → 스트림 변환
FILE *fdopen(int fd, const char *mode);
```

**스트림 버퍼의 역할:** `getc()` 첫 호출 시 한 번의 `read()`로 버퍼 채움 → 이후 호출은 버퍼에서 직접 서빙 → 시스템 콜 수 최소화.

---

## 10.11 어떤 I/O 함수를 써야 하나?

```
Unix I/O (open/read/write/stat/close)
    ↑
Rio (rio_readn/writen, rio_readlineb/readnb)    ← 네트워크 I/O 권장
    ↑
Standard I/O (printf/scanf/fread/fwrite 등)    ← 디스크/터미널 권장
```

**가이드라인:**

**G1. 가능하면 표준 I/O 사용**
- 디스크 파일, 터미널에서는 표준 I/O가 편리하고 충분
- `stat` 같이 표준 I/O에 없는 기능만 Unix I/O 직접 사용

**G2. 이진 파일에 `scanf`나 `rio_readlineb` 사용 금지**
- 이 함수들은 텍스트 파일용. 이진 데이터에는 `0xa`가 임의로 존재할 수 있어 잘못된 줄 종료 판단

**G3. 네트워크 소켓 I/O에는 Rio 함수 사용**

표준 I/O는 소켓에서 두 가지 제약이 있다:

| 제약 | 설명 |
|------|------|
| Restriction 1 | 출력 후 입력: 사이에 `fflush`, `fseek`, `fsetpos`, `rewind` 필요 |
| Restriction 2 | 입력 후 출력: 사이에 `fseek`, `fsetpos`, `rewind` 필요 (EOF 제외) |

`lseek`는 소켓에 사용 불가 → Restriction 2 우회 불가. 두 스트림을 같은 소켓에 열면 각각 `fclose` 시 동일한 fd를 두 번 닫는 문제.

```c
// 소켓에서 잘못된 표준 I/O 사용 패턴 (금지!)
FILE *fpin = fdopen(sockfd, "r");
FILE *fpout = fdopen(sockfd, "w");
// ... fclose(fpin); fclose(fpout); → 두 번 close 오류!

// 올바른 방법: Rio 사용
Rio_readlineb(&rio, buf, MAXLINE);    // 텍스트 줄 읽기
sscanf(buf, "%s %s", field1, field2); // 파싱은 sscanf로
sprintf(buf, "HTTP/1.1 200 OK\r\n");  // 출력 포맷팅은 sprintf로
Rio_writen(fd, buf, strlen(buf));     // 소켓에 쓰기
```

---

## 파일 공유 핵심 시나리오 정리

```
시나리오 1: 같은 파일을 두 번 open
  fd1 = open("f") / fd2 = open("f")
  → 서로 독립된 파일 테이블 항목 → 독립된 파일 위치
  → read(fd1) 후 read(fd2)는 처음부터 읽음

시나리오 2: fork 후 파일 공유
  fork() 전에 fd 열면 부모/자식이 동일 파일 테이블 항목 공유
  → 자식이 읽으면 부모의 파일 위치도 이동!

시나리오 3: dup2로 리다이렉션
  dup2(fd_file, STDOUT_FILENO)
  → stdout이 fd_file과 같은 파일 테이블 항목 가리킴
  → printf → 파일로 리다이렉션
```

---

## 요약

| 주제 | 핵심 |
|------|------|
| Unix I/O | 모든 I/O 장치를 파일로 모델링, fd로 식별 |
| 파일 타입 | regular / directory / socket |
| open/close | 최소 미사용 fd 반환, O_RDONLY/WRONLY/RDWR + O_CREAT/TRUNC/APPEND |
| read/write | Short count 가능 → 네트워크에서 반드시 처리 |
| Rio 패키지 | `rio_readn/writen` (비버퍼), `rio_readinitb/readlineb/readnb` (버퍼), 네트워크 필수 |
| stat | st_size(파일 크기), st_mode(타입+권한), S_ISREG/ISDIR/ISSOCK |
| 파일 공유 | 디스크립터 테이블(프로세스별), 파일 테이블/v-node(공유), fork 시 위치 공유 |
| I/O 리다이렉션 | `dup2(oldfd, newfd)` — newfd를 oldfd가 가리키는 파일로 덮어씀 |
| 표준 I/O | FILE* 스트림, 디스크/터미널에 적합, 소켓에는 부적합 |
| I/O 선택 | 디스크/터미널: 표준 I/O, 네트워크 소켓: Rio, 메타데이터: stat |
