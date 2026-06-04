# Chapter 6: The Memory Hierarchy

## 핵심 주제

메모리 시스템은 단일 수치로 특성화되지 않는다. 프로그램이 접근하는 데이터의 **지역성(locality)** 에 따라 수십 배 이상의 성능 차이가 발생한다. 스토리지 기술의 특성을 이해하고, 캐시 메모리의 동작 원리를 파악하며, 이를 캐시 친화적(cache-friendly) 코드 작성에 활용하는 것이 이 챕터의 목표다.

---

## 6.1 Storage Technologies

### 6.1.1 RAM

**SRAM (Static RAM)**

| 특성 | 값 |
|------|----|
| 셀 구조 | 트랜지스터 6개 (bistable 래치) |
| 리프레시 필요 여부 | 불필요 |
| 접근 시간 | ~1ns |
| 가격 | ~$500–1000/GB |
| 용도 | L1/L2/L3 캐시 |

비스태이블(bistable) 특성 덕분에 전원이 유지되는 한 상태를 유지한다. 트랜지스터 6개로 이루어진 셀은 두 개의 안정 상태(0, 1)를 가지며, 중간 상태는 불안정하여 진동하다가 어느 한쪽으로 수렴한다.

**DRAM (Dynamic RAM)**

| 특성 | 값 |
|------|----|
| 셀 구조 | 트랜지스터 1개 + 커패시터 1개 |
| 리프레시 필요 여부 | ~64ms 주기로 필요 |
| 접근 시간 | ~10ns |
| 가격 | ~$10/GB |
| 용도 | 메인 메모리 |

커패시터의 전하 누설로 인해 주기적으로 리프레시(refresh)해야 한다. 메모리 컨트롤러가 모든 행(row)을 순차적으로 읽고 다시 써서 전하를 복원한다.

**DRAM 내부 구조:**
```
DRAM 칩
 ┌─────────────────────────────────────────┐
 │  d × w DRAM                             │
 │  ┌───────────────────────────────────┐  │
 │  │  r rows × c cols supercell array  │  │
 │  │  (각 supercell = w bits)          │  │
 │  └───────────────────────────────────┘  │
 │          addr ─→  row 핀               │
 │          addr ─→  col 핀               │
 │          data  ←→  data 핀 (w bits)    │
 └─────────────────────────────────────────┘
```

**Enhanced DRAM 종류:**
- **FPM DRAM (Fast Page Mode)**: 동일한 행 내 연속 접근 시 열 주소만 변경
- **EDO DRAM (Extended Data Out)**: FPM 확장, 열 주소 스트로브 간격 단축
- **SDRAM (Synchronous)**: CPU 클록과 동기화
- **DDR SDRAM (Double Data Rate)**: 클록 상승 및 하강 엣지 모두 사용
  - DDR3: 최대 ~1600 MT/s, DDR4: 최대 ~3200 MT/s, DDR5: 최대 ~6400 MT/s
- **LPDDR (Low Power DDR)**: 모바일 기기용

**비휘발성 메모리 (Nonvolatile Memory):**

| 종류 | 재프로그래밍 | 용도 |
|------|------------|------|
| ROM (Read-Only Memory) | 불가 | firmware |
| PROM | 1회 가능 (fusible link) | 일회성 설정 |
| EPROM | UV 광선으로 삭제 | 개발/테스트 |
| EEPROM | 전기적 삭제 (byte 단위) | BIOS 설정 |
| Flash memory | 블록 단위 삭제 | SSD, 임베디드 |

### 6.1.2 Disk Storage

**디스크 물리 구조:**
```
   ┌────────────────────────────┐
   │      Disk Drive            │
   │   ┌──────────────────┐     │
   │   │   Platter (원판) │     │
   │   │  ┌────────────┐  │     │
   │   │  │  Surface   │  │     │
   │   │  │  ┌──────┐  │  │     │
   │   │  │  │Track │  │  │     │
   │   │  │  │┌──┐  │  │  │     │
   │   │  │  ││Se│  │  │  │     │   Sector: 최소 주소 지정 단위
   │   │  │  │└──┘  │  │  │     │   (전통적으로 512 bytes,
   │   │  │  └──────┘  │  │     │    최근 4096 bytes)
   │   │  └────────────┘  │     │
   │   └──────────────────┘     │
   │   ≡≡≡ spindle ≡≡≡         │
   └────────────────────────────┘

계층: Disk → Platter → Surface → Track → Sector
동일한 반지름의 트랙들 = Cylinder
```

