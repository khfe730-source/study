# Chapter 14: File-System Implementation

## 14.1 파일 시스템 구조 (File-System Structure)

### 계층형 설계 (Layered Design)

파일 시스템은 여러 계층으로 구성되며, 각 계층은 하위 계층의 기능을 활용해 상위 계층에 서비스를 제공한다.

```
┌─────────────────────────────────┐
│       Application Programs      │
├─────────────────────────────────┤
│    Logical File System          │  ← 메타데이터, FCB/inode, 보호
├─────────────────────────────────┤
│    File-Organization Module     │  ← 논리 블록 → 물리 블록 매핑, free-space
├─────────────────────────────────┤
│    Basic File System            │  ← 블록 단위 I/O, 버퍼/캐시 관리
├─────────────────────────────────┤
│    I/O Control                  │  ← 디바이스 드라이버, 인터럽트 핸들러
├─────────────────────────────────┤
│       Devices (HDD / NVM)       │
└─────────────────────────────────┘
```

| 계층 | 역할 |
|------|------|
| **I/O Control** | 디바이스 드라이버 + 인터럽트 핸들러; "retrieve block 123" → 하드웨어 명령 변환 |
| **Basic File System** | 논리 블록 주소로 드라이브에 명령 발행; 메모리 버퍼/캐시 관리; I/O 요청 스케줄링 |
| **File-Organization Module** | 파일의 논리 블록(0~N) → 물리 블록 매핑; free-space manager 포함 |
| **Logical File System** | 메타데이터(파일 구조, 디렉토리, 권한) 관리; FCB/inode 유지 |

**주요 파일 시스템 예시**
- UNIX: UFS (Berkeley Fast File System 기반)
- Windows: FAT, FAT32, NTFS
- Linux: ext3, ext4 (표준), + 130개 이상 지원
- Google: GFS (대규모 분산 스토리지)
- FUSE: 사용자 공간(user-space)에서 파일 시스템 구현 가능

---

## 14.2 파일 시스템 운영 (File-System Operations)

### 온-스토리지 구조 (On-Storage Structures)

```
Volume Layout
┌──────────────────────────────────────────────────┐
│  Boot Control Block  │ volume 당 1개               │
│  (boot block / partition boot sector)             │
├──────────────────────────────────────────────────┤
│  Volume Control Block │ volume 당 1개              │
│  (superblock in UFS / MFT in NTFS)               │
│  • 총 블록 수, 블록 크기                            │
│  • free-block 수 + 포인터                          │
│  • free-FCB 수 + 포인터                            │
├──────────────────────────────────────────────────┤
│  Directory Structure  │ file system 당 1개         │
│  (파일명 + inode 번호 / MFT 행)                    │
├──────────────────────────────────────────────────┤
│  Per-file FCB (inode) │ 파일 당 1개                │
│  • 소유권, 권한, 파일 내용 위치                     │
│  • NTFS: MFT에 relational DB 행으로 저장           │
└──────────────────────────────────────────────────┘
```

### 인-메모리 구조 (In-Memory Structures)

마운트 시 로드 → 파일 시스템 연산 중 업데이트 → 마운트 해제 시 소거

```
┌──────────────────────────────────────────────────────────────┐
│  In-Memory Mount Table                                        │
│  (각 마운트된 볼륨 정보)                                       │
├──────────────────────────────────────────────────────────────┤
│  Directory-Structure Cache                                    │
│  (최근 접근 디렉토리 캐시; 마운트 포인트 → 볼륨 테이블 포인터) │
├──────────────────────────────────────────────────────────────┤
│  System-Wide Open-File Table                                  │
│  (열린 파일의 FCB 복사본 + 추가 정보)                          │
│  ┌──────┬──────┬─────┬──────────────────┐                    │
│  │ FCB  │ lock │ ... │  [파일 A FCB]     │                    │
│  │ 복사 │      │     │  [파일 B FCB]     │                    │
│  └──────┴──────┴─────┴──────────────────┘                    │
├──────────────────────────────────────────────────────────────┤
│  Per-Process Open-File Table                                  │
│  (system-wide table 항목 포인터 + 프로세스별 정보)             │
│  Process A: [fd 0 → stdin] [fd 1 → stdout] [fd 3 → 파일 A]   │
│  Process B: [fd 3 → 파일 A] [fd 4 → 파일 B]                  │
└──────────────────────────────────────────────────────────────┘
```

