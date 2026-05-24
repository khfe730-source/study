# Chapter 13: File-System Interface

## 13.1 파일 개념 (File Concept)

파일(file)은 OS가 정의하고 구현하는 추상 데이터 타입으로, 2차 저장장치에 기록된 관련 정보의 명명된 집합이다.

### 13.1.1 파일 속성 (File Attributes)

| 속성 | 설명 |
|------|------|
| **Name** | 사람이 읽을 수 있는 유일한 식별자 |
| **Identifier** | 파일시스템 내부 고유 번호 (non-human readable, e.g., inode 번호) |
| **Type** | 파일 형식 (텍스트, 바이너리, 실행파일 등) |
| **Location** | 장치와 파일의 물리적 위치 포인터 |
| **Size** | 현재 크기 (bytes/words/blocks) |
| **Protection** | 접근 제어 정보 (read/write/execute 권한) |
| **Timestamps** | 생성/수정/최종 접근 시간 |
| **User ID** | 소유자 식별 |

- 파일 속성 정보는 **디렉토리 구조(directory structure)**에 저장 → 장치에 상주, 필요 시 메모리에 로드

### 13.1.2 파일 연산 (File Operations)

**기본 7가지 연산:**

```
create()    → 파일시스템에 공간 할당 + 디렉토리에 엔트리 추가
open()      → 디렉토리 탐색 + open-file table에 엔트리 복사 → file handle/descriptor 반환
write()     → write pointer 위치에 데이터 기록, write pointer 갱신
read()      → read pointer 위치에서 데이터 읽기, read pointer 갱신
             (read/write pointer 통합 → current-file-position pointer)
lseek()     → 포인터 재배치 (실제 I/O 없음, file seek)
delete()    → 디렉토리 엔트리 탐색 → 파일 공간 해제 → 엔트리 삭제/무효화
              (hard link 존재 시 마지막 링크 삭제 때만 실제 삭제)
truncate()  → 파일 내용 삭제, 속성 유지 (길이만 0으로)
```

**Open-File Table (열린 파일 테이블):**

```
Per-Process Open-File Table        System-Wide Open-File Table
┌──────────────────┐               ┌────────────────────────────────┐
│ fd 0 → entry ptr ├──────────────→│ 파일 위치 / 접근 날짜 / 크기   │
│ fd 1 → entry ptr ├──────────────→│ open count: 2                  │
│ fd 2 → entry ptr │               │ 디스크 내 위치                 │
│   ...            │               │   ...                          │
└──────────────────┘               └────────────────────────────────┘
  - file pointer (프로세스별)
  - access rights
  - accounting info
```

- `open()`: 디렉토리 탐색 → 엔트리를 system-wide table로 복사, per-process table에 포인터 추가
- `close()`: open count 감소, 0이 되면 system-wide table에서 제거
- 파일 연산은 파일명 대신 **file descriptor(fd) / file handle** 사용 → 반복 탐색 불필요

**파일 락 (File Locking):**

| 유형 | 설명 | OS 예시 |
|------|------|---------|
| **Shared lock** (reader lock) | 여러 프로세스 동시 획득 가능 | - |
| **Exclusive lock** (writer lock) | 한 번에 하나만 획득 가능 | - |
| **Mandatory** | OS가 잠금 무결성 강제 (잠금 중 다른 접근 차단) | Windows |
| **Advisory** | 소프트웨어가 잠금 규약 준수 필요 | UNIX |

```java
// Java 파일 락 예시
FileChannel ch = new RandomAccessFile("file.txt", "rw").getChannel();
FileLock exclusiveLock = ch.lock(0, raf.length()/2, false);  // exclusive
FileLock sharedLock    = ch.lock(raf.length()/2+1, raf.length(), true);  // shared
// ... 작업 후
exclusiveLock.release();
sharedLock.release();
```

### 13.1.3 파일 유형 (File Types)

**파일 유형 인식 방법:**

| 방법 | 설명 | OS |
|------|------|-----|
| **확장자(extension)** | `resume.docx`, `server.c` | 모든 OS (UNIX에서는 hint) |
| **Magic number** | 파일 앞부분의 특수 바이트 시퀀스 (e.g., ELF, `#!/bin/sh`) | UNIX/Linux |
| **Creator attribute** | 파일을 생성한 프로그램 이름 저장 | macOS |

**일반적인 파일 유형:**

```
.exe/.com/.bin  → 실행 파일 (바이너리)
.obj/.o         → 컴파일된 오브젝트
.c/.java/.py    → 소스 코드
.lib/.so/.dll   → 라이브러리
.zip/.tar/.rar  → 압축 아카이브
.gif/.jpg/.pdf  → 출력/뷰 파일
.bat/.sh        → 셸 스크립트
.xml/.html      → 마크업
```