**디스크 용량 계산:**
```
용량 = bytes/sector × sectors/track × tracks/surface
      × surfaces/platter × platters/disk
```

**디스크 접근 시간:**
```
T_access = T_seek + T_rotation + T_transfer

T_seek (탐색 시간):
  - 헤드를 해당 실린더로 이동
  - 평균 ~9ms (일반적인 7200 RPM 드라이브)

T_rotation (회전 지연):
  - 헤드가 해당 섹터 위에 올 때까지 대기
  - 평균 = (1/2) × (1/RPM) × (60 sec/min)
  - 7200 RPM → T_avg_rotation ≈ 4ms

T_transfer (전송 시간):
  - 섹터 내 데이터를 읽는 시간
  - T_avg_transfer = 1/RPM × 1/sectors_per_track × 60
  - 일반적으로 ~0.02ms
```

**접근 시간 비교 (7200 RPM 기준):**
| 장치 | 접근 시간 |
|------|---------|
| SRAM | ~1ns |
| DRAM | ~10ns |
| SSD | ~50μs |
| 회전 디스크 | ~10ms (seek + rotational) |

→ 디스크는 SRAM 대비 **40,000배 이상** 느리다.

**논리 디스크 블록 (Logical Disk Block):**
- OS는 물리 구조를 추상화하여 연속된 논리 블록 배열로 노출
- 디스크 컨트롤러가 논리 블록 ↔ 물리 섹터 매핑 담당
- CHS (Cylinder-Head-Sector) 주소 지정은 LBA (Logical Block Address)로 대체됨

**디스크 I/O 흐름:**
```
CPU    ──→  I/O 버스  ──→  디스크 컨트롤러  ──→  디스크
           (PCI/PCIe)

1. CPU가 디스크 컨트롤러에 명령 레지스터에 쓰기
2. 컨트롤러가 논리 블록 주소를 물리 주소로 변환 후 데이터 읽기
3. DMA(Direct Memory Access): 버스를 통해 주메모리로 직접 전송
4. 전송 완료 시 컨트롤러가 CPU에 인터럽트 발생
→ CPU는 I/O 대기 중 다른 작업 가능
```

### 6.1.3 SSD (Solid State Disk)

```
SSD 내부 구조
 ┌──────────────────────────────────┐
 │  SSD                            │
 │  ┌──────────────────────────┐   │
 │  │  Flash 메모리 칩들       │   │
 │  │  ┌───────────────────┐   │   │
 │  │  │  Block (수백 KB)  │   │   │ ← 삭제 단위
 │  │  │  ┌─────────────┐  │   │   │
 │  │  │  │ Page (4KB)  │  │   │   │ ← 읽기/쓰기 단위
 │  │  │  └─────────────┘  │   │   │
 │  │  └───────────────────┘   │   │
 │  └──────────────────────────┘   │
 │  Flash Translation Layer (FTL)  │ ← 주소 변환 계층
 └──────────────────────────────────┘
```

| 연산 | 시간 | 특이사항 |
|------|------|---------|
| 페이지 읽기 | ~50μs | 랜덤 접근 |
| 페이지 쓰기 | ~250μs | 먼저 블록 전체 삭제 필요 |
| 블록 삭제 | ~2ms | ~100,000회 이후 마모 |
| 순차 읽기 | ~550 MB/s | |
| 순차 쓰기 | ~470 MB/s | |

**SSD 쓰기 제약:**
- 쓰기 전에 반드시 블록(수백 KB) 전체를 먼저 삭제해야 한다
- FTL이 wear leveling (마모 평준화)를 관리
- Write amplification: 작은 쓰기도 전체 블록 재작성을 유발할 수 있음

**HDD vs SSD:**
| 특성 | HDD | SSD |
|------|-----|-----|
| 랜덤 접근 | ~10ms | ~50μs (~200배 빠름) |
| 순차 처리량 | ~120 MB/s | ~550 MB/s |
| 용량 대비 가격 | 저렴 | 상대적으로 비쌈 |
| 내구성 | 기계적 마모 | 전기적 마모 (쓰기 한계) |

---

## 6.2 Locality

좋은 지역성(locality)을 가진 프로그램은 캐시를 효과적으로 활용한다.

### 시간 지역성 (Temporal Locality)