### open() / read() 파일 연산 흐름

```
open("a/b/c", O_RDONLY)
  1. per-process table에서 "a/b/c" 탐색
  2. system-wide table에서 검색
  3. 없으면: 디렉토리에서 파일 탐색 → FCB를 system-wide table에 적재
  4. per-process table에 항목 생성 → file descriptor 반환

read(fd, buf, n)
  1. per-process table[fd] → system-wide table 항목 포인터
  2. FCB에서 위치 정보 확인 → 블록 읽기
  3. 읽은 블록 → 버퍼 캐시 경유 → 사용자 공간으로 복사
```

---

## 14.3 디렉토리 구현 (Directory Implementation)

### 선형 리스트 (Linear List)

- 파일 이름 + 데이터 블록 포인터를 선형 배열로 저장
- 디렉토리 탐색: **O(n)** (파일 수에 비례)
- 단순 구현, 정렬/캐시로 성능 개선 가능
- 파일 삭제: 항목을 "삭제됨"으로 표시하거나 마지막 항목으로 덮어씀

### 해시 테이블 (Hash Table)

```
파일명 → hash() → 버킷 인덱스
                    ↓
              [항목 리스트] ← chained-overflow로 충돌 처리
```

- 평균 탐색: **O(1)**
- 테이블 크기 고정 → 크기 변경 시 rehashing 필요
- **체인 오버플로(Chained-Overflow)**: 각 버킷을 연결 리스트로 연결

---

## 14.4 할당 방법 (Allocation Methods)

### 14.4.1 연속 할당 (Contiguous Allocation)

```
start=14, length=3

Block:  14  15  16
         ↓   ↓   ↓
        [A] [A] [A]  ← 파일 A의 연속된 블록
```

- 디렉토리 항목: `(start, length)`
- **직접 접근(direct access)**: 블록 i → `start + i` (단 1회 I/O)
- **외부 단편화(external fragmentation)** 심각; compaction 비용 높음
- **Extents**: 큰 연속 청크 단위로 할당하여 단편화 완화
  - ext4, HFS+ 등에서 사용: `(start, length)` extent 목록

**First-fit / Best-fit**: 연속 할당 공간 탐색 전략 (외부 단편화 trade-off 상이)

### 14.4.2 연결 할당 (Linked Allocation)

```
Directory: jeep → block 9
block 9:  [data...][next→16]
block 16: [data...][next→1]
block 1:  [data...][next→10]
block 10: [data...][next→25]
block 25: [data...][next→-1] ← EOF
```

- 디렉토리 항목: `(start_block, end_block)`
- **직접 접근 불가**: i번째 블록 접근 시 i번 디스크 I/O
- **신뢰성 문제**: 포인터 손상 → 파일 손상; doubly linked로 부분 해결
- **클러스터(Cluster)**: 여러 블록을 묶어 포인터 오버헤드 감소, seek 감소

**FAT (File Allocation Table) — 연결 할당의 변형**

```
Volume 시작부에 FAT 위치:

FAT Index:   0    ...  217   ...  618   ...  339   ...
FAT Entry:   ...       618        339        EOF        ...

Directory: test → start=217

접근: 217→FAT[217]=618→FAT[618]=339→FAT[339]=EOF
```

- MS-DOS에서 유래; FAT12/FAT16/FAT32
- 랜덤 접근 성능 개선 (FAT 캐시 시 디스크 seek 없이 블록 위치 파악)
- FAT 전체를 메모리에 캐시하면 효율적

### 14.4.3 인덱스 할당 (Indexed Allocation)

```
파일의 index block:
┌────────────────────┐
│  ptr[0] → block 9  │
│  ptr[1] → block 16 │
│  ptr[2] → block 1  │
│  ptr[3] → block 10 │
│  ...               │
└────────────────────┘
Directory: jeep → index_block=19
```

- 외부 단편화 없음, 직접 접근 지원
- 작은 파일에서 index block 낭비

**대용량 파일을 위한 확장 방식:**

