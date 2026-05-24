# Chapter 11: Mass-Storage Structure

## 11.1 대용량 저장장치 개요 (Overview of Mass-Storage Structure)

### 11.1.1 하드 디스크 드라이브 (Hard Disk Drives)

```
                    spindle
                      │
         ┌────────────┼────────────┐
         │  ●●● track │ ●●●       │ ← platter (자기 물질 도포)
         │   sector ──┘           │
         │                        │
arm ─────┼──[read-write head]     │ cylinder = 같은 암 위치의 모든 트랙
         │                        │
         └────────────────────────┘
                   rotation →
```

**핵심 성능 지표:**

| 지표 | 설명 | 일반 수치 |
|------|------|----------|
| 회전 속도 (RPM) | 디스크 회전 속도 | 5,400 / 7,200 / 10,000 / 15,000 RPM |
| 탐색 시간 (seek time) | 암이 원하는 실린더로 이동하는 시간 | 수 ms |
| 회전 지연 (rotational latency) | 원하는 섹터가 헤드 아래로 오는 시간 | 수 ms |
| 전송 속도 (transfer rate) | 헤드↔컨트롤러 데이터 전송 속도 | 수십~수백 MB/s |

- **위치 지정 시간(positioning time)** = seek time + rotational latency = **random-access time**
- 섹터 크기: 전통적으로 512B → 2010년 이후 4KB로 전환 중
- 헤드 크래시(head crash): 헤드가 디스크 표면에 접촉 → 자기 데이터 손상, 복구 불가

### 11.1.2 비휘발성 메모리 장치 (Nonvolatile Memory Devices)

**NAND Flash 특성:**

```
NVM Device (SSD) 내부 구조:
┌─────────────────────────────┐
│  Controller                 │
│  ┌──────────────────────┐   │
│  │  Flash Translation   │   │  ← FTL: 논리 블록 → 물리 페이지 매핑
│  │  Layer (FTL)         │   │
│  └──────────────────────┘   │
│                             │
│  Die 0    Die 1    Die 2    │
│  ┌──┐     ┌──┐     ┌──┐    │
│  │B0│     │B0│     │B0│    │  ← Block (수 페이지)
│  │P0│     │P0│     │P0│    │  ← Page (읽기/쓰기 단위)
│  │P1│     │P1│     │P1│    │
│  └──┘     └──┘     └──┘    │
└─────────────────────────────┘
```

**NAND 연산 특성:**

| 연산 | 단위 | 속도 | 특이사항 |
|------|------|------|----------|
| Read | Page | 가장 빠름 | |
| Write (Program) | Page | 중간 | 덮어쓰기 불가, 반드시 erase 후 write |
| Erase | Block (수 페이지) | 매우 느림 | 약 100,000회 한계 |

**수명 측정:** DWPD (Drive Writes Per Day) — 하루에 드라이브 용량을 몇 번 쓸 수 있는지

**FTL 핵심 알고리즘:**
- **Garbage Collection**: 무효 페이지가 섞인 블록에서 유효 데이터를 다른 곳으로 이동 → 블록 erase
- **Over-provisioning**: 총 용량의 약 20%를 항상 쓰기 가능 공간으로 예비
- **Wear Leveling**: erase 횟수를 전체 블록에 균등 분산 → 수명 연장
- **Write Amplification**: GC로 인해 실제 쓰기 I/O가 애플리케이션 요청보다 훨씬 많아지는 현상

**NVM vs HDD 비교:**

| 항목 | HDD | NVM(SSD) |
|------|-----|----------|
| 임의 읽기 (IOPS) | 수백 | 수십만~수백만 |
| 순차 읽기 | 빠름 | HDD와 유사~10배 |
| 쓰기 성능 변동 | 일정 | 수명/용량에 따라 변동 |
| 전력 소비 | 높음 | 낮음 |
| 비용/GB | 낮음 | 높음 |
| 인터페이스 | SATA | NVMe/PCIe (직결, 고속) |

### 11.1.3 휘발성 메모리 (Volatile Memory) — RAM Drive