### 13.1.4 파일 구조 (File Structure)

- **UNIX 철학**: 파일 = 8비트 바이트의 순열 (OS가 내부 구조 해석 안 함) → 최대 유연성
- OS는 반드시 **실행 파일 구조** 지원 (로더가 메모리 로드 위치, 첫 명령어 위치 파악 필요)

### 13.1.5 내부 파일 구조 (Internal File Structure)

```
논리 레코드(logical record): 1 byte (UNIX 기준)
물리 블록(physical block):  512 bytes or 4KB

┌──────────────────────────────────────────┐
│ Block 0: [byte 0~511]                    │
│ Block 1: [byte 512~1023]                 │
│ ...                                      │
│ Block N: [byte N*512 ~ file_end + 패딩]  │  ← 마지막 블록: 내부 단편화
└──────────────────────────────────────────┘
```

- **내부 단편화(internal fragmentation)**: 블록 단위 할당 → 마지막 블록 일부 낭비
  - 1,949 byte 파일, 512B 블록 → 4블록 할당 (99 byte 낭비)

---

## 13.2 접근 방법 (Access Methods)

### 13.2.1 순차 접근 (Sequential Access)

```
┌──────────────────────────────────┐
│ beginning    current     end     │
│   ┌─────────────↑────────────┐   │
│   │ read/write next...       │   │
└──────────────────────────────────┘

read_next()  → 현재 위치 읽기 + 포인터 전진
write_next() → 현재 위치 쓰기 + 포인터 전진
reset()      → 처음으로 이동
```

- 테이프 모델 기반, 편집기/컴파일러 등 가장 일반적 사용 패턴

### 13.2.2 직접 접근 (Direct Access)

- 디스크 모델 기반: `read(n)`, `write(n)`, `position(n)` + `read_next()`
- **상대 블록 번호(relative block number)**: 파일 시작을 0으로 하는 인덱스
  - `record N` 접근 → `seek to L * N bytes` (L = 논리 레코드 크기)
- 순차 접근 시뮬레이션:
  ```c
  cp = 0;              // reset
  read(cp); cp += 1;   // read_next
  write(cp); cp += 1;  // write_next
  ```

### 13.2.3 기타 접근 방법 (Index-Based)

```
대형 파일에 대한 인덱스 구조:

Master Index (메모리 유지)
  └── Secondary Index Blocks (디스크)
        └── Data Blocks (실제 데이터)

IBM ISAM (Indexed Sequential Access Method):
  1. 마스터 인덱스 이진 탐색 → 보조 인덱스 블록 번호
  2. 보조 인덱스 블록 읽기 → 데이터 블록 번호
  3. 데이터 블록 순차 탐색
  → 최대 2번의 직접 접근으로 임의 레코드 위치
```

---

## 13.3 디렉토리 구조 (Directory Structure)

디렉토리 = 파일명 → 파일 제어 블록(FCB) 변환 심볼 테이블

**필수 디렉토리 연산**: 탐색, 생성, 삭제, 목록, 이름 변경, 파일시스템 순회

### 13.3.1 단일 수준 디렉토리 (Single-Level Directory)

```
/
└── cat  bo  a  test  data  mail  cont  hex  records  ...

문제: 모든 파일명이 전역적으로 유일해야 함
      다중 사용자 환경에서 충돌 불가피
```

### 13.3.2 2단계 디렉토리 (Two-Level Directory)

```
MFD (Master File Directory)
├── user1/ → UFD1: {cat, a, test, x}
├── user2/ → UFD2: {data, a}
├── user3/ → UFD3: {a, test}
└── user4/ → UFD4: {data, a}
```

- 각 사용자가 독립적인 UFD(User File Directory) 보유
- 다른 사용자 파일 접근: `/username/filename` 형식의 경로명
- **Search Path**: 시스템 파일 디렉토리(user 0) 자동 탐색으로 확장

### 13.3.3 트리 구조 디렉토리 (Tree-Structured Directories)

```
/                        ← root
├── spell/
│   ├── mail/
│   │   ├── prt/
│   │   │   ├── first
│   │   │   └── last
│   │   └── list
│   ├── prog
│   └── ...
├── bin/
│   ├── find
│   ├── count
│   └── hex
└── programs/
    ├── e
    └── ...

절대 경로명: /spell/mail/prt/first
상대 경로명: (현재 디렉토리 = /spell/mail) → prt/first
```