최근에 참조한 메모리 위치를 가까운 미래에 다시 참조할 가능성이 높다.

```c
int sum = 0;
for (int i = 0; i < n; i++)
    sum += a[i];   // sum은 매 반복마다 참조 → 높은 시간 지역성
```
`sum`은 매 반복마다 읽고 쓰이므로 컴파일러가 레지스터에 캐시한다.

### 공간 지역성 (Spatial Locality)

참조된 메모리 위치에서 가까운 위치를 가까운 미래에 참조할 가능성이 높다.

```c
// 좋은 공간 지역성: stride-1
int sumvec(int v[N]) {
    int sum = 0;
    for (int i = 0; i < N; i++)
        sum += v[i];   // v[i] → v[i+1] → v[i+2] ... (stride-1)
    return sum;
}
```

**stride-k 패턴의 미스율:**
```
miss/iteration = min(1, wordsize × k / B)

k=1 (stride-1): min(1, 4/16) = 0.25   → 75% 히트율 (B=16 bytes 가정)
k=2 (stride-2): min(1, 4×2/16) = 0.5  → 50% 히트율
k=4 (stride-4): min(1, 4×4/16) = 1.0  → 100% 미스율
```

**2D 배열 순회 비교:**
```c
// 좋은 지역성: row-major (C 배열 저장 순서와 일치)
int sumarrayrows(int a[M][N]) {
    int sum = 0;
    for (int i = 0; i < M; i++)
        for (int j = 0; j < N; j++)
            sum += a[i][j];   // stride-1
    return sum;
}

// 나쁜 지역성: column-major
int sumarraycols(int a[M][N]) {
    int sum = 0;
    for (int j = 0; j < N; j++)
        for (int i = 0; i < M; i++)
            sum += a[i][j];   // stride-N (캐시보다 큰 배열 → 100% 미스)
    return sum;
}
```

`sumarrayrows`는 `sumarraycols`보다 **최대 25배 빠를 수 있다**.

---

## 6.3 Memory Hierarchy

### 메모리 계층 구조

```
빠름/비쌈/소용량                          느림/저렴/대용량
─────────────────────────────────────────────────────────
  L0: CPU 레지스터 (수십 바이트, ~0 cycles)
  L1: L1 캐시 SRAM (32 KB, ~4 cycles)
  L2: L2 캐시 SRAM (256 KB, ~10 cycles)
  L3: L3 캐시 SRAM (수 MB, ~40-75 cycles)
  L4: 메인 메모리 DRAM (수 GB, ~200 cycles)
  L5: 로컬 디스크 (수 TB, ~10ms)
  L6: 원격 스토리지 (수 PB, ~수십 ms)
─────────────────────────────────────────────────────────
```

**캐시의 원리:**
- 상위 레벨 k가 하위 레벨 k+1의 일부를 저장하는 캐시 역할
- 계층 어디에나 존재: TLB, L1/L2/L3, VM, 버퍼 캐시, 네트워크 캐시 등

**캐시 히트/미스 종류:**

| 미스 종류 | 원인 | 해결책 |
|---------|------|--------|
| Cold miss (Compulsory miss) | 처음 접근 → 캐시 비어있음 | 불가피 |
| Conflict miss | 서로 다른 블록이 같은 set에 매핑 | 연관도 증가, 패딩 |
| Capacity miss | 작업 집합(working set)이 캐시보다 큼 | 알고리즘 개선 |

---

## 6.4 Cache Memories

### 6.4.1 캐시 메모리 구조

캐시는 **(S, E, B, m)** 4-tuple로 정의된다:

| 파라미터 | 의미 | 도출 |
|---------|------|------|
| S = 2^s | 세트(set) 수 | s = log₂(S) |
| E | 세트당 라인(line) 수 | (associativity) |
| B = 2^b | 블록(block) 크기 (bytes) | b = log₂(B) |
| m | 물리 주소 비트 수 | M = 2^m |
| C = S×E×B | 캐시 용량 | |
| t = m − (s+b) | 태그(tag) 비트 수 | |

**주소 분할:**
```
m 비트 주소
┌──────────────┬─────────────┬──────────────┐
│  t 태그 비트  │  s 세트 인덱스 │  b 블록 오프셋 │
│   (상위)     │   (중간)     │    (하위)     │
└──────────────┴─────────────┴──────────────┘
 m-1           s+b           b              0
```