- DRAM 일부를 블록 장치로 노출 (`/dev/ram`, macOS `diskutil`, Linux `/tmp` tmpfs)
- 시스템 크래시/전원 off 시 데이터 소실
- 용도: 고속 임시 파일 공유, 부팅 시 `initrd` (초기 루트 파일시스템)

### 11.1.4 연결 방식 (Connection Methods)

```
CPU ─── System Bus (PCI/PCIe)
          │
          ├── NVMe SSD (직결, 최고속)
          │
          └── I/O Bus Controller
                │
                ├── SATA (HDD, 일반 SSD)
                ├── SAS (서버급 HDD)
                └── USB / Thunderbolt (외장 장치)
```

- **NVMe (NVM Express)**: PCIe 직결, 저지연 고처리량
- **Host Controller** ↔ **Device Controller** 간 명령 전송 → DMA로 캐시↔DRAM 전송

### 11.1.5 주소 매핑 (Address Mapping)

- **LBA (Logical Block Address)**: 장치를 1차원 논리 블록 배열로 추상화
- HDD: LBA → (cylinder, track, sector) 매핑 (실제로는 드라이브 내부 처리)
- NVM: LBA → (chip, block, page) 매핑 (FTL이 담당)
- **CLV (Constant Linear Velocity)**: CD/DVD — 트랙 밀도 균일, 회전 속도 가변
- **CAV (Constant Angular Velocity)**: HDD — 회전 속도 고정, 외곽 트랙에 데이터 밀도 낮음

---

## 11.2 HDD 스케줄링 (HDD Scheduling)

목표: **access time 최소화** + **대역폭 최대화** (= 총 전송 바이트 / 총 시간)

요청 큐 예시: 헤드 위치 53, 큐 = {98, 183, 37, 122, 14, 124, 65, 67}

### 11.2.1 FCFS

```
53 → 98 → 183 → 37 → 122 → 14 → 124 → 65 → 67
     ←45→←85→←146→←85→←108→←110→←59→←2→
총 이동: 640 실린더  (비효율적, 공정하지만 성능 최악)
```

### 11.2.2 SCAN (Elevator Algorithm)

```
방향: 53 → 0 방향 이동 중
 0 ← 14 ← 37 ← 53 (반환)
 53 → 65 → 67 → 98 → 122 → 124 → 183
총 이동: 236 실린더
```

- 헤드가 디스크 끝까지 이동 후 방향 전환
- 방금 지나온 쪽의 실린더는 반환점 이후 처리 → 최대 대기 = 왕복 시간

### 11.2.3 C-SCAN (Circular SCAN)

```
53 → 65 → 67 → 98 → 122 → 124 → 183 → 199 (끝)
→ 0으로 즉시 복귀 (서비스 없음) → 14 → 37
```

- 복귀 시 서비스 없음 → **균일한 대기 시간** 제공
- 실린더를 원형 목록으로 취급

### 11.2.4 스케줄링 알고리즘 선택

**Linux I/O 스케줄러:**

| 스케줄러 | 설명 | 적합 대상 |
|---------|------|----------|
| **Deadline** | 읽기/쓰기 별도 큐, FCFS 큐 타임아웃(기본 500ms) 감시, C-SCAN 순서 배치 | 일반 HDD (RHEL 7 기본) |
| **NOOP** | FCFS + 인접 요청 병합 | NVM 장치, CPU 바운드 시스템 |
| **CFQ** (Completely Fair Queueing) | 프로세스별 큐, 실시간/최선노력/유휴 3단계 우선순위, 지역성 예측 대기 | SATA HDD |

---

## 11.3 NVM 스케줄링

- 기계적 이동 없음 → FCFS 기반이 기본
- **Linux NOOP**: FCFS + **인접 쓰기 요청 병합** (읽기는 FCFS 그대로)
- 읽기 서비스 시간: 균일 / 쓰기 서비스 시간: **불균일** (GC 상태에 따라 변동)
- **Write Amplification** 상세:

```
애플리케이션 쓰기 1회 →
  ① 새 데이터 페이지 쓰기
  ② GC: 유효 페이지들 읽기 (n회)
  ③ GC: 유효 페이지들 새 위치에 쓰기 (n회)
  ④ 빈 블록 erase
  = 실제 I/O >> 요청 I/O
```

