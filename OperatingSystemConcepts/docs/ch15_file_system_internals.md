# Chapter 15: File-System Internals

## 15.1 파일 시스템 (File Systems)

현대 범용 OS는 단일 파일 시스템이 아닌 다수의 파일 시스템 타입을 동시에 지원한다.

**Solaris 파일 시스템 예시:**

| 파일 시스템 | 유형 | 설명 |
|------------|------|------|
| `tmpfs` | 가상(Virtual) | 휘발성 메인 메모리에 생성; 재부팅/크래시 시 내용 소거 |
| `objfs` | 가상 | 커널 인터페이스; 디버거에 커널 심볼 접근 제공 |
| `ctfs` | 가상 | 부팅 시 시작되는 프로세스의 "계약(contract)" 정보 관리 |
| `lofs` | 루프백(Loopback) | 하나의 파일 시스템을 다른 위치에서 접근 가능하게 함 |
| `procfs` | 가상 | 모든 프로세스 정보를 파일 시스템 인터페이스로 노출 |
| `ufs` | 범용 | UNIX File System |
| `zfs` | 범용 | Zettabyte File System |

```
일반적인 스토리지 구성:

  Physical Disk
  ├── Partition 0  ──→  Volume  ──→  File System (ext4)
  ├── Partition 1  ──→  Volume  ──→  File System (zfs)
  └── Partition 2  ──→  Raw (swap, DB 전용)

  Volume Manager를 사용하면:
  ├── Partition 0 ─┐
  └── Partition 1 ─┤──→  Volume (span)  ──→  File System
```

---

## 15.2 파일 시스템 마운팅 (File-System Mounting)

파일 시스템은 사용 전 반드시 **마운트(mount)** 되어야 한다. OS에 디바이스 이름과 마운트 포인트(mount point)를 제공하면 OS가 유효한 파일 시스템인지 검증 후 디렉토리 구조에 붙인다.

```
마운트 전:
/
├── home/
├── users/    ← 빈 디렉토리 (마운트 포인트)
└── etc/

/device/dsk 에 있는 볼륨:
.
├── bill/
├── fred/
└── sue/

마운트 후 (mount /device/dsk /users):
/
├── home/
├── users/        ← 마운트 포인트
│   ├── bill/     ← /device/dsk 내용
│   ├── fred/
│   └── sue/
└── etc/
```

**OS별 마운트 처리:**

| OS | 방식 |
|----|------|
| **UNIX/Linux** | 명시적 `mount` 명령; `/etc/fstab`으로 부팅 시 자동 마운트 |
| **macOS** | 새 디스크 감지 시 `/Volumes` 아래 자동 마운트 |
| **Windows** | 드라이브 레터(C:, D:) 할당 방식; 최신 버전은 디렉토리 마운트도 지원 |

**UNIX 마운트 구현:**
- 마운트 포인트 디렉토리의 inode 인-메모리 복사본에 플래그 설정
- 플래그가 있는 inode는 마운트 테이블 항목을 가리킴
- 마운트 테이블 항목 → 해당 장치 파일 시스템의 superblock 포인터
- 이로써 OS가 파일 시스템 타입이 달라도 디렉토리 구조를 투명하게 탐색 가능

---

## 15.3 파티션과 마운팅 (Partitions and Mounting)

### Raw vs. Cooked 파티션

```
파티션 유형:

  Raw (파일 시스템 없음)              Cooked (파일 시스템 있음)
  ┌────────────────────┐              ┌────────────────────┐
  │  UNIX swap space   │              │  boot block        │
  │  (자체 포맷 사용)   │              │  superblock        │
  │  Database raw I/O  │              │  inodes            │
  │  RAID bitmap       │              │  data blocks       │
  └────────────────────┘              └────────────────────┘
```

### 부트 파티션과 부트 로더

부팅 가능한 파티션은 부트 정보를 별도 포맷으로 포함한다. 부팅 시 파일 시스템 코드가 아직 로드되지 않았으므로 부트 정보는 파일 시스템 포맷과 독립적으로 구성된다.