| 방식 | 설명 |
|------|------|
| **Linked Scheme** | index block 마지막 포인터 → 다음 index block |
| **Multilevel Index** | 1단계 index → 2단계 index → 데이터 (2단계: 4GB 지원) |
| **Combined Scheme** | UNIX inode 방식 |

**UNIX inode 구조 (Combined Scheme)**

```
inode
├── 12 direct block pointers       → 최대 12 × 4KB = 48KB
├── 1 single indirect pointer      → 1024 × 4KB   = 4MB
├── 1 double indirect pointer      → 1024² × 4KB  ≈ 4GB
└── 1 triple indirect pointer      → 1024³ × 4KB  ≈ 4TB
```

- 4KB 블록, 4-byte 포인터 가정
- 소규모 파일: direct block만으로 처리 → 빠름
- ZFS: **128-bit 파일 포인터** 지원 → 이론상 무한대 파일 크기
- NVM용 최적화: 디스크 헤드 이동 불필요 → seek 최소화 알고리즘 불필요, CPU 경로 단축에 집중

---

## 14.5 자유 공간 관리 (Free-Space Management)

### 비트 벡터 / 비트맵 (Bit Vector / Bitmap)

```
Block:  0  1  2  3  4  5  6  7  8  9 10 11 ...
Bit:    1  1  0  0  0  0  1  1  0  0  0  0  ...
        ↑  ↑                 ↑  ↑
       used                 used
       (0=free, 1=used 또는 반대 convention)

첫 번째 free block 탐색:
  bit_vector & (-bit_vector) → 첫 번째 0비트 위치 (CPU 명령 활용)
```

- 간단, CPU 지원으로 빠른 free block 탐색
- **단점**: 전체 비트맵을 메모리에 유지해야 함 (1TB 디스크, 4KB 블록 → 32MB 비트맵)

### 연결 리스트 (Linked List of Free Blocks)

```
free-list head → [block 2] → [block 3] → [block 4] → [block 5] → ... → null
```

- 순차 탐색만 효율적; 임의 블록 할당 느림
- 블록 포인터는 볼륨 내에 저장 → 비휘발성 유지

### 그루핑 (Grouping)

- 첫 번째 free block에 n개의 free block 주소 저장
- 마지막 주소는 다음 그룹 블록 포인터
- 한 번에 여러 free block 주소 확보 가능

### 카운팅 (Counting)

- 연속 free block이 많은 시스템에 적합
- `(first_free_block, count)` 쌍으로 관리

### ZFS Space Map (메타슬랩 기반)

```
Metaslab (할당 단위):
  Metaslab 0: [space map log]
  Metaslab 1: [space map log]
  ...

Space Map = 로그 구조 균형 트리 (log-structured balanced tree)
  → 할당/해제를 로그에 기록 후 나중에 병합
  → 연속 free 영역을 효율적으로 추적
```

- ZFS는 메타슬랩(metaslab) 단위로 저장 공간 분할
- 각 메타슬랩은 독립적인 space map 유지
- **TRIM/Unallocate**: NVM 장치에서 삭제된 블록을 장치에 알려 GC(Garbage Collection) 지원

---

## 14.6 효율성과 성능 (Efficiency and Performance)

### 클러스터링 (Clustering)

- 여러 블록을 클러스터 단위로 묶어 할당
- **장점**: 디스크 헤드 seek 감소, 블록 할당/free-list 관리 오버헤드 감소
- **단점**: 내부 단편화 증가, 소량 랜덤 I/O 성능 저하

### 통합 버퍼 캐시 (Unified Buffer Cache)

```
Before Unified Buffer Cache:
┌─────────────────────┐  ┌─────────────────────┐
│   Page Cache        │  │   Buffer Cache       │
│ (mmap/memory I/O)   │  │ (read()/write() I/O) │
└─────────────────────┘  └─────────────────────┘
→ Double Caching 문제: 같은 데이터가 두 곳에 저장됨

After Unified Buffer Cache (Linux, Solaris 등):
┌──────────────────────────────────────┐
│         Unified Buffer Cache         │
│  mmap I/O ←→ read()/write() I/O    │
└──────────────────────────────────────┘
→ 중복 캐싱 제거, 메모리 효율 향상
```