---

## 11.4 오류 검출 및 정정 (Error Detection and Correction)

**계층별 오류 보호:**

| 기법 | 원리 | 감지 범위 |
|------|------|----------|
| **Parity bit** | 바이트 XOR → 홀수/짝수 | 단일 비트 오류 |
| **CRC** (Cyclic Redundancy Check) | 해시 함수 기반 | 다중 비트 오류 |
| **ECC** (Error-Correcting Code) | 추가 데이터로 오류 위치 특정 후 복원 | 소수 비트 오류 → 정정 |

**저장장치 ECC 동작:**
- 쓰기 시: 섹터/페이지 데이터 → ECC 계산 후 함께 저장
- 읽기 시: ECC 재계산 → 저장값과 비교
  - 일치: 정상
  - 불일치 + 정정 가능: **Recoverable soft error** → 자동 정정
  - 불일치 + 정정 불가: **Non-correctable hard error** → 데이터 손실

---

## 11.5 저장장치 관리 (Storage Device Management)

### 저장장치 초기화 3단계

```
① Low-level Formatting (물리 포맷)
   → 섹터/페이지 구조 생성 (header + data + trailer + ECC)
   → NVM: FTL 초기화, 불량 페이지 매핑

② Partitioning (파티션 분할)
   → fdisk (Linux), Disk Management (Windows)
   → 파티션 정보: 장치 고정 위치에 저장
   → /etc/fstab: 파티션별 마운트 설정

③ Logical Formatting (파일시스템 생성)
   → 파일시스템 메타데이터(빈 공간 맵, 초기 디렉토리) 기록
```

**Volume 관리:**
- `lvm2` (Linux): 여러 파티션/장치를 묶어 논리 볼륨 구성
- `ZFS`: 볼륨 관리 + 파일시스템을 통합 제공

### 11.5.2 부트 블록 (Boot Block)

```
전원 ON
  │
  ▼
Firmware (NVM/Flash, 모든보드 고정 주소)
  → 간단한 Bootstrap Loader 실행
  │
  ▼
MBR (Master Boot Record) — 장치의 첫 번째 섹터
  ┌──────────────────────────┐
  │  Boot Code               │
  │  Partition Table         │
  │  Boot Partition Flag     │
  └──────────────────────────┘
  │
  ▼
Boot Partition의 첫 섹터 (Boot Sector)
  → 커널 위치 가리킴
  │
  ▼
Kernel 로드 → OS 부팅 완료
```

- Linux 기본 부트로더: **grub2**
- 부트 코드는 바이러스에 의해 감염 가능 (firmware 쓰기 가능)

### 11.5.3 불량 블록 (Bad Blocks)

**HDD 불량 블록 처리:**

| 기법 | 설명 |
|------|------|
| **Sector Sparing** (forwarding) | 불량 섹터 → 여분 섹터로 논리 재매핑 (컨트롤러 담당) |
| **Sector Slipping** | 불량 섹터 이후 섹터들을 한 칸씩 밀어서 빈자리 생성 후 재매핑 |

- 스페어 섹터는 **같은 실린더** 내에서 우선 선택 (seek 최소화)
- NVM: 불량 페이지 표를 FTL에서 관리, over-provisioning 영역으로 대체

---

## 11.6 스왑 공간 관리 (Swap-Space Management)

### 스왑 공간 위치 선택

| 방식 | 장점 | 단점 |
|------|------|------|
| **파일시스템 내 파일** | 관리 편리, 동적 크기 변경 | 성능 오버헤드 (FS 레이어 경유) |
| **Raw 파티션** | 고속 (FS 우회), 내부 단편화 감수 | 재파티션 필요 시 복잡 |

**Linux 스왑 구조:**

```
Swap Area (raw partition 또는 swap file)
┌─────┬─────┬─────┬─────┬──── ...
│slot0│slot1│slot2│slot3│
└─────┴─────┴─────┴─────┴──── ...
   │     │
Swap Map (정수 배열)
[  1,   0,   3,   0, ...]
    ↑           ↑
  사용 중      3개 프로세스가 공유하는 스왑 페이지
```

