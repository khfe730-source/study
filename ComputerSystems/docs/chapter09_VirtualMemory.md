# Chapter 9: Virtual Memory

## 핵심 주제

가상 메모리(VM)는 하드웨어 예외, 주소 변환, 메인 메모리, 디스크, 커널 소프트웨어의 상호작용으로 각 프로세스에 크고 균일하며 독립적인 주소 공간을 제공한다. 세 가지 핵심 역할:
1. **캐시 도구**: 메인 메모리를 디스크에 저장된 주소 공간의 캐시로 활용
2. **메모리 관리 도구**: 프로세스마다 균일한 주소 공간 제공, 링킹·로딩·공유 단순화
3. **메모리 보호 도구**: 프로세스 간 주소 공간 격리

---

## 9.1 물리 주소 지정 vs 가상 주소 지정

```
물리 주소 지정 (Physical Addressing):
CPU → [물리 주소] → 메인 메모리
(초기 PC, DSP, 임베디드에서 사용)

가상 주소 지정 (Virtual Addressing):
CPU → [가상 주소 VA] → MMU → [물리 주소 PA] → 메인 메모리
                        ↑
                  MMU(Memory Management Unit):
                  CPU 칩 내부의 하드웨어 주소 변환기
                  페이지 테이블을 통해 VA → PA 변환 (on-the-fly)
```

---

## 9.2 Address Spaces

- **가상 주소 공간**: N = 2ⁿ 개의 주소 {0, 1, ..., N−1} (현대 시스템: n=48, 256TB)
- **물리 주소 공간**: M = 2ᵐ 개의 주소 {0, 1, ..., M−1} (Core i7: m=52, 4PB)

각 바이트는 가상 주소와 물리 주소를 모두 가진다. VM의 핵심: 동일한 데이터 객체가 여러 주소 공간에서 서로 다른 주소를 가질 수 있다.

---

## 9.3 VM as a Tool for Caching

### DRAM 캐시 구조

가상 메모리는 디스크에 저장된 N 바이트 배열이며, 메인 메모리(DRAM)가 이 배열의 캐시 역할을 한다.

- **가상 페이지(VP, Virtual Page)**: P = 2ᵖ bytes 크기의 블록 (Linux: P = 4KB)
- **물리 페이지(PP, Physical Page) = 페이지 프레임**: 동일한 P bytes 크기

**가상 페이지의 세 가지 상태:**

```
Unallocated: 아직 OS가 할당하지 않음 (디스크 공간도 없음)
Cached:      DRAM에 현재 캐시됨
Uncached:    할당됐지만 현재 DRAM에 없음 (디스크에 있음)
```

**DRAM 캐시 특성 (SRAM 캐시와 비교):**
| 특성 | SRAM 캐시 | DRAM 캐시(VM) |
|------|---------|------------|
| 미스 패널티 | 수십~수백 cycles | 수십 ms (디스크) |
| 블록 크기 | 64 bytes | 4KB~2MB (매우 큰 이유) |
| 연관도 | 8/16-way | **완전 연관(fully associative)** |
| 교체 정책 | LRU (HW) | 정교한 소프트웨어 알고리즘 |
| 쓰기 정책 | write-through or write-back | **항상 write-back** |

### 페이지 테이블 (Page Table)

MMU가 VA→PA 변환 시 참조하는 자료구조. **페이지 테이블 항목(PTE, Page Table Entry)** 배열.

```
페이지 테이블 구조:

       valid bit  물리 페이지 번호(PPN) 또는 디스크 주소
PTE 0 [  0  |   null   ]    → 미할당 (VP 0)
PTE 1 [  1  |  PP 0    ]    → DRAM에 캐시됨 (VP 1 → PP 0)
PTE 2 [  1  |  PP 1    ]    → DRAM에 캐시됨 (VP 2 → PP 1)
PTE 3 [  0  | disk addr]    → 디스크에 있음 (VP 3, 미캐시)
PTE 4 [  1  |  PP 3    ]    → DRAM에 캐시됨
...
PTE 7 [  1  |  PP 2    ]    → DRAM에 캐시됨

valid=1: PPN이 DRAM 내 물리 페이지 번호
valid=0 + null: 미할당
valid=0 + addr: 할당됐지만 디스크에 있음
```

### 페이지 히트 vs 페이지 폴트

**페이지 히트**: 참조한 가상 주소의 VP가 DRAM에 캐시됨 → PTE.valid=1 → MMU가 PA 계산