```
부팅 순서:
  BIOS/UEFI
      ↓
  Boot Loader (MBR 또는 EFI 파티션)
  ├── 파일 시스템 구조를 이해
  ├── 커널 위치 파악 및 메모리 로드
  └── 커널 실행 시작

  멀티부트(Dual Boot):
  Boot Loader (GRUB 등)
  ├── OS 1 선택 → 해당 파티션 커널 로드
  └── OS 2 선택 → 다른 파티션 커널 로드
```

**루트 파티션(Root Partition):** 부트 로더가 선택한 파티션; OS 커널과 시스템 파일 포함, 부팅 시 자동 마운트

---

## 15.4 파일 공유 (File Sharing)

### 다중 사용자 환경에서의 파일 공유

다중 사용자 시스템에서는 **소유자(owner)**와 **그룹(group)** 개념으로 파일 공유와 보호를 관리한다.

```
파일 메타데이터:
  owner_id  = 1000   ← 파일 소유자 (최대 권한)
  group_id  = 200    ← 그룹 멤버들과 공유
  permissions = rwxr-xr--

접근 판별:
  요청 사용자 ID == owner_id  → 소유자 권한 적용
  요청 사용자 ID ∈ group      → 그룹 권한 적용
  그 외                       → 기타 사용자 권한 적용
```

**이동식 디스크 주의사항:** 시스템 간 ID가 다를 수 있으므로 이식 시 소유권 재설정이 필요하다.

---

## 15.5 가상 파일 시스템 (Virtual File Systems, VFS)

### VFS 계층 구조

```
┌──────────────────────────────────────────────────────┐
│          File-System Interface                        │
│      (open, read, write, close, file descriptors)    │
├──────────────────────────────────────────────────────┤
│              VFS Interface                            │
│  ┌─────────────────────────────────────────────┐     │
│  │  vnode: 네트워크 전체 유일 파일 표현 구조체  │     │
│  └─────────────────────────────────────────────┘     │
├───────────────┬──────────────────┬───────────────────┤
│ Local FS      │ Local FS         │ Remote FS         │
│ Type 1 (ext4) │ Type 2 (zfs)     │ Type 1 (NFS)      │
│ [disk]        │ [disk]           │ [network]         │
└───────────────┴──────────────────┴───────────────────┘
```

**VFS의 두 가지 핵심 역할:**

1. **추상화(Abstraction)**: 파일 시스템 고유 구현을 VFS 인터페이스 뒤로 숨김 → 여러 파일 시스템 타입 동시 마운트 가능
2. **네트워크 고유 식별**: `vnode`는 네트워크 전체에서 유일한 파일 식별자를 포함 (UNIX inode는 단일 파일 시스템 내에서만 유일)

### Linux VFS 4대 객체

| 객체 | 표현 대상 |
|------|---------|
| `inode` object | 개별 파일 |
| `file` object | 열린(open) 파일 |
| `superblock` object | 전체 파일 시스템 |
| `dentry` object | 개별 디렉토리 항목 |

각 객체 타입은 **함수 포인터 테이블**을 가지며, VFS는 객체 타입을 몰라도 해당 함수를 호출할 수 있다:

```c
// file object의 함수 테이블 예시 (linux/fs.h의 struct file_operations)
struct file_operations {
    int     (*open)  (...);
    int     (*close) (...);
    ssize_t (*read)  (...);
    ssize_t (*write) (...);
    int     (*mmap)  (...);
    // ...
};

// VFS가 read를 호출할 때:
file->f_op->read(file, buf, count, pos);
// → ext4 구현, zfs 구현, NFS 구현 중 해당 함수가 자동 호출됨
```

이 **객체 지향(object-oriented)** 설계 덕분에 VFS는 실제 파일이 로컬 디스크인지, 원격 NFS인지 알 필요 없이 동일한 인터페이스로 처리한다.

---

## 15.6 원격 파일 시스템 (Remote File Systems)

### 발전 단계

```
1세대: ftp
  Client → [ftp] → Server
  파일을 명시적으로 전송; 투명한 접근 불가

2세대: DFS (Distributed File System)
  Client → [mount] → Server
  원격 디렉토리를 로컬처럼 투명하게 접근 가능

3세대: WWW
  ftp와 유사한 명시적 전송 방식으로 회귀
  브라우저 기반, 투명성 없음

현재: Cloud Storage
  API 기반 접근; 위치 투명성 제공
```