- 값 0: 슬롯 사용 가능 / 양수: 해당 페이지를 매핑한 프로세스 수
- **Linux/Solaris**: 익명 메모리(anonymous memory — heap, stack, BSS)만 스왑 → 코드 세그먼트는 파일시스템에서 재로드
- Solaris: 물리 메모리에서 페이지가 방출될 때만 스왑 공간 할당 (지연 할당)

---

## 11.7 저장장치 연결 방식 (Storage Attachment)

### 연결 유형 비교

```
┌─────────────────────────────────────────────────────────┐
│           Storage Attachment 유형                        │
│                                                         │
│  HOST-ATTACHED    NETWORK-ATTACHED     CLOUD            │
│  ┌───────────┐    ┌──────────────┐    ┌───────────┐     │
│  │   HDD     │    │   NAS        │    │  Amazon   │     │
│  │   SSD     │    │  NFS(Unix)   │    │    S3     │     │
│  │   USB     │    │  CIFS(Win)   │    │  iCloud   │     │
│  │   FC      │    │  iSCSI       │    │  Dropbox  │     │
│  └───────────┘    └──────────────┘    └───────────┘     │
│   직접 I/O         LAN via RPC         WAN via API       │
└─────────────────────────────────────────────────────────┘
```

**NAS 프로토콜:**
- **NFS**: Unix/Linux — TCP/UDP over IP
- **CIFS**: Windows — TCP/UDP over IP
- **iSCSI**: IP 네트워크로 SCSI 프로토콜 전송 → 원격 스토리지를 로컬처럼 사용

**Cloud Storage:**
- NAS: 기존 파일시스템 프로토콜 사용 → OS가 투명하게 처리
- Cloud: **API 기반** (WAN 지연/실패 처리 포함) → 애플리케이션이 직접 처리

### SAN (Storage-Area Network)

```
      Server A    Server B    Server C
         │            │            │
         └────────────┼────────────┘
                      │
               SAN Switch (FC 또는 iSCSI)
                      │
         ┌────────────┼────────────┐
         │            │            │
    Storage Array  RAID Array   JBOD
```

- 사설 스토리지 네트워크 (FC 또는 iSCSI)
- 여러 호스트 ↔ 여러 스토리지 동적 할당 가능
- **Storage Array**: 자체 컨트롤러(CPU+메모리+소프트웨어) + RAID + 스냅샷 + 압축 + 중복제거

---

## 11.8 RAID 구조 (RAID Structure)

### 신뢰성 이론: MTBF (Mean Time Between Failures)

- 단일 디스크 MTBF = 100,000시간
- 100개 디스크 어레이 MTBF = 100,000 / 100 = **1,000시간 ≈ 41.6일**
- 미러링 시 데이터 손실 MTBF:
  ```
  MTBF_mirrored = MTBF² / (2 × MTTR) = 100,000² / (2×10) = 500,000,000시간 ≈ 57,000년
  ```

### RAID 레벨별 비교

```
RAID 0 (Striping, 중복 없음):
  Drive1: A1  A3  A5
  Drive2: A2  A4  A6
  → 최고 성능, 신뢰성 없음

RAID 1 (Mirroring):
  Drive1: A1  A2  A3
  Drive2: A1  A2  A3  (복사본)
  → 높은 신뢰성, 2배 비용

RAID 4 (Block Striping + Dedicated Parity):
  Drive1: A1  A5  A9
  Drive2: A2  A6  A10
  Drive3: A3  A7  A11
  Drive4: A4  A8  A12
  DriveP: P1  P2  P3   ← 패리티 드라이브 집중
  → 패리티 드라이브 병목

RAID 5 (Block-Interleaved Distributed Parity):
  Drive1: A1  A5   P   A13
  Drive2: A2   P  A9   A14
  Drive3:  P  A6  A10  A15
  Drive4: A4  A7  A11  P
  Drive5: A3  A8  A12  A16
  → 패리티 분산, 병목 해소 (가장 일반적)

RAID 6 (P+Q Redundancy):
  → XOR + Galois Field 연산으로 2개 패리티 블록
  → 동시에 2개 드라이브 장애 허용

RAID 0+1 vs 1+0:
  0+1: Stripe → Mirror  (스트라이프 1개 장애 시 전체 스트라이프 손실)
  1+0: Mirror → Stripe  (개별 드라이브 장애 → 미러 파트너만 영향)
       ↳ 이론적으로 1+0이 더 안전
```