**중간 비트 인덱싱(middle-bit indexing):**  
세트 인덱스에 중간 비트를 사용하면 인접한 메모리 블록이 서로 다른 세트에 분산된다. 상위 비트를 사용하면 연속된 C/B 개 블록이 동일한 세트로 몰려 공간 지역성 활용이 불가능해진다.

**캐시 검색 과정 (3단계):**
```
1. Set Selection  : 세트 인덱스 비트로 세트 선택
2. Line Matching  : 세트 내 라인들을 탐색하여 (valid=1 AND tag 일치) 확인
3. Word Extraction: 블록 오프셋으로 해당 바이트 추출
```

### 6.4.2 Direct-Mapped Cache (E = 1)

각 세트에 라인이 정확히 하나:
```
┌───────┬───────┬──────────────────────┐
│ Valid │  Tag  │   Data Block (B bytes) │
└───────┴───────┴──────────────────────┘
  1 bit  t bits       B bytes
```

**예시: (S,E,B,m) = (4,1,2,4)**
```
주소 4비트: [태그 1비트][세트인덱스 2비트][블록오프셋 1비트]

Addr 0 = 0|00|0 → 세트 0, 블록 0, 오프셋 0
Addr 8 = 1|00|0 → 세트 0, 블록 4, 오프셋 0
→ 주소 0과 주소 8은 동일 세트에 매핑 (conflict 발생 가능)
```

**Conflict Miss (Thrashing):**
```c
float x[8], y[8];  // 각각 32 bytes
// x[0]–x[7]: 주소 0–28
// y[0]–y[7]: 주소 32–60
// 캐시가 2세트 × 16byte 블록 = 32 bytes일 때:
// x[i]와 y[i]가 동일 세트에 매핑 → 모든 접근이 미스!

// 해결책: 패딩
float x[12];  // x[8]–x[11]: dummy padding
// 이제 y[0]–y[3]이 세트 1로 이동 → conflict 해소
```

### 6.4.3 Set-Associative Cache (1 < E < C/B)

**2-way set-associative 예시:**
```
Set i:
┌───────┬──────┬────────────────┐   ← Line 0
│ Valid │ Tag  │  Data Block    │
├───────┼──────┼────────────────┤   ← Line 1
│ Valid │ Tag  │  Data Block    │
└───────┴──────┴────────────────┘
```

라인 교체 정책 (Replacement Policy):
- **LRU (Least Recently Used)**: 가장 오래 전에 접근된 라인 교체
- **LFU (Least Frequently Used)**: 접근 빈도가 가장 낮은 라인 교체
- **Random**: 무작위 교체 (하드웨어 오버헤드 없음)

계층이 낮아질수록(CPU에서 멀어질수록) 미스 패널티가 크므로 정교한 교체 정책 투자가 정당화된다.

### 6.4.4 Fully Associative Cache (E = C/B)

단일 세트에 모든 라인 포함:
```
주소 분할: [tag t bits][block offset b bits]
           (세트 인덱스 비트 없음)
```
모든 라인을 병렬로 태그 비교해야 하므로 하드웨어 비용이 크다. TLB(Translation Lookaside Buffer)와 같은 소용량 캐시에만 적합하다.

### 6.4.5 쓰기 정책 (Write Policies)

**Write Hit 처리:**

| 정책 | 동작 | 장점 | 단점 |
|------|------|------|------|
| Write-through | 즉시 하위 레벨에 쓰기 | 구현 단순, dirty bit 불필요 | 모든 쓰기마다 버스 트래픽 |
| Write-back | 라인 evict 시에만 하위에 쓰기 | 버스 트래픽 감소, DMA 대역폭 확보 | dirty bit 필요, 복잡도 증가 |

**Write Miss 처리:**

| 정책 | 동작 | 주로 사용 |
|------|------|---------|
| Write-allocate | 블록을 캐시에 로드 후 업데이트 | Write-back 캐시와 조합 |
| No-write-allocate | 캐시 우회, 직접 하위 레벨에 쓰기 | Write-through 캐시와 조합 |

**실용적 관점:** 대부분의 현대 캐시는 write-back + write-allocate 조합을 사용한다. 읽기와 동일하게 지역성을 활용한다.

### 6.4.6 Intel Core i7 캐시 계층