**페이지 폴트 (DRAM 캐시 미스)**:
```
1. CPU가 VA 생성 → MMU가 PTE 읽음 → valid=0 → 페이지 폴트 예외 발생
2. 커널 페이지 폴트 핸들러 실행:
   - 희생 페이지(victim page) 선택 (예: VP 4 → PP 3)
   - VP 4가 수정(dirty)됐으면 디스크에 기록
   - PTE 4 수정: valid=0, 디스크 주소 저장
3. VP 3을 디스크에서 PP 3으로 복사
   PTE 3 수정: valid=1, PPN=3
4. 페이지 폴트 핸들러 복귀 → 폴트 명령어 재실행 → 이번엔 히트
```

**스와핑(Swapping) / 페이징(Paging)**: 디스크 ↔ DRAM 간 페이지 이동
- Swap in (page in): 디스크 → DRAM
- Swap out (page out): DRAM → 디스크
- **Demand paging**: 페이지 미스 시에만 디스크에서 로드 (현대 시스템 모두 채택)

**스래싱(Thrashing)**: 작업 집합(working set)이 물리 메모리를 초과하면 페이지를 계속 스왑 in/out → 성능 급락

---

## 9.4 VM as a Tool for Memory Management

OS는 프로세스마다 **독립적인 페이지 테이블**을 유지한다.

```
프로세스 i 페이지 테이블:     프로세스 j 페이지 테이블:
VP 1 → PP 2                  VP 1 → PP 7 (공유 가능!)
VP 2 → PP 7 (공유 가능!)     VP 2 → PP 10
```

VM이 메모리 관리를 단순화하는 방법:

**링킹 단순화**: 모든 프로세스가 동일한 가상 메모리 구조 사용 (코드=0x400000 시작). 링커가 물리 주소와 무관하게 실행 파일 생성 가능.

**로딩 단순화**: 로더가 `.text`/`.data` 영역을 위한 가상 페이지를 생성하고 PTE를 파일 위치로 설정. 실제 데이터는 **demand paging**으로 처음 참조 시에만 로드. 디스크에서 메모리로의 실제 복사 없음.

**공유 단순화**: 커널 코드, libc.so 같은 공용 라이브러리를 물리 메모리에 단 한 벌만 두고, 여러 프로세스의 서로 다른 가상 페이지가 동일한 물리 페이지를 가리키게 함.

**메모리 할당 단순화**: `malloc`이 k 연속 가상 페이지를 요청하면, OS는 임의의 k개 물리 페이지를 매핑. 물리 메모리에서 연속 공간 불필요.

---

## 9.5 VM as a Tool for Memory Protection

PTE에 권한 비트(permission bits) 추가:

```
PTE 형식 (권한 비트 포함):
┌─────┬──────┬───────┬─────────────────────────┐
│ SUP │ READ │ WRITE │ PPN/disk address         │
└─────┴──────┴───────┴─────────────────────────┘

SUP:   커널(supervisor) 모드에서만 접근 가능?
READ:  읽기 권한
WRITE: 쓰기 권한
```

권한 위반 시: CPU가 일반 보호 폴트(general protection fault) 발생 → 커널이 SIGSEGV 전송 → "Segmentation fault"

---

## 9.6 Address Translation

### 기호 정리

| 기호 | 의미 |
|------|------|
| N = 2ⁿ | 가상 주소 공간 크기 |
| M = 2ᵐ | 물리 주소 공간 크기 |
| P = 2ᵖ | 페이지 크기 (bytes) |
| VPN | 가상 페이지 번호 (Virtual Page Number) |
| VPO | 가상 페이지 오프셋 (Virtual Page Offset) |
| PPN | 물리 페이지 번호 (Physical Page Number) |
| PPO | 물리 페이지 오프셋 (= VPO, 항상 동일) |
| TLBI | TLB 세트 인덱스 |
| TLBT | TLB 태그 |

### 기본 주소 변환

```
가상 주소 (n bits):
┌─────────────────────────────┬─────────────┐
│     VPN (n−p bits)          │  VPO (p bits) │
└─────────────────────────────┴─────────────┘

PTBR(Page Table Base Register) → 페이지 테이블 배열[VPN] → PTE
PTE.valid=1 → PPN 추출

물리 주소 (m bits):
┌──────────────────────────┬─────────────┐
│    PPN (m−p bits)        │  PPO (p bits) │
└──────────────────────────┴─────────────┘
              ↑
         PTE에서 추출           = VPO (그대로)
```