**RAID 레벨 선택 가이드:**

| RAID 레벨 | 적합 사용 사례 | 특징 |
|-----------|--------------|------|
| 0 | 과학 계산, 재현 가능 데이터 | 최고 성능, 신뢰성 없음 |
| 1 | DB 로그, 부트 파티션 | 빠른 복구, 2배 비용 |
| 5 | 일반 파일서버 | 좋은 성능/비용 균형 |
| 6, MD-6 | 대용량 스토리지 어레이 | 2중 장애 허용 |
| 0+1 / 1+0 | 성능+신뢰성 동시 요구 | 비용 큼 |

### RAID 구현 계층

```
Software RAID  ←  OS/커널 레이어 (lvm2, mdadm)
Hardware RAID  ←  HBA (Host Bus Adapter) 수준
Array RAID     ←  Storage Array 내부 컨트롤러
SAN RAID       ←  SAN 스위치/가상화 장치
```

**RAID의 한계 — ZFS 솔루션:**

일반 RAID가 보호하지 못하는 것:
- 파일 포인터 손상 (소프트웨어 버그)
- Torn writes (전원 중단 중 불완전 쓰기)
- RAID 컨트롤러 자체 버그

**ZFS 접근법:**
```
inode
  ├── checksum(data_block_1)  ─→  data_block_1
  ├── checksum(data_block_2)  ─→  data_block_2
  └── ...

directory_entry
  └── checksum(inode)  ─→  inode

→ 모든 메타데이터/데이터에 체크섬 → 자동 감지+복구
→ ZFS Pooled Storage: RAID set → Pool → 여러 File System
   (malloc/free 방식 공간 할당, 인위적 크기 제한 없음)
```

### Hot Spare

- 데이터 저장 없이 대기하는 예비 드라이브
- 드라이브 장애 발생 즉시 자동으로 리빌드에 투입
- RAID 레벨 자동 복원 (인력 개입 불필요)

### 11.8.7 오브젝트 스토리지 (Object Storage)

**파일시스템 vs 오브젝트 스토리지:**

| 항목 | 파일시스템 | 오브젝트 스토리지 |
|------|-----------|-----------------|
| 접근 방식 | 경로명 탐색 | Object ID |
| 사용자 | 사람 중심 | 프로그램/API 중심 |
| 확장성 | 수직 확장 | **수평 확장** (노드 추가) |
| 데이터 보호 | RAID | **N벌 복제** (다른 노드에 분산) |
| 데이터 형식 | 구조화 | **비구조화** (self-describing) |

**대표 구현:** HDFS (Hadoop), Ceph
**활용:** Google 검색 인덱스, Facebook 사진, Spotify 음악, AWS S3

```
Object Storage 접근 패턴:
  Create object → object_id
  Access(object_id) → data
  Delete(object_id)
  
특징: Content-addressable storage (내용 기반 주소 지정)
```

---

## 핵심 요약

```
Mass Storage 계층:
  DRAM (RAM drive) ← 최고속, 휘발성
       ↓
  NVM/SSD ← 고속, NAND 한계(erase, wear)
       ↓
  HDD ← 대용량, 기계적 지연
       ↓
  Tape / Cloud ← 아카이브, 백업

스케줄링:
  HDD: FCFS < SCAN < C-SCAN  (seek 최소화)
  NVM: FCFS (+ 인접 병합)    (seek 없음)

신뢰성:
  ECC → 섹터 오류 정정
  RAID 1/5/6 → 드라이브 장애 허용
  ZFS 체크섬 → 소프트웨어 오류까지 검출
  Object Storage N-copy → 노드 장애 허용
```