```
┌────────────────────────────────────┐
│          Processor Package         │
│  ┌──────────────┬──────────────┐   │
│  │   Core 0     │    Core 3    │   │
│  │  Regs        │   Regs       │   │
│  │  L1 d-cache  │   L1 d-cache │   │  32KB, 8-way, 4 cycles
│  │  L1 i-cache  │   L1 i-cache │   │  32KB, 8-way, 4 cycles
│  │  L2 unified  │   L2 unified │   │  256KB, 8-way, 10 cycles
│  └──────────────┴──────────────┘   │
│         L3 unified (shared)        │  8MB, 16-way, 40-75 cycles
└────────────────────────────────────┘
              │
         Main Memory (DRAM)           ~200 cycles
```

| 캐시 | 접근 시간 | 크기 | 연관도 | 블록 크기 | 세트 수 |
|------|---------|------|------|---------|------|
| L1 i-cache | 4 cycles | 32 KB | 8-way | 64 B | 64 |
| L1 d-cache | 4 cycles | 32 KB | 8-way | 64 B | 64 |
| L2 unified | 10 cycles | 256 KB | 8-way | 64 B | 512 |
| L3 unified | 40–75 cycles | 8 MB | 16-way | 64 B | 8,192 |

**I-cache vs D-cache 분리의 이유:**
- 명령어와 데이터를 동시에 읽기 가능 (처리량 증가)
- I-cache는 읽기 전용 → 구현 단순
- 각기 다른 접근 패턴에 최적화 가능
- 명령어 미스가 데이터 캐시에 간섭하지 않음

### 6.4.7 캐시 파라미터 영향

| 파라미터 | 크면 | 작으면 |
|---------|------|--------|
| 캐시 크기 C | 히트율 증가, 히트 시간 증가 | 히트율 감소, 히트 시간 감소 |
| 블록 크기 B | 공간 지역성 활용 증가, 미스 패널티 증가 | 시간 지역성 활용 증가 |
| 연관도 E | 충돌 미스 감소, 히트 시간/미스 패널티 증가 | 충돌 미스 증가, 하드웨어 단순 |

**연관도 설계 선택:**
- L1 (미스 패널티 낮음): 8-way → 히트 시간 최소화 우선
- L3 (미스 패널티 높음): 16-way → 미스율 최소화 우선

---

## 6.5 Writing Cache-Friendly Code

캐시 친화적 코드의 두 가지 원칙:

1. **공통 경로를 빠르게 (Make the common case go fast):** 코어 함수의 내부 루프에 집중
2. **내부 루프의 미스 최소화:** 메모리 접근 수가 같을 때 미스율이 낮은 루프가 빠름

### stride-1 참조 패턴

```c
// sumvec: stride-1, 블록 4 words(16 bytes) 가정
// 미스 패턴: v[0]m, v[1]h, v[2]h, v[3]h, v[4]m, ...
// 미스율 = 1/4 = 25%
int sumvec(int v[N]) {
    int sum = 0;
    for (int i = 0; i < N; i++)
        sum += v[i];  // stride-1: 4개 중 3개 히트
    return sum;
}
```

### 2D 배열과 행 우선 순서

```c
// 좋음: row-major traversal (stride-1)
// 미스율 = 1/4 (블록 4개 int 가정)
int sumarrayrows(int a[M][N]) {
    int sum = 0;
    for (int i = 0; i < M; i++)
        for (int j = 0; j < N; j++)  // ← inner loop: j 변화 = stride-1
            sum += a[i][j];
    return sum;
}

// 나쁨: column-major traversal (stride-N)
// 배열이 캐시보다 크면 미스율 ≈ 100%
// sumarrayrows보다 최대 25배 느림
int sumarraycols(int a[M][N]) {
    int sum = 0;
    for (int j = 0; j < N; j++)
        for (int i = 0; i < M; i++)  // ← inner loop: i 변화 = stride-N
            sum += a[i][j];
    return sum;
}
```

**미스 패턴 비교 (4×8 배열, 블록 4 words):**
```
sumarrayrows (row-major):
a[i][j]  j=0  j=1  j=2  j=3  j=4  j=5  j=6  j=7
i=0      1[m] 2[h] 3[h] 4[h] 5[m] 6[h] 7[h] 8[h]
i=1      9[m] ...                              → 미스율 1/4

sumarraycols (col-major, 배열 >> 캐시):
a[i][j]  j=0   j=1   j=2  ...
i=0      1[m]  5[m]  9[m] ...
i=1      2[m]  6[m]  10[m]...   → 미스율 1 (모두 미스)
```