**페이지 히트 처리 (하드웨어만):**
```
CPU → VA → MMU → PTE 주소 계산 → 캐시/메모리에서 PTE 읽기
→ valid=1 → PPN 추출 → PA 구성 → 캐시/메모리에서 데이터 반환 → CPU
```

**페이지 폴트 처리 (하드웨어 + OS):**
```
CPU → VA → MMU → PTE.valid=0 → 페이지 폴트 예외
→ OS 폴트 핸들러:
   1. 희생 페이지 선택 → dirty면 스왑 아웃
   2. 새 페이지 스왑 인
   3. PTE 업데이트
→ 폴트 명령어 재실행 → 이번엔 히트
```

### TLB (Translation Lookaside Buffer)

MMU 내부의 소형 PTE 캐시. 모든 VA→PA 변환마다 PTE를 메모리에서 읽는 오버헤드(수십~수백 cycles)를 제거.

```
가상 주소 분할 (TLB 접근):
┌────────────────────────┬────────────┬───────────────┐
│  TLBT (VPN 상위 비트) │ TLBI (하위) │ VPO           │
└────────────────────────┴────────────┴───────────────┘
              VPN

TLB 히트:  MMU가 TLB에서 PPN 즉시 획득 → PA 구성
TLB 미스: MMU가 L1 캐시/메모리에서 PTE 읽기 → TLB 갱신 → PA 구성
```

TLB는 일반적으로 높은 연관도(4-way 이상)로 구성된다.

### 다단계 페이지 테이블 (Multi-Level Page Tables)

**문제**: 32-bit 주소 공간 + 4KB 페이지 + 4-byte PTE = 4MB 페이지 테이블 상시 메모리 상주 필요. 64-bit에서는 불가능한 크기.

**해결: k-단계 페이지 테이블 계층**

```
2단계 (32-bit 예시):
가상 주소: [VPN1(10) | VPN2(10) | VPO(12)]

1단계 테이블: 1,024 PTEs, 각 PTE가 4MB 커버
  → 할당된 영역만 2단계 테이블 존재 (대부분 null)
2단계 테이블: 각각 1,024 PTEs, 각 PTE가 4KB 커버

이점:
1. 1단계 PTE가 null이면 2단계 테이블 자체가 불필요
2. 1단계 테이블만 항상 메모리 상주; 2단계는 필요 시 페이지인
→ 전형적인 프로그램은 할당된 페이지가 적으므로 대부분 null
```

```
k단계 일반화:
VA: [VPN₁|VPN₂|...|VPNₖ|VPO]

각 VPNᵢ → i단계 페이지 테이블의 오프셋
마지막(k단계) PTE → 물리 페이지 PPN + PPO = PA

TLB가 다단계 PTE를 캐시하므로 실제 성능 저하 미미.
```

### End-to-End 주소 변환 예시 (14-bit VA, 12-bit PA)

```
파라미터: n=14, m=12, P=64(2⁶), TLB 4-way 16entry, L1 직접매핑 16set 4B블록

VA = 0x03d4 = 0000 1111 01 0100
     ├─ VPN = 0x0F = 0b00001111
     │   ├─ TLBI = VPN[1:0] = 0b11 = 0x3
     │   └─ TLBT = VPN[7:2] = 0b000011 = 0x3
     └─ VPO = 0x14 = 0b010100

TLB 검색: set 0x3, tag 0x3 → 히트 → PPN = 0x0D

PA = PPN:PPO = 0x0D:0x14 = 0x354
   = 0000 1101 0101 00
     ├─ CT = PA[11:6] = 0x0D
     ├─ CI = PA[5:2]  = 0x5
     └─ CO = PA[1:0]  = 0x0

L1 캐시: set 0x5, tag 0x0D → 히트 → data = 0x36 반환
```

---

## 9.7 Intel Core i7 / Linux 메모리 시스템

### Core i7 주소 변환