### 15.6.1 클라이언트-서버 모델 (Client-Server Model)

- **서버**: 파일 시스템을 볼륨/디렉토리 단위로 export
- **클라이언트**: 마운트 요청으로 원격 파일 시스템 접근
- 클라이언트 신원 확인 어려움 → IP 스푸핑(spoofing) 위험

**NFS 인증 방식:**
- 기본값: 클라이언트 네트워킹 정보 기반 (user ID 일치 필요)
- 보안 강화: 암호화 키(encrypted keys) 사용

```
보안 이슈 시나리오:
  Client의 user ID = 1000
  Server의 실제 해당 user ID = 2000

  → Server는 ID 1000으로 권한 검사
  → 잘못된 인증으로 접근 허용/거부 결정
```

### 15.6.2 분산 정보 시스템 (Distributed Information Systems)

클라이언트-서버 시스템 관리 단순화를 위한 통합 네임스페이스 제공:

| 시스템 | 제공 기능 |
|--------|---------|
| **DNS** | 호스트명 → 네트워크 주소 변환 (전 인터넷) |
| **NIS** (Sun, 구 Yellow Pages) | 사용자명/비밀번호/호스트명 중앙화; 비암호화 전송으로 보안 취약 |
| **NIS+** | NIS 보안 강화판; 복잡성으로 미채택 |
| **CIFS + Active Directory** | Microsoft; Kerberos 기반 인증; 단일 로그인(single sign-on) |
| **LDAP** | 경량 디렉토리 접근 프로토콜; Active Directory 기반; 현재 산업 표준화 추세 |

LDAP의 목표: 조직 내 모든 사용자와 자원 정보를 단일 분산 디렉토리에 통합 → 안전한 SSO(Single Sign-On) 구현

### 15.6.3 장애 모드 (Failure Modes)

**로컬 파일 시스템 장애 원인:** 드라이브 고장, 메타데이터 손상, 디스크 컨트롤러 장애, 케이블 장애 등

**원격 파일 시스템 추가 장애 원인:**
```
장애 시나리오:
  Client ←── 네트워크 단절 ──→ Server
  Client ←── 서버 크래시   ──→ (서버 없음)
  Client ←── 서버 점검 종료──→ (계획된 중단)

처리 전략:
  Option A: 즉시 모든 원격 FS 연산 종료 (데이터 손실 위험)
  Option B: 서버 복구까지 연산 지연 (대부분의 DFS 선택)
```

**NFS v3 접근:** 상태 없는(stateless) DFS → 크래시 후 복구 용이하나 보안 취약
**NFS v4:** 상태 유지(stateful) → 보안·성능·기능성 개선

---

## 15.7 일관성 의미론 (Consistency Semantics)

파일을 공유할 때, 한 사용자의 변경이 다른 사용자에게 언제 보여지는지를 정의하는 규칙

### 15.7.1 UNIX 의미론 (UNIX Semantics)

```
User A: write(fd, "hello")  ──→ 즉시 반영
User B: read(fd, buf)       ←── "hello" 읽음 (즉시 가시성)

공유 파일 포인터:
  User A가 포인터를 10 이동시키면
  User B도 동일한 파일 포인터 위치에서 읽음
```

- 파일은 **단일 물리 이미지** → 모든 접근이 직렬화
- **가장 엄격한 일관성**, 파일 접근 충돌 시 지연 발생

### 15.7.2 세션 의미론 (Session Semantics)

Andrew File System (OpenAFS) 사용:

```
User A: open("file")  → 자신만의 이미지 생성
        write(...)    → A의 이미지에만 반영
        (User B는 이 변경 즉시 볼 수 없음)
        close("file") → 변경 사항 커밋

User B: 이후 새로 open("file")  → A의 변경 사항 반영된 버전 확인
        (이미 열어둔 B의 인스턴스는 변경 사항 미반영)
```

- 파일이 동시에 여러 다른 이미지로 존재 가능
- 지연 없이 동시 읽기/쓰기 허용
- 분산 파일 시스템에 적합 (캐시 활용)

### 15.7.3 불변 공유 파일 의미론 (Immutable-Shared-Files Semantics)