### 핵심 원칙 요약

| 원칙 | 설명 |
|------|------|
| 지역 변수 반복 참조 | 컴파일러가 레지스터에 캐시 → 최상위 계층 활용 |
| stride-1 참조 | 캐시 블록 내 모든 데이터 활용 |
| 다차원 배열 행 우선 | C의 row-major 저장 순서와 일치 |

---

## 6.6 Putting It Together: Impact of Caches on Program Performance

### 6.6.1 메모리 마운틴 (The Memory Mountain)

**읽기 처리량(read throughput)** 을 시간 지역성(working set size)과 공간 지역성(stride) 두 축으로 측정한 함수.

```
                     메모리 마운틴 (Core i7 Haswell)
Read                     │
Throughput (MB/s)        │  L1 ridge
  14,000 ─────────────── ┤  ~~~~~~~~~~~~~~
  12,000                 │      L2 ridge
  10,000                 │          ~~~~~~~~~~
   8,000                 │               L3 ridge
   6,000                 │                   ~~~~~~~~~~
   4,000                 │                        Mem ridge
   2,000                 │                             ~~~~~~~~~~
       0 ────────────────┴──────────────────────────────────────
                         stride=1 ──────────────── stride=12
                         (공간 지역성)                (↓낮음)
       size: 16KB (좌) ─────────────────── 128MB (우)
             (시간 지역성 높음)              (낮음)
```

**주요 관측:**
- L1 피크: ~14 GB/s, 메인 메모리 저점: ~900 MB/s → **15배 이상 차이**
- stride-1 수평 능선: ~12 GB/s (L2, L3 초과해도 유지) → **하드웨어 프리페처(prefetcher)** 덕분
- 동일 메모리 레벨에서도 stride 증가 시 처리량 감소 (공간 지역성 효과)

```c
// 메모리 마운틴 측정 함수 (4×4 loop unrolling)
int test(int elems, int stride) {
    long i, sx2 = stride*2, sx3 = stride*3, sx4 = stride*4;
    long acc0=0, acc1=0, acc2=0, acc3=0;
    long limit = elems - sx4;

    for (i = 0; i < limit; i += sx4) {
        acc0 = acc0 + data[i];
        acc1 = acc1 + data[i+stride];
        acc2 = acc2 + data[i+sx2];
        acc3 = acc3 + data[i+sx3];
    }
    return ((acc0 + acc1) + (acc2 + acc3));
}
```

### 6.6.2 루프 재배열로 공간 지역성 향상 (Matrix Multiply)

n×n 행렬 곱 C = A×B의 6가지 루프 순서 버전:

```c
// (a) ijk 버전
for (i = 0; i < n; i++)
    for (j = 0; j < n; j++) {
        sum = 0.0;
        for (k = 0; k < n; k++)
            sum += A[i][k] * B[k][j];  // A: stride-1, B: stride-n
        C[i][j] += sum;
    }

// (b) jik 버전 - ijk와 동일한 미스 패턴

// (c) jki 버전 (AC class - 최악)
for (j = 0; j < n; j++)
    for (k = 0; k < n; k++) {
        r = B[k][j];
        for (i = 0; i < n; i++)
            C[i][j] += A[i][k] * r;  // A: stride-n, C: stride-n
    }

// (e) kij 버전 (BC class - 최선)
for (k = 0; k < n; k++)
    for (i = 0; i < n; i++) {
        r = A[i][k];
        for (j = 0; j < n; j++)
            C[i][j] += r * B[k][j];  // B: stride-1, C: stride-1
    }
```

**내부 루프 미스율 분석 (블록 크기 32 bytes = 4 doubles):**

| 버전 (class) | 로드 | 스토어 | A 미스 | B 미스 | C 미스 | **총 미스/반복** |
|------------|------|------|------|------|------|------------|
| ijk & jik (AB) | 2 | 0 | 0.25 | 1.00 | 0.00 | **1.25** |
| jki & kji (AC) | 2 | 1 | 1.00 | 0.00 | 1.00 | **2.00** |
| kij & ikj (BC) | 2 | 1 | 0.00 | 0.25 | 0.25 | **0.50** |

```
미스 분석 설명:
- AB class: A[i][k] stride-1 (0.25 미스), B[k][j] stride-n (1.00 미스)
- AC class: A[i][k] stride-n, C[i][j] stride-n → 각각 1.00 미스
- BC class: B[k][j] stride-1, C[i][j] stride-1 → 각각 0.25 미스
```