```
48-bit 가상 주소 (VA):
┌──────┬──────┬──────┬──────┬─────────────┐
│ VPN1 │ VPN2 │ VPN3 │ VPN4 │ VPO (12bit) │
│ 9bit │ 9bit │ 9bit │ 9bit │             │
└──────┴──────┴──────┴──────┴─────────────┘

→ 4단계 페이지 테이블 계층
CR3 → L1 PGD → L2 PUD → L3 PMD → L4 PTE → PPN
(Page Global Dir) (Page Upper Dir) (Page Middle Dir) (Page Table)

각 단계의 PTE 커버 범위:
L1: 512GB/entry, L2: 1GB/entry, L3: 2MB/entry, L4: 4KB/entry
```

**Core i7 캐시/TLB 계층:**
| 구성요소 | 크기 | 연관도 | 블록 크기 |
|---------|------|--------|---------|
| L1 d-cache | 32KB | 8-way | 64B |
| L1 i-cache | 32KB | 8-way | 64B |
| L2 unified | 256KB | 8-way | 64B |
| L3 unified | 8MB | 16-way | 64B |
| L1 d-TLB | 64 entries | 4-way | |
| L1 i-TLB | 128 entries | 4-way | |
| L2 unified TLB | 512 entries | 4-way | |

**PTE 비트 필드 (Level 4):**
```
63    52  51      12  11  9  8  7  6  5  4  3  2  1  0
┌─────┬────────────┬────┬──┬──┬──┬──┬──┬──┬──┬──┬──┐
│ XD  │  PPN(40bit)│ OS │G │0 │D │A │CD│WT│U/S│R/W│P│
└─────┴────────────┴────┴──┴──┴──┴──┴──┴──┴──┴──┴──┘

P:   현재 물리 메모리에 존재?
R/W: 읽기 전용 or 읽기/쓰기
U/S: 유저 or 커널 모드 접근
A:   참조 비트 (MMU가 읽기/쓰기 시 설정)
D:   더티 비트 (MMU가 쓰기 시 설정)
G:   전역 페이지 (태스크 스위치 시 TLB에서 제거 안 함)
XD:  실행 금지 비트 (NX bit, 버퍼 오버플로우 방어)
```

### Linux 가상 메모리 시스템

**Linux 프로세스 메모리 구조:**
```
┌──────────────────────────────────────┐ 2^48−1
│  Kernel virtual memory               │
│  (프로세스별: 페이지 테이블, 스택,    │
│   task/mm struct)                    │
│  (공통: 커널 코드, 전역 데이터)       │
├──────────────────────────────────────┤
│  User stack (↓성장)                  │ ← %rsp
├──────────────────────────────────────┤
│  Memory-mapped region (shared libs)  │
├──────────────────────────────────────┤ ← brk
│  Run-time heap (malloc, ↑성장)        │
├──────────────────────────────────────┤
│  .bss (uninitialized data)           │
│  .data (initialized data)            │
│  .text, .init (code)                 │
└──────────────────────────────────────┘ 0x400000
```

**Linux 커널 자료구조:**
```
task_struct (프로세스당)
  └─ mm_struct
       ├─ pgd → CR3에 저장되는 L1 페이지 테이블 포인터
       └─ mmap → vm_area_struct 연결 리스트

vm_area_struct (각 가상 메모리 영역):
  ├─ vm_start: 영역 시작 주소
  ├─ vm_end:   영역 끝 주소
  ├─ vm_prot:  읽기/쓰기/실행 권한
  ├─ vm_flags: 프라이빗/공유 여부 등
  └─ vm_next:  다음 영역 포인터
```

**Linux 페이지 폴트 처리 3단계:**
```
1. 주소 A가 vm_area_struct에 속하지 않음
   → 세그멘테이션 폴트(Segmentation fault) → 프로세스 종료

2. 접근 권한 위반 (예: 읽기 전용 페이지에 쓰기)
   → 보호 예외(Protection fault) → 프로세스 종료

3. 합법적인 주소이고 권한도 맞음
   → 희생 페이지 선택 → dirty면 스왑 아웃
   → 새 페이지 스왑 인 → PTE 갱신
   → 폴트 명령어 재실행
```

---

## 9.8 Memory Mapping

Linux는 가상 메모리 영역을 디스크 상의 객체와 연관시켜 초기화한다.

### 두 가지 매핑 유형

**1. 일반 파일 (Regular file):**
- 실행 파일의 `.text`, `.data` 섹션 등
- 파일 섹션을 페이지 크기로 분할 → 각 가상 페이지의 초기 내용
- Demand paging: 처음 참조될 때만 디스크에서 로드