- 파일이 shared로 선언되면 **이름 재사용 불가 + 내용 변경 불가**
- 완전한 읽기 전용 공유
- 분산 시스템에서 구현 단순 (쓰기 없으므로 동기화 불필요)

### 세 의미론 비교

| 의미론 | 쓰기 가시성 | 동시성 | 일관성 수준 | 사용 환경 |
|--------|:----------:|:------:|:----------:|---------|
| UNIX | 즉시 | 낮음 (직렬화) | 엄격 | 단일 시스템 |
| Session (AFS) | close 후 | 높음 | 완화 | 분산 FS |
| Immutable | 없음 (읽기 전용) | 최고 | 해당 없음 | 정적 데이터 공유 |

---

## 15.8 NFS (Network File System)

NFS는 LAN(또는 WAN)에서 원격 파일에 투명하게 접근하기 위한 클라이언트-서버 프로토콜이다. ONC+ 표준의 일부로, 대부분의 UNIX 벤더가 지원한다. 여기서는 NFS Version 3을 기준으로 설명한다.

### 15.8.1 개요

```
독립적인 세 파일 시스템 (마운트 전):
  U:/           S1:/           S2:/
  ├── usr/      ├── usr/       ├── usr/
  │   └── local │   └── shared │   └── dir2
  └── ...       │       └── dir1└── ...
                └── ...

마운트 후 (U에서 S1:/usr/shared → /usr/local):
  U:/
  ├── usr/
  │   └── local/       ← S1:/usr/shared로 대체
  │       └── dir1/    ← S1의 dir1 접근 가능
  └── ...

캐스케이딩 마운트 (S2:/usr/dir2 → U:/usr/local/dir1):
  U:/usr/local/dir1/ ← 이제 S2의 dir2 내용
```

**설계 목표:**
- 이기종 환경(다른 OS, 아키텍처) 지원
- RPC(Remote Procedure Call) + XDR(eXternal Data Representation)로 구현 독립성 달성
- 두 독립 프로토콜: **마운트 프로토콜** + **NFS 프로토콜**

### 15.8.2 마운트 프로토콜 (Mount Protocol)

```
마운트 과정:
  Client                          Mount Server (서버 측)
    │                                    │
    │─── mount request ─────────────────→│
    │    (원격 디렉토리 이름, 서버 이름)  │
    │                              export list 확인
    │                              (/etc/dfs/dfstab)
    │←── file handle ────────────────────│
    │    (파일 시스템 식별자 + inode 번호)│
    │                                    │
  이후 모든 파일 접근에 file handle 사용
```

- 서버의 **export list** (`/etc/dfs/dfstab`): 마운트 허용 파일 시스템과 허용 클라이언트 목록
- **file handle**: 서버가 파일을 구분하는 키 (파일 시스템 ID + inode 번호)
- 정적 마운트 설정: `/etc/vfstab` (Solaris)

### 15.8.3 NFS 프로토콜 (NFS Protocol)

NFS 서버는 **상태 없는(stateless)** 설계 — 서버가 클라이언트 세션 상태를 유지하지 않는다.

**지원 연산:**
- 디렉토리 내 파일 탐색
- 디렉토리 항목 목록 읽기
- 링크/디렉토리 조작
- 파일 속성 접근
- 파일 읽기/쓰기

**open/close 연산이 없는 이유:**

```
Stateless 서버:
  - 클라이언트 정보 미보관 → 크래시 후 복구 간단
  - 각 요청에 완전한 정보 포함 (파일 식별자 + 절대 오프셋)
  - 멱등성(Idempotent): 동일 연산 반복 수행 = 1회 수행과 동일 결과
  - 시퀀스 번호로 중복/누락 요청 감지

  단점:
  - 수정 데이터를 서버 디스크에 커밋 후 결과 반환 (동기 쓰기 강제)
  - 캐싱 이점 상실 → 성능 패널티
  - 해결: NVRAM 캐시(배터리 백업 메모리) → 논리적 동기 쓰기 + 실제 비동기 플러시
```

**동시성 제한:**
- NFS 자체에 동시성 제어 메커니즘 없음
- `write()` 시스템 콜이 여러 RPC write로 분할될 수 있음
- 잠금(lock) 관리는 NFS 외부 서비스 사용 (Solaris의 잠금 관리자)