**실측 성능 (Core i7):**
```
CPE (Cycles Per inner-loop Iteration)
  100 ┤ jki ●──────────────●
      │ kji ●
      │
   10 ┤ ijk ●──────────────●
      │ jik ●
      │ kij ●──────────────●  ← 최선 (하드웨어 프리페처 활용)
      │ ikj ●
    1 ┤
      └────────────────────────
          50      300      600
               n (배열 크기)

→ 최악(jki/kji) 대비 최선(kij/ikj)이 최대 40배 빠름
```

**핵심 인사이트:**
- 메모리 참조 횟수보다 **미스율**이 더 중요한 성능 예측 지표
- BC class는 AB class보다 메모리 연산이 많지만(2 load + 1 store vs 2 load) 미스율이 낮아 더 빠름
- kij/ikj는 n이 커져도 성능이 일정 → 하드웨어 프리페처가 stride-1 패턴 감지

### 블로킹 (Blocking) 기법

시간 지역성을 극대화하기 위한 루프 변환 기법. 큰 행렬을 캐시에 들어가는 블록 단위로 나눠 처리한다.

```c
// 일반 행렬 곱 → 각 행/열을 n번 읽음
// 블로킹 적용 행렬 곱 → 각 블록을 한 번 읽고 모든 연산 완료
#define B_SIZE 32  // 캐시 크기에 맞춘 블록 크기

for (int ii = 0; ii < n; ii += B_SIZE)
    for (int jj = 0; jj < n; jj += B_SIZE)
        for (int kk = 0; kk < n; kk += B_SIZE)
            // B_SIZE × B_SIZE 서브행렬에 대해 내부 3중 루프 수행
            for (int i = ii; i < min(ii+B_SIZE, n); i++)
                for (int j = jj; j < min(jj+B_SIZE, n); j++) {
                    sum = 0.0;
                    for (int k = kk; k < min(kk+B_SIZE, n); k++)
                        sum += A[i][k] * B[k][j];
                    C[i][j] += sum;
                }
```

> 참고: Core i7은 정교한 프리페처 덕분에 블로킹이 행렬 곱 성능을 개선하지 않지만, 프리페처가 없는 시스템에서는 큰 성능 향상을 가져온다.

### 6.6.3 프로그램 최적화 권고

```
1. 내부 루프에 집중
   - 대부분의 연산과 메모리 접근이 내부 루프에서 발생
   - 외부 루프는 성능에 미치는 영향이 적음

2. 공간 지역성 극대화
   - stride-1 순서로 데이터 읽기
   - 메모리에 저장된 순서대로 접근 (C: row-major)

3. 시간 지역성 극대화
   - 한 번 메모리에서 읽은 데이터를 최대한 많이 활용
   - 작업 집합(working set)이 캐시에 들어오도록 알고리즘 설계
```

---

## 핵심 공식 정리

```
캐시 파라미터:
  C = S × E × B           캐시 용량
  t = m − (s + b)         태그 비트 수

stride-k 미스율:
  misses/iter = min(1, wordsize × k / B)

디스크 접근 시간:
  T_access = T_seek + T_rotation + T_transfer
  T_avg_rotation = (1/2) × (1/RPM) × 60 (sec)

속도 비교:
  CPU 레지스터 < L1(4cyc) < L2(10cyc) < L3(50cyc)
  < 메인 메모리(200cyc) << SSD(50μs) <<< HDD(10ms)
```

---

## 요약

| 주제 | 핵심 내용 |
|------|---------|
| SRAM vs DRAM | SRAM: 빠름/비쌈/캐시, DRAM: 느림/저렴/메인 메모리 |
| 디스크 접근 | Seek + Rotation + Transfer ≈ 10ms, SSD는 200배 빠름 |
| 지역성 | 시간(temporal): 반복 참조, 공간(spatial): stride-1 |
| 캐시 구조 | (S,E,B,m): 세트/연관도/블록/주소 크기 |
| 미스 종류 | Cold, Conflict (thrashing), Capacity |
| 쓰기 정책 | Write-back+Write-allocate = 현대 표준 |
| 캐시 친화적 코드 | Row-major 순회, stride-1, 지역 변수 활용 |
| 행렬 곱 최적화 | BC class (kij/ikj) = 0.5 misses/iter, 최대 40배 빠름 |