**2. 익명 파일 (Anonymous file):**
- 커널이 생성하는 모든 0으로 채워진 파일 (실제 파일 없음)
- 스택, 힙, BSS 영역에 사용
- 처음 참조 시 "demand-zero page" 생성 (디스크 I/O 없음)

### Shared vs Private 객체 (Copy-on-Write)

**공유 객체 매핑:**
```
프로세스 1      물리 메모리      프로세스 2
가상 공간    ┌──────────────┐    가상 공간
  VP a ─────→│  shared obj  │←───── VP x
  VP b ─────→│              │←───── VP y
             └──────────────┘
→ 단 하나의 물리 복사본을 여러 프로세스가 공유
→ 어느 쪽의 쓰기든 모든 프로세스에 보임 (+ 디스크 반영)
```

**프라이빗 객체 (Copy-on-Write):**
```
초기 상태:
프로세스 1     물리 메모리     프로세스 2
  VP a ──(읽기전용)→ obj ←(읽기전용)── VP x

쓰기 발생 시:
1. 보호 폴트 발생
2. 커널이 해당 페이지 복사 → 새 물리 페이지 생성
3. PTE를 새 페이지로 갱신, 쓰기 권한 부여
4. CPU가 쓰기 명령어 재실행
→ 각 프로세스는 자신만의 독립 복사본 보유
```

### fork()와 VM

`fork()` 호출 시:
1. 자식 프로세스의 `mm_struct`, area structs, 페이지 테이블을 부모에서 **복사**
2. 양쪽 프로세스의 모든 페이지를 **읽기 전용**으로 표시
3. 모든 area struct를 **private copy-on-write**로 표시

→ 실제 쓰기가 발생할 때만 페이지 복사 (지연 복사, 효율적)

### execve()와 VM

`execve("a.out", ...)` 호출 시:
1. 기존 유저 영역 삭제
2. 새 프라이빗 영역 생성:
   - `.text`/`.data` → a.out 파일의 해당 섹션에 매핑 (private file-backed)
   - `.bss` → 익명 파일 (demand-zero)
   - 스택/힙 → 익명 파일 (demand-zero, 초기 크기 0)
3. 공유 영역 매핑: `libc.so` 등 동적 라이브러리 (shared file-backed)
4. PC를 a.out 진입점으로 설정

### mmap / munmap

```c
#include <sys/mman.h>

void *mmap(void *start, size_t length, int prot, int flags, int fd, off_t offset);
// start: 힌트 주소 (보통 NULL)
// length: 매핑 크기
// prot: PROT_READ | PROT_WRITE | PROT_EXEC | PROT_NONE
// flags: MAP_ANON (익명) | MAP_PRIVATE (CoW) | MAP_SHARED (공유)
// fd: 파일 디스크립터 (MAP_ANON이면 0)
// offset: 파일 내 오프셋

int munmap(void *start, size_t length);  // 가상 메모리 영역 삭제

// 예: 읽기 전용 익명 페이지 할당
char *p = mmap(NULL, 1024, PROT_READ, MAP_PRIVATE|MAP_ANON, 0, 0);
```

---

## 9.9 Dynamic Memory Allocation

### 힙과 할당기

```
힙 레이아웃:
┌────────────┐ ← 힙 끝 (brk pointer)
│            │
│  free      │
│            │
├────────────┤
│ allocated  │
├────────────┤
│ allocated  │
├────────────┤
│            │
│  free      │
│            │
└────────────┘ ← 힙 시작 (.bss 끝)
```

**할당기 두 종류:**
- **명시적 할당기(Explicit)**: 애플리케이션이 직접 할당/해제. C의 `malloc/free`, C++의 `new/delete`
- **묵시적 할당기(Implicit) = Garbage Collector**: 할당기가 사용 안 된 블록 자동 감지/해제. Java, Python, Lisp 등

### malloc / free / sbrk

```c
#include <stdlib.h>

void *malloc(size_t size);
// 최소 size bytes의 블록 반환 (8-byte 정렬 보장 in 32-bit, 16-byte in 64-bit)
// 초기화 안 함. 실패 시 NULL + errno 설정

void free(void *ptr);  // malloc/calloc/realloc이 반환한 포인터여야 함

void *calloc(size_t n, size_t size);  // 0으로 초기화된 n*size bytes 블록
void *realloc(void *ptr, size_t size); // 이미 할당된 블록 크기 변경

// 내부 구현에서 사용
void *sbrk(intptr_t incr);  // 힙 크기를 incr bytes 확장/축소
```