### 15.8.4 경로명 변환 (Path-Name Translation)

```
/usr/local/dir1/file.txt 접근:

  컴포넌트 분리: [usr] [local] [dir1] [file.txt]

  1. lookup(root_vnode, "usr")     → usr_vnode    (로컬)
  2. lookup(usr_vnode, "local")    → local_vnode  (로컬)
     * 마운트 포인트 탐지 → 이후 NFS RPC 사용
  3. RPC_lookup(server, dir_vnode, "dir1")  → dir1_vnode  (원격)
  4. RPC_lookup(server, dir1_vnode, "file.txt") → file_vnode (원격)
```

- 마운트 포인트를 넘으면 **컴포넌트마다 별도 RPC** 발생 → 비용 높음
- **디렉토리-이름 룩업 캐시(directory-name-lookup cache)**: 원격 디렉토리 이름 → vnode 캐시로 성능 개선
  - 서버의 속성(attribute)과 캐시 불일치 시 캐시 무효화

### 15.8.5 원격 연산 (Remote Operations)

```
NFS 아키텍처 (VFS 통합):

Client                              Server
─────────────────────────────────────────────────────
  System Call (read/write)
       ↓
  OS Layer → VFS operation
       ↓
  VFS: 원격 파일 감지
       ↓
  NFS Client ─────── RPC/XDR ──────→ NFS Server
                      network               ↓
                                      VFS layer
                                            ↓
                                      Local FS (UNIX FS)
                                            ↓
                                      disk I/O
                                            ↓ 결과 반환
  ←────────── RPC response ──────────────
```

**캐싱 전략:**

| 캐시 종류 | 내용 | 유효성 |
|----------|------|-------|
| **파일 속성 캐시** | inode 정보 | 기본 60초 후 폐기; 서버에서 새 속성 도착 시 갱신 |
| **파일 블록 캐시** | 파일 데이터 | 속성 캐시가 최신일 때만 사용 |

- **read-ahead** + **delayed-write** 기술 사용
- 지연 쓰기 블록은 서버 확인 전까지 해제하지 않음
- **NFS 일관성 한계**: 새 파일이 30초간 다른 클라이언트에 보이지 않을 수 있음 → UNIX 의미론도, 세션 의미론도 완전히 따르지 않음 → 그럼에도 불구하고 가장 널리 사용되는 다중 벤더 분산 파일 시스템

---

## 핵심 비교 요약

### 파일 시스템 접근 방식 비교

```
로컬 FS 접근:
  App → syscall → VFS → [ext4|zfs|...] → disk driver → disk

원격 FS 접근 (NFS):
  App → syscall → VFS → NFS client → [network] → NFS server → VFS → local FS → disk
```

### 일관성 의미론 vs. NFS 실제 동작

```
이론                 실제 NFS 동작
──────────────────────────────────────────────────
UNIX semantics       ✗ (delayed write로 즉시성 없음)
Session semantics    ✗ (이미 열린 파일에 변경 반영 불완전)
Immutable semantics  ✗ (쓰기 지원)

NFS의 실제: "근사한 UNIX 의미론"
  - 새 파일: 최대 30초 지연 노출
  - 쓰기: 캐시로 인해 다른 클라이언트에 지연 반영
  - 실용성과 성능을 위해 엄격한 일관성 희생
```

### VFS가 제공하는 추상화 효과

```
VFS 없을 때:
  open_ext4(), read_ext4(), write_ext4()
  open_zfs(),  read_zfs(),  write_zfs()
  open_nfs(),  read_nfs(),  write_nfs()
  ... (파일 시스템마다 별도 코드)

VFS 있을 때:
  open(), read(), write()  ← 단일 인터페이스
       ↓
  vnode->f_op->read()  ← 파일 시스템별 함수 포인터 자동 호출
```

### NFS Stateless 설계의 장단점

| 항목 | 내용 |
|------|------|
| **장점** | 서버 크래시 후 복구 간단, 구현 단순 |
| **단점** | 동기 쓰기 강제로 성능 패널티, 보안 취약 (위조 요청 허용 가능) |
| **NFS v4 개선** | Stateful 설계 → 보안/성능/기능 향상 |