### 순차 접근 최적화

| 기법 | 설명 |
|------|------|
| **Free-Behind** | 다음 페이지 요청 시 이전 페이지를 버퍼에서 즉시 제거 (재사용 가능성 없음) |
| **Read-Ahead** | 현재 페이지 + 후속 여러 페이지를 미리 읽어 캐시 (디스크 전송 1회로 처리) |

LRU 교체 알고리즘은 순차 접근에 부적합 → 최근 사용 페이지가 다시 사용될 가능성이 낮음

### 동기/비동기 쓰기 (Synchronous vs. Asynchronous Writes)

| 구분 | 설명 | 용도 |
|------|------|------|
| **Synchronous** | 스토리지에 실제 도달할 때까지 호출 반환 안 함 | DB atomic transaction, 메타데이터 |
| **Asynchronous** | 캐시에 저장 후 즉시 반환; OS가 나중에 플러시 | 대부분의 일반 파일 쓰기 |

> 쓰기는 거의 비동기 → 파일 시스템 쓰기가 직관과 반대로 소량 I/O에서 읽기보다 빠름

---

## 14.7 복구 (Recovery)

### 14.7.1 일관성 검사 (Consistency Checking)

- **fsck** (UNIX), **chkdsk** (Windows): 디렉토리 구조, 메타데이터, free-space 비트맵을 스캔하여 불일치 탐지·수정
- 파일 시스템 마운트 전 `status bit` 확인 → 비정상 종료 시 fsck 실행
- **문제점**:
  - 대용량(수 TB) 검사에 수 시간 소요
  - 복구 불가능한 손상 가능
  - 사람의 개입 필요할 수 있음

### 14.7.2 저널링 / 로그 구조 파일 시스템 (Log-Structured / Journaling FS)

DB의 로그 기반 복구 기법을 파일 시스템 메타데이터 갱신에 적용

```
메타데이터 변경 흐름:

[변경 연산] → [로그(저널)에 순차 기록] → committed
                      ↓
              [실제 파일 시스템 구조에 비동기 반영]
                      ↓
              [완료 시 로그 포인터 업데이트]

로그 = 순환 버퍼 (circular buffer)
```

**크래시 복구:**
- 로그에 커밋된 미완료 트랜잭션 → 재실행(redo)
- 커밋 전 중단된 트랜잭션 → 롤백(undo)
- fsck 불필요 → 부팅 시간 대폭 단축

**성능 이점:**
- 동기 랜덤 메타데이터 쓰기 → 동기 순차 로그 쓰기로 변환
- 순차 I/O가 랜덤 I/O보다 HDD에서 훨씬 빠름

**채용 파일 시스템:**
- Windows: **NTFS**
- Linux: **ext3**, **ext4**
- Solaris: **UFS** (저널링 옵션), **ZFS**
- Veritas File System

### 14.7.3 덮어쓰지 않는 방식 (Never-Overwrite: WAFL / ZFS)

```
기존 방식:
  [old data] → overwrite → [new data]  ← 크래시 시 손상 위험

Never-Overwrite:
  [old data] 유지
  [new data] → 새로운 블록에 기록
  메타데이터 포인터를 새 블록으로 원자적 업데이트
  이전 블록 해제
```

- **스냅샷(Snapshot)**: 이전 블록 포인터(old root inode)를 유지하면 특정 시점 파일 시스템 뷰 제공
- ZFS: 모든 데이터·메타데이터 블록에 **체크섬(Checksum)** 추가 → 일관성 검사 불필요

### 14.7.4 백업과 복원 (Backup and Restore)

```
전형적 백업 스케줄:
  Day 1: Full Backup  (모든 파일)
  Day 2: Incremental Backup (Day 1 이후 변경된 파일)
  Day 3: Incremental Backup (Day 2 이후 변경된 파일)
  ...
  Day N: Incremental → 다시 Day 1으로 순환

복원: Full + 각 Incremental 순서대로 적용
```

- 디렉토리 항목의 **마지막 수정 시각** 비교로 변경 파일만 백업
- Full Backup 주기적 보관 ("영구 보관용")
- 오프사이트(off-site) 백업으로 재해 대비

---

## 14.8 예제: WAFL 파일 시스템 (Write-Anywhere File Layout)