### 할당기 설계 목표와 제약

**두 가지 목표 (상충 관계):**
1. **처리량 최대화**: 초당 처리할 수 있는 malloc/free 요청 수
2. **메모리 이용률 최대화**: 최대 활용도 = max(Pᵢ) / Hₖ (Pᵢ: i번째 요청까지의 할당 페이로드 합, Hₖ: 현재 힙 크기)

**제약:**
- 임의 순서의 요청 처리 (요청 순서 예측 불가)
- 즉각 응답 (버퍼링/재정렬 불가)
- 힙에서만 자료구조 저장
- 정렬 요건 준수 (어떤 타입도 저장 가능해야 함)
- 할당된 블록 이동 불가

### 단편화 (Fragmentation)

**내부 단편화 (Internal Fragmentation):**
```
할당된 블록 크기 > 요청된 페이로드

원인: 정렬 요건, 최소 블록 크기
측정: Σ(블록 크기 - 페이로드 크기)
```

**외부 단편화 (External Fragmentation):**
```
여러 작은 빈 블록들의 합이 요청 크기보다 크지만,
단일 연속 빈 블록이 요청을 만족하지 못하는 경우

예: 빈 공간이 16 bytes인데 8bytes + 8bytes로 분리되어 있을 때
    10bytes 요청 → 실패! (힙 확장 필요)
```

### 묵시적 자유 리스트 (Implicit Free List)

```
블록 형식:
┌──────────────────────────┬───┐  ← 헤더 (4 bytes)
│  block size (29 bits)    │ a │  a: allocated bit (0=free, 1=alloc)
├──────────────────────────┴───┤
│         payload               │
│         (malloc 반환 포인터→) │
├───────────────────────────────┤
│      padding (선택)           │
└───────────────────────────────┘

예: 24-byte 할당 블록 헤더: 0x00000018 | 0x1 = 0x00000019
    40-byte 자유 블록 헤더: 0x00000028 | 0x0 = 0x00000028
```

```
힙 전체 레이아웃 (묵시적 자유 리스트):
│패딩│ 8/1 헤더 │ 8/1 푸터 │ ... 블록들 ... │ 0/1 에필로그 헤더 │
  unused  prologue block        regular blocks     epilogue
```

묵시적 자유 리스트: 헤더의 크기 필드를 따라 전체 블록을 순회하여 자유 블록 탐색. 장점: 단순함. 단점: 배치(placement) 시간이 전체 블록 수에 선형.

### 배치 정책 (Placement Policy)

| 정책 | 방법 | 특징 |
|------|------|------|
| **First fit** | 리스트 처음부터 첫 적합 블록 | 뒤쪽에 큰 블록 보존, 앞쪽에 파편 누적 |
| **Next fit** | 이전 탐색 위치부터 시작 | first fit보다 빠르지만 이용률 낮음 |
| **Best fit** | 가장 딱 맞는 블록 선택 | 이용률 최고, 전체 탐색 필요 |

### 분할 (Splitting)

자유 블록이 요청보다 충분히 크면 두 부분으로 분할:
```
before: [  free block 32 bytes  ]
after:  [ alloc 16 ] [ free 16  ]
```

### 합체 (Coalescing) with Boundary Tags

해제된 블록 주변에 인접한 자유 블록과 병합 → 거짓 단편화(false fragmentation) 방지

**경계 태그(Boundary Tag)**: 블록 끝에도 헤더의 복사본(footer) 추가 → 이전 블록을 O(1)에 병합 가능.

```
블록 형식 (경계 태그 포함):
┌────────────────────────┐ ← 헤더: 크기 + 할당 비트
│ payload / 자유 블록 내용 │
└────────────────────────┘ ← 푸터: 헤더의 복사본

4가지 합체 케이스:
Case 1: 이전/다음 모두 할당됨 → 합체 없음, 현재 블록만 자유로 표시
Case 2: 이전 할당, 다음 자유  → 현재+다음 병합 (O(1))
Case 3: 이전 자유, 다음 할당  → 이전+현재 병합 (footer로 이전 블록 위치 파악)
Case 4: 이전/다음 모두 자유   → 3개 모두 병합
```

최적화: 할당된 블록에는 footer 불필요 → 이전 블록의 alloc 비트를 현재 블록 헤더의 여분 비트에 저장.

### 단순 할당기 구현 핵심 코드