- 각 프로세스는 **현재 디렉토리(current directory)** 보유
- 로그인 셸의 초기 디렉토리: accounting 파일에서 지정
- 디렉토리 삭제 정책:
  - 보수적: 비어 있을 때만 삭제
  - 재귀적: `rm -r` (편리하지만 위험)

### 13.3.4 비순환 그래프 디렉토리 (Acyclic-Graph Directories)

```
공유(sharing)를 위해 트리 구조 확장:

/
├── dict/         ← 실제 디렉토리
│   └── count     ← 실제 파일
├── spell/
│   └── count ──→ (dict/count를 가리키는 링크)
└── ...
```

**구현 방법:**

| 방법 | 설명 | 삭제 처리 |
|------|------|----------|
| **Symbolic Link** (소프트 링크) | 다른 파일의 경로명을 저장하는 특수 파일 | 링크만 삭제, 원본 유지. 원본 삭제 시 dangling pointer |
| **Hard Link** | 동일 파일의 디렉토리 엔트리 추가 | 참조 카운트 감소, 0이 되면 실제 삭제 |
| **디렉토리 엔트리 복제** | 정보를 두 곳에 중복 저장 | 일관성 유지 어려움 |

**Hard Link 참조 카운트 (UNIX inode):**

```
inode {
  reference_count: 3,   ← 3개의 디렉토리 엔트리가 이 inode를 가리킴
  ...
}

link 삭제 → reference_count-- → 0이 되면 inode + 데이터 블록 해제
디렉토리에는 hard link 금지 (순환 방지)
```

### 13.3.5 일반 그래프 디렉토리 (General Graph Directory)

- 링크 추가로 사이클 발생 가능
- 탐색 시 무한 루프 방지: **링크 무시** 또는 **방문 횟수 제한**
- 참조 카운트로 삭제 처리 불가 (사이클 내 셀프 참조로 카운트 > 0 유지)
- **가비지 컬렉션(Garbage Collection)** 필요:
  1. 전체 파일시스템 탐색 → 접근 가능한 모든 항목 마킹
  2. 마킹 안 된 항목 → 자유 공간 목록에 추가
  - 디스크 기반에서는 매우 느림 → 사이클 허용 시 비용 큼

---

## 13.4 보호 (Protection)

### 13.4.1 접근 유형 (Types of Access)

제어 가능한 접근 유형:
- **Read**: 파일 내용 읽기
- **Write**: 파일 내용 쓰기/덮어쓰기
- **Execute**: 파일 로드 후 실행
- **Append**: 파일 끝에 추가
- **Delete**: 파일 삭제 + 공간 해제
- **List**: 파일명 및 속성 나열
- **Attribute change**: 속성 변경

### 13.4.2 접근 제어 (Access Control)

**ACL (Access-Control List):**
- 각 파일/디렉토리에 (사용자명, 접근유형) 쌍 목록 연결
- 장점: 세밀한 접근 제어 / 단점: 가변 길이, 관리 복잡

**UNIX 압축형 접근 제어 (9비트):**

```
파일: book.tex
       owner  group  other
r w x  r w -  r - -
1 1 1  1 1 0  1 0 0  = 764 (8진수)

각 필드:
  r = 읽기 허용
  w = 쓰기 허용
  x = 실행 허용 (디렉토리: 접근/탐색 허용)

ls -l 예시:
-rw-rw-r-- 1 pbg staff 31200 Sep 3 08:30 intro.ps
drwx------ 5 pbg staff   512 Jul 8 09:33 private/
drwxrwxr-x 2 pbg staff   512 Jul 8 09:35 doc/

d: 디렉토리, -: 일반 파일
```

**Solaris 확장 (ACL 통합):**
```
-rw-r--r--+ 1 jim staff 130 May 25 22:13 file1
            ↑ "+"는 ACL 설정됨 표시
# ACL 관리
setfacl -m u:visitor:r file.txt
getfacl file.txt
```

**Windows NTFS**: GUI 기반 ACL, 명시적 allow/deny 설정 가능

**ACL 충돌 해결**: 더 세밀한 ACL 항목이 우선 (Solaris 기준)

**잠금(Mandatory vs Advisory):**

| | Mandatory | Advisory |
|---|---|---|
| 메커니즘 | OS가 강제 (잠금 위반 접근 차단) | 프로그래머가 규약 준수 필요 |
| 대표 OS | Windows | UNIX |

### 13.4.3 기타 보호 방법