NetApp의 WAFL은 **랜덤 쓰기 최적화** 목적으로 설계된 네트워크 파일 서버 전용 파일 시스템이다.

### 기본 구조

```
WAFL 파일 시스템 트리 (root inode 기반):

root inode
  ├── inode file          ← 모든 inode 포함
  ├── free-block map file ← 블록 할당 상태
  ├── free-inode map file ← inode 할당 상태
  └── ... (일반 파일들)

각 inode: 16개 포인터 (데이터 블록 또는 간접 블록)
```

- BFS(Berkeley Fast File System) 기반, 크게 수정
- 메타데이터를 일반 파일로 관리 → 위치 제약 없음 (어디든 배치 가능)
- NVRAM 캐시를 쓰기 앞단에 배치 → 안정적 쓰기 성능
- NFS, CIFS, iSCSI, ftp, http 프로토콜 지원

### 스냅샷 메커니즘

```
(a) 스냅샷 이전:
    root inode → [A][B][C][D][E]

(b) 스냅샷 직후 (블록 변경 전):
    root inode ─────────────────────┐
    snapshot inode ─────────────────┤
                                    ↓
                          [A][B][C][D][E]

(c) 블록 D 수정 후:
    root inode → [A][B][C][D'][E]   ← 새 블록 D'
    snapshot inode → [A][B][C][D][E] ← 원본 블록 D 유지
```

**스냅샷 생성**: root inode 복사본 생성 → 이후 모든 변경은 새 블록에 기록

**Free-block map 특성:**
- 블록당 1비트가 아닌 **다중 비트 비트맵** (스냅샷별 사용 여부 추적)
- 모든 스냅샷이 해당 블록을 해제할 때만 실제 free

### 클론 (Clone)

- **Read-Write 스냅샷**: 스냅샷을 기반으로 읽기-쓰기 가능한 복사본 생성
- 쓰기 시 새 블록에 기록 + 클론 포인터 업데이트
- 원본 스냅샷은 불변 유지
- 테스트, 버전 관리, 복제(replication)에 활용

### 쓰기 성능 최적화

- **Write-Anywhere**: 항상 가장 가까운 free 블록에 기록 → HDD 헤드 이동 최소화
- NVRAM 캐시: 쓰기 요청을 즉시 수용, 배치로 플러시
- 랜덤 쓰기 부하(NFS/CIFS 서버 환경)에 특히 효율적

---

## 핵심 비교 요약

### 할당 방법 비교

| 방법 | 직접 접근 | 외부 단편화 | 신뢰성 | 대표 사례 |
|------|:-------:|:---------:|:-----:|---------|
| 연속(Contiguous) | ✅ O(1) | 심각 | 높음 | CD-ROM, 초기 FS |
| 연결(Linked) | ❌ O(n) | 없음 | 낮음 (포인터 손상) | FAT (변형) |
| FAT | △ O(n)* | 없음 | 중간 | MS-DOS, FAT32 |
| 인덱스(Indexed) | ✅ O(1) | 없음 | 높음 | UNIX UFS, ext4 |

*FAT 캐시 시 O(1) 가능

### 자유 공간 관리 비교

| 방법 | 탐색 성능 | 메모리 사용 | 적합 환경 |
|------|:--------:|:---------:|---------|
| 비트 벡터 | O(n/w)* | 중간 | 범용 |
| 연결 리스트 | O(free) | 낮음 | 순차 할당 |
| 그루핑 | O(1) 배치 | 낮음 | 대규모 할당 |
| 카운팅 | O(1) 연속 | 낮음 | 연속 free 영역 많을 때 |
| ZFS Space Map | O(log n) | 가변 | 대형 풀 스토리지 |

*w = 워드 비트 수 (32/64)

### 복구 방식 비교

| 방식 | 복구 시간 | 안전성 | 대표 FS |
|------|:--------:|:-----:|--------|
| Consistency Check (fsck) | 느림 (수 시간) | 손상 가능 | 초기 UNIX FS |
| Journaling | 빠름 (로그 재실행) | 높음 | NTFS, ext3/4 |
| Never-Overwrite + Checksum | 즉시 | 매우 높음 | ZFS, WAFL |