```c
/* 기본 상수 및 매크로 */
#define WSIZE     4               // 워드 크기 (bytes)
#define DSIZE     8               // 더블 워드 크기
#define CHUNKSIZE (1<<12)         // 힙 확장 기본 단위 (4096 bytes)

#define PACK(size, alloc)  ((size) | (alloc))      // 헤더/푸터 값 생성
#define GET(p)             (*(unsigned int *)(p))   // 주소 p의 워드 읽기
#define PUT(p, val)        (*(unsigned int *)(p) = (val)) // 쓰기

#define GET_SIZE(p)   (GET(p) & ~0x7)  // 헤더/푸터에서 블록 크기 추출
#define GET_ALLOC(p)  (GET(p) & 0x1)   // 할당 비트 추출

/* 블록 포인터 bp로부터 헤더/푸터/이전/다음 블록 포인터 계산 */
#define HDRP(bp)       ((char *)(bp) - WSIZE)
#define FTRP(bp)       ((char *)(bp) + GET_SIZE(HDRP(bp)) - DSIZE)
#define NEXT_BLKP(bp)  ((char *)(bp) + GET_SIZE(((char *)(bp) - WSIZE)))
#define PREV_BLKP(bp)  ((char *)(bp) - GET_SIZE(((char *)(bp) - DSIZE)))

/* 블록 해제 + 합체 */
void mm_free(void *bp) {
    size_t size = GET_SIZE(HDRP(bp));
    PUT(HDRP(bp), PACK(size, 0));  // 헤더 자유로 표시
    PUT(FTRP(bp), PACK(size, 0));  // 푸터 자유로 표시
    coalesce(bp);                   // 인접 자유 블록 합체
}

/* 할당 */
void *mm_malloc(size_t size) {
    size_t asize;  // 조정된 블록 크기
    if (size <= DSIZE)  asize = 2*DSIZE;          // 최소 블록: 16 bytes
    else  asize = DSIZE * ((size + DSIZE + DSIZE-1) / DSIZE); // 8-byte 정렬

    char *bp = find_fit(asize);      // 자유 리스트 탐색
    if (bp != NULL) { place(bp, asize); return bp; }

    // 적합한 블록 없음 → 힙 확장
    size_t extendsize = MAX(asize, CHUNKSIZE);
    bp = extend_heap(extendsize/WSIZE);
    place(bp, asize);
    return bp;
}
```

### 고급 자유 리스트 구조

**명시적 자유 리스트 (Explicit Free List):**
- 자유 블록 내부에 predecessor/successor 포인터 저장 → 이중 연결 리스트
- first-fit 시간: O(자유 블록 수) (묵시적 O(전체 블록 수)보다 빠름)
- 최소 블록 크기 증가 (포인터 공간 필요)

**분리 자유 리스트 (Segregated Free List):**
- 크기 클래스별 별도 자유 리스트 유지
- 예: {1-8}, {9-16}, {17-32}, ..., {4097-∞}
- 할당 시 해당 크기 클래스 리스트 우선 탐색 → 없으면 다음 클래스
- GNU malloc (glibc), 현대 표준 할당기가 채택
- best-fit에 근접한 이용률 + 빠른 탐색 속도

---

## 9.10 Garbage Collection (가비지 컬렉션)

묵시적 할당기. 프로그램이 명시적으로 `free`를 호출하지 않아도 사용하지 않는 메모리를 자동 회수.

**Mark & Sweep 알고리즘:**
```
Mark 단계: 루트 집합(스택, 전역 변수)에서 도달 가능한 모든 블록 표시
Sweep 단계: 표시되지 않은 블록 모두 해제

루트 집합 → [블록 A] → [블록 C]
             [블록 B] → [블록 D]  ← D는 루트에서 도달 불가 → 해제
```

C 프로그램에서의 보수적 GC: 포인터 값처럼 보이는 값이 있으면 해당 블록을 살아있는 것으로 보존. 포인터 계산(pointer arithmetic) 때문에 오인 가능성.

---

## 9.11 C 프로그램의 흔한 메모리 버그

### 1. 잘못된 포인터 역참조

```c
int val;
scanf("%d", val);   // 버그: val이 아닌 &val이어야 함
                    // 임의 주소에 쓰기 → 세그폴트 또는 데이터 오염
```

### 2. 초기화되지 않은 메모리 읽기