- **파일별 패스워드**: 기억 부담, 유출 시 전체 노출
- **디렉토리별 패스워드**: 파일 그룹 단위 보호
- **암호화(Encryption)**: 파티션 전체 또는 개별 파일 → 강력하지만 키 관리 중요

---

## 13.5 메모리 매핑 파일 (Memory-Mapped Files)

### 기본 메커니즘

```
기존 방식:
  open() → read() → ... → write() → close()
  매 연산 = 시스템 콜 + 디스크 I/O

Memory-Mapped 방식:
  mmap() → 파일을 가상 주소 공간에 매핑
         → 이후 파일 접근 = 메모리 읽기/쓰기 (시스템 콜 없음)

┌─────────────────────────────────┐
│  Process Virtual Address Space   │
│  ...                            │
│  [mapped region] ←─────────────┐│
│    addr = 0x7f000000           ││
│    len  = file_size            ││
│  ...                            ││
└─────────────────────────────────┘│
                                   │
                    demand paging  │
                                   ↓
                         Physical Memory
                              ↕ (page fault 시)
                          Disk File
```

**동작 원리:**
1. `mmap()` 호출 → 가상 주소 공간에 파일 매핑 (아직 물리 메모리에 없음)
2. 최초 접근 시 **page fault** 발생 → 파일의 해당 블록을 물리 페이지로 로드
3. 이후 접근 → 일반 메모리 접근 (TLB hit, 매우 빠름)
4. 파일 닫힐 때 변경된 페이지를 디스크에 write-back

**Solaris 접근법**: `open()/read()/write()`로 열어도 내부적으로 커널 주소 공간에 메모리 매핑 → 모든 파일 I/O를 메모리 서브시스템으로 처리

**다중 프로세스 파일 공유:**

```
Process A virtual memory      Process B virtual memory
  [mapped region]               [mapped region]
       │                              │
       └──────────┬───────────────────┘
                  ↓
         Physical Memory (공유 페이지)
                  ↕
              Disk File

→ A가 수정한 내용을 B가 즉시 볼 수 있음
→ COW(Copy-on-Write)로 쓰기 시 개별 복사본 생성 가능
```

**공유 메모리 구현 기반**: POSIX 공유 메모리 객체 → `mmap()`으로 각 프로세스의 가상 주소에 매핑

### Windows API 메모리 매핑 예시

```c
// Producer (생산자)
HANDLE hFile = CreateFile("temp.txt",
    GENERIC_READ | GENERIC_WRITE, 0, NULL,
    OPEN_ALWAYS, FILE_ATTRIBUTE_NORMAL, NULL);

HANDLE hMapFile = CreateFileMapping(hFile,
    NULL, PAGE_READWRITE, 0, 0, TEXT("SharedObject"));

LPVOID lpMapAddr = MapViewOfFile(hMapFile,
    FILE_MAP_ALL_ACCESS, 0, 0, 0);

sprintf(lpMapAddr, "Shared memory message");  // 공유 메모리에 쓰기
UnmapViewOfFile(lpMapAddr);
CloseHandle(hFile);
CloseHandle(hMapFile);

// Consumer (소비자)
HANDLE hMapFile = OpenFileMapping(FILE_MAP_ALL_ACCESS,
    FALSE, TEXT("SharedObject"));  // 기존 매핑 열기

LPVOID lpMapAddr = MapViewOfFile(hMapFile,
    FILE_MAP_ALL_ACCESS, 0, 0, 0);

printf("Read: %s", lpMapAddr);  // 공유 메모리에서 읽기
UnmapViewOfFile(lpMapAddr);
CloseHandle(hMapFile);
```

---

## 핵심 요약

```
파일 추상화:
  파일명 → 파일 속성 + 데이터 블록
  open() → file descriptor → 모든 I/O 연산
  open-file table: per-process + system-wide 2단계

접근 방법:
  순차(Sequential) → 대부분 기본
  직접(Direct/Random) → DB, random access 파일
  인덱스(ISAM) → 대형 파일의 효율적 키 탐색

디렉토리 구조 진화:
  단일 → 2단계 → 트리 → 비순환 그래프 → 일반 그래프
  공유: 소프트 링크(dangling 위험) vs 하드 링크(reference count)

보호:
  UNIX: owner/group/other × rwx = 9비트
  ACL: 세밀한 제어, Solaris 기본 + ACL 확장
  Mandatory(Windows) vs Advisory(UNIX) locking

메모리 매핑:
  mmap() = 파일 블록을 가상 주소에 매핑 (demand paging 기반)
  장점: 시스템 콜 오버헤드 없음, 효율적 공유
  공유 메모리 = 메모리 매핑 파일의 응용
```