```c
int *p = malloc(n * sizeof(int));
for (int i=0; i<n; i++) sum += p[i]; // malloc은 초기화 안 함!
// 해결: calloc 사용 또는 memset으로 초기화
```

### 3. 스택 버퍼 오버플로우

```c
void func(char *s) {
    char buf[64];
    strcpy(buf, s);  // s가 64 bytes 초과 시 → 스택 손상
    // 해결: strncpy(buf, s, sizeof(buf)-1)
}
```

### 4. 포인터와 가리키는 객체 크기 혼동

```c
int **p = malloc(n * sizeof(int));   // 버그: sizeof(int*) 여야 함
int **p = malloc(n * sizeof(int *)); // 정상
```

### 5. Off-by-One 오류

```c
char s[n];
for (int i=0; i<=n; i++) s[i] = 0;  // s[n] 초과 접근!
// 해결: i < n
```

### 6. 포인터 대신 포인터가 가리키는 객체를 참조

```c
int *GetArr(int **listp, int *sizep, int n) {
    *listp = realloc(*listp, n * sizeof(int));  // 정상
    listp = realloc(listp, n * sizeof(int));    // 버그: listp는 로컬 포인터
}
```

### 7. 포인터 산술 오해

```c
int *p = malloc(10 * sizeof(int));
p += sizeof(int);   // 버그: 4 × sizeof(int) = 16 bytes 이동
p += 1;             // 정상: sizeof(int) = 4 bytes 이동
```

### 8. 존재하지 않는 변수 참조

```c
int *stackref() {
    int val = 100;
    return &val;     // 버그: 함수 반환 후 val은 소멸
}
```

### 9. 해제된 힙 블록 참조 (Use-after-free)

```c
int *p = malloc(n * sizeof(int));
free(p);
/* ... 나중에 ... */
for (int i=0; i<n; i++) p[i] = 0;  // 버그: 이미 해제된 메모리
```

### 10. 메모리 누수 (Memory Leak)

```c
void func() {
    int *p = malloc(100);
    // ... free(p) 없이 반환 → 메모리 누수
    // 장기 실행 프로그램에서 치명적
}
```

---

## 핵심 공식/구조 정리

```
주소 변환:
  가상 주소 n bits: [VPN(n-p bits) | VPO(p bits)]
  물리 주소 m bits: [PPN(m-p bits) | PPO(p bits)]  (PPO = VPO)
  PTE 개수 = 2^(n-p)

TLB 주소:
  TLBT = VPN[상위 (n-p-t) bits]
  TLBI = VPN[하위 t bits]  (T = 2^t sets)

Core i7:
  4단계 페이지 테이블, VPN = 4 × 9 bits, VPO = 12 bits
  CR3 → L1 PGD → L2 PUD → L3 PMD → L4 PTE → PPN

malloc 블록 크기 조정 (double-word 정렬):
  size ≤ 8:  asize = 16
  size > 8:  asize = 8 × ceil((size + 8) / 8)

합체 케이스:
  Case 1: prev=alloc, next=alloc → 변경 없음
  Case 2: prev=alloc, next=free  → 현재+다음 병합
  Case 3: prev=free,  next=alloc → 이전+현재 병합
  Case 4: prev=free,  next=free  → 3개 전체 병합
```

---

## 요약

| 주제 | 핵심 |
|------|------|
| VM 역할 3가지 | 캐시(DRAM)/메모리 관리/보호 |
| DRAM 캐시 | 완전 연관, write-back, 큰 페이지(4KB), demand paging |
| 페이지 테이블 | PTE: valid bit + PPN/disk addr; 페이지 히트/폴트 |
| TLB | MMU 내 PTE 캐시, TLBI/TLBT로 접근 |
| 다단계 페이지 테이블 | 빈 가상 영역 → L1 PTE null → L2 테이블 불필요 |
| Core i7 | 4단계, 48-bit VA, 각 9-bit VPN, CR3, XD/A/D 비트 |
| Memory Mapping | 파일 백드/익명, CoW, fork/execve 구현 기반, mmap |
| malloc | 명시적 할당기, 묵시적 자유 리스트, 경계 태그 합체 |
| 배치 정책 | first/next/best fit 트레이드오프 |
| 분리 자유 리스트 | glibc 채택, best-fit 근사 + 빠른 탐색 |
| 메모리 버그 | 초기화 안 됨, off-by-one, use-after-free, 메모리 누수 |
